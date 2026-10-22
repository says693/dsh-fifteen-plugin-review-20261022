# dsh-session-pin 0.7.19 实测报告（PerryLink/dsh-session-pin）

被测件：npm 发布件 `dsh-session-pin@0.7.19`（tarball sha1 `f551e6123d41cdab62ce7aefac8e6bd11d65b6b3`，sha512 与 registry `dist.integrity` 逐字节一致，SLSA provenance attestation 存在）。宿主 `@deepseek-ai/dsh` 0.2.0-rc.2，Windows x64 + Edge（Playwright 1.64.1 headless），隔离 `DSH_HOME`，`--host 127.0.0.1 --port 0 --no-open`。源码对照提交 `ed1a8ad0ce2724a9a783842e4d1b1e270fcfdad` 仅用于阅读定位，实际运行的是发布件。

## 一、先说结论：8 处与 README 不一致

按影响从大到小。每条都给了可复跑读数与证据文件名。

### 1. 导航条任何点击都会把「已置顶」面板关掉 —— 看板/标签/已存视图在鼠标路径下不可用

面板与导航条是两棵独立的 DOM 树：导航条 `div.__dsh-session-pin-nav__` 由 `document.createElement` 直接挂在 `document.body`（`lib/client.js:1163-1166`，`position:fixed;top:48px;right:300px`），面板 `div.__dsh-session-pin-panel__` 由 `shell.overlay` 槽渲染。而面板在打开时注册了 document 级 capture 阶段 click 监听：

```js
const onClick = (event) => {
  const target = event.target;
  if (target instanceof Node && panelRef.current?.contains(target)) return;
  ui.setOpen(false);
};
document.addEventListener("click", onClick, true);   // client.js:1682-1688
```

导航条不在面板内，所以点它必然命中 `setOpen(false)`。

读数（`evidence/run1-72-74-panel-close.json`）：

```text
t1  footer 点击          → panelOpen true
t2  点 nav 过滤输入框     → panelOpen false, document.activeElement null
   随后 type('请假')      → input.value 仍为 ""
t3  点 All chip          → panelOpen false
t4  点面板内 ⋯ 管理按钮    → panelOpen true, editor true   ← 阳性对照
```

对照截图：`run1-72-nav-click.png`、`run1-73-chip-click.png`、`run1-74-manage-in-panel.png`。

数据层其实是通的：`isolated-home/profiles/pin/cordis.patch.yml` 里能看到 `boards.byId`、`views` 被正确写入；面板内的 ⋯ 编辑器也能选看板、填标签、保存成功并把组头从「未分组」改成「复习 (1)」（`evidence/run3-01-assign-board.json`）。所以这是**点击路径**缺陷，不是存储缺陷：用户能建看板、能归组，但每次想点过滤器或看板 chip，面板先消失了。

### 2. 置顶后归档会话，pin 不清理，面板留下点不动的幽灵条目

用宿主的「归档会话」按钮归档一个已置顶会话后，侧栏不再显示它，但 footer 仍显示「已置顶的会话 (5)」（侧栏只剩 4 条）。刷新整页、重开面板，被归档的会话依然列在「已置顶」里。

```text
run1（归档 /goto 请假）
  sidebar  after: 5 条 → 4 条（不含归档项）
  panel    after reload: 5 行，含 /goto 请假
  footer   「已置顶的会话 (5)」
run3（归档 帮我整理一份关于海洋塑料污）— 独立复现
  afterArchivePanel.stillInPanel: false（当次 DOM 里临时消失）
  afterReload.panelRows 仍含「帮我整理一份关于海洋塑料污」
  patch yml 的 pinned 数组仍保留该 id
```

证据：`evidence/run1-80-81-anomalies.json`、`evidence/run1-82-83-health-reload.json`、`evidence/run3-02-03-filter-archive.json`、`run1-83-reload-prune.png`。

README 承诺 `pruneStale` 默认 true 时「Drop pins and colors for entities absent from a ready list」。源码 `onListChange()` 用 `list.ids` 作 live 集合（`client.js:634-648`），所以是否剪枝取决于宿主把归档会话从 `list.ids` 移除的时机；实测在刷新后仍未剪掉，用户看到的是「一个点了没反应的条目」。

### 3. `/goto` 在 0.2.0-rc.2 上完全不触发

在输入框输入 `/goto 请假` 回车：输入框被清空，当前会话不变，没有弹提示。这个会话的标题最后变成了 `/goto 请假`——说明那行字被当成普通消息文本处理了。

```text
composerKind: { tag:"DIV", isTextarea:false, closestScroll:true }
beforeEnter : text="/goto 请假", cur=session-6854de43…
afterEnter  : text="",            cur=session-6854de43…   ← 未切换
dialogs     : []                                          ← 无 alert
对照 dispatch KeyboardEvent('Enter') → defaultPrevented=true（宿主自己处理，插件未介入）
```

