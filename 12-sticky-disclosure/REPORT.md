# dsh-sticky-disclosure 1.0.1 技术报告

对象：`Han-1413141/dsh-sticky-disclosure`，固定提交 `d844605756b9d4c2f5e723ee67ea0f65476d7085`（package.json `1.0.1`，MIT）。
**npm 上无该包的发布件（`npm view` 返回 E404），本报告以本地目录方式安装该提交的源码树，全部结论只覆盖源码树，未测发布包。**

转写说明：本文件由 `evidence/` 下 68 个原始读数转写而来，没有新起宿主、没有新开探针。所有读数都能在 `evidence/` 里找到对应文件。

---

## 一、结论先说

**没找到可以提交的缺陷。** 插件承诺的三件事——常驻按钮带实时计数、一键收起全部展开区块、可改可存的快捷键——在官方宿主 0.2.0-rc.2 上逐项复现成功，读数与 UI 实况一致到「四方对齐」的程度（DOM 实况 / `data-count` 属性 / 可见文本 / 插件自报 `window.dshStickyDisclosure.expanded()`）。

同时有三块事实需要维护者知道：

1. **刷新不保留展开态**（`stateRemembered=false`）。刷新前展开 3 个，刷新后全部收回、计数归零。README 没有承诺记住展开态，所以这不算违约，但从「我读到一半刷新了」的角度看是清晰可感的行为，建议在 README「行为细节」里补一句。
2. **40 个区块时的收起是瞬时的**（`clickMs=14`），但**逐个展开**的开销是探针侧的（39116ms），与本插件无关，别被截图时间误导。
3. **一条读数不能用**：`21-probe11-reload.txt` 最后一阶段按了 `Ctrl+Shift+K` 却断言 `hotkey` 仍是 `Ctrl+Alt+C`——那是探针脚本的键位和断言不匹配（该轮没有先改绑），不是「刷新后快捷键失效」。已在 `RESULTS.json` 记为 `uncertain`。

---

## 二、验到的行为（带读数）

### 2.1 常驻按钮与计数

开机后 `[data-sticky-disclosure-control]` 就已经在页面上（此时没有任何会话，`rows=0`）：

- rect `{x:1286, y:952, w:96, h:32}`，`visible=true`，`z-index: 15`
- 文本「全部收起」，`title="收起全部展开区块 · Ctrl+Alt+C"`，`aria-label="收起全部展开区块"`
- `data-count="0"`，`.dshSd_count` 文本为空串（不是「·0」）
- 证据：`evidence/05-probe01.txt`、`evidence/run01-01-boot.json`

计数在 6 轮会话（18 行 = 6 个 turn-process + 6 个 Think + 6 个工具卡）里逐次展开，每一步四方对齐：

| 展开数 | DOM 实况 | `data-count` | 可见文本 | `expanded()` | 一致 |
|---|---|---|---|---|---|
| 0 | 0 | 0 | （空） | 0 | true |
| 1 | 1 | 1 | `·1` | 1 | true |
| 2 | 2 | 2 | `·2` | 2 | true |
| 3 | 3 | 3 | `·3` | 3 | true |
| 4 | 4 | 4 | `·4` | 4 | true |

证据：`evidence/16-probe06-core.txt`、`evidence/run01-06-core.json`

按钮尺寸随位数变化但钉位不变：0 个时 `96×32`（x=1286），有计数时 `105×32`（x=1277），40 个时 `112×32`（x=1270），`y` 恒为 952；右缘距滚动区右边 -56px、下缘距底边 -16px。

### 2.2 一键收起

用的是真实鼠标事件（`boundingBox` 中心 `page.mouse.click`），不是 DOM 派发：

- 展开 4 个 → 点击后 `pillClickMs=5`，`domExpanded=0`、`data-count="0"`、文本回空串、`expanded()=0`、`dockChips=0`
- 40 个（20 Think + 20 工具卡）→ 点击后 `clickMs=14`，`expanded=0`、`leftoverCount=0`，没有漏收
- 证据：`evidence/16-probe06-core.txt`、`evidence/19-probe09-many.txt`、`evidence/06-core-c-collapsed.png`、`evidence/09-many-b-collapsed.png`

### 2.3 快捷键全链路

