# dsh-navbar 0.4.0 真实功能快测报告

对象：`@vlln/dsh-navbar@0.4.0`（npm 发布件），仓库 `vlln/dsh-navbar` @ `e53adfc8`（仅作源码对照）。宿主 `@deepseek-ai/dsh` 0.2.0-rc.2，Edge 155 + Playwright 1.64.1 headless，隔离 `DSH_HOME`，端口 0。

## 一、发现与影响（先看这里）

### 1. 在本机宿主上，导航条完全不会出现（最重要）

在 1440×1000 下打开一条有 25 条用户消息的会话，导航条 `nav[data-dsh-navbar]` 的 computedStyle 是 `display: none`，节点数 0。

原因不是插件没装好——同一个插件在本会话里正常渲染了 25 个「精选」按钮进消息动作栏、注入了 5466 字符的 style、挂好了预览卡元素。问题在于它唯一的数据锚点不存在：

- 插件源码的取数口径是 `document.querySelectorAll('[data-time-hover-root]')`，README 写的是「driven only by official anchor attributes (`data-time-hover-root`, on user rows since 0806)」。
- 本机 dsh 0.2.0-rc.2 的 chat 客户端**完全不产出这个属性**。在 `dsh-client-ui-chat/lib/client.js` 与整份 dsh 安装目录里检索 `data-time-hover-root`，0 命中。
- 实际 user 行携带的是 `data-chat-flow-kind="user"` + `data-chat-anchor-key` + `data-chat-turn`，气泡类名 `Sixlwa_bubble`。

读数：3 条不同会话（25 条 / 8 条 / 1 条用户消息），`[data-chat-flow-kind=user]` 分别 = 25 / 8 / 1，而 `[data-time-hover-root]` 恒 = 0，`dots` 恒 = 0，`nav display` 恒 = `none`。把锚点补到 user 行上（只改 DOM）后，导航条立刻按预期出现。

影响：对 0.2.0-rc.2 用户来说，装上这个插件等于没装——功能全体静默失效，且不会报错、没有任何提示。这是 0 成本可发现、0 成本可修的一条（把取数口径换成 `[data-chat-flow-kind=user]`，或做双口径兜底）。

### 2. 滚动时当前节点高亮会整段卡住

补上锚点后，从会话底部向上滚，逐段采样 active 节点：

| 步骤 | scrollTop | active | 期望 | 一致 |
|---|---|---|---|---|
| 1 | 4135 | 20 | 20 | ✓ |
| 2 | 3635 | 18 | 18 | ✓ |
| 3 | 3135 | 18 | 16 | ✗ |
| 4 | 2635 | 18 | 13 | ✗ |
| 5 | 2135 | 18 | 11 | ✗ |
| 6 | 1635 | 18 | 9 | ✗ |
| 7 | 1135 | 6 | 6 | ✓ |
| 8 | 635 | 4 | 4 | ✓ |

同一现象在直接赋值 scrollTop 时也复现（0.45 / 0.6 / 0.75 三档都停在 active=7，期望 11 / 14 / 17）。卡住期间 scrollTop 一直在正常变化。active 数量始终为 1，不存在多高亮。成因未定位（IntersectionObserver 回调节流、`render()` 里 `builtRows` 的早退分支、`updateActive` 的 `next === activeIndex` 短路都只是候选），因此只写现象不写成因。

### 3. 超长消息会把悬停预览卡撑成一整屏

单条 6720 字的用户消息，悬停预览时：

- `getComputedStyle(preview).webkitLineClamp` = `"6"`（CSS 声明生效了）
- 但实测 `preview` 高度 7462px，`scrollHeight === clientHeight === 7462`，即 `truncated = false`，截断没发生

预览卡本该是 6 行的小卡片，实际铺满整屏。普通长度消息下预览正常（244 宽、6 行），所以是长文本分支的问题。

### 4. 窄窗口下导航条压到正文上

| 视口宽 | nav x | 对话流 flow.right | 重叠 |
|---|---|---|---|
| 1440 | 1240 | 1228 | 0 |
| 1000 | 877 | 865 | 0 |
| 760 | 712 | 721 | **9px** |
| 600 | 552 | 561 | **9px** |

三档都没有横向溢出、nav 右边界也都在视口内。760 及以下开始与正文重叠 9px（nav 宽 40px，其中 9px 落在正文区域上）。重复点击仍能命中，可点性没坏，但节点圆点会盖住正文右边缘的字。

