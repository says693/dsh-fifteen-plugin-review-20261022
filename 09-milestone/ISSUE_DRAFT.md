{"project":"bench","target":"SnowCrescenter-tech/dsh-milestone","form":"intake"}

# dsh-milestone 0.7.2 在 DSH 0.2.0-rc.2 上圆点时间轴整条不渲染：`useSession(s => s.queue)` 取 `.length` 抛错，被自带兼容边界收成"已停用"提示

装了插件、打开一个有 3 条用户消息的会话，右侧圆点时间轴**一个圆点都没有**，右下角出现"dsh-milestone 与当前 DSH 版本不兼容，已停用。请升级插件。"。控制台第一条错误是 `TypeError: Cannot read properties of undefined (reading 'length')`，栈顶 `MilestoneRail`。

## 复现（最小步骤）

1. 用 `--from-default-profile web` 派生隔离 profile（直接 `add` 建出来的 profile 缺 `dsh-web-app`，宿主静默起不来）。
2. profile 依赖里加 `dsh-milestone@0.7.2`，并把 `@deepseek-ai/dsh-session` / `dsh-session-projection` / `dsh-llm` 等核心包钉回宿主版本 `0.2.0-rc.2`（不钉会先撞下面"安装期问题"，看不到本条）。
3. `DSH_HOME` 隔离，`--host 127.0.0.1 --port 0 --no-open` 启动；Edge + Playwright headless 打开，点开任意含用户消息的会话。
4. 读数：`[data-rail-dot]` 计数 0；`[data-chat-flow-kind="user"]` 行数 3；`[data-chat-anchor-key]` 行数 12；`[data-rail-list]` 不存在；不兼容提示条 `x=1100,y=892,326×94`。

对照：同一会话在**不装插件**的 profile 里同样是 3 条用户消息行、12 个锚点行、`scrollHeight` 4261、0 个 `pageerror`，只是没有 rail 宿主和提示条。所以不是夹具没消息，是插件自身在渲染阶段抛错。

## 定位

`src/client/MilestoneRail.tsx:632`（固定提交 `3bdd8d1327f30b64a1119aeb62c752ef1e12c713`，产物 `lib/client.js` 同义）：

```ts
const awaitingInput = (useSession(s => s.queue) as unknown[]).length > 0
```

宿主 `@deepseek-ai/dsh-client-ui-session@0.2.0-rc.2` 的会话快照只有 `readonly pendingInteraction: SessionPendingInteraction | undefined`，没有 `queue`；选择器返回 `undefined`，`.length` 直接抛错。产物里 `.queue)` 全文件仅此一处。

修法建议（未代改）：`Boolean(useSession(s => s.pendingInteraction))`，并给同类选择器统一加空值保护。

## 测试环境

- 宿主 `@deepseek-ai/dsh 0.2.0-rc.2`，Node v24.19.0
- Edge 155.0.4283.62 headless / Playwright 1.64.1
- 发布件 `dsh-milestone@0.7.2`：本地 tarball SHA1 `05bff0576f22c73f89c62a46d167559876bb814b`、SHA512 `sha512-F/mWJyfPObZLcyUT4v0bGWF6No4hE3brQEjS/mQYY0yMWkdlwgrIgUI7gsblWjAbs73AfPL6i1SbIA5Vln8J6Q==`，与 npm 元数据一致
- 会话为手写夹具经官方 jsonl 持久化写入，未调用任何模型

## 附带的安装期问题（与上面分列，不混为一谈）

按 README 的 `dsh plugin --profile <n> add dsh-milestone` 安装后，profile 内出现 `dsh-session@0.1.5-rc.2`、`dsh-session-projection@0.1.5-rc.2`、`dsh-llm@0.1.5-rc.2`，遮蔽宿主 0.2.0-rc.2。后果是宿主 stderr `2 entries did not activate`、`/api/session/list` 返回 `gateway/definition-unavailable`、侧栏"暂无会话"。钉住核心包版本后消失。属依赖区间问题，不是本条 UI 缺陷，但对新手而言是"装完什么都打不开"的第一道坎。

## 判定

- **confirmed**：
  1. 圆点时间轴在 0.2.0-rc.2 上整条不渲染，0 圆点，且同屏存在 3 条用户消息行（对照 profile 同读数）。
  2. 根因为 `MilestoneRail.tsx:632` 读取已移除的 `queue` 字段。
  3. 按 README 安装会遮蔽宿主核心包，导致会话列表整体读不出。
- **untested（被上面阻断，无法观测，不写成通过或失败）**：圆点数与消息数对应、点击跳转、滚动高亮跟手、长会话（44 条）密度与重叠、窄窗 1000/760/600 与深色主题可见性、0/1 条边界、超长会话快速滚动。窄窗有附带读数：1000/760/600 下 rail 宿主不再布局（`rail_rect` 为 null），但不兼容提示条照旧渲染。
- **uncertain**：刷新一次后状态相同（0 圆点、无列表、提示条在）；冷重启与偏好保留未单独跑。
- **不做**：未改插件源码、未跑插件自带测试、未调用模型、未投稿上传。

## 证据（相对本目录）

- `evidence/p07-rootcause-queue.json` —— 产物片段、`.queue)` 唯一出现下标、宿主 `.d.ts` 字段对照
- `evidence/ab-w3-api-session.json` —— 装插件：3 用户行 / 12 锚点行 / 0 圆点 / 提示条在
- `evidence/ab-ctl-api-session.json` —— 对照不装：同读数、无提示条
- `evidence/ab-w3-incompatible.png`、`evidence/ab-w3-session-open.png` —— 截图
- `evidence/ab-w3-long-1000-light.json`、`-760-light.json`、`-600-dark.json` —— 窄窗与深色
- `evidence/h3-host.stderr.txt` —— 依赖遮蔽导致的 `2 entries did not activate`
- `evidence/install-01-add.txt` —— 安装日志

## 建议

1. 第 632 行改读 `pendingInteraction` 并加空值保护。
2. 全量排查同类 `useSession(s => ...)` 直接取属性/长度的写法。
3. 兼容提示条补上当前宿主版本号，并说明通常是插件与宿主版本不匹配。
4. 把宿主核心包从 `dependencies` 移到 `peerDependencies` 或放宽到主版本区间。
