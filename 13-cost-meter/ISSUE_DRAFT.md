<!-- intake:v1 {"project":"bench","target":"Sttrevens/dsh-cost-meter","form":"intake"} -->

# 【intake 输入】dsh-cost-meter 0.1.0-rc.3 实测：徽标在宿主 0.2.0-rc.2 上一个都不出现；折叠算得对、落盘也对，只有客户端视图被丢掉；另附价目表键名对不上宿主真实模型 id 的读数

## 性质声明

本件是 **intake 输入（外部贡献申报）**，不是悬赏：不 `/claim`、不自行加 `bounty` 标签、不自行派发结算。
本件是**测评报告**；不排名、不给总分、不用星级。

## 一句话结论

装上之后无论是头部的会话总成本，还是每条回复结尾的该轮成本，**一次都没有出现过**；而宿主侧的折叠数值和落盘缓存都是对的——问题出在投影注册时字段名与宿主约定不一致，导致客户端视图从未被产出。

## 结论速览（每句都指到读数位置）

| # | 结论 | 判定 | 读数位置 |
|---|---|---|---|
| 1 | 6 个合成会话 × 4 个视口 × 明暗两主题 × 刷新/切会话/冷重启，`.dsh-cost-badge` 计数恒为 **0** | confirmed | §3.1、§6 |
| 2 | 根因：插件传给 `ctx.sessionProjections.register` 的键是 `schema` / `view`，宿主读的是 `stateSchema` / `wire.viewSchema`；宿主侧最终 `stateSchema===undefined`、`wire===undefined`，客户端视图被跳过 | confirmed | §3.2、§6 `p06-run2.txt` |
| 3 | 折叠本身正确：同一段 usage 折叠出 `input 12000 / output 3400 / cacheRead 2048`，`priced=true`，投影缓存里存了 `sessionCost` 行 | confirmed | §3.2 |
| 4 | 宿主真实模型 id 是 `deepseek-flash`（默认）与 `deepseek-v4-pro`，插件表里写的是 `deepseek-v4-flash`：`deepseek-flash` 静默回退 default 价，1M 输入+1M 输出算成 **$2.61**，官方 off-peak 为 **$0.75**（约 3.48 倍） | confirmed | §3.3 |
| 5 | 极小用量显示为 `$0.00000`（1 token = 6.525e-7 USD），与"这一轮免费"无法区分 | confirmed | §3.4 |
| 6 | 默认安装会被宿主拒绝（peerDependencies `^0.1.0-rc.6` vs 宿主 `0.2.0-rc.2`），需 `allow-version --accept-risk` | confirmed | §2 |
| 7 | 悬停分项、窄窗口/深色可读性、遮挡输入区 | untested | §4 |

## 1 环境与锚定

| 项 | 值 |
|---|---|
| 对象 | npm 发布件 `@steven-wu/dsh-cost-meter@0.1.0-rc.3`（`npm pack` 解包，未改一行） |
| shasum / integrity | `6ebd7cba8e57ff4870687ac4f503afa834df875b` / `sha512-ZLvTNe2YoeKkK5BVXutIoJUqMxuRvrczgCj6aDYLsM5zG4+av4vkAcdIlSyT8gJLWhowHx3VL7WJFJ1TRRTJ1g==` |
| 源码提交 | `9f8a924f2496856f671a508aca44bff7d3377770`（与包内 `gitHead` 一致） |
| LICENSE | MIT（`Copyright (c) 2026 Sttrevens`），sha256 `5d5b3a22…1178` |
| 宿主 | `@deepseek-ai/dsh` **0.2.0-rc.2** |
| 浏览器 | Edge headless via Playwright 1.64.1 |
| 隔离 | `DSH_HOME`/`HOME`/`USERPROFILE`/`APPDATA`/`LOCALAPPDATA`/`XDG_*`/npm cache 全在任务目录；`--host 127.0.0.1 --port 0 --no-open` |
| 会话数据 | 官方 `dsh-session-persistence-jsonl` 写合成会话；**不调用模型、不产生费用、不读真实会话/余额/密钥** |

## 2 安装面：默认会被拒

