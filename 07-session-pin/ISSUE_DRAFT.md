# [dsh-session-pin 0.7.19] 导航条点击会关掉「已置顶」面板，看板/标签/已存视图在鼠标路径下不可用（另附 7 条实测不一致）

<!-- marker: PerryLink/dsh-session-pin -->

打开「已置顶的会话」面板后，只要点一下导航条上的过滤输入框、看板标签或「+ view」，面板立刻关闭，输入框连焦点都拿不到；而看板/标签/已存视图这三个功能恰好只存在于导航条上，所以它们在真实鼠标操作下用不了。另查出击穿置顶主承诺的几条：置顶后行不前移、置顶后归档会话留下点不动的幽灵条目、`/goto` 在 0.2.0-rc.2 上完全不触发、工作区行没有置顶控件、面板行点击不切会话、健康摘要恒为空、会话头置顶按钮不渲染。

被测件为 npm 发布件 `dsh-session-pin@0.7.19`（tarball sha1 `f551e6123d41cdab62ce7aefac8e6bd11d65b6b3`、sha512 与 registry `dist.integrity` 逐字节一致、带 SLSA provenance），作者 PerryLink，Apache-2.0。宿主 `@deepseek-ai/dsh` 0.2.0-rc.2，Windows x64 + Edge 155（Playwright 1.64.1 headless），隔离 `DSH_HOME`，`--host 127.0.0.1 --port 0 --no-open`。

---

## 主条目（confirmed）：导航条点击关闭面板

**操作与现象**

1. 侧栏底部点「已置顶的会话」，面板打开（`rect(1130,48,298,205)`，`role=dialog`）。
2. 点导航条上的过滤输入框 `input.__dsh-session-pin-nav__input`。
3. 面板立刻消失，`document.activeElement` 变成 `null`；继续输入无效果（`value` 仍为空）。
4. 同样地，点 `All` 看板 chip 也会关面板。
5. 对照：点面板内部的 `⋯` 管理按钮，面板保持打开、编辑器正常出现。

**读数**（`evidence/run1-72-74-panel-close.json`）

```text
t1 footer 点击            → panelOpen true
t2 点 nav 过滤输入框       → panelOpen false, activeElement null
   type('请假')            → input.value ""
t3 点 All chip            → panelOpen false
t4 点面板内 ⋯ manage      → panelOpen true, editor true
```

**成因**（源码可对上）

导航条挂在 `document.body` 上（`lib/client.js:1163-1166`），面板由 `shell.overlay` 槽渲染；面板打开时注册了 document 级 capture click 监听，目标不在面板内就关面板（`lib/client.js:1682-1688`）：

```js
const onClick = (event) => {
  const target = event.target;
  if (target instanceof Node && panelRef.current?.contains(target)) return;
  ui.setOpen(false);
};
document.addEventListener("click", onClick, true);
```

导航条不是面板的子节点，因此必然命中 `setOpen(false)`。

**影响**：README 宣称的 boards / tags / saved views 三个能力无法在鼠标路径下使用。数据层是通的（`isolated-home/profiles/pin/cordis.patch.yml` 能看到 `boards.byId`、`views` 被写入，面板内 ⋯ 也能成功归组），所以是点击路径问题，不是存储问题。

**建议**：把导航条渲染进面板子树（或在关闭判定里同时认 nav bar 的 ref）；`panelRef.current?.contains(target)` 改成 `panelRef.current?.contains(target) || navRef.current?.contains(target)`。

---

## 其余 7 条（均为 confirmed，逐条附读数）

### 2. 置顶后行不前移（README：pinned session moves to the front of its account）

```text
orderBy=updated：置顶最末行 → y 仍 378；再置顶中间行 → y 仍 276
orderBy=manual ：清空 pin → 置顶最末行 → y 仍 378；置顶中间行 → 同样不前移
```

`localStorage['dsh.workspace.view.v5'].orderBy` 切换后回读确认为 `manual`；宿主 `workspace/insertSessionBefore` 返回 200 但列表未重排。证据：`evidence/run1-28-manual-seq.json`、`evidence/run2-02-03-header-order.json`。

### 3. 置顶后归档会话，pin 不清理，面板留下幽灵条目

