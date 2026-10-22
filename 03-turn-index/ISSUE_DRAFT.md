# [dsh-turn-index 0.1.1] 装得上、插件页也认，但右侧轮次索引面板在 DSH 0.2.0-rc.2 上从未出现（根因：`s.current` 恒为 undefined）

**给主 agent 的结论先行**：本轮 confirmed 缺陷 2 条（P1 面板不渲染 + P2 无宿主版本约束），另 1 条几何观察标 uncertain。可提交内容在下文「候选 A」「候选 B」。P1 有直接读数支撑，不是推断。

被测件：`dsh-turn-index@0.1.1`（Simon314620/dsh-turn-index，固定提交 `a324ecde201a219448c23c1c8bbd872bdab512b4`，发布件 `gitHead` 与该提交一致，MIT）。**本轮测的是 npm 发布件**。宿主 `@deepseek-ai/dsh 0.2.0-rc.2`，Windows 11 26200 + Edge 155.0.4283.62（Playwright 1.64.1 headless），隔离 `DSH_HOME` 与 profile，端口 0。

查重：`review.lingshulab.com` 逐页拉全量 9 页 / 170 条 review（`?q=` 不生效），全文匹配 `turn-index|Simon314620|轮次索引` 命中 0；`FuRongJun-1999/dsh-bounty` issues 分页取 1000 条 + search API 精确查询（`turnindex` 0 条、`Simon314620` 0 条，`turn-index` 53 条均为无关条目），命中 0。两站均查清，**首报**。

---

## 候选 A（状态：**confirmed，可提交**）P1 — 右侧轮次索引面板从不渲染

**操作与现象**：在隔离宿主里 `dsh plugin --profile web add dsh-turn-index@0.1.1`（exit 0），重启 `dsh web`，刷新页面。按 README「刷新页面即可在右侧看到索引栏」，此时应在右侧出现 232px 宽的轮次索引面板。实际：页面上**没有任何索引节点**，控制台无报错，插件页显示「已安装 1 / dsh-turn-index」。

**复现**

1. 隔离环境装插件：`dsh plugin --profile web add dsh-turn-index@0.1.1`，退出码 0；profile `package.json` 的 `dsh.profile.bundles` 追加 `dsh-turn-index`。
2. 起宿主：`dsh --profile web --host 127.0.0.1 --port 0 --no-open`，取 stdout 里的带 token URL。
3. 打开任意有 ≥1 轮的会话，等 5 秒。
4. 读 DOM：`document.querySelector('.dsh_ti_root')` → `null`；`document.querySelectorAll('[class*="dsh_ti"]').length` → `0`；`document.querySelector('[data-slot="shell.overlay"]').children.length` → `0`。
5. 对照组：在同一个 `shell.overlay` 座位里注册一个只读探针组件，探针正常渲染（`children.length=1`、DOM 可见）。

**读数**

```text
插件已加载的证据（说明不是安装/加载问题）：
  style[data-plugin-css="dsh-turn-index/styles"]        存在（injectCss 只在 apply() 里调用）
  boot manifest 条目  {"id":"dsh-turn-index","url":"plugins/??dsh-turn-index/client.js&rev=2a624d65dbe1"}
  插件页文案          「已安装 1 / dsh-turn-index / 对话轮次索引：…」

面板缺席的读数：
  .dsh_ti_root                              null
  [class*="dsh_ti"]                         0
  [data-slot="shell.overlay"].children      0（对照组探针在位时为 1）
  MutationObserver 5 秒 everPresent         false
  pageerror / console.error / unhandledrejection   全空
  [data-slot-error] / [data-factory-error]  无

根因探针（只读，未改被测源码）：
  useSessions(s => s.current)               undefined
  useSessions(s => s)                       {ids, byId, phase, projectionsBySession}   ← 没有 current
  ctx.sessions 自有键                        [rootCtx, searchResultLimit, list, manager,
                                            scopes, retainObservers, scopeDrops, closed]  ← 没有 current
  sessions.binding('session-cded7315-…')    "object"      （活绑定可取）
  sessions.binding(undefined)               "undefined"
```

**成因**：`lib/client.js:292` `var sessionId = useSessions(function (s) { return s.current; })`，配合 `lib/client.js:388` `if (sessionId === undefined) return null;`。0.2.0-rc.2 的 `SessionListState` 只有 `ids` / `byId` / `phase` / `projectionsBySession`，不提供 `current`，所以 `sessionId` 恒为 `undefined`，组件恒定 `return null` —— 不抛错，所以连错误面都没有。宿主自己的 `ui-layout` 是从 `byId` 推导当前会话的：`Object.values(state.byId).find(s => (s.retainedBy.mainView ?? 0) > 0)?.id`。

**影响**：插件的全部价值就是这个侧栏。装完等于什么都没发生，且无任何提示，用户会误以为是自己没重启或装错。

