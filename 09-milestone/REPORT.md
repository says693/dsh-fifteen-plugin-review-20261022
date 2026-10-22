# dsh-milestone 0.7.2 技术报告（宿主 @deepseek-ai/dsh 0.2.0-rc.2）

## 一、先说结论

在实测的宿主版本上，dsh-milestone 的圆点时间轴**整条不出现**，这不是"在某些会话里不显示"，而是组件在渲染阶段就抛异常，插件自带的兼容边界把它收成一条"与当前 DSH 版本不兼容，已停用"的提示条。

一处代码即可解释全部现象：

```ts
// src/client/MilestoneRail.tsx:632（固定提交 3bdd8d13）
const awaitingInput = (useSession(s => s.queue) as unknown[]).length > 0
```

`useSession` 的选择器在本宿主上返回 `undefined`——0.2.0-rc.2 的会话快照已经用 `pendingInteraction` 取代了 `queue`——随后取 `.length` 抛 `TypeError: Cannot read properties of undefined (reading 'length')`。这行位于组件顶层渲染路径，React 的 error boundary 只能力保宿主不崩，代价是整条 rail 被停用。

除此之外还有一层**安装期**问题，容易和上面混为一谈，单独列出：按 README 的命令安装会把宿主的核心包替换成旧版，导致连会话列表都读不出来。钉住依赖版本后这层问题消失，上面那条才是插件自身的缺陷。

## 二、发现与影响

### 发现 1（阻断）：`queue` 字段消失导致 rail 抛错停用

| 项 | 读数 |
|---|---|
| rail 宿主元素 | 存在，8px 宽，`x=276..284`，`z-index:11` |
| `[data-rail-list]` | 不存在 |
| `[data-rail-dot]` 计数 | 0 |
| 同一时刻屏幕上的用户消息行 | 3 |
| 同一时刻的 `[data-chat-anchor-key]` 行 | 12 |
| 不兼容提示条 | 存在，`x=1100,y=892,326×94`，文案"dsh-milestone 与当前 DSH 版本不兼容，已停用。请升级插件。" |
| 控制台错误 | 2 条，第一条栈顶为 `MilestoneRail` |
| `pageerror` | 0（异常被 error boundary 接住，没有冒泡到 window） |

**关键对照。** 同一份会话数据、同一个宿主版本，在**不装插件**的对照 profile 里同样是 3 条用户消息行、12 个锚点行、滚动高度 4261px、0 个 `pageerror`。也就是说会话渲染本身完全正常，缺的只是圆点。这排除了"夹具坏了导致没消息"的可能。

**根因链。** 产物里 `.queue)` 只出现一次（`evidence/p07-rootcause-queue.json` 记录了下标与上下文），宿主 `@deepseek-ai/dsh-client-ui-session@0.2.0-rc.2` 的类型声明里只有 `readonly pendingInteraction: SessionPendingInteraction | undefined`，全文没有 `queue` 字段。三点合起来构成闭环，不是推测。

**为什么这很隐蔽。** 插件作者写了 `CompatBoundary`，本意是"宿主升级时不要白屏、给用户一句人话"。它确实做到了——宿主没事，用户看到一条提示。副作用是：`pageerror` 为 0，控制台只有 2 条 console error，如果只看"页面有没有崩"，会误判为"插件只是不显示"。实际是整条功能在错误边界后面短路了。

### 发现 2（安装期，非 UI 缺陷）：装插件会遮蔽宿主的核心包

| 项 | 读数 |
|---|---|
| `dsh plugin --profile web add dsh-milestone` 后 profile 内核心包 | `dsh-session@0.1.5-rc.2`、`dsh-session-projection@0.1.5-rc.2`、`dsh-llm@0.1.5-rc.2`、`dsh-brand@0.1.5-rc.3` |
| 宿主自带版本 | 全部 `0.2.0-rc.2` |
| 宿主 stderr | `dsh: warning: 2 entries did not activate`（typert-loader、image-offload） |
| 界面后果 | 侧栏"暂无会话"；`/api/session/list` 返回 `gateway/definition-unavailable`；`scrollHeight` 1000、行数 0 |

插件把 `@deepseek-ai/dsh-session`、`dsh-session-projection`、`dsh-llm` 写进 `dependencies` 并固定在 `0.1.5-rc.2`（peerDependencies 只对 `dsh-client-*` 开了 optional 区间）。loader 从 profile 目录解析同名 bundle 行，就命中了这些旧副本。

