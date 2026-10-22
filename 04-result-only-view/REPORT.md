# dsh-result-only-view 1.7.0（npm）实测报告

被测件：**npm 发布件 `dsh-result-only-view@1.7.0`**（`dist-tags.latest`，2026-10-01 发布，MIT，作者 YEYEYEYESHIFU）。
仓库固定提交 `47cfe87a7d183be2506fcca8ff9ed2e5f6069b2c`（package.json **1.6.4**）**仅用于源码差异核对**，没有安装进宿主运行过。本文所有运行读数都来自 1.7.0。

宿主 `@deepseek-ai/dsh` 0.2.0-rc.2，`--profile web --host 127.0.0.1 --port 0 --no-open`；隔离 `DSH_HOME`/`HOME`/`APPDATA`/npm cache；Edge 无头 + Playwright，1440×1000，网络只放行本机。用户正在使用的 GUI（1241 / 2538 / 4756）全程未访问。

---

## 一、结论先行

**本轮没有发现可提交的功能缺陷。** 主路径逐项与 README 描述一致：

| 用户会做的事 | 实测结果 |
|---|---|
| 找到开关 | 输入框左侧工具行，文案「◉ 只看结果」/「○ 只看结果」，rect (655,936,79,22) |
| 点一下 | 同一 tick 内思考行即 `display:none`，下一帧按钮状态翻转为开 |
| 看回答 | 回答正文、代码块、表格一字不少，思考正文不再出现在可见文本里 |
| 再点一下 | 注入样式被移除，所有元素 `display` 逐项回到初始值，无残留内联样式 |
| 刷新 | 开/关都能被正确读回并复现 |
| 换会话 | 状态是全局的，新会话立刻按当前状态渲染 |
| 长内容 | 2852 字正文 + 代码块 + 4 行表格完整，未被裁 |
| 展开中途切换 | 不留孤儿元素，切回来后内容还在 |
| 快速连点 20 次 | 停在与点击次数奇偶一致的状态，样式标签不累积，无报错 |

三项未取得读数（见第四节），一项因测试环境限制记不确定（冷重启）。

---

## 二、被测对象与两种版本的区分

| | npm 发布件 | 仓库固定提交 |
|---|---|---|
| 版本 | **1.7.0** | 1.6.4 |
| 标识 | `shasum aca77025e02ed3d31f07ba0d84bfccd7e7a57d9f`；`gitHead 4653ee8d…` | `47cfe87a7d183be2506fcca8ff9ed2e5f6069b2c` |
| 本轮用途 | **运行时被测件** | 仅源码差异核对 |

`dist.integrity` 已在本地用 tarball 复算一致（sha512），tarball 本地 sha256 `28167b44…`。

逐文件比对（`evidence/diff-01-1.7.0-vs-47cfe87.txt`）：

- **完全相同**：`lib/index.js`、`cordis.patch.yml`、`LICENSE`
- **不同**：`lib/client.js`（1.7.0 = 59,767 字节 / 1.6.4 = 53,761 字节）、`package.json`、`README.md`、`CHANGELOG.md`、`scripts/verify.mjs`

1.7.0 的变更日志说明这是一个**桌面端支持 + 0.2.0 DOM 适配**版本，关键几点（`evidence/diff-02-keys-and-features.txt`）：

1. `conversation.chat.turnTail` 在 0.1.x 是 chain 槽、0.2.0 是 list 槽，list 槽注册必须带 `id`，否则抛 `list slot "conversation.chat.turnTail" requires options.id`——修之前痕迹行在桌面端从不渲染，且以页面错误的形式暴露。
2. 0.2.0 把每一步的过程包进嵌套 group seat（`[data-step-process]` / `[data-chat-group-key]`），走查逻辑改成按 flow 的 DOM 顺序、并以两条 turn-tail 之间为界；步数统计从 `Processed 0 steps` 修到真实值。
3. 新增 `preparing` 工具状态，计入「进行中」。
4. 0.2.0 回合头部的 `Completed in …` 控制行被当过程行折叠，且排除在 reveal key 之外。
5. peer 依赖范围从 `*` 收紧为 `^0.1.1-rc.2 || ^0.2.0-rc.2`。

