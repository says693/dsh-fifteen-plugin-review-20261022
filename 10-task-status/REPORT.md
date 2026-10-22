# dsh-task-status 0.3.1 实测报告

被测对象：`@vlln/dsh-task-status@0.3.1`（npm 发布件），仓库 `vlln/dsh-task-status`，固定提交 `3751096f38eeb52df85b2cfae15a239d4197d643`。
宿主：`@deepseek-ai/dsh` 0.2.0-rc.2，web profile `ts2`。浏览器：Edge 155.0.4283.62 headless（Playwright 1.64.1）。
本报告只抄录 `evidence/` 下已有读数，未重新起宿主、未跑浏览器、未新增探针。

---

## 一、发现与影响

### D1 · 有主（owned）后台任务被 `/tasks` 路由静默丢弃

插件 `collectTasks()` 的第一个循环写的是：

```js
for (const agent of ctx.agents.list())
  for (const snapshot of jobs.list(agent)) {
    if (snapshot.ownerSession === void 0 || seen.has(snapshot.id)) continue
    ...
  }
```

宿主 `dsh-jobs-local` 给快照打的归属字段叫 `owner`，不叫 `ownerSession`。整个 DSH 安装树里 `ownerSession` 命中 0 个文件。

读数（`evidence/run4-11-owned-vs-unowned.json`）：

| 字段 | 值 |
|---|---|
| ownerId | `session-8edbe7f6-98a0-421b-aa96-32383adfe63e` |
| ownedJobId | `probe-2` |
| unownedJobId | `probe-3` |
| idsSeenByPluginRoute | `["probe-1","probe-3"]` |
| ownedMissingFromPluginRoute | `true` |
| unownedPresentInPluginRoute | `true` |
| ownerFieldPresentInAnyHostRow | `false` |

同一次读数里，`probe-2` 在 `jobs.list(agent)` 与 `jobs.list()` 两处都能查到，唯独不在插件路由的响应里；不带主的 `probe-3` 反而正常出现。

**影响**：模型侧真正起的后台任务都挂在某个会话名下，也就是都带主。这批任务恰恰是状态条要显示的那批。结果是状态条即便渲染出来，也只会显示无主任务，而正常使用中几乎不存在无主任务。这是「功能对目标用户不可用」，不是显示瑕疵。

**旁证**：`kill` 同一个 owned 任务返回 500 `job probe-2 belongs to another session`，说明宿主侧的归属判断是正常工作的——问题在插件读错了字段名。

### D2 · 状态条在有 running 任务时也从不渲染

client 用一条 DOM 属性判断「当前是否在 Chat 视图」：

```js
setInChat(document.querySelector('[data-chat-flow=""]') !== null)
```

该属性在宿主 0.2.0-rc.2 里不存在。DOM 全量 dump 的 `dataAttrs` 只有六种：`conversationScroll`、`inputScroll`、`composerInput`、`phase`、`placeholder`、`lexicalEditor`；`chatFlowAny=0`、`chatFlowEmpty=0`（`evidence/run2-05-dom-dump.json`）。判据为假，组件 `return null`。

读数（`evidence/run5-12-final-matrix.json`，三个 running 任务同时在跑）：

| 阶段 | hasChatFlow | barCount |
|---|---|---|
| 00-after-new-session | false | 0 |
| 01-three-running-tasks | false | 0 |
| 02-after-bar-click | false | 0 |
| 03-narrow-900 | false | 0 |
| 04-narrow-640 | false | 0 |
| 05-after-reload | false | 0 |
| 06-after-kill-all | false | 0 |

同一时刻 `/plugins/dsh-task-status/tasks` 返回 `count=3`、`statuses=["running","running","running"]`。轮询也确实在跑：`polls=62`。请求全 200，界面始终空白。

`run2-03`（11 次轮询）、`run3-08`、`run4-08`（35 次）、`run4-09`（28 次）四个实例读数一致，`barCount` 全 0。

**影响**：插件的主功能——输入框上方的任务进度条——在本宿主版本下不可见。用户装了它，看不到任何东西，也没有报错提示。

### D3 · `/output` 对 `/tasks` 里存在的 running 任务返回 404

`evidence/run5-12-final-matrix.json`：

- `tasksAlive`：200，`count=3`，`ids=["probe-1","probe-2","probe-3"]`，`keys=["id","kind","label","status","startedAt"]`
- 紧接着用第一个 id 读输出：`outputAlive` = **404**，body `{"error":"task probe-1 not found"}`（`bodyLen=34`）

`run2-02` 里同样：`burstOutputHead` = `{"error":"task probe-3 not found"}`，而 `probe-3` 刚在 `/tasks` 里以 `completed` 出现过。

