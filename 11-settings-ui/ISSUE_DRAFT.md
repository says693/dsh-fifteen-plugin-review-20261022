<!-- intake:v1 {"project":"bench","target":"weibaohui/dsh-settings-ui","form":"intake"} -->

# [intake 输入] dsh-settings-ui 0.2.10：自定义宽高输入框无法清空重输，想输 900 会得到 48000

在宿主设置窗口 →「设置界面」→「自定义」里，点进「宽 × 高（px）」输入框，全选删除后敲 `900`，输入框变成 `48000`；高度栏从 960 敲 `600`，变成 `360600`。键盘没法把这个框清空重输，只能用粘贴或方向键绕过。同一个框还只有下界没有上界，脏值会被写进 profile 配置。

## 一、复现步骤

1. 宿主 `@deepseek-ai/dsh@0.2.0-rc.2`，隔离 profile，装入 `@weibaohui/dsh-settings-ui@0.2.10`，起在 `127.0.0.1` 的随机端口。
2. 首次启动如遇「添加一个 API Key」引导弹窗，先点「稍后配置」，否则左下「设置」按钮点了没反应（宿主行为，非本插件）。
3. 打开左下「设置」→ 左侧导航点「设置界面」→ 点尺寸 chip「自定义」。
4. 点击宽度输入框，`Ctrl+A` 再 `Delete`：值从 `1280` 变 `480`（被 `Math.max(480, …)` 夹到下界）。
5. 依次敲 `9`、`0`、`0`：得到 `4809` → `48090` → `480900`。改为 `Ctrl+A` 后直接打 `900`：得到 `48000`。
6. 高度栏重复：原值 `960`，敲 `600` → `360600`。
7. 越界对照：输入 `0` → `480×360`（正确夹住）；输入 `-100` → `480100`；输入 `99999` → `4809999`，且 `customWidth`/`customHeight` 原样写进 profile 的 `cordis.patch.yml`。

## 二、测试环境

- 被测件：`@weibaohui/dsh-settings-ui@0.2.10`（npm 发布件，非源码目录安装）
- shasum `1be422ae19a42072906fc6a3f26095eb2a796a93`，integrity `sha512-o/YWP3yB3ABlCR537SNA68HPgcc/xmdM/OKaJ3AXWCDLxEWEWm4OyiUqFezQ8Alxm//PeAE1loGDbT4LBR6Baw==`，均与 registry 一致；解包 5 个文件的 sha256 与固定提交 `f0e79f9` 同路径文件逐字节一致
- 宿主：`@deepseek-ai/dsh 0.2.0-rc.2`，`--host 127.0.0.1 --port 0 --no-open`
- 浏览器：Microsoft Edge 155.0.4283.62，Playwright 1.64.1 headless，独立 context，仅放行本机地址
- 系统：Windows 11 26200，Node v24.19.0；测试用 `DSH_HOME` / `HOME` / `APPDATA` / `LOCALAPPDATA` 全部重定向到任务目录

## 三、证据

- `evidence/p19-input-rootcause.json`：`type=number`；`setSelectionRange(0,2)` 抛 `InvalidStateError: The input element's type ('number') does not support selection.`；`selStart/selEnd = null / null`；粘贴 900 → 值 `900`、API `customWidth:900`
- `evidence/p19-run.txt`：同一次探测的原始 stdout，含退出码
- `evidence/p06-typing.json`：`4809 → 48090 → 480900` 的逐键读数与每次注入的 CSS
- `evidence/p15-typing-anomaly.json`：`-100 → 480100`、`99999 → 4809999`、落库 `customWidth=48000 customHeight=360600`
- `evidence/p15-typing-result.png`、`evidence/p06-after-typing-900.png`：输入框内的实际值
- `evidence/p05-custom.json`：`0 → 480×360` 的下界对照，以及服务端 `customWidth:-5 → 480`、`opacity:0 → 30`、`size:'huge'` 丢弃、非法 JSON → 400 的读数
- `evidence/p19-a11y.png`、`evidence/p19-input-rootcause.json`（`keyboard.aria`）：`chipsGroupRole=null`，滑块与两个 color input 的 `aria-label / aria-labelledby / id` 全为 null
- `evidence/npm-pack.log`、`evidence/npm-view-0.2.10.json`：tarball 只有 5 个文件，无 LICENSE

