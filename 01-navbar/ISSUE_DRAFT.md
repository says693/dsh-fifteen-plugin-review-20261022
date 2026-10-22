{"project":"bench","target":"vlln/dsh-navbar","form":"intake"}

# dsh-navbar 0.4.0：在本机宿主上导航条完全不出现（取数锚点 `data-time-hover-root` 已不存在）；另有滚动高亮整段卡住、超长消息预览未截断、窄窗与正文重叠 9px

装好 `@vlln/dsh-navbar@0.4.0` 后新建/打开一条有多轮对话的会话，期望右侧出现节点导航条，实际什么都没有：页面里 `nav[data-dsh-navbar]` 元素存在但 `display:none`，节点数 0，控制台无任何报错。同一插件在同一个页面里已经正常渲染了 25 个「精选」按钮进消息动作栏、注入了 5466 字符样式表，说明加载没问题，是它唯一的数据来源取不到。

## 已验（confirmed）

### 1. 导航条在 dsh 0.2.0-rc.2 上不出现：`data-time-hover-root` 已不被产出

插件源码取数口径（`lib/client.js`）：

```js
const allRows = () => [...document.querySelectorAll("[data-time-hover-root]")].filter(row => !row.hasAttribute("data-pending-steering"));
```

README 也写明：`Zero data-channel dependencies: driven only by official anchor attributes (data-time-hover-root, on user rows since 0806)`。

实测读数（Edge 155 headless，1440×1000，三条不同会话）：

| 会话 | user 消息数 `[data-chat-flow-kind=user]` | `[data-time-hover-root]` | `button[data-vlln-dot]` | `nav` computedStyle.display |
|---|---|---|---|---|
| 45 轮（历史窗加载 25 条） | 25 | **0** | **0** | `none` |
| 8 轮 | 8 | **0** | **0** | `none` |
| 1 轮 | 1 | **0** | **0** | `none` |

本机 dsh 0.2.0-rc.2 的 chat 客户端不产出该属性：在 `node_modules/@deepseek-ai/dsh-client-ui-chat/lib/client.js` 与整份 dsh 安装目录检索字符串 `data-time-hover-root`，**0 命中**。当前 user 行实际携带的属性是：

```
class, data-chat-anchor-key, data-chat-flow-key, data-chat-paging-anchor,
data-chat-node-key, data-chat-flow-kind=user, data-chat-turn
```

气泡元素类名形如 `Sixlwa_bubble`（含 `bubble` 子串，这一半判据是满足的）。

**对照实验**：在同一页面把 `data-time-hover-root` 手动补到 `[data-chat-flow-kind=user]` 且含 bubble 子元素的行上（只改 DOM，不动插件源码），导航条立刻出现：`display:flex`、rect `x=1240 w=40 h=251`、`dots=11 + 2 个省略点`、标签 `user #17（点击跳转）…`。这证明插件逻辑本身可用，卡住它的只有锚点。

### 2. 滚动时当前节点高亮会整段卡住

补锚点后，从会话底部用真实滚轮向上滚，逐段采样（每段等 420ms）：

| 步骤 | scrollTop | active | 期望（视口内最靠上的 user 行） | 一致 |
|---|---|---|---|---|
| 1 | 4135 | 20 | 20 | ✓ |
| 2 | 3635 | 18 | 18 | ✓ |
| 3 | 3135 | 18 | 16 | ✗ |
| 4 | 2635 | 18 | 13 | ✗ |
| 5 | 2135 | 18 | 11 | ✗ |
| 6 | 1635 | 18 | 9 | ✗ |
| 7 | 1135 | 6 | 6 | ✓ |
| 8 | 635 | 4 | 4 | ✓ |
| 9 | 135 | 2 | 2 | ✓ |
| 10 | 0 | 1 | 1 | ✓ |

`scrollTop` 全程在正常变化（3135→2635→2135→1635），只有 active 不动。直接赋值 `scrollTop` 时同样复现：0.45 / 0.6 / 0.75 三档都停在 active=7（期望 11 / 14 / 17）。整个过程中 active 数量恒为 1（无多高亮）。**成因未定位**，只报现象：候选包括 IntersectionObserver 回调的 rAF 节流、`render()` 中 `builtRows` 早退分支、`updateActive()` 里 `next === activeIndex` 短路，但都没有单独验证。

### 3. 超长消息的悬停预览没有截断，撑成一整屏

单条 6720 字用户消息悬停时：

- `getComputedStyle(preview).webkitLineClamp` = `"6"`（CSS 声明在）
- `preview.getBoundingClientRect().height` = **7462px**
- `preview.scrollHeight` = `preview.clientHeight` = **7462**（`truncated = false`）

普通长度消息下预览正常（244 宽、6 行）。README 承诺「Hovering a node shows a message preview card (6-line truncation…)」。

### 4. 窄窗口（≤760）导航条与正文重叠 9px

| 视口宽 | nav x | 对话流 flow.right | 重叠 | 横向溢出 | nav 右边界越界 |
|---|---|---|---|---|---|
| 1440 | 1240 | 1228 | 0 | 否 | 否 |
| 1000 | 877 | 865 | 0 | 否 | 否 |
| 760 | 712 | 721 | **9px** | 否 | 否 |
| 600 | 552 | 561 | **9px** | 否 | 否 |

nav 宽 40px，其中 9px 落在正文区上；重复点击仍能命中，不阻断操作，但圆点会压住正文右边缘的文字。