```
$ dsh plugin --profile cm add @steven-wu/dsh-cost-meter@0.1.0-rc.3
dsh: installation rejected: Plugin @steven-wu/dsh-cost-meter@0.1.0-rc.3 is incompatible
     with dsh 0.2.0-rc.2: peerDependencies {"@deepseek-ai/dsh-client-runtime":"^0.1.0-rc.6",
     "@deepseek-ai/dsh-session-projection":"^0.1.0-rc.6",
     "@deepseek-ai/dsh-client-ui-conversation":"^0.1.0-rc.6"}.
dsh: nothing was installed.
$ echo $LASTEXITCODE
1
```

宿主机实际装的是 `dsh-session-projection 0.2.0-rc.2` 与 `dsh-client-ui-conversation 0.2.0-rc.2`，不满足 `^0.1.0-rc.6`。
按宿主提示加 `--accept-risk` 后可以装上（exit 0），pnpm 报 `[WARN] Issues with peer dependencies found`。

这一条是**版本声明面的事实**，单独记录，不当作 UI 缺陷；但它决定了"普通用户能不能装上"。

## 3 实测

### 3.1 徽标始终为 0

同一页面上：CSS 注入成功（`style[data-plugin]` 存在，`costCssLoaded=true`），客户端已加载，两个目标槽位也都在宿主里存在——
`conversation.session.header.utilities`（该槽位实测有 2 个子节点，来自别的插件）、
`conversation.chat.assistant-actions`（存在，来自本插件的子节点 0 个）。
但 `.dsh-cost-badge` 计数在以下全部采样点都是 **0**：1440 主样本、刷新后、切到别的会话、切回来、1000/760/600 px、深色主题、冷重启。

用量数据本身宿主是认的：极小样本页面正文出现 `用量 1 tok`，说明 `assistant/message.usage` 已被解析渲染。

### 3.2 根因：注册字段名与宿主约定不一致

把**未修改的**发布件 `lib/index.js` 用假 ctx 截获它传给注册表的对象，再注册进宿主真实的 `SessionProjectionRegistry`：

| 侧 | 实际字段 |
|---|---|
| 插件传入 | `apply, init, key, **schema**, stateVersion, **view**` |
| 宿主存入 | `apply, init, key, **stateSchema**, stateVersion, **wire**` |
| 宿主实际取值 | `stateSchemaType = "undefined"`，`wireType = "undefined"` |

宿主源码（`dsh-session-projection/lib/index.js`）：

```js
register(definition) {
  const wire = definition.wire;
  const erased = { key: definition.key,
                   stateSchema: definition.stateSchema,   // ← 插件给的是 schema
                   ...
                   wire: wire === void 0 ? void 0 : {...} }; // ← 插件给的是顶层 view
```

而所有客户端视图出口都有这一句：

```js
if (def.wire === void 0) continue;   // 第 247 行，viewCheckpoint / 快照同款
```

于是 `useProjection("sessionCost")` 在浏览器里恒为 `undefined`，两个徽标组件都命中
`if (!cost || cost.priced !== true) return null`。

值得强调的是**这不是"算错了"，是"发不出去"**。同一段 usage 喂给插件自己的 `apply`，折叠结果完全正确：

```
totals { input: 12000, output: 3400, cacheRead: 2048, cacheWrite: 0 }
usd    { inputUsd: 0.00783, outputUsd: 0.0066555, cacheReadUsd: 0.000044544 }
byTurn["1"] 同上，priced: true
```

投影缓存文件里也确实落了一行（`sess-cm-tiny.json`）：

```json
"sessionCost": { "ver": 3, "seq": 11, "val": { "totals": {"input":1,...},
                 "usd": {"inputUsd":6.525e-7,...}, "priced": true, "model": "deepseek-v4-pro" } }
```

**建议修法**（未改插件源码，仅建议）：`makeProjection` 的返回对象把 `schema` 改名 `stateSchema`、把 `view` 包成 `wire: { viewSchema, view }`。

### 3.3 价目表键名对不上宿主真实模型 id

宿主 `/api/session/modelCatalog` 实测只提供两个模型：
`deepseek-flash`（默认）与 `deepseek-v4-pro`。
插件 `DEFAULT_PRICING.models` 的键是 `deepseek-official/deepseek-v4-pro` 与 `deepseek-official/deepseek-v4-flash`。