两版都包含 `dsh.result-only-view.enabled`、`dsh.result-only-view.settings`、`conversation.input.left`、`conversation.chat.turnTail`、`settings.general.item` 与 auto/manual 折叠模式，因此本报告测到的行为都落在两版共有的代码路径上——但**这只说明路径相同，不能反过来当成「1.6.4 也一定这样」的运行时结论**，1.6.4 没有跑过。

---

## 三、主要读数

### 3.1 开关位置与被误判的风险

页面上有 **4 个** `button[aria-pressed]`：自动换行、好的回答、有问题的回答、只看结果。被测开关是第 4 个：

```
text        = ◉ 只看结果
aria-pressed= true
title       = 只看结果已开启：过程已隐藏，运行中以实时摘要芯片展示正在进行的步骤（问答与插件审批卡片除外）。点击恢复完整单行折叠。
rect        = (655, 936, 79, 22)
borderRadius= 11px
祖先槽       = conversation.input.left ← conversation.composer.bar ← conversation.composer ← main.conversation ← main ← root
```

这解释了本轮一次探针自伤：早期版本用 `document.querySelector('button[aria-pressed]')` 取第一个匹配，命中的是「自动换行」，于是「点击后思考行没变」看起来像缺陷。做控件清点后才定位到真正目标（`evidence/run1-04-inventory.json`），probe-05 起全部重测。**`evidence/run1-03-core.json` 的读数作废，只作过程留痕，不用于任何结论。**

### 3.2 点击即时性

```
点前      pressed=false  styleTags=0  thinkHidden=0  thinkTotal=1
同一 tick pressed=false  styleTags=1  thinkHidden=1   ← 隐藏已生效，点击本身耗时 1.9ms
下一 rAF  pressed=true   label=◉ 只看结果
+50ms     pressed=true   styleTags=1  thinkHidden=1
+150ms    同上
+400ms    同上
```

即：**隐藏同步生效，按钮状态在下一帧翻转**。用户看到的是一次点击、画面立刻收起。

### 3.3 隐藏了什么、保留了什么（sess-rov-one）

开启后逐项：

| 元素 | 开 | 关 |
|---|---|---|
| 用户消息行 | block, 76px | block, 76px |
| turn-process | **none** | block, 33px |
| assistant-step（只有思考，无正文） | **none** | block, 0px |
| assistant-step（真正的回答） | block, **549px** | block, 549px |
| `[data-variant="think"]` | **none, 0px** | flex, 24px |
| 注入 `style[data-plugin=…]` | 1 个 | 0 个 |

会话可见文本：开启 386 字 / 关闭 385 字，两边都含 `END-OF-ANSWER-ONE`、表头「状态」、代码行 `function greet`、用户提问「给我解释一下」；开启态**不含**思考正文（`thinkInText=0`）。

### 3.4 还原保真

三态（关 / 开 / 再关）逐项比较：

- 5 个 flow 项 + think 行的 `display` 与 inline `style` 完全一致 → `identical=true`
- 关闭态所有被查元素的 `inline style` 均为 `null`（插件从来没写内联样式，全靠样式表注入/移除）
- 差异只出现在开启态：turn-process none、thinking-only assistant-step none、think none、turn-tail 高度 32px→68px（多出的痕迹行按钮）

### 3.5 刷新与切换会话

- 关 → `stored="0"` → reload → `pressed=false`、`styleTags=0`、think 可见
- 开 → `stored="1"` → reload → `pressed=true`、`label=◉ 只看结果`、`styleTags=1`、`thinkDisplayed=0`
- 关态下 one→multi：pressed 仍 false，3 个 think 全可见
- 开态下 multi→one：pressed 仍 true，think 全隐藏

一个实现细节值得使用者知道：**切会话时按钮 DOM 会被重建**。脚本里如果缓存了元素句柄，读到的会是脱离文档的旧节点，状态看起来「没变」。人用不受影响。

### 3.6 长内容、代码块、表格（sess-rov-long）

