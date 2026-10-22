<!-- intake:v1 {"project":"bench","target":"Jimmy0123-ux/dsh-token-pet","form":"intake"} -->

# dsh-token-pet 0.4.2（npm，仓库 Jimmy0123-ux/dsh-token-pet）体验快测：玩偶浮层吃掉发送按钮的点击；浮窗"模型未知"在本宿主恒不消失

## 我做了什么

在隔离环境（独立 `DSH_HOME`/`HOME`/`APPDATA`/npm cache，官方 `dsh 0.2.0-rc.2`，`--host 127.0.0.1 --port 0 --no-open`）里装 `dsh-token-pet@0.4.2`，用自造的 5 条合成会话（已知 token 量）在 Edge headless 里打开，逐项比对插件读数与我从会话 envelope 手算的真值。

先说一件必须先讲清的事：**任务给的坐标 `pk7j7sqryy-ops/dsh-token-pet@b498347` 与 npm 上的 `dsh-token-pet@0.4.2` 不是同一个实现。** npm 全部 9 个版本的 `repository` 都指向 `Jimmy0123-ux/dsh-token-pet`；`pk7j7sqryy-ops` 那个仓库是 2026-08-15 的另一份实现（0.1.0、7 个文件、client bundle 21,551 字节，并自称带天气/日期模块），而 0.4.2 的 23 MB bundle 里「天气/weather/日期」命中 0 次。两者我都装了、都测了，下面的结论严格分开。

## 现象一：点发送没反应，宠物却在"打招呼"

打开带消耗的会话，在输入框打一句草稿，点右下角发送按钮——**没有任何反应**。与此同时，右侧那只玩偶的 aria-label 从「用量小宠物，空闲」变成「用量小宠物，打招呼」，统计浮窗自己弹了出来。

读数（1440×900）：

- 玩偶 HUD：`rect {x:1302, y:682, w:120, h:175}`
- 发送按钮：`rect {x:1202, y:828, w:34, h:34}`
- 两者相交 20×29 px
- `document.elementFromPoint(1219, 845)`（发送按钮正中心）返回的不是按钮，而是一个 `position:fixed`、`z-index:9998`、`pointer-events:auto` 的**透明** `div`，`rect {x:1174, y:665, w:248, h:217}`，React 链为 `div > TokenPetWindow > ErrorBoundary > RootEntry`
- 真实鼠标点击后：`draft` 仍是 `T-1440`、`petLabel` 变成「用量小宠物，打招呼」、`panelOpen: true`
- 同一按钮改用 JS `b.click()` 绕过命中测试：草稿被清空、对话行数 8 → 13，发送成功

四档宽度全部复现：

| 宽度 | 玩偶 HUD | 发送按钮 | 相交 | 发送中心命中 |
|---|---|---|---|---|
| 1440 | (1302,682) 120×175 | (1202,828) 34×34 | 20×29 | TokenPetWindow |
| 1000 | (862,682) 120×175 | (839,828) 34×34 | 23×29 | TokenPetWindow |
| 760 | (622,682) 120×175 | (695,828) 34×34 | 47×29 | TokenPetWindow |
| 600 | (462,682) 120×175 | (535,828) 34×34 | 48×29 | TokenPetWindow |

网格采样量化：视口内 550 个 10 px 采样点落在插件浮层上，其中 **346 个（63%）位于可见玩偶之外**——用户点到"空气"也会被吞。

## 现象二：浮窗顶部恒显示「模型未知」

账本页与模型页都正确写着 `fixture-provider · fixture-model-a`，但"当前上下文"卡片那行固定是「模型未知」。

定位：`buildPetView` 里 `model = timeline?.model`，`timeline` 来自 `useProjection('contextTimeline')`。本宿主实际只注册了四个投影：

```
contextPressure    => dsh-token-meter
contextBreakdown   => dsh-token-meter
tokenUsage         => dsh-token-meter
sessionStats       => dsh-session-stats
contextTimeline    => (NOT PROVIDED BY HOST)
todayUsageBuckets  => (NOT PROVIDED BY HOST)
```

所以 `model`/`provider` 恒 `undefined` → 恒渲染「模型未知」。同一原因也让趋势卡在这台机器上拿不到当日数据。这一条**看起来像 bug，实际是宿主少了一个投影**；但插件没有回退路径，用户无法分辨。

## 现象三：索引状态自相矛盾

点「首次建立索引」→ `POST /token-pet/index/build` 返回 200，UI 变「索引已就绪 · 0 个已索引」，磁盘 `session-usage-index.json` 只有 27 字节。同一屏里，「最耗 Token 的会话」仍写「首次建立历史索引后显示」，「本日用量趋势」写「暂无趋势数据」；而设置页明写「维护状态：当前宿主的会话持久化接口不支持趋势索引 / 失败：`sessionPersistence does not expose listSnapshots`」。

降级本身是插件有意设计（CHANGELOG 0.3.1 有记录），**不是缺陷**；但「已就绪」与「首次建立后显示」并置会让新手以为没建完、反复点。

## 复现

见 `REPRODUCE.md`。最小路径：

```powershell
# 1) 派生 web profile（必须，否则 profile 缺 @deepseek-ai/dsh-web-app，boot EXIT 1 且 stdout/stderr 全空）
node bin.js --profile pet --from-default-profile web --dump-config
node bin.js plugin --profile pet add dsh-token-pet@0.4.2
# 2) 造合成会话
node make-sessions.mjs
# 3) 起宿主（端口 0）
node host.mjs hp3 pet
# 4) 发送拦截
$env:REVIEW_HOST_TAG='hp3'; node probe-13-send.mjs
```

## 已验证（confirmed）

