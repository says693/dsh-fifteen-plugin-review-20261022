# dsh-conversation-minimap 1.3.4 技术报告

被测对象：`dsh-conversation-minimap@1.3.4`（npm 发布件，作者 Braidy-Wu，MIT）。
源码对照：GitHub 提交 `4b2ee4704de51b2409ff80bbba54445cdbc2ba67`，受测五个文件与发布件逐字节相同。
宿主：官方 `@deepseek-ai/dsh` 0.2.0-rc.2，web profile，`--host 127.0.0.1 --port 0 --no-open`。
浏览器：Edge + Playwright 1.64.1，headless，独立 context，只放行本机地址。

---

## 一、发现与影响

### 1（阻断）迷你地图在真实宿主上完全不出现，而且不报错

**现象。** 装好 1.3.4、打开一段有 8 轮提问的会话，左侧没有任何导航条。`document.querySelectorAll('.dsh-mm-seat').length === 0`，`.dsh-mm-anchor` 也是 0。五个合成会话（用户消息 0 / 1 / 3 / 8 / 25 条）都是 0；五种窗口与主题组合也是 0。控制台零错误、零警告，页面上也看不出任何异常。

宿主侧其实是活的：`window.__DSH_MINIMAP_CONFIG__` 正常注入为 `{enabled:true,minPrompts:4,anchorSize:3}`，`<style id="dsh-conversation-minimap-style">`（2297 字符）也在 DOM 里。也就是说宿主半、客户端 bundle 的 `apply()`、`MountManager.start()` 都跑过了，唯独界面没出来。

**根因。** `client.js` 的 `MountManager.currentSessionId()` 读的是：

```js
this.ctx.sessions.list.getSnapshot().current
```

而宿主 0.2.0-rc.2 里 `SessionListState` 的形状是：

```ts
{ ids: SessionId[], byId: Record<SessionId, SessionSummary>, phase: SessionListPhase, projectionsBySession: ... }
```

没有 `current` 字段。store 的初值也一样（`{ids: [], byId: {}, phase: 'pending', projectionsBySession: {}}`）。官方自己的 UI 取当前会话走的是 `byId` 加它自己的 binding source，不是 `snapshot.current`。

于是 `currentSessionId()` 永远返回 `null`，`check()` 每次都是「destroy 并直接 return」，`new MinimapController(...)` 从来不会执行，seat 也就永远不会被创建。整条链路被 try/catch 包着，所以失败是完全静默的。

**为什么作者的自测没发现。** 仓库里 `test/smoke.html` 给 `getSnapshot()` 塞的桩就是 `{ current: 's1' }` —— 恰好是真实宿主没有的那个字段。桩和真机不一致，自测自然全绿。

**差分验证。** 同一份逐字节相同的 `client.js`，喂两种桩：

| 桩 | applied | seats | anchors | userRows | loadOlder 调用 |
|---|---|---|---|---|---|
| `nocurrent`（严格照宿主形状） | true | 0 | 0 | 8 | 0 |
| `withcurrent`（多一个 `current:'s1'`） | true | 1 | 8 | 8 | 1 |

只补一个字段，功能从完全不可用变成完全可用。

**影响。** 对使用者来说，这个插件在 0.2.0-rc.2 上等于没装：没有报错、没有降级提示、没有任何线索。用户会以为是自己没装好。

**建议。** 会话 id 不要从 `list.getSnapshot().current` 取。可以改用 `uiSession` 暴露的当前 binding（`adapter.current` / `bindingSource`），或者从 `byId` 里按 `retainedBy.mainView > 0` 反查，再不行让用户在配置里指定 sessionId。同时在 `check()` 里加一条可见的降级日志（`console.warn` 一次即可），不要静默 return。自测桩要照真实 `SessionListState` 的形状写。

---

### 2（中）连续快速点击后，早期消息的高亮描边永久残留

`onJump` 每次会清掉上一根绿点（`dsh-mm-jumped`）和上一个 `highlightTimer`，但**上一条消息身上的 `dsh-mm-jump-highlight` 描边只由它自己那 2 秒的定时器移除**。第二次点击把这个定时器 `clearTimeout` 掉，第一条消息的描边就再也没人清。

实测：300 ms 间隔连点第 2、4、6 个块，等 3.5 秒（2.0 s 高亮 + 0.3 s 淡出 + 余量）后，`u2` 与 `u4` 两行仍然带 `.dsh-mm-jump-highlight`，永不消失。同一夹具单次点击则正常清除。