| provider/model | 1M 输入 + 1M 输出，插件算 | 官方 off-peak | 说明 |
|---|---|---|---|
| `deepseek-official/deepseek-flash`（宿主默认） | **$2.61** | **$0.75** | 未命中，回退 default 价，高估约 3.48× |
| `deepseek-official/deepseek-v4-pro` | $2.61 | $2.64 | 命中，差约 1.1% |
| `deepseek-official/deepseek-v4-flash`（插件键） | $0.8700 | — | 宿主不会发这个 id |
| `some-vendor/mystery-model-9000` | $2.61 | — | 与 v4-pro 完全同价，界面无任何"未定价"标记 |

官方价目表来源：<https://api-docs.deepseek.com/quick_start/pricing>，抓取时间 **2026-10-22T01:48Z**；
页面注明 off-peak 为 peak 的一半，peak 时段为 UTC 周一至周五 01:00–04:00 与 06:00–10:00；
并注明 `deepseek-flash` 才是模型名，旧的 `deepseek-v4-flash` 已退役。

### 3.4 精度：极小用量显示成零

`formatUsd` 三档：`>=1` 两位、`>=0.01` 四位、否则五位。

| 输入 | 显示 |
|---|---|
| 0.0000006525（1 token） | `$0.00000` |
| 0.0000065 | `$0.00001` |
| 0.004187136 | `$0.00419` |
| 0.03457293 | `$0.0346` |
| 2.61 | `$2.61` |
| 55.0188 | `$55.02` |

算术都对，但 `$0.00000` 与"这一轮完全免费"在屏幕上无法区分。
千万级用量（48M/12M/9.6M/2M）算出 `$55.02`，无溢出、无 NaN、无科学计数法。

## 4 未测 / 不确定 / 不做

- **未测**：悬停分项（没有徽标可悬停，`.dsh-cost-tip` 计数 0）；徽标在窄窗口与深色主题下的真实可读性；流式过程中的实时刷新。
- **不确定**：刷新/切会话/冷重启的"数值是否保留"。后端缓存确实在、确实跨重启存活，但屏幕上没有东西可比对，这一半无法判定。
- **不做**：真实账单核对（未调用模型，所有金额都是按价目表推算）；真实 API Key / 余额读取；投稿。

## 5 测试环境

见 §1。补充：Node v24.19.0、npm 11.17.0、Windows 10 19045。

## 6 证据

全部在 `evidence/` 下（相对文件名）：

| 文件 | 内容 |
|---|---|
| `install-01-add.txt` | 默认安装被拒的原文与 exit 1 |
| `install-02-allow-version.txt` / `install-03-add-after-allow.txt` | 豁免后安装成功 |
| `p01-boot.json` | 首屏：CSS 注入、槽位清单、badge 0 |
| `p03-open-sessions.json` | 6 个会话逐个打开后的读数 |
| `p03-run2.txt` | 同上的一次可读回放 |
| `p06-run2.txt` | 注册面键名对照（插件传入 vs 宿主存入） |
| `p07-values.txt` | 插件自身 `view` 的全部数值与格式化输出 |
| `p08-model-catalog.json` | 宿主真实模型 id |
| `p09-pricing.txt` | 宿主模型 id × 插件价目表的定价对照 |
| `p10-layout-theme-persist.json` | 视口/主题/刷新/切会话/悬停探测 |
| `p11-cold-restart.json` | 冷重启后的读数 |
| `p10-1440-main.png`、`p10-w600.png`、`p10-dark-1440.png`、`p11-cold-row1.png` | 截图 |
| `stray-created-by-my-probe-cost-pricing.json` | 我自己的探针误写文件的留档，已从真实主目录删除 |

## 7 复现

见同目录 `REPRODUCE.md`（可逐条照跑）。

## 8 建议

1. **P0（阻断）**：把投影定义改成宿主认的字段名（`stateSchema` + `wire:{viewSchema, view}`）。不改这一条，插件在 0.2.x 上等于没装。
2. **P1**：价目表补 `deepseek-official/deepseek-flash` 这一行（或改成用宿主 catalog 的 id 匹配），并在未命中时给出可见提示，避免静默用 default 价。
3. **P2**：极小金额不要落到 `$0.00000`，可用 `< $0.00001` 或提高精度；`peerDependencies` 的版本范围需要覆盖当前宿主。