## 二、能正常工作的部分

补上锚点后，插件的主体逻辑是扎实的：

- **位置**：1440 宽下 nav x=1240，对话流 flow.right=1228，差 12px，与源码 `flow.right + 12` 一致；不压正文。
- **点击跳转 6/6 命中**：用「悬停预览文本 vs 跳转后滚动容器顶部第一条 user 行文本」比对，不用索引。预览「第 21 段」→ 跳转后顶部行也是「第 21 段」，与容器顶差值 0px；22/23/24/25/26 段同样 delta=0。scrollTop 逐次 46→261→476→691→906→1121 单调递增。
- **滑动窗口**：45 条会话中段 dots=11 + 2 个省略点，顶部段 6 点 + 1 省略，底部段 9 点 + 1 省略；窗口首尾编号随滚动变化（7—17 → 15—25 → 17—25）。
- **密度**：相邻节点 top 差 17px（= CSS gap 10 + 节点高 7），nav 整体高 251，未溢出视口，节点串内部无滚动条。
- **边界**：1 条消息时 nav `display:none`（与 README 的 `rows.length < 2` 一致），控制台 0 error；快速连点 5 个不同节点无异常，最终收敛到第 5 次点击的目标。
- **滚轮切换**：光标在条上滚一次，scrollTop 0→261、active 0→1，切换一条。

## 三、复现步骤

见 `REPRODUCE.md`。关键三步：起隔离宿主（`node host.mjs <tag> navbar`）→ 用官方 session-persistence 造合成会话 → Playwright 打开并量 DOM。

## 四、环境与限制

- 宿主 `@deepseek-ai/dsh` 0.2.0-rc.2；Edge 155.0.4283.62；Playwright 1.64.1；Node v24.19.0。
- 会话全部是**合成会话**（官方 `dsh-session-persistence-jsonl` 写入，不调用模型、不读真实会话）。
- **第 1、2、4 条的读数是在补打 `data-time-hover-root` 后取得的**。补锚点只补了「取数入口」，插件其余逻辑原样跑；但正因为原生环境没有这个属性，这些读数应理解为「插件逻辑正确性」，而不是「用户开箱即可用」。第 1 条本身读的就是原生状态。
- 造合成会话踩了三处坑（envelope 要 `{seq,time}`、turn/step 从 1 起、`turn/end` 必须带 `reason.kind`），第三处会让宿主对话页整页空白（`Cannot read properties of undefined (reading 'kind')`），已用无插件 profile 做对照确认是夹具问题。
- 未测：刷新后重建、冷重启后重建。原因同上——原生本来就不出现，测「重建」没有意义。
- 本轮没测：多会话切换时节点是否残留、pin（精选）状态的写入/读回往返、深色主题、`prefers-reduced-motion`。

## 五、给维护者的建议（按性价比排序）

1. 换锚点或做双口径兜底（`[data-time-hover-root]` ∪ `[data-chat-flow-kind=user]`）——这一步决定 0.2.0-rc.2 上能不能用。
2. 修滚动高亮的整段卡顿。
3. 修超长消息预览的 line-clamp 未生效。
4. 窄窗（<760）时把 nav 的定位改成「贴在滚动容器右内边距内」，避免 9px 压字。

## 六、证据索引

`evidence/` 下按探针编号组织，`.json` 是读数、`.png` 是截图：

- `nb2-18-edges.json`、`nb2-10-rendered-dom.json`、`nb2-11-hover-anchor.json`、`nb2-12-hover-top.json` — 第 1 条（原生不出现 + 锚点缺失）
- `nb2-17-wheel-up.json`、`nb2-15-comprehensive.json` — 第 2 条（高亮卡顿）
- `nb2-15-ultra-hover.png`、`nb2-15-comprehensive.json` — 第 3 条（预览撑满）
- `nb2-19-jump-narrow.json`、`nb2-19-narrow-760.png`、`nb2-19-narrow-600.png` — 第 4 条（窄窗重叠）
- `nb2-20-semantic-jump.json` — 跳转语义命中
- `nb2-21-visual.json`、`nb2-21-nav-visible.png`、`nb2-21-hover-preview.png` — 位置/密度/预览正常态
- `nb2-14-anchor-inject2.json` — 滑动窗口、滚轮切换
- `nbctl-09-variants.json` — 无插件对照
- `dedup-reviews-all.json`、`dedup-plugins-all.json` — 两站查重全量数据