影响：翻长对话时多点几下，屏幕上会留下一串蓝色描边，看起来像"选中了多个地方"。

建议：跳转前先把 `document.querySelectorAll('.dsh-mm-jump-highlight')` 全部清掉，或用一个数组记住所有待清理的行。

---

### 3（轻）深色主题下灰色块对比度只有 2.08:1

块的底色写死 `rgba(128,128,128,0.5)`，没有走主题变量。深色页面 `rgb(21,21,23)` 上合成后是 `rgb(75,75,76)`，对比度 **2.08:1**；浅色页面白底上合成约 `rgb(192,192,192)`，约 1.8:1。WCAG 对非文字 UI 的建议是 3:1。当前蓝色活动块用了 `var(--dsw-static-deepseek-500)`，在两种主题下都清楚。

影响：深夜模式下列导航条几乎看不见，只剩一个蓝点。

建议：块底色改走主题变量（如 `--dsw-alias-label-caption` 或 `--dsw-alias-border-l2`），深色主题下提高不透明度。

---

### 4（观察，非缺陷）块的长短不反映消息内容

README 写的是 "one evenly-spaced horizontal capsule per user prompt"，源码常量也是 `BASE_W=12`、`ANCHOR_H=3`、`ANCHOR_GAP=12`。实测八个块全部 **12×3 px**、间距 15 px，与提示长度（2 / 42 / 202 / 802 / 12 / 402 / 102 / 52 字，对应行高 46 / 46 / 86 / 246 / 46 / 146 / 66 / 46 px）**完全无关**；user 与 steering 两种角色样式也相同，assistant 行不参与。

也就是说：这条导航条是"第几轮"的索引，不是"对话内容的缩略图"。对找位置够用，但不要期待它一眼看出哪轮问得长。唯一随内容变化的是 hover 时的鱼眼宽度（实测峰值 44 px）。

如果作者想让"缩略图"名副其实，可以按每轮字符数给块宽做一个有限的映射（比如 12–28 px），或者按比例把块间距拉开。

---

### 5（轻）窄窗口 760 px 及以下，导航条静默消失

`@media (max-width:767px)` 直接 `display:none!important`。实测 760 px 与 600 px 下，8 个块仍在 DOM 里但可见盒子是 0×0，屏幕上什么都没有，也没有任何提示。

1000 px 下正常：seat 盒 x=10、宽 72，消息正文从 x=96 开始，重叠行数 0，不挡字。

影响：把窗口拖窄一点，功能就没了，用户不知道是被隐藏还是坏了。建议至少在隐藏时留一个可点的小把手，或在发布说明里写清断点。

---

### 6（观察）0 条消息时会留下一个空的 seat 元素

空会话下 `seat` 依然 `display:block`，只是没有子元素。它宽 72 px、定位在左侧，因为没内容所以看不见，但元素确实在。属于整洁度问题，不影响使用。

---

## 二、验证过的正常行为

以下读数在受控夹具里测得（真实宿主因第 1 条不渲染，见"限制"）。

- **块数与提问数严格一一对应**：8 轮→8 块，48 轮→48 块，1 轮（minPrompts=0）→1 块。每个块的 `data-mm-key` 与对应行的 `data-chat-anchor-key` 一致。
- **悬停鱼眼与预览**：指针停在第 3 块，宽度序列 `[20,35,44,35,20,13,12,12]`，峰值正好 44 px，1 个 `dsh-mm-hot`、5 个 `dsh-mm-enlarged`，预览面板 380×160 显示 "Prompt + 全文"。
- **点击跳转**：3/3 正确。第 7 块 `scrollTop 0→557`，目标行中心与视口中心偏差 **0 px**；第 4 块偏差 0 px；跳第 1 条时 `scrollTop 557→0`，中心偏差 -461 px（已到滚动上限，行被正确高亮）。三次都出现了 2 秒描边和绿点。
- **滚动跟随**：滚动到 100% / 75% / 50% / 25% / 0% 时活动块依次是 `u7/u6/u5/u4/u3`，单调不跳号；所有采样点活动块都在 rail 范围内。
- **长会话性能**：48 块挂载 2026 ms；连续滚动 90 帧，中位 **5.6 ms**、p95 **5.7 ms**、最大 5.8 ms，**超过 50 ms 的帧 0 个**。列高溢出 824 px 的 rail 时 `translateY(58px)` 加遮罩淡出，块不越界。
- **超长消息**：8000 字提问夹具，4 块，中位 5.6 ms / p95 5.8 ms / 最大 5.9 ms / 超 50 ms 帧 0。
- **开关与阈值**：`enabled:false` → 0 seat 0 块；`minPrompts=4` 下 1 轮与 3 轮均 `display:none` 且 0 块；`minPrompts=0` 下 1 轮出 1 块。
- **配置生效**：`anchorSize:8` → 块高 8 px、间距仍 12 px。
- **快速滚动**：48 块夹具上 30 次 `scrollTop` 跳变，0 页面错误、0 控制台告警，块结构完好。
- **减少动效**：`prefers-reduced-motion: reduce` 下仍渲染 8 块，过渡被关闭，跳转改为 `behavior:'auto'`。
- **不遮挡正文**：1440 / 1000 px 下 seat 盒与消息行的重叠数均为 0。

