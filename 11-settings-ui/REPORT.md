# dsh-settings-ui 0.2.10 技术评估

被测件：`@weibaohui/dsh-settings-ui@0.2.10`（npm 发布件）。宿主 `@deepseek-ai/dsh 0.2.0-rc.2`，Windows 11 26200 + Edge 155.0.4283.62（Playwright 1.64.1 headless），隔离 `DSH_HOME` 与 profile，端口 0。

发布件身份已核对：`shasum` = `1be422ae19a42072906fc6a3f26095eb2a796a93`、`integrity` = `sha512-o/YWP3yB3ABlCR537SNA68HPgcc/xmdM/OKaJ3AXWCDLxEWEWm4OyiUqFezQ8Alxm//PeAE1loGDbT4LBR6Baw==`，均与 registry 一致；解包后 5 个文件的 sha256 与固定提交 `f0e79f9` 的同路径文件**逐字节相同**（`gitHead` 是 `51180fdb`，即 0.2.10 的发布提交，比任务指定的比对提交早 9 天，两者之间只差 LICENSE 与发布工作流）。

## 一、结论先看

插件**能装、能用、界面在**。核心四项——换尺寸、调不透明度、换背景色、恢复默认——在真宿主上都是即时生效的，刷新、冷重启、换浏览器都不丢，换 profile 不串味，四个窄视口都不溢出，深色下对比度充足，全程零控制台报错。这在同批插件里属于完成度偏高的一档。

问题集中在两个地方，而且都不难修：

1. **自定义宽高输入框用键盘没法清空重输**——它是 `<input type="number">`，浏览器不支持在这个 type 上做选区，于是 `Ctrl+A` 无效、每个数字都被追加。想设 900 会得到 `48000`。有绕过办法（粘贴 / 方向键），但用户不会猜到。
2. **数值只有下界没有上界**——`-100` 变 `480100`，`99999` 变 `4809999`，脏值会一路写进 profile。视觉上被 CSS 的 `min()` 兜住了，所以不炸，但用户下次打开会看到一个荒唐数字，而且改不回去（回到问题 1）。

另有两条轻微的：控件缺少 `aria-label`（读屏用户听到三个无名控件），npm 包里没带 LICENSE 文本。

**任务书里有三处预设与本版本实际不符**，先说清楚以免误判：

- **悬浮球不存在**。关键词扫描 + 几何扫描都是 0 命中，插件注入的 DOM 只有两个 `<style>`。这不是「悬浮球坏了」，是这个版本没有这个功能。
- **图片背景已下线**。API 传 `bgMode:'image'` 会被 `normalizeBgMode` 回落成 `default`，源码注释写着「图片背景已下线」。因此「图片加载失败」这条异常**没有可测对象**。
- **设置不存浏览器**。`localStorage` 里只有宿主的三个键，插件的设置走宿主 settings 文档，落盘在 profile 的 `cordis.patch.yml`。这是更好的设计（换浏览器不丢），但也意味着换 profile 就换一套。

**设置窗口拖不动**也不是缺陷——面板由宿主居中定位，插件改的是宽高不是位置，本来就没承诺可拖动。

## 二、确认可用（附读数）

### 尺寸

五档全部即时生效，不用关窗，切换即变：

| 档位 | 面板 rect（1440×1000 视口） | computed |
|---|---|---|
| 默认 | `x320 y100 w800 h800` | `width:800px; height:800px` |
| 大 | `x180 y110 w1080 h780` | `width:1080px; height:780px` |
| 特大 | `x80 y24 w1280 h952` | `height:952px` ← CSS 声明 960，被 `min(960px, calc(100vh - 48px))` 夹到 952 |
| 全屏 | `x0 y0 w1440 h1000` | `max-width:none; border-radius:0px` |
| 自定义 | 随宽高变化，见下 | `width:min(Xpx, calc(100vw - 48px))` |

从全屏切回默认时注入的 CSS 被清空（长度 149 → 0），没有残留声明。

### 不透明度

滑到 70% → computed `background-color` 变 `color(srgb 1 1 1 / 0.7)`，注入 `color-mix(in srgb, var(--dsw-alias-bg-layer-2) 70%, transparent)`；30% → `/0.3`；100% → 回 `rgb(255, 255, 255)` 且注入清空。截图里能看见聊天区文字透过设置面板。

### 背景色 + 亮暗分存

切「纯色」→ 面板变 `rgb(238, 241, 245)`（`#eef1f5`）。把浅色色板改成 `#ff8800` → 面板变 `rgb(255, 136, 0)`，注入 CSS 变 `background-color:#ff8800`，暗色那一色 `#1e2a38` 静止不动。

切主题不用刷新：切到深色，注入串立刻换 `background-color:#1e2a38`、面板变 `rgb(30, 42, 56)`；切回浅色又变 `#ff8800`。这是 `MutationObserver` 盯 `body[data-ds-dark-theme]` 的结果。

### 持久化（三条链路都验了）