根因可对到源码（`client.js:1181-1184`）：

```js
if (!options.enableGoto || event.key !== "Enter" || event.isComposing) return;
const composer = event.target instanceof HTMLTextAreaElement
  && event.target.closest("[data-input-scroll]") !== null ? event.target : void 0;
if (composer === void 0) return;
```

本底座的 composer 是 `div[contenteditable=true][data-composer-input]`，不是 `HTMLTextAreaElement`，第一个判断就把事件挡掉了。证据：`evidence/run1-48-49-ws-goto.json`、`evidence/run1-41-43-nav-goto.json`、`run1-49-goto.png`。

因为入口没成立，README 里「多命中选择列清单」「无命中给说明」两个分支本轮无法验证（记 untested）；「命令行不进入模型」这条虽然会话日志里确实没有该文本，但那是命令根本没被识别的结果，不能算承诺兑现。

### 4. 工作区行没有置顶控件，工作区级置顶用不了

README 把「pin whole workspaces」列为两级置顶之一。实测工作区行 `div[role=treeitem][aria-expanded]` 内 `querySelector('[class*=__dsh-session-pin-badge__]')` 为 null，hover 后该行只有宿主自带的两个按钮（工作区操作 / 在该工作区新建会话）。

同一页面上会话行的 `[pin][swatch]` 是正常的，所以不是 overlay 整体没跑起来。源码 `workspaceIdFor` 靠「行内第一个非空 span 的文本命中 workspaces 快照 title」来认行（`client.js:1302-1307`），当前宿主把工作区标题放在 `span.YDXeBa_projectText` 与 `span.YDXeBa_title` 里，实测未命中。证据：`evidence/run1-48-49-ws-goto.json`、`evidence/run1-50-ws-health-detail.json`。

### 5. 点「已置顶」面板里的行不会切到那个会话

先把当前会话切到 A（用侧栏行点击，`dsh.sessions.current` 确实变成 A），再打开面板点第一行 B：

```text
before      : cur = session-08fbbad4…（A，海洋塑料）
clicked row : 写一封给班主任的请假邮件（B，session-3c8badf3…）
after       : cur = session-08fbbad4…（仍 A）
              sidebar aria-selected 仍 A
              会话头标题仍 A 的标题
```

同一 context 内用侧栏点 A 时立刻切换（cur 变 A、会话头出现 A 的标题）。两次独立复现（`evidence/run1-44-panel-nav.json`、`evidence/run1-39-panel-click.json`）。面板行的 `openRow` 最终走 `c.sessions.retain(id,{source:'gateway'})`；未定位该 retain 在本基线为何不生效。

### 6. 健康摘要恒为空

5 条已置顶行都有 `span.__dsh-session-pin-nav__health`，但 `innerText` 恒为空。试过：逐条把会话切成当前会话后重开面板（4 次）、把组头折叠再展开触发重算、刷新页面后重读——始终为空。而这些会话在会话区确实有用户消息。

源码 `summarizeHealth` 只数 `user/message` 与 `assistant/message`，数据来自 `c.sessions.binding(id).session.getSnapshot().nodes`，未绑定的会话返回 `undefined` → 文本为空（`client.js:206-223`、`996-1013`、`2032-2041`）。观察到「所有行都空」，但没确认是绑定从未建立还是节点 kind 不匹配，成因部分记 uncertain。证据：`evidence/run1-85-health-precise.json`、`run1-85-health-precise.png`。

### 7. 置顶后行不前移（updated 与 manual 两种排序都测了）

README 的核心承诺之一是「a pinned session to the front of its account」。实测：

```text
orderBy=updated：置顶最末行 → 该行 y 仍 378；再置顶中间行 → y 仍 276
orderBy=manual ：清空所有 pin → 置顶最末行 → y 仍 378，其它行 y 不变
                置顶中间行 → 同样不前移
```

`localStorage['dsh.workspace.view.v5']` 的 `orderBy` 每次切换后都回读确认过（`updated` → `manual`）。宿主侧 `workspace/insertSessionBefore` 返回 200，但列表没重排。未定位是 `reorderMoves` 的判据（`client.js:357-364`，要求 head 前缀全部等于 present 才跳过）还是宿主未采用插入结果。证据：`evidence/run1-10-pin-basic.json`、`evidence/run1-28-manual-seq.json`、`evidence/run2-02-03-header-order.json`。

### 8. 会话头部的置顶按钮没渲染

README 的四类置顶界面里有一项「a pin toggle in the session header」。两次独立上下文里 `button.__dsh-session-pin-header__` 均为 null，虽然槽位 `conversation.session.header.actions` 在 DOM 中存在。证据：`evidence/run1-39-panel-click.json`、`evidence/run1-44-panel-nav.json`、`evidence/run2-02-03-header-order.json`。

