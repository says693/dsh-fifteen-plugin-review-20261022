# dsh-token-pet 实测评测报告

**被测对象**：`npm dsh-token-pet@0.4.2`（下文简称 **0.4.2**）与任务坐标 `pk7j7sqryy-ops/dsh-token-pet@b498347`（下文简称 **pk-0.1.0**）。
**宿主**：`@deepseek-ai/dsh 0.2.0-rc.2`，`--host 127.0.0.1 --port 0 --no-open`，完全隔离的 `DSH_HOME/HOME/APPDATA/npm cache`。
**浏览器**：Edge 155.0.4283.62（headless）+ Playwright 1.64.1，1440×900 起。
**测试方式**：自造合成会话（已知 token 量）+ 手算对照 + DOM/React fiber 归属判定 + 命中测试，全程不调用模型、不读真实会话或密钥。

---

## 0. 一句话结论

0.4.2 的数字是可信的——上下文占用、会话累计、终身账本、构成明细逐项与宿主真值和我的手算值一致，长开 5 分钟零泄漏；但它的玩偶浮层用一个 **248×217 的透明 `position:fixed` 容器**盖住了发送按钮，真实鼠标点击会被浮层吃掉。任务坐标 pk-0.1.0 是**另一份完全不同的实现**，它能正常渲染，并在天气接口不可用时干净降级。

---

## 1. 先要解决的问题：任务坐标与 npm 发布件不是同一个东西

这是本轮最先必须落笔的核对，因为它决定了后面每条结论该挂到谁头上。

| 项 | `pk7j7sqryy-ops/dsh-token-pet` | `dsh-token-pet` on npm |
|---|---|---|
| 版本 | 0.1.0（`b498347`） | 0.4.2（latest） |
| `package.json` 的 author | `pk7j7sqryy-ops` | 空 |
| repository | `pk7j7sqryy-ops/dsh-token-pet` | `Jimmy0123-ux/dsh-token-pet`（**全部 9 个版本都是这个**） |
| 仓库创建 / 最后推送 | 2026-08-15 / 2026-08-15 | 2026-09-01 / 2026-09-21 |
| stars | 3 | 56 |
| client bundle | 21,551 字节 | 23,054,338 字节（内嵌 WebP 逐帧条带） |
| 文件数 | 7（lib 2 个 + dynamic 2 个 + 3 个元文件） | 53 |
| 天气 / 日期模块 | **有**（ipwho.is + Open-Meteo + 3 天预报 + 极端天气预警 + 日期周几） | **没有**（bundle 里「天气/weather/气温/temperature/日期」命中 0 次） |
| 形态 | 会话头部右上角内嵌药丸 | 右下角常驻悬浮玩偶 + 三标签浮窗 |

结论：**任务给的坐标是另一份更早、更小的实现**。按任务要求"优先测 npm 发布件"，主线测 0.4.2；但任务书第 4 项（天气/日期模块）只有 pk-0.1.0 才具备，所以两者都装、都测，结论严格分开，绝不互相套用。

证据：`evidence/step01-npm-view.json`、`evidence/npm-versions-repository.txt`、`evidence/pk0.1.0-manifest.txt`。

---

## 2. 主要发现

### 2.1 【P1】玩偶浮层拦截发送按钮的点击（0.4.2，四档宽度复现）

**现象**：在会话里打了草稿，点发送按钮没有任何反应；宠物反而"打招呼"并且弹出统计浮窗。

**读数**：

| 宽度 | 玩偶 HUD rect | 发送按钮 rect | 相交 | 发送中心命中的组件 |
|---|---|---|---|---|
| 1440 | (1302,682) 120×175 | (1202,828) 34×34 | 20×29 px | `TokenPetWindow` |
| 1000 | (862,682) 120×175 | (839,828) 34×34 | 23×29 px | `TokenPetWindow` |
| 760 | (622,682) 120×175 | (695,828) 34×34 | 47×29 px | `TokenPetWindow` |
| 600 | (462,682) 120×175 | (535,828) 34×34 | 48×29 px | `TokenPetWindow` |

**定位**：插件通过 portal 在 `document.body` 末尾挂了一个容器：

