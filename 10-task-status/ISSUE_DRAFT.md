<!-- intake:v1 {"project":"bench","target":"vlln/dsh-task-status","form":"intake"} -->

# [intake 输入] dsh-task-status 0.3.1：装好后有后台任务在跑，输入框上方始终不出现状态条；`/tasks` 还会漏掉有主任务

装的是 npm 发布件 `@vlln/dsh-task-status@0.3.1`，宿主 `@deepseek-ai/dsh` 0.2.0-rc.2，web profile。装完重启，靠 `ctx.jobs.start()` 起了 1 个、后来 3 个 running 的后台任务，`/plugins/dsh-task-status/tasks` 里能看到它们，页面输入框上方却一直是空的——`[data-task-status-bar]` 计数全程为 0，没有报错、没有占位、什么都没有。进一步对比有主 / 无主任务时，发现 `/tasks` 只返回无主的那批；有主（`owner` 指向当前会话）的那条被静默丢掉。另外用 `/tasks` 里确实存在的 running 任务 id 去读 `/plugins/dsh-task-status/output?id=…`，返回 404 `task <id> not found`。

## 现象

三个现象，都能用原始响应直接复现：

**1. 有主任务从 `/tasks` 里消失。** 同一个宿主、同一个会话，先起一条 owned 任务（`owner=session-8edbe7f6-98a0-421b-aa96-32383adfe63e`，id `probe-2`），再起一条 unowned 任务（id `probe-3`）。`/tasks` 返回的 `idsSeenByPluginRoute` 是 `["probe-1","probe-3"]`——`probe-2` 不在，`probe-3` 在。而同一次调用 `/probe-ts/list`（探针原样返回 `jobs.list(agent)` 与 `jobs.list()`）两处都能查到 `probe-2`。响应里 `ownerFieldPresentInAnyHostRow=false`，任务对象的键只有 `id/kind/label/status/startedAt/finishedAt`。

静态对照：`collectTasks()` 的过滤条件是 `if (snapshot.ownerSession === void 0 || seen.has(snapshot.id)) continue`，宿主给快照打的归属字段叫 `owner`。`ownerSession` 这个名字在整棵 DSH 安装树里命中 0 个文件。

**2. 有 running 任务时状态条仍然不渲染。** 三个 running 任务同时在跑（`/tasks` 的 `count=3`、`statuses=["running","running","running"]`），页面在七个阶段连续采样，`hasChatFlow` 全为 `false`、`barCount` 全为 0。轮询确实在跑：同一次会话里记到 `polls=62`，全部 200。

静态对照：client 用 `document.querySelector('[data-chat-flow=""]') !== null` 判断是否在 Chat 视图，不在就 `return null`。DOM 全量 dump 里 `dataAttrs` 只有 `conversationScroll / inputScroll / composerInput / phase / placeholder / lexicalEditor` 六种，`chatFlowAny=0`、`chatFlowEmpty=0`——这个属性在当前宿主版本里压根不产出。

**3. `/output` 对 `/tasks` 里存在的任务返回 404。** `tasksAlive` 返回 200、`count=3`、`ids=["probe-1","probe-2","probe-3"]`；紧接着拿 `probe-1` 读输出，得到 **404**、body `{"error":"task probe-1 not found"}`、`bodyLen=34`。另一个实例里，刚跑完的 burst 任务读出来是 `{"error":"task probe-3 not found"}`。控制台相应留下 `Failed to load resource: the server responded with a status of 404 (Not Found)`。

根因已经拿到：插件给 `ctx.jobs.read` 打的镜像补丁，读的是返回值的 `text` 与 `snapshot` 两个字段，而宿主的 `readJob()` 返回的是 `{chunks, lossy, job}`。直接调用打过补丁的 `ctx.jobs.read` 实测：

```json
{"resultKeys":["text","snapshot"],"hasChunks":false,"hasLossy":false,"hasJob":false,
 "hasText":true,"hasSnapshot":false,"textLen":0,"officialToolInputsUsable":false}
```

`text` 恒为空、`snapshot` 恒 undefined，于是 `readTaskOutput()` 返回 `snapshot=undefined`，路由的 `if (read === null || read.snapshot === undefined)` 成立 → 404。同一处还波及官方工具：`dsh-tool-jobs` 按 `renderModelDelta(read.chunks, read.lossy, read.job.output.spillPaths ?? [])` 消费，补丁后这三个入参全是 undefined。

## 复现

```
# 1. 隔离宿主（HOME/USERPROFILE/DSH_HOME/APPDATA/npm cache 全部指向任务目录）
node host.mjs run5 ts2 0          # 端口 0，宿主自选；就绪后写入 evidence/run5-host.local.json

# 2. 起 3 个 running 的进程内任务（走宿主公开 seam，不调用模型）
GET /probe-ts/start?mode=long&kind=probe&label=MULTI-A&intervalMs=250&lines=200
GET /probe-ts/start?mode=long&kind=probe&label=MULTI-B&intervalMs=300&lines=200
GET /probe-ts/start?mode=long&kind=probe&label=MULTI-C&intervalMs=350&lines=200

# 3. 读任务列表（期望 3 条 running，实际也是 3 条）
GET /plugins/dsh-task-status/tasks

# 4. 用第一步拿到的 id 读输出（期望 200 + {text, full:true, snapshot}，实际 404）
GET /plugins/dsh-task-status/output?id=probe-1

# 5. 页面上数状态条（期望 >=1，实际 0）
node probe-10-final-matrix.mjs run5
```

