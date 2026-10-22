# [dsh-conversation-minimap 1.3.4] 装上后迷你地图在 DSH 0.2.0-rc.2 上完全不出现，也不报错：插件读 `sessions.list.getSnapshot().current`，而宿主没有这个字段

marker: `intake:v1` · plugin: `dsh-conversation-minimap` · version: `1.3.4` · author: `Braidy-Wu` · host: `@deepseek-ai/dsh 0.2.0-rc.2` · first report（两站查重见文末）

## 一句话

装了 1.3.4、打开一段 8 轮的会话，左侧导航条一个像素都没有；`.dsh-mm-seat` 与 `.dsh-mm-anchor` 计数都是 0，控制台零错误零警告。原因是插件取当前会话 id 的位置在宿主里根本不存在，于是挂载链路静默退出。

## 操作与现象

1. 隔离 profile（从官方 web 模板派生）里装 `dsh-conversation-minimap@1.3.4`。
2. 用官方 `dsh-session-persistence-jsonl` 造 5 个合成会话：0 / 1 / 3 / 8 / 48 轮提问。
3. 起宿主 `--host 127.0.0.1 --port 0 --no-open`，用 Edge（Playwright headless）打开会话。

结果：五个会话全部 0 块、0 seat。宿主半边其实是活的 —— `window.__DSH_MINIMAP_CONFIG__` 正常注入 `{enabled:true,minPrompts:4,anchorSize:3}`，`<style id="dsh-conversation-minimap-style">`（2297 字符）也在 DOM 里，说明客户端 bundle 的 `apply()` 与 `MountManager.start()` 都执行过。1440/1000/760/600 四种宽度与深色主题共 5 组，也全是 0。

## 根因（源码对读）

`client.js` 的 `MountManager.currentSessionId()`：

```js
var snap = this.ctx.sessions.list.getSnapshot()
return snap && snap.current ? snap.current : null
```

宿主 0.2.0-rc.2 的 `SessionListState` 是：

```ts
// dsh-api-session-controller/lib/types/client/sessions/service.d.ts
interface SessionListState {
  ids: SessionId[]
  byId: Record<SessionId, SessionSummary>
  phase: SessionListPhase
  projectionsBySession: ...
}
```

没有 `current`。store 初值同样是 `{ids: [], byId: {}, phase: 'pending', projectionsBySession: {}}`（`service.js:166`）。官方自己的 UI 取当前会话用的是 `byId` + 自己的 binding source（`dsh-client-ui-session/lib/client.js:281-283`），不是 `snapshot.current`。

于是 `currentSessionId()` 恒为 `null` → `check()` 走 `destroy()` 后直接 `return`，`new MinimapController(...)` 永远不执行 → seat 永不创建。整条链路被 try/catch 包住，所以失败完全静默。

## 差分验证（同一份 client.js，只换桩）

把 npm tarball 里的 `client.js`（sha256 `451f4ff4…`，与 GitHub `4b2ee470` 逐字节相同）原样服务的两页：

| 桩的 `getSnapshot()` 形状 | applied | `.dsh-mm-seat` | `.dsh-mm-anchor` | 用户行 | `loadOlder` |
|---|---|---|---|---|---|
| `{ids, byId, phase, projectionsBySession}`（照宿主） | true | 0 | 0 | 8 | 0 |
| 同上 + `current:'s1'` | true | 1 | 8 | 8 | 1 |

只多一个字段，从完全不可用变成完全可用。`withcurrent` 下首块 `aria-label` 为「跳转到第 1 轮：…」，块数 8 = 用户行 8。

## 为什么作者自测没发现

仓库 `test/smoke.html` 的桩恰好是 `getSnapshot: function(){ return { current: 's1' } }` —— 正是真实宿主没有的那个字段。桩与真机不一致，自测自然全绿。

## 复现

```sh
# 1) 建 web 派生 profile 并装插件（直接 plugin add 建出的 profile 缺 dsh-web-app，不会打印 URL）
node <dsh>/lib/bin.js --profile mmweb --from-default-profile web --dump-config
node <dsh>/lib/bin.js plugin --profile mmweb add dsh-conversation-minimap@1.3.4

# 2) 造合成会话（官方 jsonl 后端，zstd，不调模型）
node make-sessions.mjs <DSH_HOME>/sessions <fixtures/workspace>

# 3) 起宿主
node <dsh>/lib/bin.js --profile mmweb --host 127.0.0.1 --port 0 --no-open

# 4) 浏览器打开会话后读数
document.querySelectorAll('.dsh-mm-seat').length     // 0
document.querySelectorAll('.dsh-mm-anchor').length   // 0
document.querySelectorAll('[data-chat-flow-kind=user]').length  // 8
window.__DSH_MINIMAP_CONFIG__                        // {enabled:true,minPrompts:4,anchorSize:3}
# 控制台：无 error 无 warning

# 5) 差分对照（同一份 client.js，只换桩）
node p12-stub.mjs
# nocurrent  -> seats 0, anchors 0
# withcurrent-> seats 1, anchors 8
```