- **C12 玩偶浮层拦截发送按钮**：`elementFromPoint` + 真实点击效果 + 四档宽度 + JS 直点对照，四项证据齐全。
- **C14 浮窗「模型未知」**：本宿主缺 `contextTimeline` 投影，四个投影实测 + 插件取值为唯一来源。
- **C16 索引状态矛盾**：200 + 27 字节空索引 + 三处并置文案 + 设置页明写不支持。
- **C02 玩偶渲染无 404**：逐帧 WebP 内嵌 data: URI（naturalWidth 11520×540，1.85 MB base64），页面 img 全部 `complete:true`，无任何图片请求。
- **C03/C04/C05 读数与真值一致**：`50000/100000=50%`、`95200`、`2,223,600`、缓存读 `942,000`、缓存写 `4,000` 全部逐位相符。
- **C07 断网降级**：`/token-pet/**` 全 abort 后玩偶与 chip 仍在，浮窗逐块显示「读取失败 / 重试」，连点刷新不转圈、无未捕获异常。
- **C08/C09 性能**：219 帧 mean 5.56 ms、p95 5.70 ms、>33 ms 帧 0；306 秒采样 DOM 节点恒定 795、堆 96.9→90.7 MB 回落。
- **C10 空会话**：「上下文 –%」+「— | 上下文窗口读取中」，不假报 0%。
- **C15 跨零点**：「10月21日 23:50」与「00:10」分属两天。
- **C18 pk-0.1.0 天气降级**：`POST /token-pet/weather` 405 → 卡片「🌡️ 天气不可用」，不崩不空白。
- **C01 身份不一致**：npm 9 个版本 repository 全指向 `Jimmy0123-ux`。

## 未验（untested）

- 提示词增强 / 完成提示音 / 成本估算 / 皮肤 ZIP 导入：需要真实模型调用、真实回复边界或用户提供的 ZIP，按边界要求不做。
- 趋势与会话排行在本宿主不可得（缺 `listSnapshots`），只验证了降级文案。
- 发送按钮拦截是否在 DSH Desktop 或其它版本宿主同样发生。

## 不确定（uncertain）

- **C24**：pk-0.1.0 卡片里「上下文占用 ~50.1K」与「上下文构成 0+135+218=353」同屏并列，相差两个数量级。前者是 `contextPressure`（provider 回报），后者是 `contextBreakdown`（宿主按 4 字符/token 启发式估算），本就不同源；卡片没给口径说明，新手会以为构成相加应等于占用。未找到文档是否解释，不硬判。

## 不做

- 不调用任何模型、不产生费用、不读真实余额/API Key/真实会话。
- 不碰用户正在使用的 GUI（1241/2538/4756）。
- 不修改插件源码、不为通过而打补丁。

## 建议

1. **优先级最高**：把 portal 根容器改成 `pointer-events: none`，只给可见玩偶与卡片子节点 `pointer-events: auto`；或把 248×217 容器收紧到可见区域。这是会直接打断用户"发消息"这一核心动作的问题。
2. `contextTimeline` 缺失时回退到账本 `cumulative.byModel` 取模型名，或把文案改成「本宿主不提供模型信息」，别让用户以为是插件读不出来。
3. 宿主不支持趋势索引时，把「首次建立索引」按钮与「索引已就绪」文案统一成一种口径，避免同屏两种说法。
4. README 里的安装说明建议补一句：`dsh plugin add` 建出的新 profile 缺 `@deepseek-ai/dsh-web-app`，需先 `--from-default-profile web` 派生——这会直接省掉一批"装了看不见"的困惑。

## 测试环境

| 项 | 值 |
|---|---|
| 宿主 | `@deepseek-ai/dsh` 0.2.0-rc.2 |
| 浏览器 | Edge 155.0.4283.62（headless） |
| 驱动 | Playwright 1.64.1 |
| Node | v24.19.0 |
| 0.4.2 shasum | `3fe75ccdd3c5f129c20337663a3357584d19f006` |
| 0.4.2 integrity | `sha512-+8eAE8PmjITyPVTY7X0vjvCIc6ct5XfQA7g2kIrByrWNAEx6PVrsrnxQJX4j6yvuVEsC2g5rdUHjkAg4B0Df0Q==` |
| pk-0.1.0 client sha256 | `9044162168544499314F4469FAD68FFCBCC1777DE4AD7953A6A4EB8F9CBD3C5D` |
| 隔离 | `DSH_HOME`/`HOME`/`USERPROFILE`/`APPDATA`/`LOCALAPPDATA`/`XDG_*`/npm cache 全指向任务目录；端口 0 |
| 查重 | 悬赏站精确词 0 命中；评测站 12 页插件目录 + 9 页评测列表全量扫描 0 命中 |

## 证据

- `evidence/p12-hit-test.json` / `p12-click-effect.json` / `p13-send-interception.json` / `p13-after-send-click.png` / `p14-send-intercept-widths.json` / `p25-overlay-map.json`（现象一）
- `evidence/p26-projections.txt` / `p21-sess-pet-unknown.png` / `p22-ledger-and-degrade.json`（现象二）
- `evidence/p22-after-build.png` / `p16-endpoints.json`（现象三）
- `evidence/p21-clean-readings.json` / `p22-expanded.png`（读数对照）
- `evidence/p17-performance.json`（性能与泄漏）
- `evidence/p23-offline.json`（断网降级）
- `evidence/p29-pk-widget.json` / `p29-pk-open.png` / `p31-*.json` / `p32-pk-fixtures.json` / `p33-pk-empty-compare.json`（pk-0.1.0）
- `evidence/step01-npm-view.json` / `npm-versions-repository.txt` / `pk0.1.0-manifest.txt`（身份核对）