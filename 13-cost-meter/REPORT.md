# dsh-cost-meter 0.1.0-rc.3 技术实测报告

对象：`@steven-wu/dsh-cost-meter@0.1.0-rc.3`（npm 发布件，未改源码）
宿主：`@deepseek-ai/dsh 0.2.0-rc.2`
源码对照提交：`9f8a924f2496856f671a508aca44bff7d3377770`（与包内 `gitHead` 一致）
测试时间：2026-10-22

---

## 一、发现与影响

### 1. P0 — 徽标在 0.2.0-rc.2 上从不出现（confirmed）

**用户看到什么**：装好、重启、打开任何有 token 用量的会话，头部没有会话总成本，每条回复结尾也没有该轮成本。没有报错，控制台干净，就是没有。

**读数**：6 个合成会话 × {1440, 1000, 760, 600} px × {light, dark} × {首屏, 刷新后, 切走再切回, 冷重启}，`.dsh-cost-badge` 与 `.dsh-cost-tip` 计数恒为 **0**。
排除项：CSS 已注入（`costCssLoaded=true`）、客户端已加载、`conversation.session.header.utilities` 槽位实测有 2 个子节点（别的插件）、`conversation.chat.assistant-actions` 槽位存在、用量数据宿主已渲染（页面正文出现「用量 1 tok」）。

**根因**（已定位到具体字段）：投影定义字段名与宿主 `SessionProjectionRegistry` 的约定不一致。

| 侧 | 字段 |
| --- | --- |
| 插件 `makeProjection()` 返回 | `key, **schema**, init, apply, **view**, stateVersion` |
| 宿主 `register(definition)` 读取 | `definition.key, **definition.stateSchema**, definition.init, definition.apply, **definition.wire.viewSchema / wire.view**, definition.stateVersion` |

宿主把 `stateSchema` 与 `wire` 都取成 `undefined`，而所有客户端视图出口都有

```js
if (def.wire === void 0) continue;   // dsh-session-projection/lib/index.js:247
```

于是 `useProjection("sessionCost")` 在浏览器里恒为 `undefined`，`SessionCostBadge` 与 `TurnCostBadge` 都命中
`if (!cost || cost.priced !== true) return null`。

**关键区分**：折叠和持久化都是好的。把同样的 usage 事件喂给插件自己的 `apply`：

```
totals { input: 12000, output: 3400, cacheRead: 2048, cacheWrite: 0 }
usd    { inputUsd: 0.00783, outputUsd: 0.0066555, cacheReadUsd: 0.000044544 }
priced: true
```

投影缓存 `isolated-home/storages/session_projcache/sessions/sess-cm-tiny.json` 里也确实落了
`sessionCost: { ver: 3, seq: 11, ... inputUsd: 6.525e-7, priced: true }`。
**算得对、存得住，只是发不出去。**

**建议**：`makeProjection` 返回 `stateSchema`（原 `schema`）与 `wire: { viewSchema, view }`（原顶层 `view`）。

### 2. P1 — 宿主默认模型走错价目行，成本高估约 3.48 倍（confirmed）

宿主 catalog 实测只有两个模型：`deepseek-flash`（默认）与 `deepseek-v4-pro`。
插件价目表的键是 `deepseek-official/deepseek-v4-pro` 与 `deepseek-official/deepseek-v4-flash`。
`deepseek-flash` 查不到 → `priceOf` 回退 `defaultPrice`（即 v4-pro 那一行）。

| provider/model | 插件算（1M 输入 + 1M 输出） | 官方 off-peak | 偏差 |
| --- | --- | --- | --- |
| `deepseek-official/deepseek-flash`（宿主默认） | **$2.61** | **$0.75** | ≈ +248%（3.48×） |
| `deepseek-official/deepseek-v4-pro` | $2.61 | $2.64 | ≈ −1.1% |
| `some-vendor/mystery-model-9000` | $2.61 | — | 静默同价，无标记 |

官方来源 <https://api-docs.deepseek.com/quick_start/pricing>，抓取 **2026-10-22T01:48Z**；off-peak 为 peak 一半，peak 为 UTC 周一至周五 01:00–04:00 与 06:00–10:00；页面注明 `deepseek-flash` 才是模型名，`deepseek-v4-flash` 已退役按 Flash 价计费。

### 3. P2 — 极小金额落到 `$0.00000`（confirmed）