## 建议

1. 不要从 `list.getSnapshot().current` 取会话 id。改用 `uiSession` 的当前 binding（`adapter.current` / `bindingSource`），或从 `byId` 按 `retainedBy.mainView > 0` 反查，或允许配置里显式指定 sessionId。
2. `check()` 里 `sessionId === null` 的分支加一次性 `console.warn`（或页内降级提示），不要静默 return —— 现在用户完全无法判断是没装好还是插件坏了。
3. 自测桩照真实 `SessionListState` 形状写，别手搓 `current`。

## 附带确认的两条（同一插件、同一轮实测）

### A. 快速连点后，早期消息的高亮描边永久残留

300 ms 间隔连点第 2/4/6 块，等 3.5 秒后 `u2`、`u4` 两行仍带 `.dsh-mm-jump-highlight`，永不消失；单次点击则正常清除。

原因：`onJump` 会清掉上一根绿点和上一个 `highlightTimer`，但上一条**消息行**的描边只由它自己那 2 秒定时器移除；第二次点击把定时器 `clearTimeout` 掉，描边就没人清了。

建议：跳转前先 `document.querySelectorAll('.dsh-mm-jump-highlight').forEach(el=>el.classList.remove('dsh-mm-jump-highlight'))`，或用一个集合记住所有待清理的行。

复现读数（`evidence/p15-followups.json`）：

```text
三次连点后 3.5s：staleHighlights = ['u2','u4']（count 2）
单次点击后 2.6s：staleHighlights = []（count 0）
```

### B. 深色主题下灰块对比度 2.08:1

块底色写死 `rgba(128,128,128,0.5)`，不走主题变量。深色页面 `rgb(21,21,23)` 上合成 `rgb(75,75,76)`，对比度 **2.08:1**（WCAG 非文字 UI 建议 3:1）。建议改走 `--dsw-alias-*` 系列变量。

## 不是缺陷，但值得说明的两点

- **块的长短不反映消息长度**：8 个块实测全是 12×3 px、间距 15 px，与提示长度（2/42/202/802/12/402/102/52 字，行高 46/46/86/246/46/146/66/46 px）无关。这与 README 的 "one evenly-spaced horizontal capsule per user prompt" 一致，属于设计取舍 —— 但叫 "缩略图" 容易让人以为能看出长短。若想名副其实，可按字符数做有限映射（12–28 px）。
- **760 px 及以下静默消失**：`@media (max-width:767px)` 直接 `display:none!important`，760/600 px 下 8 个块仍在 DOM 但可见盒子 0×0，无任何提示。

## 已验 / 未验 / 不确定 / 不做

**已验（真机）**

- 5 个合成会话（0/1/3/8/48 轮）全部 0 块 0 seat，配置与样式已注入，控制台零错误。
- 5 组窗口/主题组合（1440 light、1440 dark、1000、760、600）全部 0 块。
- 刷新前后、宿主整进程冷重启（新端口 3938）前后，均为 0 块；宿主半与 bundle 重启后仍正常加载。
- npm 发布件校验：shasum `17a4c86c8479b02e6bc3077a53c5c225bdbb2da9`、integrity `sha512-bZhe5D2I…`、8 文件 290513 字节、MIT；与 GitHub `4b2ee470` 五个受测文件逐字节相同。

**已验（受控夹具，用逐字节相同的 client.js）**

- 块数与提问数一一对应：8→8、48→48、1→1（minPrompts=0）。
- 鱼眼与预览：宽度序列 `[20,35,44,35,20,13,12,12]`，峰值 44 px，预览 380×160。
- 点击跳转 3/3 正确，最准的一次行中心与视口中心偏差 0 px。
- 滚动跟随：100/75/50/25/0% → `u7/u6/u5/u4/u3`，活动块始终在 rail 内。
- 48 块挂载 2026 ms；90 帧滚动中位 5.6 ms、p95 5.7 ms、超 50 ms 帧 0。
- 8000 字提问：4 块，中位 5.6 ms，超 50 ms 帧 0。
- `enabled:false` → 0 seat；1/3 轮在 minPrompts=4 下 `display:none`；`anchorSize:8` → 块高 8 px。
- 快速滚动 30 次跳变：0 错误 0 告警。
- 1440/1000 px 下 seat 与消息行重叠数 0（不挡字）。

**未验**

- `steering` 类型消息是否也生成块：需要真实模型轮次里的中途插话，涉及付费模型调用，未做。

**不确定**

- 一个「能挂载」的版本在刷新/冷重启后是否会自动重建：真机上挂载本身被阻断，观察不到目标行为。
- 真实宿主上 `loadOlder` 跨 120 页上限的历史同步：同样被阻断。

**不做**