把核心包钉回 `0.2.0-rc.2`（`pnpm-workspace.yaml` 的 `overrides`，pnpm 11 已不再读 `package.json` 的 `pnpm.overrides`）后，宿主 stderr 变空、会话正常列出、对话正常渲染——此时发现 1 才暴露出来。

**这条不算插件的 UI 缺陷，但它决定了新手照 README 装完会看到什么。** 报告把它单列，就是不想让两层问题互相背锅。

## 三、被测了但拿不到读数的部分

圆点不存在，下列检查在物理上无法观测，一律按"被阻断"处理，不写成"通过"也不写成"失败"：

- 圆点数与用户消息数是否一一对应（屏幕上有 3 条用户消息，圆点 0 个，只能说明阻断，不能反推对应关系）
- 点击圆点跳转
- 滚动时当前点高亮、是否跟手
- 44 条提问长会话的密度、间距、重叠
- 窄窗 1000 / 760 / 600 与深色主题下的可见性
- 0 条 / 1 条消息的边界
- 超长会话快速滚动

窄窗还有一个附带读数值得记：1440px 时 rail 宿主占 `x=276..284` 与正文列相交 4px；到 1000 / 760 / 600 时 rail 宿主**根本没有被布局**（`rail_rect` 为 null），但不兼容提示条照旧渲染。也就是说这条提示在窄窗下反而成了唯一可见的插件痕迹。

刷新一次后重测：仍是 0 圆点、无列表、提示条在。冷重启与"偏好是否保留"没有单独跑，这一项标为不确定。

## 四、复现步骤

1. 用 `--from-default-profile web` 派生一个隔离 profile（直接 `dsh plugin --profile <新名> add` 建出来的 profile 缺 `dsh-web-app`，宿主会静默起不来）。
2. 在 profile 的 `package.json` 里加 `dsh-milestone@0.7.2`，并把 `dsh-session` / `dsh-session-projection` / `dsh-llm` / `dsh-brand` / `dsh-timeout` / `dsh-typert-protocol` / `dsh-util-*` 全部钉回 `0.2.0-rc.2`。
3. `DSH_HOME` 指向隔离目录，`--host 127.0.0.1 --port 0 --no-open` 启动宿主。
4. Edge + Playwright headless 打开宿主给的地址，点开任意有用户消息的会话。
5. 观察：右下角出现不兼容提示条；`document.querySelectorAll('[data-rail-dot]').length === 0`；`document.querySelectorAll('[data-chat-flow-kind="user"]').length > 0`；控制台第一条错误栈顶是 `MilestoneRail`。

对照：同样的步骤去掉第 2 步的插件依赖，会话渲染读数与上面**完全相同**，只是没有提示条、也没有 rail 宿主。

完整的可复跑步骤见 `REPRODUCE.md`。

## 五、环境与限制

- 宿主 `@deepseek-ai/dsh 0.2.0-rc.2`，Node v24.19.0，Edge 155.0.4283.62（headless，Playwright 1.64.1）。
- 发布件 `dsh-milestone@0.7.2`，本地 tarball SHA1 `05bff0576f22c73f89c62a46d167559876bb814b`、SHA512 `sha512-F/mWJyfPObZLcyUT4v0bGWF6No4hE3brQEjS/mQYY0yMWkdlwgrIgUI7gsblWjAbs73AfPL6i1SbIA5Vln8J6Q==`，与 npm 元数据逐字符一致。
- 全部会话内容都是手写夹具，经官方 jsonl 持久化写入，**没有调用任何模型**。因此"悬停卡片里的 token 数、TTFT"这类依赖真实推理的数据不在本次范围内。
- 没有修改插件源码，没有安装兜底实现，没有跑插件自带的测试套件。
- 只清理了本任务创建的宿主进程（按 `evidence/*-host.local.json` 里的 pid）。

## 六、建议（给维护者）

1. 第 632 行改成读 `pendingInteraction` 并做空值保护，例如 `Boolean(useSession(s => s.pendingInteraction))`。这一处改动即可让 rail 在 0.2.0-rc.2 上重新挂载。
2. 同类隐患值得全量查一遍：`useSession(s => ...)` 的返回值凡是直接取属性/长度的，都在跨版本时会变成同一类崩溃。
3. `CompatBoundary` 的提示文案可以补一句"这通常是插件与宿主版本不匹配"，并附上当前宿主版本号，用户才知道该升级谁。
4. 把 `@deepseek-ai/dsh-session` 等核心包从 `dependencies` 挪到 `peerDependencies`，或至少放宽到宿主主版本区间，否则每次安装都会遮蔽宿主自己的包。

上面 4 条都只是建议，本次没有代改任何代码。