`readTaskOutput()` 第一步就是 `collectTasks(ctx).some(...)`，而 D1 的字段错配让这个自检对 owned 任务恒为假；即使任务在列表里出现过，也会被判成 not found。

**影响**：README 承诺的「实时输出 tail」读不到内容。这个 404 还会在控制台留下一条 `Failed to load resource: the server responded with a status of 404 (Not Found)`（`run5-12`、`run4-10` 都有记录）。

### D4 · npm 元数据里的仓库 URL 是未求值的模板字面量

`evidence/01-npm-view.json`：

```
repository: git+https://github.com/vlln/+process.argv[1]+.git
homepage:   https://github.com/vlln/+process.argv[1]+#readme
bugs:       https://github.com/vlln/+process.argv[1]+/issues
```

三处都把仓库名写成了 `+process.argv[1]+`，看起来是发布脚本拼 URL 时漏了模板求值。

**影响**：从 npm 页面点「仓库」跳不到仓库，人只能自己去搜。不影响安装。

### 没有变成缺陷的现象

- **`/tasks` 的 unowned 通路是对的**：`probe-1/2/3` 三条都正常返回，`status` 从 `running` → `completed`/`killed` 的变化、`detail`、`finishedAt` 都对（`evidence/run2-02-unowned-jobs.json`）。
- **参数校验是对的**：缺 id → 400 `missing task id`；不存在的 id → 404 `task X not found`（三个实例一致）。
- **前端轮询节奏是对的**：约每秒一次，与声明一致。
- **宿主侧的 `tool-jobs` 被置 disabled 不是插件的问题**（见下文环境限制）。

---

## 二、怎么测的

### 1. 锁身份

- 发布件：`dsh-task-status-0.3.1.tgz`，15676 字节，10 个文件，解包 47324 字节。sha1 `62c1deb9…` 与 `dist.shasum` 一致，sha512 `sha512-rNIzEO…` 与 `dist.integrity` 一致，sha256 `21f7c648…`。registry 上有签名（keyid `SHA256:DhQ8wR5APBv…`）。
- 仓库：HTTP 200，MIT，8 stars，提交 `3751096f…` HTTP 200，提交信息「docs: 归档说明——官方已内置后台任务 UI，本插件冻结」。
- 发布件的 `gitHead` 是 `849d73a5…`，早于固定提交。两者逐文件差异没有比对，源码只用于读插件声明的工作方式。

### 2. 隔离

HOME / USERPROFILE / DSH_HOME / APPDATA / LOCALAPPDATA / XDG_* / npm cache 全部指向任务目录内的 `isolated-home/` 与 `temp/npm-cache`；宿主 cwd 是 `fixtures/workspace`（workspace id `ws-fixtures-task-status`）。未读写日常配置、真实会话、真实工作区；未访问用户正在使用的 DSH Web GUI。

### 3. 起宿主

`dsh --profile ts2 --host 127.0.0.1 --port 0 --no-open`，端口由宿主自选。共五个实例：run1 `:6481`、run2 `:3917`、run3 `:14623`、run4 `:14928`、run5 `:8798`。

### 4. 造真实后台任务（关键）

web profile 下 `jobs.start()` 直接失败：

```
500 {"error":"Error: background jobs unavailable: no job controller serves this agent
     (load @deepseek-ai/dsh-tool-jobs in its composition)"}
```

原因在 `evidence/04-profile-initialize.log`：

```yaml
- id: tool-jobs
  name: '@deepseek-ai/dsh-tool-jobs'
  disabled: true
```

这是 `@deepseek-ai/dsh-web-app` 给 base 打的 patch 段。组合里没有 job controller，任何 `jobs.start()` 都会被拒（`evidence/run1-02-unowned-jobs.json`，`agentCount=0`，三次 start 全 500）。

于是自建了一个探针插件 `dsh-probe-ts`（**不是改被测插件**），把 `ctx.jobs` 暴露成 HTTP，并调用宿主公开的 `ctx.jobs.attachController('probe-ts')` 补上 controller。之后 `/probe-ts/start` 一律 200，能按参数起任务：`long`（持续追加）、`instant`（同一 tick 结束）、`burst`（一次性灌 200000 字节）、`fail`。全程零模型调用、零密钥。

探针的 `/probe-ts/list` 原样返回 `jobs.list()` 与每个 agent 的 `jobs.list(agent)`，用来和被测插件的 `/tasks` 做逐字段比对——D1 就是这样定出来的。

### 5. 读页面

Edge headless，独立 context，`serviceWorkers: 'block'`，只放行 `127.0.0.1 / localhost / [::1]`。每次快照读四类东西：