| 步骤 | 读数 |
|---|---|
| 默认 `Ctrl+Alt+C` | 展开 1 个 → 按后 0，`pass=true` |
| 齿轮打开面板 | `role=dialog`，`320×197` @ `(1102,747)`，文本「收起全部快捷键 / Ctrl + Alt + C / 设置 / 恢复默认 / 快捷键保存在此浏览器中，刷新后仍然有效」，`focus=capture` |
| 点「设置」 | `data-armed` 置位，按钮文案变「请按新组合键…」 |
| 按 `Ctrl+Shift+K` | kbd 变 `[Ctrl, Shift, K]`；`localStorage["dsh-sticky-disclosure:hotkey"]={"ctrl":true,"meta":false,"alt":false,"shift":true,"code":"KeyK"}`；`hotkey()="Ctrl+Shift+K"` |
| 旧键失效 | 展开 1 个后按 `Ctrl+Alt+C` → 仍为 1 |
| 新键生效 | 按 `Ctrl+Shift+K` → 0 |
| 「恢复默认」 | kbd 与 storage 都回 `Ctrl+Alt+C` / `{ctrl:true,alt:true,code:KeyC}` |
| 刷新后 | `hotkey()` 仍是 `Ctrl+Shift+K`，storage 未丢 |

证据：`evidence/17-probe07-hotkey.txt`、`evidence/run01-07-hotkey.json`、`evidence/07-hotkey-a-panel.png`、`evidence/07-hotkey-b-rebound.png`、`evidence/07-hotkey-c-reset.png`

### 2.4 异常对照

- **0 个可折叠块**：`rows=0`，按钮照旧常驻 `96×32`、`data-count="0"`、文本「全部收起」；点它 `clickOk=true`，之后 `expanded=0`、`pill="0"`，无异常、无控制台错误。证据：`evidence/18-probe08-abnormal.txt`、`evidence/08-abnormal-a-zero.png`
- **40+ 块**：`rows=60 / think=20 / tool=20 / turnProcess=20`，展开 40 个后三方一致；一次点击清空，`leftover=[]`。证据：`evidence/19-probe09-many.txt`、`evidence/09-many-a-expanded.png`
- **顶部 chip**：展开 2 个时 `dockChips=2`，40 个时 `dockChips=30`；截图里能看到顶部成排的 chip（「搜索文件内容 p1 / 思考 / …」）。证据：`evidence/08-abnormal-c-many-expanded.png`

### 2.5 视觉与主题

| 视口 | 主题 | 按钮位置 | 对比度 | 是否被遮挡 | 横向溢出 |
|---|---|---|---|---|---|
| 1440×1000 | 浅色 | (1277, 952) 105×32 | 5.8:1 | 否（topmost 是 `dshSd_label` 自己） | 无 |
| 1440×1000 | 深色 | (1277, 952) 105×32 | 9.25:1 | 否 | 无 |
| 1280×900 | 浅色 | (1117, 852) 105×32 | 5.8:1 | 否 | 无 |
| 320×700 | 浅色 | (157, 652) 105×32 | 5.8:1 | 否 | 无 |
| 320×700 | 深色 | (157, 652) 105×32 | 9.25:1 | 否 | 无 |

深色下 `bg=rgb(44,44,46)`、`color=rgb(207,211,214)`、`border=rgba(255,255,255,0.16)`，说明它确实吃的是宿主的 `--dsw-*` 令牌而不是硬编码白底。证据：`evidence/20b-probe10-visual.txt`、`evidence/run01-10-visual.json`

### 2.6 刷新

刷新前 `expanded=3`、`105×32`、文本「全部收起 ·3」、`dockChips=2`；刷新后 `expanded=0`、`96×32`、文本「全部收起」、`dockChips=0`；随后重新展开 2 个 → `expanded=2`、文本「·2」，计数恢复正常。`stateRemembered=false`。证据：`evidence/21-probe11-reload.txt`、`evidence/11-reload-a-before.png`、`evidence/11-reload-b-after.png`

---

## 三、没测到 / 不能定的