归档一个已置顶会话后侧栏不再显示它，但 footer 仍「已置顶的会话 (5)」；刷新重开面板仍列出该会话。两次独立复现（run1 与 run3）。README 承诺 `pruneStale` 默认 true 时「Drop pins and colors for entities absent from a ready list」。证据：`evidence/run1-80-81-anomalies.json`、`evidence/run1-82-83-health-reload.json`、`evidence/run3-02-03-filter-archive.json`。

### 4. `/goto` 在 0.2.0-rc.2 上完全不触发

```text
composerKind: { tag:"DIV", isTextarea:false, closestScroll:true }
输入 /goto 请假 + Enter → cur 不变、无 alert、输入框被清空
该会话标题最终变成「/goto 请假」，会话 zstd 日志无该文本事件
```

源码要求 `event.target instanceof HTMLTextAreaElement`（`lib/client.js:1181-1184`），本底座 composer 是 `div[contenteditable=true][data-composer-input]`，条件不成立直接 return。对照：对同一 contenteditable dispatch Enter，`defaultPrevented=true`（宿主自己处理，插件未介入）。证据：`evidence/run1-48-49-ws-goto.json`。

### 5. 工作区行没有置顶控件

`div[role=treeitem][aria-expanded]` 内 `querySelector('[class*=__dsh-session-pin-badge__]')` 为 null；hover 后只有宿主自带按钮。同页会话行控件正常。源码 `workspaceIdFor` 用行内 span 文本匹配 workspaces 快照 title（`lib/client.js:1302-1307`），本底座标题在 `span.YDXeBa_projectText` / `span.YDXeBa_title` 中，未命中。证据：`evidence/run1-48-49-ws-goto.json`、`evidence/run1-50-ws-health-detail.json`。

### 6. 点已置顶面板里的行不切换会话

```text
当前 A = session-08fbbad4…；点面板首行 B = session-3c8badf3…
after: dsh.sessions.current 仍 A；侧栏 aria-selected 仍 A；会话头仍是 A 标题
对照: 侧栏点 A → cur 立刻变 A，会话头出现 A 的标题
```

两次独立复现。证据：`evidence/run1-44-panel-nav.json`、`evidence/run1-39-panel-click.json`。

### 7. 健康摘要恒为空

5 条已置顶行的 `span.__dsh-session-pin-nav__health` 的 `innerText` 全为空（逐条切会话 4 次 + 折叠展开重算 + 刷新后重读均为空），而这些会话在会话区确有用户消息。源码 `summarizeHealth` 只数 `user/message` 与 `assistant/message`，走 `c.sessions.binding(id).session.getSnapshot().nodes`（`lib/client.js:206-223`、`2032-2041`）。证据：`evidence/run1-85-health-precise.json`。

### 8. 会话头置顶按钮不渲染

`button.__dsh-session-pin-header__` 两次独立上下文均为 null，虽然 `conversation.session.header.actions` 槽在 DOM 中存在。证据：`evidence/run1-44-panel-nav.json`、`evidence/run2-02-03-header-order.json`。

### 9. 附：每次置顶一条 404（有降级，属噪音）

`POST /api/session.setPinned` → 404，随后降级 `POST /api/settings/mutate` → 200（`ns=session-pin`、`path=["pinned"]`），状态最终正确；控制台每次置顶留一条 404。证据：`evidence/run1-40-rpc-trace.json`。

---

## 已验正常（支撑判断，逐项读数见 `RESULTS.json`）

- 发布件完整性：sha1 `f551e612…` = `dist.shasum`；sha512(base64) = `dist.integrity`；14 文件 / unpacked 504916；安装 exit 0，bundles `[dsh-base, dsh-web-app, dsh-session-pin]`。
- 会话行 `[pin][swatch]` 出现（34×16 容器、两个 16×16 按钮），hover 显形；空白未发消息的会话行不挂控件。
- 置顶态：`aria-pressed` true/false 与 `title`「置顶会话/取消置顶」同步，可多条并存。
- 取色：`data-color=c0→c1`，行背景 `rgba(249,115,22,0.1)→rgba(234,179,8,0.1)`，`box-shadow: rgb(234,179,8) 3px 0 0 0 inset`。
- Shift+点击清色：`data-color` 消失、背景与色条清零、aria-pressed 仍 true。
- 取消置顶回原位：5 条全置顶后逐条取消，行 key 序列与基线一致（`orderRestored=true`），y 242/276/310/344/378 复原。
- 持久化：状态写宿主设置（`cordis.patch.yml` 的 `- id: session-pin` 段），localStorage 无 pin 键；刷新保留；冷重启（origin 8677→7552）后置顶、颜色、看板、视图全部保留。
- 看板分组：面板内 ⋯ 编辑器选看板 + 标签保存后 `data-board`/`data-tags` 正确，组头变「▾ 复习 (1)」「▾ 未分组 (4)」。
- 过滤：输入「请假」→ 5 行剩 2 行；清空恢复；侧栏 6 条不受影响。
- 异常·多置顶：连点 5 条全部置顶，footer「已置顶的会话 (5)」，无报错（maxPins 默认 0）。
- 窄窗：1440/1100/900/760/640 均无横向溢出；900 以下宿主收侧栏为 55px 图标栏，面板与导航条自身仍完整可见。