- `document.querySelectorAll('[data-task-status-bar]')` 的数量与矩形、文本、子节点数
- `[data-chat-flow=""]` 是否存在、子节点数、文本长度
- 页面横向溢出（`scrollWidth > clientWidth`）
- `localStorage` 键、控制台 error/warning、所有 `/plugins/dsh-task-status/*` 与 `/probe-ts/*` 的网络响应

截图 33 张，覆盖 1440×1000 / 900×800 / 640×700 三档。

---

## 三、环境限制（重要）

**无法在不调用模型的前提下造出「真实」的后台任务。**

这一点必须说清楚，因为它决定了哪些结论能下、哪些不能：

1. web profile 把 `tool-jobs` 置为 disabled，模型侧没有 `run_in_background` 通道。所以「模型真的起了一个 bash 后台任务，然后看状态条怎么显示」这条路，在本环境里走不通。
2. 走通的那条路是 `ctx.jobs.attachController` + `jobs.start(spec)`，造出来的是**进程内合成任务**：它和真实任务共用同一个 `ctx.jobs` 注册表、同一套字段（`id/kind/label/status/startedAt/finishedAt/owner`）、同一套生命周期（running → completed/killed/failed），但 `kind` 是 `probe` 而不是 `bash` 或 `agent`。
3. 所以：凡是只依赖「注册表里有一个 running 任务」的结论（D1、D2、D3 都是这一类），成立；凡是要依赖「模型侧真实产出」的结论（比如真实 bash 任务的输出格式会不会被 tail 正确渲染、任务名与命令摘要怎么显示），都不能下。
4. 页面上还压着一层与本插件无关的阻塞：会话停在「添加一个 API Key 开始使用」的引导上。`run2` 用「继续」关掉预览说明后停在这一层；`run3`/`run4` 的 `bodyTextLen` 恒为 65 / 200，点「新会话」、展开工作区都不改变现状。填 Key 需要真实密钥，按边界不填。这让「首屏能不能进入对话流」这一条只能记 `uncertain`，不能全算到插件头上。

另外两条限制：

- `run3` 的 profile 没干净启动：4 条未激活，`workspace 'eb42bca0-…' is absent from registry order`，`session-controller` / `workspace-controller` pending，浏览器里出现 `session/control: active Service "sessionController" is unavailable`。run3 的读数只作参考，结论以 run4/run5 为主。
- 未做两站查重（评测站与 bounty issues），因此本报告不声称首报。

---

## 四、四态分布

`RESULTS.json` 共 28 项：confirmed 21、uncertain 2、untested 5。

**untested 的 5 项都是「状态条从不渲染」的下游**，没有状态条就没有可展开的行、没有可量的滚动区、没有可观察的刷新重建：

| id | 项 | 为什么测不到 |
|---|---|---|
| C23 | 多任务排列与计数 | 三个 running 任务在 `/tasks` 里，但状态条 0 个，没有可点的行 |
| C24 | 展开后 tail 是否每秒增长 | 同上，`02-after-bar-click` 阶段 `barCount` 仍为 0，点击落空 |
| C25 | 窄窗 160px 封顶与横向溢出 | 状态条不出现，窄窗只剩宿主自身重排 |
| C26 | 刷新与冷重启后的重建/清理 | 基线本来就是 0，reload 后仍是 0，两个 0 没有区分力 |
| C27 | 两站查重 | 本轮只允许转写已有证据，未联网 |

**uncertain 的 2 项**：C19（零任务时是否隐藏——有任务的场景同样是 0，无法证明隐藏是条件判断的结果）、C22（首屏停在 API Key 引导，归因不一）。

---

## 五、给维护者的建议

1. `collectTasks()` 的过滤字段改成宿主实际投影的 `owner`（或两个字段都认）。这一条不修，功能对正常用户完全不可用。
2. `[data-chat-flow=""]` 这个判据在 0.2.0-rc.2 里已经不存在了，需要换一个更稳的判据（例如宿主 conversation 槽的挂载状态），或至少在锚点缺失时降级为「按当前会话渲染」而不是直接 `return null`。
3. `readTaskOutput()` 的自检不要复用 `collectTasks()` 的过滤结果——用 `jobs.get(id)` 之类直接查，避免「任务在列表里、却读不到输出」的自相矛盾。
4. 修一下发布脚本里的仓库 URL 模板。
5. 仓库里的归档说明已经写明「官方已内置后台任务 UI，本插件冻结」。既然冻结，前三条是否还修，由作者定；但文档里最好把「在 0.2.0-rc.2 上不可用」写进 README，免得后来的人装了看不到东西。

---

## 六、复跑

见 `REPRODUCE.md`。所有命令都在任务目录内执行，不触碰日常配置与真实会话。