**建议**：改 `lib/client.js:292` 的取值方式（用 `byId` + `retainedBy.mainView`，或用会话作用域绑定）；同时在取不到当前会话时打一条 `console.warn`，让失败可诊断。

**证据**：`evidence/p01-boot.json`、`evidence/p04-watch.json`、`evidence/p05-run.txt`、`evidence/p07-scan.json`、`evidence/p09-run.txt`、`evidence/p09-probe.png`、`evidence/p10-boundary.json`、`evidence/p12-reload.json`、`evidence/p13-cold-restart.json`。

---

## 候选 B（状态：**confirmed，可提交**）P2 — 没有宿主版本约束，不兼容被推迟到运行时且不留痕

**操作与现象**：`package.json` 无 `engines` 字段，`peerDependencies` 只有可选的 `react`（`peerDependenciesMeta.react.optional = true`），没有任何宿主版本范围。README 的 Compatibility 表写「支持版本 DSH `0.1.0-rc.6`，最后验证 2026-08-14」。

**读数**

```text
package.json  engines                  （缺省）
package.json  peerDependencies         {"react":"^18.0.0 || ^19.0.0"}
package.json  peerDependenciesMeta     {"react":{"optional":true}}
安装结果                                exit 0，无任何版本告警
```

**影响**：同批次其他插件常用 peer 范围在安装阶段就被宿主拒装（用户当场知道不兼容）；本插件在 0.2.0-rc.2 上照装、照加载，失败推迟成「装好了但没界面」。这与候选 A 叠加，构成最差的失败形态：**静默**。

**建议**：加 `engines`（或在 peer 里声明宿主包版本范围），让不兼容在安装阶段可见。

**证据**：`evidence/npm-view-0.1.1.json`、`evidence/install-01-add.txt`、解包后的 `package.json`（`PROVENANCE.json` 里有其 sha256）。

---

## 候选 C（状态：**uncertain，不建议单独提交**）几何上可能与宿主自带轮次导轨、输入框重叠

宿主自带右侧轮次导轨实测 `rect={x:1393, y:443, w:28, h:62}`（7 个 mark）。按插件 CSS（`right:20px; width:232px`）推算面板 `rect` 为 `x=1188, top=80, bottom=1004`：与导轨相交 **1674 px²**，且 `bottom=1004` 超出 1000 高视口、纵向压到 `[data-composer-seat]`（`y=872, h=128`）。

**为什么不定性**：面板从未渲染，这是几何推算不是观测。要变成可提交条目，得先让面板能渲染（例如在能渲染的宿主版本上复测），再量实际相交。

**证据**：`evidence/p14-rail.json`、`evidence/p14-rail-1440.png`。

---

## 已验 / 未验 / 不确定 / 不做

**已验（confirmed）**

- 发布件安装成功（exit 0），profile bundles 追加插件（A1）。
- 宿主识别插件、boot manifest 下发 bundle、client bundle 在浏览器执行（`injectCss` 生效，样式标签存在且 reload 后仍只有 1 份）（A2/A3）。
- 面板在 1440 宽、五种会话（0/1/6/20/超长）、三次 reload、一次冷重启下**从不渲染**，且无任何错误面（B1/B1b/B1c）。
- 根因：`s.current` 恒 `undefined`，组件恒定 `return null`（B1d）。
- 0 / 1 / 6 / 20 轮与超长单轮下，会话数据齐备（`userRows` 分别 0/1/6/20/3）而面板恒缺席（F1/F2/F3/I1）。
- 刷新与冷重启后读数完全一致（`styleTags=1`、`panel=false`、`prefKeys=[]`）（G1/G2）。
- 快速来回滚动 10 次无崩溃、无 pageerror（I2）。
- 无宿主版本约束（J1）。

**未验（untested）**

- 滚动联动高亮是否跟手、是否漏/多高亮（C1/C2）：无条目可观测，11 步滚动每步 `activeItemKeys=[]`，**不能**写成「高亮不跟手」。
- 点击条目跳转（D1）：可点条目数 0。
- 折叠/展开状态保留（E1）：`.dsh_ti_toggle` 不存在，`data-collapsed` 为 `null`。
- 窄窗口 1000/760/600 下插件窄条降级是否遮挡（H2）：文档无横向溢出是宿主自身行为，插件侧无对象。
- 「加载更早的轮次」分页、轨迹视图兜底提示与「切回对话」、双语文案、details 左移避让：全部依赖面板存在。
- 0.1.0-rc.6 宿主上是否同样失效：本轮只有 rc.2，未回测。

**不确定（uncertain）**

- 候选 C 的几何重叠：按 CSS 推算，非实测。

**不做**

- 不改插件源码、不建仓库、不投稿、不碰用户正在使用的 GUI（1241/2538/4756）、不读真实密钥或真实会话。
- 不用 mock 页面替代真实宿主来「证明」未测到的三项。