有主 / 无主对照（现象 1）的复现：

```
GET /probe-ts/start?mode=long&kind=probe&label=OWNED-running&owner=session-8edbe7f6-98a0-421b-aa96-32383adfe63e
GET /probe-ts/start?mode=long&kind=probe&label=UNOWNED-running
GET /plugins/dsh-task-status/tasks     # 只出现 UNOWNED 那条
GET /probe-ts/list                     # owned / unowned 两处都能看到 OWNED 那条
```

## 环境

| 项 | 值 |
|---|---|
| 被测件 | `@vlln/dsh-task-status@0.3.1`（npm 发布件，sha1 `62c1deb9…` / sha512 `rNIzEO…`，与 registry 一致） |
| 仓库 | `vlln/dsh-task-status`，提交 `3751096f38eeb52df85b2cfae15a239d4197d643`（MIT） |
| 宿主 | `@deepseek-ai/dsh` 0.2.0-rc.2，profile `ts2`（`dsh-base` + `dsh-web-app` + 本插件） |
| 浏览器 | Edge 155.0.4283.62 headless（Playwright 1.64.1），独立 context，只放行本机地址 |
| 端口 | 一律 `--port 0`；本次五个实例分别是 `:6481 / :3917 / :14623 / :14928 / :8798` |
| 隔离 | HOME / USERPROFILE / DSH_HOME / APPDATA / LOCALAPPDATA / XDG_* / npm cache 全在任务目录内 |
| 造任务方式 | 自建探针 `dsh-probe-ts` 调 `ctx.jobs.attachController` + `ctx.jobs.start`，**未改被测插件一个字节**，全程零模型调用、零密钥 |

## 证据

均在 `evidence/` 下，相对文件名：

- `run6-13-readshape.json` — 直调打过补丁的 `ctx.jobs.read`，`resultKeys=["text","snapshot"]`、`officialToolInputsUsable=false`（现象 3 的根因）
- `14-dedup.json` — 两站查重全量读数
- `run5-12-final-matrix.json` — 三个 running 任务的 `/tasks` 200 读数、`/output` 404 读数、七阶段 `barCount` 全 0、`polls=62`
- `run4-11-owned-vs-unowned.json` — 有主 / 无主任务对照，`ownedMissingFromPluginRoute=true`、`ownerFieldPresentInAnyHostRow=false`
- `run2-02-unowned-jobs.json` — 无主任务在 `/tasks` 里正常出现；`burstOutputHead` 为 not found
- `run2-05-dom-dump.json` — `chatFlowAny=0`、`chatFlowEmpty=0`、`dataAttrs` 六项清单
- `run2-03-v1-visibility.json` / `run4-08-v1-chat-visibility.json` / `run4-09-open-session.json` — 另外三个实例的 `barCount=0` 与各自轮询次数
- `04-profile-initialize.log` — `- id: tool-jobs / disabled: true`
- `run1-02-unowned-jobs.json` — 加 controller 之前 `jobs.start` 的 500 与错误文案
- `run5-final-01-three-running-tasks.png` — 三个任务在跑时的页面截图，对话区无状态条
- `01-npm-view.json` / `02-npm-pack.json` / `03-meta.json` — 版本、哈希、提交、LICENSE

## 四态边界

**已确认**：上面三个现象，每个都有 200/404 原始响应或 DOM 计数直接支撑；两个现象在 4–5 个独立宿主实例上重复出现。

**不确定**：零任务时状态条「正确隐藏」这件事，因为它和有任务时的 0 是同一个 0，看不出是条件判断的结果还是从未渲染；首屏停在「添加一个 API Key 开始使用」引导，这一层归因不清，不能全算插件。

**未验**：多任务并行时的计数与排列、展开后 tail 是否每秒增长、窄窗 160px 封顶与横向溢出、刷新与冷重启后的重建/清理——这四项都卡在「状态条不出现，没有可点的行」，不是没测，是测不到。

**查重**：两站已查（2026-10-22T02:12:18Z）。评测站 `review.lingshulab.com` 拉全量 172 条评测 + 133 个插件本地扫描，命中 0；悬赏仓 `FuRongJun-1999/dsh-bounty` 四查询（dsh-task-status=35 / vlln=2 / task-status=35 / 任务状态条=401），逐条比对标题后全部为分词误命中（dsh-task-dag、dsh-task-planner、dsh-loop、whale-girl 等别的插件），同范围重复 0。结论：两站均无同范围重复，可写首报。证据 `evidence/14-dedup.json`。

## 建议

1. `collectTasks()` 的过滤字段改成宿主实际投影的 `owner`（或同时认两个）。这一条不修，功能对正常用户完全不可用——模型侧起的任务都带主。
2. `[data-chat-flow=""]` 在 0.2.0-rc.2 里已不存在，换判据；退一步说，锚点缺失时也不该直接 `return null`，那会让整个组件静默消失。
3. `readTaskOutput()` 的自检别复用 `collectTasks()` 的过滤结果，用 `jobs.get(id)` 直接查，避免「任务在列表里却读不到输出」。
4. npm 元数据里 `repository / homepage / bugs` 三处 URL 都是 `github.com/vlln/+process.argv[1]+`，发布脚本的模板没求值，顺手修一下。
5. 仓库的归档说明写了「官方已内置后台任务 UI，本插件冻结」。若确实冻结，建议在 README 里补一句「0.2.0-rc.2 上状态条不显示」，省得后来的人装完以为是自己配错了。
