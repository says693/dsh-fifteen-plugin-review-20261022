<!-- intake:v1 {"project":"bench","target":"Han-1413141/dsh-sticky-disclosure","form":"intake"} -->

# [intake 输入] dsh-sticky-disclosure 1.0.1：常驻计数与一键收起在官方宿主上按文档工作，未复现出可提交缺陷

**未找到可提交缺陷，本文件作复核留档。**

在官方 `@deepseek-ai/dsh` 0.2.0-rc.2 的隔离宿主里装上这个插件后，右下角的「全部收起」按钮、它的实时计数、以及可改可存的快捷键都按 README 描述工作。展开几个区块时按钮从 `96×32` 长到 `105×32` 并显示「·N」，点一下（真实鼠标点击）全部收回、计数回空；40 个区块时一次点击 14ms 清空、没有漏收；0 个区块时按钮照旧在、点了不报错。唯一值得维护者留意的是一条「刷新后不保留展开态」的行为，但 README 并未承诺记住展开态，因此不作为缺陷提交。

## 复现

1. `dsh --profile sdweb --from-default-profile web --dump-config`
2. `dsh plugin --profile sdweb add H:/CODEX/fifteen-plugin-reviews-20261022/r3-02-sticky-disclosure/temp/repo`
3. 用官方 `dsh-session-persistence-jsonl` 写合成会话：`sess-sd-zero`（0 块）、`sess-sd-one`（1 轮）、`sess-sd-six`（6 轮 = 6 turn-process + 6 Think + 6 工具卡 = 18 行）、`sess-sd-many`（20 轮 = 60 行 / 40 个可折叠）
4. 起宿主：`dsh --profile sdweb --host 127.0.0.1 --port 0 --no-open`（本次 OS 分配 9578）
5. Edge headless 打开该 origin，关掉「预览版说明」欢迎层，打开 `sess-sd-six`
6. 依次展开 0→4 个区块，每步读 DOM 实况 / `data-count` / `.dshSd_count` 文本 / `window.dshStickyDisclosure.expanded()`
7. 在按钮 `boundingBox` 中心做 `page.mouse.click`，再读一次
8. 点齿轮 → 「设置」→ 按 `Ctrl+Shift+K` → 关闭面板 → 分别按旧键 `Ctrl+Alt+C` 与新键

最小复现步骤全文见 `REPRODUCE.md`。

## 环境

- 对象：`Han-1413141/dsh-sticky-disclosure` 1.0.1，MIT，固定提交 `d844605756b9d4c2f5e723ee67ea0f65476d7085`
- **npm 无发布件（`npm view dsh-sticky-disclosure version` → E404），以本地目录安装该提交，未测发布包**
- 宿主：`@deepseek-ai/dsh` 0.2.0-rc.2，`--profile sdweb --host 127.0.0.1 --port 0 --no-open`，pid 2024
- 浏览器：Edge 155.0.4283.62 headless，Playwright 1.64.1，独立 context，只放行本机地址
- 平台：Windows；Node v24.19.0；`DSH_HOME` / `HOME` / `USERPROFILE` / `APPDATA` / `npm_config_cache` 全部隔离在任务目录内
- 会话数据为手写合成 fixture，未调用模型，未使用任何真实密钥或账号

## 证据（相对 `evidence/` 的文件名）

- `05-probe01.txt`、`run01-01-boot.json`、`run01-02-open.json` —— 无会话时按钮已常驻：rect `{1286,952,96,32}`、`data-count="0"`、计数文本空串、`title` 带 `Ctrl+Alt+C`、`z-index:15`
- `16-probe06-core.txt`、`run01-06-core.json` —— 展开 0/1/2/3/4 时四方读数逐条对齐 `agree=true`；真实鼠标点击后 `pillClickMs=5`、`domExpanded=0`
- `06-core-a-baseline.png` / `06-core-b-expanded.png` / `06-core-c-collapsed.png` —— 收起前中后三态截图
- `17-probe07-hotkey.txt`、`run01-07-hotkey.json` —— 面板 `role=dialog` `320×197`、改绑写入 `localStorage["dsh-sticky-disclosure:hotkey"]`、旧键失效、新键生效、恢复默认、刷新后保持 `Ctrl+Shift+K`
- `07-hotkey-a-panel.png` / `07-hotkey-b-rebound.png` / `07-hotkey-c-reset.png` —— 面板三态截图
- `18-probe08-abnormal.txt`、`run01-08-abnormal.json` —— 0 块时 `rows=0`、按钮仍在、`clickOk=true`、点后 `expanded=0`；40 块时 `expanded=19`（首版探针未点全）、`agree=true`
- `19-probe09-many.txt`、`run01-09-many.json` —— 40 个展开 `agree=true`、`dockChips=30`、一次点击 `clickMs=14`、`leftoverCount=0`
- `08-abnormal-c-many-expanded.png` / `09-many-a-expanded.png` / `09-many-b-collapsed.png` —— 大量区块的展开/收起截图
- `20b-probe10-visual.txt`、`run01-10-visual.json`、`10-visual-{a-light-1440,b-dark-1440,c-narrow-320,d-narrow-dark-320,e-light-1280}.png` —— 五组视口/主题：按钮 `inView=true`、`topmost` 是按钮自己、无横向溢出、对比度浅色 5.8:1 / 深色 9.25:1
- `21-probe11-reload.txt`、`run01-11-reload.json`、`11-reload-a-before.png`、`11-reload-b-after.png` —— 刷新前 `expanded=3`、刷新后 `0`、`stateRemembered=false`
- `15-diag-actions.txt`、`13-diag-click2.txt`、`12-diag-click.txt` —— 宿主不可见操作栏 `div.xzv4MW_actions`（`opacity:0`、`pointer-events:auto`、`479,227,748×28`）拦截 `locator.click` 的取证
- `run01-05a-count.json`、`07-probe03-six.txt` —— 四条点击超时的原始报错
- `08-diag-overlay.txt` —— 首启的两层遮挡（预览版说明遮罩 + 「添加一个 API Key 开始使用」对话框）
- `01-derive-profile.txt`、`02-plugin-add.txt`、`03-make-sessions.txt`、`03b-make-sessions.txt`、`03c-make-sessions.txt`、`04-host-boot.txt`、`run01-host.local.json` —— 安装与启动链路
- `run01-04-six.json`（失败）与 `run01-04b-six.json`（成功）—— format v4 tool-call 身份比较导致的会话损坏与其修复