## 未验 / 不确定 / 不做

**未验**

- `/goto` 的多命中列清单与无命中说明两个分支——被入口缺陷阻断，无法触发。
- 桌面端（Electron）与远端浏览器（README 说远端回落 localStorage）行为。

**不确定**

- 已存视图的回放：`+ view` 能写入视图（patch yml 可读），但点视图 chip 会先关面板，看不到筛选结果。
- 健康摘要的内容层（是否只含计数与方向）：没取到非空 UI 读数，只有源码依据。
- 置顶不前移的成因：`reorderMoves` 判据 vs 宿主未采用插入结果，未能区分。
- 面板行不切会话的成因：`c.sessions.retain(id,{source:'gateway'})` 在本基线为何不生效，未定位。

**不做**

- 不投稿、不上传、不建公开仓库、不改插件源码。
- 未使用真实模型密钥，会话消息一律 `MISSING_CREDENTIAL` 失败（环境限制，非插件问题）。

## 环境

DSH 0.2.0-rc.2 / Node v24.19.0 / pnpm 11.7.0 / Edge 155（Playwright 1.64.1 headless）/ Windows x64 / 隔离 `DSH_HOME`、`HOME`、`APPDATA`、`LOCALAPPDATA`、npm cache / 端口 `127.0.0.1` 随机分配（`--port 0`，本轮 8677、7552、5956）。未触碰用户日常 GUI（1241 / 2538 / 4756）。

## 证据

上传证据仓后把下列相对文件名替换为链接：

```text
evidence/run1-72-74-panel-close.json        主条目：导航条点击关面板（含阳性对照）
evidence/run1-72-nav-click.png
evidence/run1-73-chip-click.png
evidence/run1-74-manage-in-panel.png
evidence/run1-28-manual-seq.json            置顶不前移（manual 排序）
evidence/run2-02-03-header-order.json       置顶不前移 / 头部按钮缺失 / Shift 清色 / 404
evidence/run1-80-81-anomalies.json          多置顶 + 置顶后归档
evidence/run1-82-83-health-reload.json      归档后刷新仍在 + health 空
evidence/run3-02-03-filter-archive.json     归档异常独立复现
evidence/run1-48-49-ws-goto.json            /goto 不触发 + 工作区无控件
evidence/run1-44-panel-nav.json             面板行点击不切会话
evidence/run1-85-health-precise.json        health 恒为空
evidence/run1-40-rpc-trace.json             session.setPinned 404 → settings/mutate 降级
evidence/run1-10-pin-basic.json             置顶/取色/顺序基础读数
evidence/run2-01-cold-restart.json          冷重启后保留
evidence/run3-01-assign-board.json          看板归组成功（阳性对照）
evidence/run1-60-62-narrow-filter.json      窄窗五档 + 过滤
evidence/npm-view-0.7.19.json               发布件元数据
evidence/hashes.json                        包内文件 sha256
evidence/install-add.txt                    安装输出
evidence/lingshulab-index-scan.json         评测站全量索引扫描
```

## 查重

- GitHub `FuRongJun-1999/dsh-bounty/issues`：`dsh-session-pin`、`session-pin`、`session pin`、`PerryLink` 四个词各 12 条命中，全部是其他插件测评或记忆类安全报告偶含 `pin` 子串，同范围重复 0。
- `review.lingshulab.com`：`?q=` 服务端不过滤，改拉 `/api/reviews` 全量 172 条（9 页 × 20）本地扫描，`dsh-session-pin|session-pin|perrylink|置顶` 四个模式命中 0（`evidence/lingshulab-index-scan.json`）。
- 就本轮可见范围无重复，可按首报写。