- **刷新**：设成「大 + 70% + #ff8800」后 reload，页面首帧 `#dsh-settings-ui-style` 就已经是完整的 `width:1080px;height:min(780px, calc(100vh - 48px));background-color:color-mix(in srgb, #ff8800 70%, transparent)`——不用先打开设置窗口。
- **冷重启**：停掉 pid 29232，新进程 pid 20288 在新端口 10932 起来，首帧注入串一模一样，面板 1080×780、range=70、选中 chip 不变。
- **换浏览器**：全新 browser context（localStorage 从空开始）打开同一宿主，注入串依然在。

落盘位置是 `profiles/su/cordis.patch.yml` 里的 `- id: dsh-settings-ui / config: {...}` 条目，不是浏览器。

### 换 profile 不串味

新建 `su2` profile（同版本、同宿主、端口 2379）：`tweakOnBoot=null`、API 全默认、面板 800×800 / 100% / 主题默认。干净。

### 恢复默认

点一下：面板立刻回 800×800、range=100、chip 回「默认」+「主题默认」、注入 CSS 清空（长度 0）、API 全默认。关窗重开一致，刷新一致，落盘文件也真的回默认了（`size: default / opacity: 100 / bgMode: default`）。

### 窄窗口

| 视口 | 面板 rect | 超出视口 | chip 行数 | 横向溢出 |
|---|---|---|---|---|
| 1440×1000 | 1280×952 | 否 | — | 无 |
| 1000×900 | 952×852 | 否 | 3 | 无 |
| 760×900 | 712×852 | 否 | 4 | 无 |
| 600×800 | 552×752 | 否 | 4 | 无 |

600 宽 + 全屏 → 600×800 铺满，无溢出无裁切。文档始终没有横向滚动条。

### 深色可读性（含半透明合成后）

按 WCAG 算有效对比度（把元素自身 `opacity` 与面板半透明的 alpha 都合成进去）：

| 状态 | 标签 | 键名 | 数值 | 提示 | chip |
|---|---|---|---|---|---|
| 深色 + 纯色 100% | 5.99 | 6.75 | 7.58 | 4.63 | 13.92 |
| 深色 + 纯色 50% | 6.48 | 7.36 | 8.32 | 4.92 | 15.78 |
| 深色 + 纯色 30% | 6.64 | 7.57 | 8.58 | 5.02 | 16.48 |
| 亮色 + 主题默认 | 4.94 | — | 7.08 | 3.55 | 18.9 |

最弱的一项是亮色默认态下的提示文案（3.55），深色下反而更好。正文类文字全部 ≥4.5，达到 AA。

### 服务端校验

直接打 API 绕过前端：

| 请求 | 结果 |
|---|---|
| `customWidth:-5` | 200 → 落库 **480**（夹到下界，正确） |
| `opacity:0` | 200 → **30**（正确） |
| `opacity:999` | 200 → **100**（正确） |
| `size:'huge'` | 200 → 丢弃，保持原值（正确） |
| `bgMode:'image'` | 200 → 回落 `default`（正确） |
| 非法 JSON body | **400** `{"error":"invalid JSON body: Expected property name or '}' ..."}` |

下界与枚举都守住了，上界没守。

## 三、确认的问题

### P1（medium）自定义宽高输入框：键盘无法清空重输

**现象**：点进「宽 × 高（px）」，全选删除，敲 `900` —— 得到 `48000`。

**读数**：

```
初始 1280
Ctrl+A → Delete        → 480       （被 min 夹到下界）
敲 "9"                 → 4809
敲 "0"                 → 48090
敲 "0"                 → 480900
Ctrl+A → 直接敲 "900"  → 48000
高度栏 960 → 敲 "600"  → 360600
```

**根因**（已定位到具体 API）：输入框是 `<input type="number" class="su-num">`。浏览器**不支持在 `type=number` 上做文本选区**：

```
setSelectionRange(0, 2) → InvalidStateError:
  Failed to execute 'setSelectionRange' on 'HTMLInputElement':
  The input element's type ('number') does not support selection.
selectionStart / selectionEnd → null / null（Ctrl+A 之后仍是）
```

所以 `Ctrl+A` 在这个控件上不产生可替换选区，每个新数字都被接到已有值末尾。叠加插件 `onChange` 里立刻 `Math.max(min, Number(v) || min)` 并 `save()`，**每次按键都重渲染并 PUT 一次**，光标位置丢失、脏值一路写盘。

**绕过办法**（已实测有效）：

- **粘贴**：`Ctrl+A` 后粘贴 `900` → 得到 `900`，落库 `customWidth:900`；粘贴 `1024` → `1024`。
- **方向键**：`ArrowUp`/`ArrowDown` 步进正常（480 → 482）。
- **Backspace** 连按可退到下界 480。

**建议**：要么把 `type` 换成 `text` + `inputMode="numeric"`（保留数字键盘、恢复选区能力），要么在 `onFocus` 时 `select()`，要么把 `onChange` 的即时保存改成 `onBlur`/`Enter` 提交。三条任选一条都能解决。

### P2（low）数值只有下界没有上界，脏值进 profile

**读数**：