## 已验

1. 常驻按钮在 0 会话时就渲染，位置 `{x:1286,y:952,w:96,h:32}`、`z-index:15`，`data-count="0"` 且计数文本为空串（不是「·0」）。
2. 计数四方一致：展开 0/1/2/3/4 时 DOM 实况、`data-count`、`.dshSd_count` 文本（空/`·1`/`·2`/`·3`/`·4`）、`window.dshStickyDisclosure.expanded()` 全部相等，每步 `agree=true`。
3. 一键收起作用且快：真实鼠标点击 `pillClickMs=5`（4 个区块）、`clickMs=14`（40 个区块），收起后 `expanded=0`、`data-count="0"`、`leftoverCount=0`。
4. 40 个区块时计数仍准确（`data-count="40"`、文本「·40」、`expanded()=40`、`agree=true`）。
5. 快捷键全链路：默认 `Ctrl+Alt+C` 生效；改绑 `Ctrl+Shift+K` 后写入 `localStorage["dsh-sticky-disclosure:hotkey"]={"ctrl":true,"meta":false,"alt":false,"shift":true,"code":"KeyK"}`；旧键失效（1→1）、新键生效（1→0）；恢复默认回 `Ctrl+Alt+C`；刷新后仍是 `Ctrl+Shift+K`。
6. 0 块对照：`rows=0` 时按钮仍在、点击不抛错、无控制台错误。
7. 视觉：1440/1280/320 三档宽度 × 浅/深两套主题，按钮都在视口内、都没被别的层盖住（`topmost` 命中 `dshSd_label` 自身）、无横向溢出；深色对比度 9.25:1。

## 未验

- **宿主冷重启**（杀进程再起）后按钮、计数、快捷键偏好是否恢复：`evidence/` 里只有一个宿主实例（run01，pid 2024），未做第二次启动。
- **页面开着时追加新轮次**，新出现的区块是否自动收起：所有探针都是先写好会话再开页面。
- **点击顶部 chip** 是否收起对应区块：只统计了 `dock.chips` 数量（2 / 30），没点过。
- **输入框聚焦时快捷键**、IME `isComposing`、AltGr 组合、Esc 取消捕获：无对应用例。
- **禁用/卸载后 DOM 与样式是否全量还原**：全程只有一个插件启用态。
- **未测发布包**：npm 没有发布件。

## 不确定

- **320px 宽下的实际可用性**：首跑 `openSession` 等 20s 超时，原因是宿主侧栏在窄窗默认折叠、`[data-row-key]` 不可见（`20-probe10-visual.txt`）；重跑（`20b-probe10-visual.txt`）拿到了按钮读数，但内容区 `scrollport` 只剩 38px 宽。按钮是可见的，但「这么窄还能不能正常用」没有结论。
- **刷新后快捷键是否失效**：`21-probe11-reload.txt` 最后一阶段按的是 `Ctrl+Shift+K`，而该轮并没有先改绑（phase 里 `hotkey` 记录仍是 `Ctrl+Alt+C`，`storage=null`）。断言与按键不匹配，属探针问题，不能读成插件失效。

## 不做

- 不修改插件源码、不加兜底实现。
- 不调用真实模型、不读任何真实密钥或登录态、不使用第三方账号。
- 不触碰用户正在使用的 DSH Web GUI（1241 / 2538 / 4756），不读它的 token / cookie / 真实会话。
- 不上传、不投稿、不新建公开仓库、不改任何线上记录。
- 不为凑「有缺陷可报」而重新探测；没有新问题就照实交付复核结果。

## 建议

给维护者，三条都属文档/体验层面，不是功能错误：

1. **README「行为细节」补一句「刷新/重开页面后展开态不保留，会回到全部收起」**。实测 `stateRemembered=false`，刷新前展开 3 个、刷新后 0 个。用户读到一半刷新会以为是插件把内容吃掉了，写清楚就不慌。
2. **`docs/COMPATIBILITY.md` 里注明窄窗行为**。320px 宽时宿主把侧栏折叠、内容区只剩 38px，插件的按钮虽然还在视口内，但那个宽度下任何折叠 UI 都很难用；说明「建议 ≥ 800px 宽使用」比让用户自己猜好。
3. **调试句柄值得在 README 里单独立一小节**。`window.dshStickyDisclosure = {version, expanded(), affixed(), hotkey(), setHotkey(spec)}` 对做自动化的用户很有用（本次实测就是靠它做四方对齐的），现在只在一行里带过。

给评测站读者的一句话：这是一个「装上就把一件小事办利索」的插件，没有花活，也没有坑。