- 不改插件源码、不建兜底实现、不投稿、不上传、不建公开仓库。
- 不读真实会话 / token / cookie，不碰用户正在用的 GUI（1241 / 2538 / 4756）。
- macOS / Linux 未测（本机 Windows + Edge）。
- 桌面端（Electron）未测，只测 web profile。

## 测试环境

DSH `@deepseek-ai/dsh` 0.2.0-rc.2（`lib/bin.js`）/ Node v24.19.0 / pnpm 11.7.0 / Edge + Playwright 1.64.1 headless / Windows x64。
隔离 `DSH_HOME`、`HOME`、`USERPROFILE`、`APPDATA`、`LOCALAPPDATA`、`npm_config_cache` 全部在任务目录内；宿主端口 0（本次观察到 4243，冷重启后 3938）。
合成会话由官方 `@deepseek-ai/dsh-session-persistence-jsonl`（zstd）写盘，事件序 `turn/start → step/start → user/message(surfaceOp:append) → assistant/message → step/end → turn/end`，不调模型、不读真实会话。
安装面记录：`dsh plugin --profile <p> add` 首次建出的 profile 缺 `@deepseek-ai/dsh-web-app`，boot 后不打印 URL（stdout 0 字节）；须先 `--profile <p> --from-default-profile web --dump-config` 派生 web profile 再 add。这是宿主 CLI 的建 profile 行为，不计为插件缺陷。

## 证据（投稿时替换为链接）

```text
evidence/p06-rail-report.json              真机 5 个会话的 seat/anchor/userRow 读数
evidence/p06-mixed.png                     真机 8 轮会话截图（左侧无导航条）
evidence/p13-realhost-matrix.json          真机 5 组窗口/主题矩阵
evidence/p13-w1440-light.png               1440 浅色
evidence/p13-w1440-dark.png                1440 深色
evidence/p13-w760-light.png                760 宽
evidence/p04-diag.json                     config/style 已注入的证据 + 控制台为空
evidence/p08-ctx-scan.json                 页面内找不到宿主快照
evidence/p09-ctx-capture.json              包 __ModuleLoader__ 后 apply 未触达的尝试
evidence/p10-baseline-result.json          真机 baseline：applyCalls 0
evidence/p11-loader-ids.json               模块加载器实际收到的 id
evidence/p12-stub-result.json              差分：nocurrent 0 块 / withcurrent 8 块
evidence/p12-stub-nocurrent.png            差分对照截图
evidence/p12-stub-withcurrent.png          差分对照截图（导航条出现）
evidence/p14-behaviour.json                受控夹具全量读数（几何/跳转/滚动/性能/断点）
evidence/p14-mixed-hover.png               鱼眼 + 预览
evidence/p14-after-jump.png                点击跳转后高亮
evidence/p14-long48.png                    48 块长会话（遮罩淡出）
evidence/p14-huge.png                      8000 字提问
evidence/p14-w760.png / p14-w600.png       窄窗静默隐藏
evidence/p15-followups.json                深色对比度 2.08 / anchorSize / 连点残留
evidence/p15-rapid.png                     连点残留描边
evidence/p15-dark.png                      深色主题
evidence/p16-reload.json                   页面刷新前后
evidence/p17-reload.json                   冷重启（新端口 3938）前后
evidence/install-01-add-npm.txt            首次 plugin add（缺 web-app 的对照）
evidence/install-03-webprofile-add.txt     web 派生 profile 安装成功
evidence/provenance-01-hashes.txt          npm vs GitHub 提交逐文件 sha256
evidence/dedupe-01-lingshu.txt             评测站全量查重
evidence/dedupe-02-github-issues.txt       悬赏仓查重
```

## 查重

- `review.lingshulab.com`：`?q=` 服务端不过滤，改为全量分页。`/api/plugins` total=133（50/50/33 三页取尽），按 displayName / repositoryFullName / description 匹配 `minimap|Braidy-Wu`，命中 **0**；`/api/reviews` total=172（50/50/50/22 四页取尽），按 title / pluginId / bodyMarkdown 匹配，命中 **0**。查询时间 `2026-10-22T01:16:28Z` / `01:16:36Z`。
- `github.com/FuRongJun-1999/dsh-bounty`：GitHub Search API，`repo:` 限定。`minimap` 0、`Braidy-Wu` 0、`dsh-conversation-minimap` 0、`迷你地图` 1（#646 是 dsh-roleplay 的测评，正文出现「迷你」但与本体无关，非同范围重复）。查询时间 `2026-10-22T01:16:43Z`。
- 结论：两站均无同插件同范围重复，按首报处理。

## 给主 agent 的说明

- 可投稿条目：**主条（C1/C2/C3）** + **A（连点残留）** + **B（深色对比度）**，三条都是 confirmed。
- 观察项（块不反映长度、760 px 静默消失、空会话留空 seat）建议作为附注，不单独发 issue。
- 不确定项（刷新/冷重启重建、真实历史同步）不要写进投稿正文当作结论。
- 本文件只是本地草稿，未投稿、未上传、未建仓。