### 附：每次置顶一条 404

每次置顶/取消置顶都会 `POST /api/session.setPinned` → 404，然后自动降级 `POST /api/settings/mutate` → 200（`ns=session-pin`、`op=set`、`path=["pinned"]`），状态最终正确。控制台因此每次置顶留一条 `Failed to load resource: 404`。这是降级路径在起作用，不算功能失效，但属于可清理的噪音。证据：`evidence/run1-40-rpc-trace.json`。

## 二、这些是好的

- **发布件干净**：sha1 / sha512 与 registry 完全一致，14 个文件 / unpacked 504916 字节，带 SLSA provenance，Apache-2.0 LICENSE 在位；`dsh plugin add` exit 0，bundles 正确（pnpm 报了 peer 依赖 WARN 但没阻断）。
- **置顶与配色本身可靠**：点 badge 后 `aria-pressed=true`、`title=取消置顶`、加 `__dsh-session-pin-pinned__`；点色块循环色板 `data-color=c0→c1`，行背景变 `rgba(249,115,22,0.1)`→`rgba(234,179,8,0.1)`，`box-shadow: rgb(234,179,8) 3px 0 0 0 inset` 左侧色条；Shift+点击清色后背景与色条一起消失、置顶态保留。
- **取消置顶能干净回原位**：5 条全置顶后逐条取消，行 key 序列与基线完全一致（`orderRestored=true`），y 坐标 242/276/310/344/378 逐项复原。
- **持久化落在正确的地方**：状态写在宿主设置里（profile patch 的 `- id: session-pin` 段：pinned 数组、colors 映射、boards、views），`localStorage` 里没有任何 pin 键。因此冷重启（origin 从 8677 变 7552）后置顶、颜色、看板、视图全部保留——这一点比 README 说的还稳。
- **看板与过滤可用（只要走面板内）**：面板内 ⋯ 编辑器能选看板、填标签、保存，组头立刻变成「▾ 复习 (1)」「▾ 未分组 (4)」，`data-board` / `data-tags` 正确；过滤输入「请假」后 5 行只剩 2 行，清空后全恢复。
- **不干扰宿主自身**：插件过滤只改面板行的 `display`，侧栏 6 条树项不动，宿主自己的「搜索会话名称」框与视图选项菜单各自独立。
- **窄窗口不溢出**：1440/1100/900/760/640 五档 `scrollWidth <= clientWidth`；900 以下宿主把侧栏收成 55px 图标栏导致行内 pin 不可见（宿主行为），面板与导航条自身仍完整可见且右对齐。

## 三、环境与限制

| 项 | 值 |
|---|---|
| 宿主 | `@deepseek-ai/dsh` 0.2.0-rc.2，`lib/bin.js`，profile `pin` |
| 启动 | `--host 127.0.0.1 --port 0 --no-open` |
| 浏览器 | Edge（Playwright 1.64.1 `chromium.launch`，headless），视口 1440×1000 为主 |
| 隔离 | `DSH_HOME`/`HOME`/`USERPROFILE`/`APPDATA`/`LOCALAPPDATA`/npm cache 全在任务目录内 |
| 端口 | 8677（run1）、7552（run2 冷重启）、5956（run3），全部只监听 127.0.0.1 |
| 模型 | 无真实密钥；会话消息一律 `MISSING_CREDENTIAL` 失败，属环境限制，与插件无关 |
| 用户 GUI | 未触碰 1241 / 2538 / 4756 |

限制与未验：

- **/goto 两个分支**（多命中列清单、无命中说明）被 `/goto` 入口缺陷阻断，未验。
- **视图回放**：`+ view` 能把过滤存成命名视图（数据层可读到），但点视图 chip 会先关面板（缺陷 1），无法在同一屏内看到筛选结果，记 uncertain。
- **health 内容层**：没取到非空文本，因此「只统计计数与方向、不含正文」只有源码依据，不是 UI 读数。
- **桌面端（Electron）与远端浏览器**未测。
- 宿主在隔离 `DSH_HOME` 下仍把内置默认工作区指到 `H:\视频\document\deepseek-harness\default-workspace`（该目录 2026-10-19 已存在，本轮只读旁证，会话文件仍落在 `isolated-home/sessions/`）。这是宿主行为，不影响插件结论。

## 四、最小复现

见 `REPRODUCE.md`。最短路径：起隔离宿主 → 用 `dsh plugin --profile pin add <tgz>` 装发布件 → 发一条消息让会话有标题 → hover 该行点置顶 → 开 footer 面板 → 点导航条任意位置（面板应保持打开，实测关闭）。