| 项 | 状态 | 原因 |
|---|---|---|
| 宿主冷重启（杀进程再起）后按钮、计数、快捷键偏好 | untested | `evidence/` 里只有一个宿主实例（run01，pid 2024）；上一执行者在冷重启前退出，本轮边界禁止起宿主 |
| 页面开着时追加新轮次，新块是否自动收起 | untested | 所有探针都是先写好会话再打开页面，没有「运行中追加」的记录；需宿主与探针协同 |
| 点击顶部 chip 是否收起原区块 | untested | 现有探针只统计 `dock.chips`，没点过 chip |
| 输入框聚焦时快捷键、IME `isComposing`、AltGr、Esc 取消捕获 | untested | 需合成 IME composition 事件与置焦用例，属新探针范围 |
| 禁用/卸载插件后 DOM 与样式是否全量还原 | untested | 全程只有一个插件启用态 |
| 320px 宽下的实际可用性 | uncertain | 首跑 `openSession` 等 20s 超时，原因是宿主侧栏在窄窗默认折叠、`[data-row-key]` 不可见；重跑拿到按钮读数，但内容区 `scrollport` 只剩 38px 宽，插件在这么窄的区里好不好用没测到 |
| 刷新后快捷键是否失效 | uncertain | `21-probe11-reload.txt` 按的是 `Ctrl+Shift+K` 却断言 `Ctrl+Alt+C`（该轮未先改绑），是探针与断言不匹配，不是插件读数 |

---

## 四、方法学备注（必须写进上下文，否则读数会被误读）

1. **展开行只能用程序化点击。** 宿主每条回复标题行上有一个 `div.xzv4MW_actions`：`opacity: 0` 但 `pointer-events: auto`，矩形 `(479,227,748×28)` 正好盖住标题行。`elementFromPoint` 命中的不是行元素，所以 Playwright `locator.click` 一律 15s 超时（见 `evidence/07-probe03-six.txt`、`evidence/run01-05a-count.json` 的四条失败记录）。改用 `element.click()` 后 `expanded` 从 0 走到 1（`evidence/13-diag-click2.txt`）。
   - 这是宿主不可见操作栏的遮挡，**不是插件缺陷**。
   - 影响范围：C3/C4/C7 的「展开动作」是程序化的；C5/C6 的「收起动作」是真实鼠标事件，不受影响。
2. **首次启动有两层遮挡。** 「预览版说明」欢迎层（`div[role=presentation]._root_o6lrb_6` 里的 `aria-hidden` 遮罩）与「添加一个 API Key 开始使用」对话框（`evidence/08-diag-overlay.txt`）。探针用 `dismissOnboarding` 关掉。本轮没有输入任何密钥、没有调用模型。
3. **会话是手写合成 fixture。** 用官方 `dsh-session-persistence-jsonl`（format v4 + zstd）写入，`assistant/message` 的 content 里带 `kind:"reasoning"` 的 block 渲染成 Think 行，`tool/call` + `tool/result` 渲染成工具卡。format v4 按**引用相等**比较 tool-call 的 `arguments`，所以 assistant 里公告的 tool-call block 与 `tool/call` 事件必须传同一个 JSON 字符串；第一版没这样做，页面直接报 `stored session "sess-sd-six" is corrupt … tool/call call-001 does not match one advertised tool call`（`evidence/run01-04-six.json`），修好后正常（`evidence/run01-04b-six.json`）。
4. **40 个区块的 39 秒不是插件性能。** `expandMs=39116` 是探针逐行点击的累计值，`clickMs=14` 才是插件收起一次的耗时。
5. **端口是 `--port 0`**，OS 分配到的 9578 只属于本次运行；用户正在使用的 4756 未被触碰。

---

## 五、环境

- 宿主：`@deepseek-ai/dsh` 0.2.0-rc.2，`--profile sdweb --host 127.0.0.1 --port 0 --no-open`，pid 2024，origin `http://127.0.0.1:9578`
- profile `sdweb` 由 `--from-default-profile web` 派生后再 `dsh plugin add`（否则缺 `dsh-web-app`、宿主不打印 URL）
- 浏览器：Edge 155.0.4283.62 headless，Playwright 1.64.1，独立 context，只放行本机地址
- Node v24.19.0；`DSH_HOME`/`HOME`/`USERPROFILE`/`APPDATA`/`LOCALAPPDATA`/`XDG_*`/`npm_config_cache` 全部指向任务目录内
- 插件以 `link:` 方式装到 `temp/repo`（提交 `d8446057…`），未改任何源码（`git status --porcelain` 为空）

## 六、限制

- 单机单次会话，未跨机器、未跨宿主版本复验。
- 未测发布包（npm 无发布件）。
- 未跑真实模型轮次，也就没测「真实模型回复里的 Think 块」与合成 fixture 的差异。
- 评测站只扫了第 1 页（`?q=` 服务端不过滤，需拉全量），查重不构成全量查清。