```
flow 文本        2852 字，含 END-OF-ANSWER-LONG 与「第 24 段」
<pre>            visible, display=block, overflowX=auto, maxHeight=none, 高 159
                 scrollHeight = clientHeight = 159  ← 没有内部截断
<table>          visible, display=table, 4 行, 742 × 172
                 父容器 overflow=auto, maxHeight=none  ← 没有裁切
滚动容器          scrollHeight 1924 / clientHeight 924
```

README 说「只看结果」不碰回答正文；这一项成立。

### 3.7 多轮与痕迹行

sess-rov-multi 有 3 条 turn-tail，各有自己的「已处理 1 步 · 5s ▸」按钮（`aria-expanded=false`）。点第 1 条 → 该按钮文案变「收起过程 ▾」、`aria-expanded=true`、第 1 个 think 变可见（`thinkVisible` 0→1），另两条仍 hidden；再点一次回到 0。**痕迹行的步数「1 步」与该回合 `turn-process` 计数 1 一致**（不是 0 步）。

关闭「只看结果」时痕迹行整体消失（`hasTrace=false`，turn-tail 只剩时间文本）。

### 3.8 两个异常

**展开中切换（7a）**：先把思考行点开（`data-expanded=true`、高 96px）→ 切开启 → `display:none`、高 0，但 `data-expanded` 保持 true → 切回关闭 → `display:flex`、高 96px、`hasBody=true`。全页孤儿扫描 `orphanCount=0`（不存在「自身可见但祖先 display:none」的元素）。

**连续快速切换（7b）**：20 次、间隔 30ms。

- 一次跑完：`finalPressed=false`（偶数次回到原点）、`styleTagCount=0`、`thinkDisplayed=1`
- 另一次跑完：停在 `pressed=true`、`styleTagCount=1`、CSS 字符数 7610（**只有一个** style 标签，没有累积泄漏）
- 全程 `errors=[]`、`consoleLog=[]`

**痕迹行展开中切开关（7c）**：痕迹行展开时关掉开关 → 痕迹行消失；再开启 → 痕迹行回来且回到折叠态（`aria-expanded=false`），点开后内容仍可读。

---

## 四、未取得读数的三项，以及为什么

### 4.1 工具调用卡是否被隐藏（untested）

三次尝试构造含工具调用的合成会话，都被宿主的会话格式校验拒绝：

1. 在 `assistant.message.content` 里放 `{type:'tool-call', …}` → `SessionFormatError: step/end leaves unresolved tool call call-1`
2. 改成 `tool/call` + `tool/result` 两条独立事件 → `format v4 tool/result at seq N requires toolCallId matching its tool source`
3. 把 `source.callId` 对齐到 `toolCallId`（写盘成功，12 条事件）→ 加载时 `SessionFormatError: tool/call call-1 has no advertised tool lifecycle`

同一个原因失败了三次，按纪律停止。**这不是插件的问题，是我们的夹具没造对**（还缺工具生命周期声明链）。三项宿主错误原文都在 `evidence/run3-10-tool.json` 与 `evidence/run5-12-tool.json` 里。

### 4.2 白名单卡片（ask_user_question / cordis_run）是否不被隐藏（untested）

本轮所有会话里都没有出现这两类卡片，没有观测对象。README 明确承诺它们不被隐藏，但**未验不等于成立**。

### 4.3 0 轮会话（untested）

`sess-rov-empty` 只有一条 `session/title` 事件、没有任何消息。它虽然登记在 `workspace.json` 的 `sessionIds` 里，侧栏就是不显示它（`treeitems` 里查不到该 `data-row-key`），因此点不进去。宿主对 0 轮会话的列表过滤是宿主行为，与本插件无关。

### 4.4 冷重启后状态保留（uncertain）

宿主用 `--port 0`，每次重启端口都变：`9798 → 4133 → 11114 → 5315 → 12495`。`localStorage` 按 origin 隔离，新 origin 读到的是空存储（`dsh.result-only-view.enabled = null`），而源码里缺省视为「开」。