安装面：`dsh plugin --profile <p> add dsh-conversation-minimap@1.3.4` 返回 0，pnpm 装 1 个包；bundle 出现在页面的模块清单里（`/plugins/??…,dsh-conversation-minimap/client.js`）。发布件校验：tarball sha1 `17a4c86c8479b02e6bc3077a53c5c225bdbb2da9` 与 registry shasum 一致，sha512 `bZhe5D2IblCPM0/N8iGqZNyvEbmgOY+r3ky2HQKj3faj2LhkTD1EG7Yea8weFNfo61baE61muNXLcEag4TtDvQ==` 与 registry integrity 一致，8 个文件 290513 字节，LICENSE 为 MIT（Copyright (c) 2026 Braidy-Wu）。

---

## 三、复现步骤

见 `REPRODUCE.md`。最短路径：

1. 建一个从 web 模板派生的隔离 profile，装 `dsh-conversation-minimap@1.3.4`。
2. 用官方 jsonl 持久化后端造一个有 ≥4 轮提问的合成会话。
3. 起宿主 `--host 127.0.0.1 --port 0 --no-open`，用 Playwright 打开会话。
4. 读 `document.querySelectorAll('.dsh-mm-seat').length` → 0；读 `window.__DSH_MINIMAP_CONFIG__` → 已注入；读控制台 → 空。
5. 想看"本该长什么样"，跑 `p12-stub.mjs`：同一份 client.js 换个桩，seats 从 0 变 1、anchors 从 0 变 8。

---

## 四、环境与限制

- 平台 Windows 10/11 + Edge，Node v24.19.0。**macOS / Linux 未测**。
- 真实宿主上因为第 1 条，导航条从不渲染。所以第 2 条之后的所有 UI 读数（块数、鱼眼、跳转、滚动跟随、性能、断点、深色对比度、anchorSize、连点残留）都是**在受控夹具里**用逐字节相同的 `client.js` + 一个照宿主 `SessionListState` 形状写、额外补了 `current` 的桩测出来的。夹具用 127.0.0.1 随机端口的静态服务，独立 Edge context。
- **这不是"插件在真机上表现良好"。** 真机上它不出现；夹具只能说明"如果它挂上了，逻辑本身大体是对的"，并暴露挂上以后才会遇到的第 2、3、5 条问题。
- 未测：`steering` 类型消息是否也生成块（需要真实模型轮次里的中途插话，涉及付费模型调用，不做）；真实宿主上跨 120 页上限的历史同步（挂载被第 1 条挡住，跑不到）。
- 刷新与冷重启：真机上刷新前后、宿主整进程重启（新端口 3938）前后都是 0 seat。能确认的是"宿主半与 bundle 在重启后依然正常加载"，不能确认"一个能挂载的版本刷新后会不会正确重建"——这一条按 `uncertain` 记录。
- 未修改插件源码，未新建兜底实现替代主流程；第 3 节第 6 条所有读数都来自原样发布的 `client.js`。
- 未读任何真实会话、token、cookie；未触碰用户正在使用的 GUI（1241 / 2538 / 4756）。

---

## 五、查重

| 站点 | 方法 | 扫描量 | 命中 |
|---|---|---|---|
| review.lingshulab.com | `/api/plugins` 与 `/api/reviews` 全量分页（`?q=` 服务端不过滤，必须全量） | 133 个插件 / 172 篇评测 | 0 |
| github.com/FuRongJun-1999/dsh-bounty | GitHub Search API，`repo:` 限定 + 关键词 | `minimap` 0、`Braidy-Wu` 0、`dsh-conversation-minimap` 0、`迷你地图` 1（#646 是 dsh-roleplay，非同范围） | 0 |

两站均无同插件同范围重复，可作首报。