```
输入 0    → 480 × 360        （正确，Math.max 生效）
输入 1    → 480 × 360        （正确）
输入 -100 → 480100 × 36050   （减号被吞，值被拼在 480 后面）
输入 99999→ 4809999 × 3609999
落库：customWidth=48000, customHeight=360600 原样写进 cordis.patch.yml
```

`buildCss` 用 `Math.max(480, Number(s.customWidth) || DEFAULTS.customWidth)`，只有下限；`src/index.js` 的 `sanitizeSettingsPatch` 里 `num('customWidth', 480)` 同样只有下限。面板没被撑破纯粹是因为 CSS 写了 `width:min(Xpx, calc(100vw - 48px))`——也就是视觉上被兜住，数据层没有。

**建议**：补一个上界（例如 `Math.min(4096, …)`），前后端各一次。

### P3（low）控件缺少可访问名称

chip 这半做得好：原生 `<button>`、`tabIndex=0`、Tab 能到、Enter 能触发、带 `aria-pressed`。缺的是另一半：

```
chip 组 role                → null（只是个 div，不是 radiogroup）
不透明度滑块 aria-label      → null
不透明度滑块 aria-labelledby → null
两个 color input aria-label  → null / null
```

读屏用户会听到三个没有名字的「滑块」「颜色选择器」，不知道哪个是背景不透明度、哪个是浅色主题。视觉用户完全不受影响。

**建议**：给滑块与两个色板加 `aria-label`，chip 组加 `role="radiogroup"`。

### P4（low）发布件不含 LICENSE 文本

`package.json` 声明 MIT，仓库在提交 `f0e79f9` 里有 LICENSE（1066 字节，`Copyright (c) 2026 weibaohui`，sha256 `8f466caa…`）。但 npm tarball 只含 5 个文件（README.md / client/bundle.js / cordis.patch.yml / package.json / src/index.js），**没有 LICENSE**。`files` 字段没带上它。

**建议**：`files` 加 `LICENSE`，或在 `prepublishOnly` 里断言存在。

## 四、不确定的部分

### 滑块鼠标拖拽

同一次拖拽有时不触发 `onChange`（目标 50% 却回到 100%），但**点轨道**和**键盘**都完全稳定：

```
点轨道：最左 → 30，中点 → 65，最右 → 100
键盘：  Home → 30，ArrowRight ×1 → 35，×2 → 40，×4 → 45，End → 100
```

只在 headless 合成鼠标下观察到此现象，无法排除是 Playwright 对 range 原生拖拽模拟的局限，也没有真人手动拖拽的对照。因此**不写成缺陷**。

### 非法颜色值

服务端对 `bgColorLight` 不做格式校验，`'not-a-color'`、`'red;position:fixed;inset:0'`、空串三种都 200 原样落库（前端读回时用 `/^#[0-9a-fA-F]{6}$/` 兜底显示）。第三种的注入串里确实出现了 `background-color:red;position:fixed;inset:0`，但浏览器把这条声明整条判无效，实测面板 `background-color` 回落主题色、`position` 仍是 `relative`、`inset` 全 0 —— **没有观察到注入生效**。

脏值能进 CSS 是事实（有读数），但「能被利用」我没有证明出来：单条声明里出现非法 token 时 CSS 解析器会丢弃整条。要把它写成安全问题，需要构造出能穿过解析的值，这一步我没做到。因此只记为「服务端缺格式校验」这一层，不升级为安全缺陷。

## 五、环境侧的两个坑（不是插件缺陷）

1. **首次启动会被宿主的「添加一个 API Key」弹窗挡住**。点左下「设置」按钮毫无反应，必须先点弹窗里的「稍后配置」。这会让新手以为插件没装上——我一开始也以为是我点错了位置。
2. **新建 profile 的安装顺序**。直接 `dsh plugin --profile <新名> add` 建出的 profile 只含 `dsh-base`、缺 `dsh-web-app`，宿主 boot 后不打印 URL。正确顺序是先 `dsh --profile <名> --from-default-profile web --dump-config` 派生，再 add。

## 六、怎么复核

见 `REPRODUCE.md`。所有读数都在 `evidence/` 下，每个探针都有 `pNN-run.txt`（原始输出 + 退出码）与 `pNN-*.json`（结构化读数）；`p01`–`p19` 编号即执行顺序。宿主进程已按 `evidence/*-host.local.json` 里记录的 pid 全部停止，端口 1530 / 3067 / 10932 / 2379 均已无监听。

## 七、局限

- 只测了 `0.2.0-rc.2` 一个宿主。作者的版本表提到 0.1.7-rc.2 与 desktop 构建，我没有条件回测。作者 0.2.10 的 release note 说正是因为 desktop 的 `hash` 变了才改用稳定属性选择器，这条我无法验证。
- 没有配 API Key（也不允许用真实密钥），所以没在真实模型对话流里跑过。插件只碰设置面板本身，理论上不相干，但确实没验。
- 假数据只有我自己造的隔离工作区，不是真实使用场景。
- 未测发布包以外的分发渠道（GitHub 直装、其它 registry 镜像）。