所以「冷重启后仍是关」这件事在本测法下**原理上就不可能成立**，无法区分「插件没做跨 origin 迁移」与「本来就不该由 localStorage 承担」。同一 origin 内的 reload 读写都正确（3.5），说明写路径本身是好的。真实部署若固定端口，origin 不变，这项应当成立——**但本轮没有用固定端口复测**，因此记 uncertain 而不是 confirmed。

---

## 五、测试链路自身的两个坑（供复查者避开）

1. **`button[aria-pressed]` 不是唯一选择器**。页面上有 4 个，取第一个会点到「自动换行」。正确做法是按文案匹配 `只看结果` / `Results only`，并断言祖先槽含 `conversation.input.left`。
2. **不要用 PowerShell 的 `Set-Content -Encoding utf8` 改写 `workspace.json`**。它会写入 UTF-8 BOM（首字节 `EF BB BF`），宿主解析失败后表现为：页面显示「暂无会话」、侧栏 0 行、控制台报 `[session-controller] control stream failed: … Service "sessionController" is unavailable`。这个症状很容易被误判成「插件把会话弄没了」。改用 Node 的 `fs.writeFileSync(path, json, 'utf8')`（首字节 `{`）后立即恢复，侧栏重新列出 5 条会话。

另外：宿主**不热扫** `sessions` 目录，会话可见性由 `storages/workspace.json` 的 `tables.workspaces.<wsId>.sessionIds` 决定。新建的合成会话必须把 id 追加进该数组，再重启宿主才看得到。

---

## 六、环境与限制

- 宿主 `@deepseek-ai/dsh` 0.2.0-rc.2（`lib/bin.js`），`--profile web`，`--host 127.0.0.1 --port 0 --no-open`
- Node v24.19.0 / npm 11.17.0 / pnpm 11.7.0（随 CLI）/ Windows x64
- Edge 无头（Playwright `H:\CODEX\dsh-code-security-review-20261019\tooling\…`），1440×1000，`deviceScaleFactor=1`，service worker 阻断，路由只放行 `127.0.0.1|localhost|[::1]`
- 隔离：`DSH_HOME`、`HOME`、`USERPROFILE`、`APPDATA`、`LOCALAPPDATA`、`XDG_*`、`npm_config_cache` 全部在任务目录内
- **没测**：桌面端（Electron）——本轮只跑 web profile；真实模型下「运行中芯片」的行为（需要真实模型流式输出，未调用模型）；不同浏览器（只用 Edge）
- 会话内容全部是手写夹具，不是真实模型输出

---

## 七、复跑

见 `REPRODUCE.md`。最短路径：装 `temp/dsh-result-only-view-1.7.0.tgz` 进隔离 profile → `node make-sessions.mjs` → `node host.mjs <tag> web` → `node probe-05-core.mjs`（记得先按 `run1-host.local.json` 的 tag 对应上）。

## 八、证据索引

全部在 `evidence/`：

- 安装面：`install-01-add.txt`、`install-02-add-tarball.txt`、`install-03-profile-package.json`
- 夹具：`fixture-01-make-sessions.txt`、`fixture-02-tool-session.txt`、`fixture-03-tool2-session.txt`
- 控件清点：`run1-04-inventory.json` / `.png`
- 核心读数：`run1-05-core.json`、`run1-05-core.stdout.txt`、`run1-05-{on,off,long-on,multi,empty}.png`
- 持久化与异常：`run1-06-persist-anomaly.json`、`run1-06-expand-toggle.png`、`run1-06-rapid.png`
- 刷新/切会话/痕迹行：`run1-07-followup.json`、`run1-07-reload-{on,off}.png`、`run1-07-trace-click.png`
- 工具卡尝试（失败留痕）：`run3-10-tool.json`、`run5-12-tool.json`
- 版本差异：`diff-01-1.7.0-vs-47cfe87.txt`、`diff-02-keys-and-features.txt`
- 查重：`dedup-01-lingshulab.txt`、`dedup-02-bounty-github.txt`
- 宿主日志：`run{1..5}-host.{stdout,stderr}.txt`、`run{1..5}-host.local.json`