- `position: fixed`、`z-index: 9998`、`pointer-events: auto`
- rect **248×217 @ (1174,665)** —— 比可见玩偶 120×175 大得多，是一块**透明**矩形
- `document.elementFromPoint(1219, 845)`（发送按钮正中心）返回该容器的子树，React 链为 `div > TokenPetWindow > ErrorBoundary > RootEntry`

**决定性对照**：

1. 先输入草稿 `T-1440`，再用真实鼠标点发送中心 → 草稿仍在（`draft='T-1440'`）、玩偶 aria-label 从「用量小宠物，空闲」变成「用量小宠物，打招呼」、浮窗被打开。
2. 同一按钮改用 JS `b.click()` 绕过命中测试 → 草稿被清空、对话行数 8 → 13，发送成功。

所以按钮本身没坏，是**浮层的透明区域吃掉了点击**。网格采样进一步量化：视口内 550 个 10 px 采样点落在插件浮层上，其中 **346 个（63%）落在可见玩偶之外**——即用户点到"空气"也会被吞。

**影响**：任何把玩偶拖到输入区附近的用户都会间歇性点不动发送；本夹具下是默认位置就直接命中。

**建议**：外层 portal 容器改用 `pointer-events: none`，只在可见玩偶/卡片子节点上 `pointer-events: auto`；或把容器尺寸收紧到实际可见区域。

证据：`evidence/p12-hit-test.json`、`evidence/p12-click-effect.json`、`evidence/p13-send-interception.json`、`evidence/p13-after-send-click.png`、`evidence/p14-send-intercept-widths.json`、`evidence/p25-overlay-map.json`。

### 2.2 【P2】浮窗"模型未知"在本宿主恒为未知（0.4.2）

卡片顶部那行固定显示「模型未知」，但账本与模型页明明正确写着 `fixture-provider · fixture-model-a`。

**定位**：`buildPetView` 里 `model = timeline?.model`，而 `timeline` 来自 `useProjection('contextTimeline')`。本宿主（0.2.0-rc.2）实际只注册了四个投影：

```
contextPressure      => dsh-token-meter
contextBreakdown     => dsh-token-meter
tokenUsage           => dsh-token-meter
sessionStats         => dsh-session-stats
contextTimeline      => (NOT PROVIDED BY HOST)
todayUsageBuckets    => (NOT PROVIDED BY HOST)
```

于是 `model` / `provider` 恒为 `undefined` → 恒渲染 `unknownModel`（「模型未知」）。同一原因也让趋势卡在这台机器上永远拿不到当日数据。

**影响**：用户会以为插件读不出模型名，而实际上模型名在账本里是准的——这是"看着像缺陷、实际是宿主少了一个投影"的典型，容易被误报。建议插件在该投影缺失时回退到账本/`cumulative.byModel`，或直接把文案改成「本宿主不提供」。

证据：`evidence/p26-projections.txt`、`evidence/p21-sess-pet-unknown.png`、`evidence/p22-ledger-and-degrade.json`。

### 2.3 【P3】索引状态自相矛盾（0.4.2）

点「首次建立索引」→ `POST /token-pet/index/build` **200**，UI 变「索引已就绪 · 0 个已索引」，磁盘 `session-usage-index.json` 只有 **27 字节**（空索引）。但同一屏：

- 「最耗 Token 的会话」仍写「首次建立历史索引后显示」
- 「本日用量趋势」写「暂无趋势数据」

而设置页同时明写「维护状态：当前宿主的会话持久化接口不支持趋势索引 / 失败：`sessionPersistence does not expose listSnapshots`」。

**性质**：降级本身是插件有意做的（CHANGELOG 0.3.1 有记录），**不是缺陷**；但「已就绪」与「首次建立后显示」并置，会让新手以为还没建完，反复点按钮。

**建议**：宿主不支持时，把「首次建立索引」按钮与「索引已就绪」文案一起换成明确的「本宿主不支持趋势索引」，不要同时保留两种口径。

证据：`evidence/p22-ledger-and-degrade.json`、`evidence/p22-after-build.png`、`evidence/p16-endpoints.json`。

### 2.4 确认做对的地方（0.4.2）