## 能正常工作的部分（也是已验）

- 点击跳转 **6/6 命中**：用「悬停预览文本 vs 跳转后滚动容器顶部第一条 user 行文本」比对（不用索引）。预览「第 21 段」→ 跳转后顶部行也是「第 21 段」，与容器顶差值 0px；22/23/24/25/26 段同样 delta=0。
- 滑动窗口：45 条会话中段 11 点 + 2 省略，顶部 6 点 + 1 省略，底部 9 点 + 1 省略。
- 密度：相邻节点 top 差 17px，nav 高 251 未溢出视口，节点串无内部滚动条。
- 1 条消息时隐藏（`display:none`，控制台 0 error），与 README `rows.length < 2` 一致。
- 快速连点 5 个不同节点：无 pageerror、0 console error，最终收敛到第 5 次点击的目标。
- 滚轮切换：光标在条上滚一次，scrollTop 0→261、active 0→1。

## 复现

```sh
# 1. 隔离宿主（DSH_HOME/HOME/APPDATA/npm_config_cache 全指向任务目录）
node host.mjs nb2 navbar            # 输出 READY http://127.0.0.1:<port>

# 2. 造合成会话（官方 session-persistence，不调用模型）
DSH_HOME=<task>/isolated-home NAVBAR_CWD=<task>/fixtures/workspace node make-sessions.mjs

# 3. 打开会话并量 DOM
node probe-18-edges.mjs nb2         # 原生状态：dots=0 / nav display=none
node probe-19-jump-narrow.mjs nb2   # 补锚点后的跳转与窄窗
node probe-20-semantic.mjs nb2      # 语义命中
```

完整步骤见本仓 `REPRODUCE.md`。

## 测试环境

- 宿主：`@deepseek-ai/dsh` 0.2.0-rc.2（`lib/bin.js`），`--profile navbar --host 127.0.0.1 --port 0 --no-open`
- profile bundles：`@deepseek-ai/dsh-base` + `@deepseek-ai/dsh-web-app` + `@vlln/dsh-navbar@0.4.0`
- 浏览器：Edge 155.0.4283.62，Playwright 1.64.1 headless，独立 context，路由只放行 127.0.0.1
- Node v24.19.0；Windows
- 发布件校验：shasum `48eb0592ba6bb94e7c55813c2ff388ca661a1496`、integrity `sha512-XCkyXxRUbBaOIghvFjKdi3y0BHC9UADpO5H4+e0oQdcG87CUjPLbmPRsCRs8qDwviKd100oyjDcskgI6/wdN1g==`，与 npm 元数据一致；7 个文件；LICENSE MIT（Sam Gao / vlln）；无 postinstall

## 未验 / 不确定 / 不做

- **未验**：刷新后重建、冷重启后重建导航条。原因是原生环境下导航条本来就不出现，测「重建」没有意义。
- **未验**：pin（精选）状态的写入/读回往返；多会话切换时节点是否残留；深色主题；`prefers-reduced-motion`。
- **不确定**：第 2、3、4 条读数是在补打锚点后取得的。补锚点只补「取数入口」，插件其余逻辑原样跑，但严格说这些是「插件逻辑正确性」而非「开箱可用」。
- **不确定**：滚动高亮卡顿的成因未定位（只报现象）。
- **不确定**：README 说滚轮「blocking conversation-area scrolling」，实测对话区 scrollTop 从 0 变 261。但滚动的正是跳转所需位移，可能只是措辞与实现口径不同，不作为缺陷列。
- **不做**：不投稿、不改插件源码、不读真实会话/密钥。

## 建议（按性价比排序）

1. 换锚点或做双口径兜底：`[data-time-hover-root]` ∪ `[data-chat-flow-kind=user]`。这一步决定 0.2.0-rc.2 上能不能用。
2. 修滚动高亮的整段卡顿。
3. 修超长文本预览的 line-clamp 未生效（当前 `-webkit-line-clamp` 在 flex/固定高度缺失时可能失效，可补 `max-height` + `overflow:hidden` 兜底）。
4. 窄窗（<760）时把 nav 收进滚动容器的右内边距，避免 9px 压字。

## 证据（相对文件名，投稿前替换为公开链接）

- `evidence/nb2-18-edges.json` — 原生状态三条会话的 dots/nav display/timeHoverRoot 读数
- `evidence/nb2-10-rendered-dom.json` — 已渲染对话区的真实属性集合（证明没有 `data-time-hover-root`）
- `evidence/nb2-11-hover-anchor.json`、`evidence/nb2-12-hover-top.json` — 悬停 user 行后该属性仍为 0
- `evidence/nb2-17-wheel-up.json` — 滚动高亮卡顿 12 次采样
- `evidence/nb2-15-comprehensive.json` — 超长预览 truncated=false、窄窗、快速连点
- `evidence/nb2-19-jump-narrow.json` — 窄窗重叠 9px、跳转
- `evidence/nb2-20-semantic-jump.json` — 跳转语义命中 6/6
- `evidence/nb2-21-visual.json`、`evidence/nb2-21-nav-visible.png`、`evidence/nb2-21-hover-preview.png` — 补锚点后的正常态视觉
- `evidence/nbctl-09-variants.json` — 无插件 profile 对照
- `evidence/install-04-add-npm-final.txt` — 安装面 EXIT=0
- `evidence/dedup-reviews-all.json`、`evidence/dedup-plugins-all.json` — 两站查重全量数据