`formatUsd`：`>=1` → 2 位、`>=0.01` → 4 位、否则 5 位。
1 token = `6.525e-7` USD → 显示 `$0.00000`，与「这一轮免费」在屏幕上无法区分。
算术无误；这是显示下限问题。建议 `< $0.00001` 这类明确写法或提高有效位。

### 4. P1 — 默认安装被宿主拒绝（confirmed，安装面事实）

```
installation rejected: Plugin @steven-wu/dsh-cost-meter@0.1.0-rc.3 is incompatible with
dsh 0.2.0-rc.2: peerDependencies {... "^0.1.0-rc.6" ...}; nothing was installed.
```

需 `dsh plugin allow-version … --accept-risk` 才能装。这不是 UI 缺陷，但决定普通用户能否装上。

---

## 二、步骤（可复跑）

1. 隔离环境：`DSH_HOME`/`HOME`/`USERPROFILE`/`APPDATA`/`LOCALAPPDATA`/`XDG_*`/npm cache 全部指向任务目录。
2. 派生 web profile：`dsh --profile cm --from-default-profile web --dump-config`
   （不派生则 boot `EXIT 1` 且 stdout/stderr 全空——这是环境缺陷，不是插件缺陷）。
3. `dsh plugin --profile cm add @steven-wu/dsh-cost-meter@0.1.0-rc.3` → 被拒 → `allow-version … --accept-risk` → 再 add。
4. `make-sessions.mjs` 用官方 `dsh-session-persistence-jsonl` 写 7 个合成会话
   （envelope 用 `{seq,time}`；`turn`/`step` 从 1 起；`request/header` 必须带 `data.reason ∈ {initial,resume,change,series}`，否则宿主报 `seed request/header at index N has an invalid …`）。
5. `host.mjs h1 cm` 起宿主（`--host 127.0.0.1 --port 0 --no-open`），写 `evidence/h1-host.local.json`。
6. `probe-01/02/03/04/05/08/10` 用 Edge + Playwright 打开、切会话、改视口/主题、刷新。
7. `probe-06/07/09` 不走浏览器，直接把未修改的发布件注册进宿主真实 `SessionProjectionRegistry`，核对字段名、折叠数值与定价。
8. 冷重启：停 h1（按 `h1-host.local.json` 的 pid），起 `h2`，`probe-11` 重测。

---

## 三、环境

| 项 | 值 |
| --- | --- |
| 宿主 | `@deepseek-ai/dsh` 0.2.0-rc.2 |
| 相关宿主包 | `dsh-session-projection` 0.2.0-rc.2、`dsh-client-ui-conversation` 0.2.0-rc.2、`cordis` 4.0.4 |
| 浏览器 | Edge headless via Playwright 1.64.1，隔离 context，非本机地址一律 abort |
| 视口 / 主题 | 1440×1000、1000×800、760×800、600×800；light / dark |
| Node / npm / OS | v24.19.0 / 11.17.0 / Windows 10 19045 |

包身份：shasum `6ebd7cba…875b`、integrity `sha512-ZLvTNe2Y…J1g==`、8 个文件、MIT（`Copyright (c) 2026 Sttrevens`）。逐个文件 sha256 见 `PROVENANCE.json`。

---

## 四、限制

- **没有调用模型**，因此所有金额都是「合成 token 数 × 价目表」的推算，不是账单；也测不到流式过程中的实时刷新。
- **悬停分项没测到**：没有徽标可悬停，`.dsh-cost-tip` 计数 0。工具提示的实现（`position:fixed`、四桶、`\n` 连接）只在源码层面读过。
- **刷新/切会话/冷重启的「数值保留」只验到后端**：缓存行跨重启存活已确认，但屏幕上没有可比对的东西，用户可见的那一半判 `uncertain`。
- **窄窗口与深色**：无横向溢出、无遮挡，但这是因为没有元素被画出来，不能当作布局合格的证据。
- 只在 Windows + Edge 上跑过；其他平台/浏览器未测。
- 只测了 npm 发布件，没有对比仓库其它提交。

---

## 五、附：一次自查与纠正

第一次跑注册面探针时没设 `DSH_HOME`，插件按设计在**真实主目录**写入了 `C:\Users\32621\.dsh\cost-pricing.json`。
核对创建时间（2026-10-22 09:44:03，与本次运行吻合、且运行前目录列表里没有该文件）后删除，副本留 `evidence/stray-created-by-my-probe-cost-pricing.json`；
此后所有运行都用 `DSH_COST_PRICING` 指到任务目录内，并在每次运行后确认该路径不存在。
这一步没有读取任何密钥或会话内容，写入内容仅为插件自带的默认价目表。