| 检查 | 手算 | 插件显示 | 结论 |
|---|---|---|---|
| 会话累计 | 12000+9000+800+400+30000+41000+2000 = **95200** | 「21k 输入 / 1.2k 输出 / 71k 缓存读 / 2.0k 缓存写」，四者相加 = 95200 | 一致 |
| 上下文占用 | (9000+41000+0)/100000 = **50%** | 「50% | 50k / 100k」 | 一致 |
| 终身账本 | 1,245,000+32,600+942,000+4,000 = **2,223,600** | 「终身用量账本 · 4 个会话 | 2.2M」，四桶 `1.2M / 33k / 942k / 4.0k` | 缓存读 942,000、缓存写 4,000 与手算**逐位相同** |
| 极端用量 | 2,000,000/100,000 = 2000% | 「100% | 2.0M / 100k」 | 夹取正确 |
| 空会话 | 无数据 | 「上下文 –%」「— | 上下文窗口读取中」「0 轮次 | 0ms」 | 有占位，不假报 0% |
| 跨零点 | 前一天 23:50 / 今天 00:10 | 「10月21日 23:50」「00:10」 | 分属两天 |

**刷新行为**：硬刷新后 chip 从占位「上下文 –%」在 **2 秒内**解析成真实百分比（3 次独立重载实测均在 t=2s 得到「上下文 50%」，0 报错）。早期探针看到的"一直 —%"发生在夹具被污染或空会话时，不是读数缺陷。

**性能**：滚动 40 次采 219 帧，mean 5.56 ms / p95 5.70 ms / max 7.40 ms，>33 ms 帧 **0**、longtask **0**；长开 306 秒采样 11 次，DOM 节点恒定 **795**、img 恒定 4、style 恒定 141，JS 堆 96.9 MB → 90.7 MB 回落——**无泄漏迹象**。

**降级**：把 `/token-pet/**` 全部 abort 后冷启动，玩偶与 chip 仍在（数据来自本地投影），浮窗逐块显示「索引读取失败；现有终身用量账本不受影响 / 重试」「读取失败」，连点刷新文案稳定、无异常。

**隐私**：全程只放行 `127.0.0.1`，`external` 请求列表为空；0.4.2 完全不联网（无天气模块）。

证据：`evidence/p21-clean-readings.json`、`evidence/p22-ledger-and-degrade.json`、`evidence/p17-performance.json`、`evidence/p23-offline.json`。

### 2.5 pk-0.1.0：能渲染，天气降级干净，但天气接口在本宿主 405

- 渲染正常：折叠药丸 `.tkp-root` 72×32 @ (1284,10) 显示「50%」+ 内联 SVG 玩偶；展开 `.tkp-card` 306×374 @ (1050,52)。
- 卡片内容：「Token 泡泡 / 📅 10月22日 · 周四 / 🌡️ 天气不可用 / 上下文占用 50% ~50.1K / 100K / 本会话累计 合计 95.2K（输入 21K、输出 1.2K、缓存读 71K、缓存写 2K）/ 上下文构成 系统提示 0、工具定义 135、消息内容 218 / 状态不错，继续冲～」。
- 天气：`POST /token-pet/weather` → **405 Method Not Allowed**，卡片降级为「🌡️ 天气不可用」，不抛错、不空白、不阻塞其它数据。宿主的 RPC 通道（`connection.rpc.handle('/token-pet', …, { authority:'loopback' })`）与这版宿主的调用语义不匹配。
- 极端用量：2,030,000 tokens → 药丸 79×32「100%」，卡片「~2M / 100K」，四桶相加与「合计 2M」一致。
- 窄窗：306 宽卡片在 1440/760/600 都完整在视口内（600 时 x=210..516），无横向滚动。
- 空会话：整块 `.tkp-root` **不存在**（但注入的 `style[data-plugin=dsh-token-pet]` 仍在，插件已加载）；这是该版的显示条件，不是崩溃。

**一条未决**：卡片里「上下文占用 ~50.1K」与「上下文构成 0+135+218=353」同屏并列，相差两个数量级。前者来自 `contextPressure`（provider 回报），后者来自 `contextBreakdown`（宿主按 4 字符/token 的启发式估算），本就不同源；但卡片没给口径说明，新手会以为构成加起来该等于占用。记为 **不确定**，不硬判缺陷。