## 四、已验 / 未验 / 不确定 / 不做

**已验（confirmed，有可复跑读数）**

1. 自定义宽高输入框键盘无法清空重输（主条目，medium）。根因定位到浏览器对 `type=number` 不支持 selection API，加上 `onChange` 里立即 `save()` 导致每次按键重渲染并 PUT 一次。
2. 数值只有下界没有上界，越界值会落进 profile（low）。前端 `buildCss` 用 `Math.max(480, …)`，宿主侧 `sanitizeSettingsPatch` 的 `num('customWidth', 480)` 同样只有下限；视觉没被撑破只是因为 CSS 写了 `width:min(Xpx, calc(100vw - 48px))`。
3. 控件缺少可访问名称（low）。chip 是原生 `<button>`、`tabIndex=0`、带 `aria-pressed`，键盘这一半是好的；缺的是 chip 组的 `role="radiogroup"`、滑块与两个色板的 `aria-label`。
4. 发布件不含 LICENSE 文本（low）。`package.json` 声明 MIT，仓库 `f0e79f9` 里有 1066 字节的 LICENSE，但 tarball 只有 README.md / client/bundle.js / cordis.patch.yml / package.json / src/index.js 五个文件。

同时确认可用（不构成缺陷，供维护者对照）：五档尺寸即时生效、70% 不透明度与纯色背景即时生效、关窗重开/整页刷新/冷重启宿主/换浏览器都不丢设置、换 profile 不串味、恢复默认干净（注入 CSS 清空、落盘回默认）、四档窄视口无溢出、深色下正文对比度 ≥4.5、全程 0 条 pageerror 与 console.error。

**未验**

- 只在 `0.2.0-rc.2` 一个宿主上跑过；作者版本表里的 `0.1.7-rc.2` 与 desktop 构建没有条件回测。
- 没有配置真实模型密钥，未在真实对话流中运行（插件只碰设置面板，理论上不相干）。

**不确定**

- 鼠标按住滑块拖动有时不触发 `onChange`、松手弹回 100%；但点轨道与键盘完全稳定。只在不带界面的合成鼠标下观察到，无法排除是我方驱动方式的局限，因此不作为缺陷提交。
- 服务端对 `bgColorLight` 不做格式校验，`'red;position:fixed;inset:0'` 能进注入串；实测浏览器把整条声明判无效（面板背景回主题色、`position` 仍是 `relative`），没有观察到可利用结果，因此不升级为安全缺陷。

**不做（本版本无此能力，非「坏了」）**

- 悬浮球：关键词与几何双扫描 0 命中，插件注入的 DOM 只有两个 `<style>`。
- 图片背景：`normalizeBgMode` 把 `image` 回落成 `default`，源码注释写「图片背景已下线」，UI 无上传入口。
- 设置窗口拖动：面板由宿主居中定位，插件只改宽高不改位置，本来就没承诺可拖动。

## 五、建议

1. 输入框改 `type="text"` + `inputMode="numeric"`（保留数字键盘，恢复选区能力）；或在 `onFocus` 时 `select()`；或把 `onChange` 的即时保存挪到 `onBlur` / `Enter` 提交。三条任选一条即可解决主条目。
2. `buildCss` 与 `sanitizeSettingsPatch` 各补一个上界（例如 `Math.min(4096, …)`），让前后端都夹住。
3. 给不透明度滑块与两个色板加 `aria-label`，chip 组加 `role="radiogroup"`。
4. `files` 字段加上 `LICENSE`，或在 `prepublishOnly` 里断言它存在。