证据：`evidence/p29-pk-widget.json`、`evidence/p29-pk-open.png`、`evidence/p31-*.json`、`evidence/p32-pk-fixtures.json`、`evidence/p33-pk-empty-compare.json`。

---

## 3. 复现要点

1. 派生 web profile（**不能**直接 `dsh plugin add`，否则 profile 缺 `@deepseek-ai/dsh-web-app`，boot 后 EXIT 1 且 stdout/stderr 全空）：
   `node bin.js --profile pet --from-default-profile web --dump-config`
2. 装 0.4.2：`node bin.js plugin --profile pet add dsh-token-pet@0.4.2`
3. 造夹具：`node make-sessions.mjs`（envelope `{seq,time}`；`turn/step` 从 1 起；`turn/end` 带 `data.reason:{kind:'completed'}`；**`request/header` 必须带 `reason:'initial'`**，否则会话被判 corrupt）
4. 起宿主：`node host.mjs <tag> pet`（端口 0，pid/url 落 `evidence/<tag>-host.local.json`）
5. 跑探针：`REVIEW_HOST_TAG=<tag> node probe-13-send.mjs`（发送拦截）、`probe-21-clean.mjs`（读数对照）、`probe-22-ledger-degrade.mjs`（账本与降级）、`probe-17-performance.mjs sess-pet-known 300000`（5 分钟）
6. pk-0.1.0：`node bin.js --profile petpk --from-default-profile web --dump-config` → `add link:<abs>` → `node host.mjs hpk petpk` → `probe-29/31/32/33`

完整步骤见 `REPRODUCE.md`。

---

## 4. 环境与版本

| 项 | 值 |
|---|---|
| 宿主 | `@deepseek-ai/dsh` 0.2.0-rc.2 |
| 浏览器 | Edge 155.0.4283.62（headless） |
| 驱动 | Playwright 1.64.1 |
| Node | v24.19.0 |
| 0.4.2 shasum | `3fe75ccdd3c5f129c20337663a3357584d19f006` |
| 0.4.2 integrity | `sha512-+8eAE8PmjITyPVTY7X0vjvCIc6ct5XfQA7g2kIrByrWNAEx6PVrsrnxQJX4j6yvuVEsC2g5rdUHjkAg4B0Df0Q==` |
| 0.4.2 tarball sha512 | `FBC78013…0740DFD1` |
| pk-0.1.0 client sha256 | `9044162168544499314F4469FAD68FFCBCC1777DE4AD7953A6A4EB8F9CBD3C5D` |

---

## 5. 限制与未测

- **不调用模型、不产生费用、不读真实余额/API Key**：提示词增强、完成提示音、成本估算、皮肤 ZIP 导入四条路径均未测。
- 趋势与会话排行在本宿主**不可得**（宿主缺 `listSnapshots`），插件如实降级，未伪造数据。
- 夹具是合成会话，不是真实模型对话；这不影响"读数是否与宿主一致"（那是投影层面的比对），但不是真实使用场景。
- 只在 dsh 0.2.0-rc.2 一版宿主上测；0.4.2 的 README 与 CHANGELOG 提到对旧宿主（缺 `listSnapshots`）的兼容分支，未回测。
- 发送按钮拦截是否在 DSH Desktop 或其它版本宿主上同样发生，未测。
- 未在真实用户 GUI（1241/2538/4756）上做任何操作。

---

## 6. 查重

- 悬赏站（GitHub issues）：`dsh-token-pet` / `pk7j7sqryy` / `Jimmy0123-ux` 均 **0 命中**；模糊词 `token-pet` 3 条命中（#645 dsh-roleplay-plugin、#569 vlln/whale-girl、#1481 @sunwindy/dsh-tool-baixiaochun），**无一同范围**。
- 评测站：12 页插件目录（133 条）+ 9 页评测列表（172 条）全量拉取本地扫描，`dsh-token-pet` / `pk7j7sqryy` / `Jimmy0123` / `token-pet` / `用量小宠物` 全部 **0 命中**。
- 判定：**首报**（两站同范围均无记录；该插件在评测站目录里尚未收录）。
