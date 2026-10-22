# dsh-turn-index 0.1.1 技术报告（Simon314620/dsh-turn-index @ a324ecd）

被测件：npm 发布件 `dsh-turn-index@0.1.1`，作者 Simon314620，MIT。发布件 `gitHead` 字段与任务指定的固定提交 `a324ecde201a219448c23c1c8bbd872bdab512b4` 一致，因此发布件即待测代码，本轮**测的是发布包**。宿主 `@deepseek-ai/dsh 0.2.0-rc.2`，隔离 `DSH_HOME`，端口 0，Edge 155.0.4283.62 + Playwright 1.64.1 headless。

## 一句话结论

插件装得上、宿主也认它、浏览器里连它的样式都注入成功了 —— 但**右侧轮次索引面板在本次全部场景里从未出现**。失效点不在安装、不在加载、也不在报错，而在它取「当前是哪个会话」的那一行写法上。

## 发现与影响

### 1（P1）面板从不渲染：`useSessions(s => s.current)` 在 0.2.0-rc.2 上恒为 `undefined`

**现象**：1440 宽视口下 `document.querySelector('.dsh_ti_root')` 为 `null`，`document.querySelectorAll('[class*="dsh_ti"]').length` 为 0。轮次索引这条侧栏在整个页面里没有任何节点。

**排除项**（都不是原因）：

| 假设 | 读数 | 结论 |
|---|---|---|
| 没装上 | 插件页「已安装 1」下有 dsh-turn-index；profile 的 `dsh.profile.bundles` 已追加 | 装上了 |
| 宿主没下发 bundle | boot manifest 里有 `{"id":"dsh-turn-index","url":"plugins/??dsh-turn-index/client.js&rev=2a624d65dbe1"}` | 下发了 |
| bundle 没执行 | 页面存在 `style[data-plugin-css="dsh-turn-index/styles"]`，且 `injectCss()` 只在 `apply()` 里调用 | 执行了 |
| 组件抛错被边界吞掉 | init 脚本捕获 `window error` / `unhandledrejection` / `console.error` 全空；DOM 无 `[data-slot-error]`、无 `[data-factory-error]` | 没抛错 |
| 渲染慢/竞态 | 21 次 250ms 采样 + MutationObserver 连看 5 秒，`everPresent=false` | 不是竞态 |
| `shell.overlay` 座位本身不可用 | 本轮另装的只读探针在同一个座位里正常渲染（`overlaySeatChildren=1`，探针 DOM 可见） | 座位是好的 |

**成因（探针读数，未改动被测源码）**：另写一个只读探针包 `dsh-probe-ti`，同样用 `ctx.slots.inject('shell.overlay', …)` 注册，在组件里读宿主真实 API：

```text
useSessions(s => s.current)  -> undefined        (currentIsUndefined = true)
useSessions(s => s)          -> { ids, byId, phase, projectionsBySession }   // 没有 current 字段
ctx.sessions 自有键           -> [rootCtx, searchResultLimit, list, manager, scopes,
                                 retainObservers, scopeDrops, closed]        // 也没有 current
sessions.binding('session-cded7315-…') -> "object"    (活绑定可取)
sessions.binding(undefined)            -> "undefined"
```

插件 `lib/client.js:292` 写的是 `var sessionId = useSessions(function (s) { return s.current; })`；`lib/client.js:388` 紧接着 `if (sessionId === undefined) return null;`。第 388 行这个早退本身是合理的防御写法，但它和第 292 行的取值方式连在一起，在这版宿主上就变成**恒定 return null**：`SessionListState` 只提供 `ids` / `byId` / `phase` / `projectionsBySession`，当前会话的身份要自己从 `byId` 里按 `retainedBy.mainView > 0` 之类的方式推导（宿主自己的 `ui-layout` 就是这么写的：`Object.values(state.byId).find(s => (s.retainedBy.mainView ?? 0) > 0)?.id`）。

**影响**：这条插件的全部价值就是这个侧栏。面板不出现，等于用户装完之后屏幕上什么也没变，而控制台没有任何报错、插件页显示「已安装」——用户会以为是自己装错了或忘了重启。

**为什么装得上却跑不出来**：`package.json` 里没有 `engines` 字段，`peerDependencies` 只有一个**可选**的 `react`，没有任何宿主版本约束。README 的 Compatibility 表写的是「支持版本 DSH 0.1.0-rc.6，最后验证 2026-08-14」。所以在 0.2.0-rc.2 上它会照常装、照常让宿主加载，失败被推迟到运行时且不留痕迹。

### 2（P2）装的时候没有任何版本提示

见上。同批次其他插件普遍靠 peer 范围在安装阶段就被拒（另有报告记录过 `peer ^0.1.0-rc.6` 拒装），而本插件把不兼容推迟成「装好了但没界面」。建议加 `engines` 或把宿主版本写进 peer 范围，让不兼容在安装阶段可见。

### 3（观察，未定性）几何上可能与宿主自带轮次导轨、输入框重叠

宿主自带一条右侧轮次导轨，实测 `rect={x:1393, y:443, w:28, h:62}`（7 个 mark）。按插件 CSS 推算它的面板位置为 `right:20px; width:232px`，即 `x=1188, top=80, bottom=1004`：与导轨相交面积 **1674 px²**，且 `bottom=1004` 超出 1000 高视口、纵向压到 `[data-composer-seat]`（`y=872, h=128`）。

**这条只标 uncertain。** 面板从未真正渲染，以上是按 CSS 与实测矩形做的几何推算，不是观测到的遮挡。要定成缺陷，需要先让面板能渲染出来再量。

窄窗口三档（1000 / 760 / 600）实测 `docOverflowX=false`、`scrollWidth` 分别 1000 / 760 / 600，但这是宿主自身行为（此时它的导轨 `rect` 已是 0×0 收起），插件声明的「≤1024px 收成 44px 窄条」因面板不存在而无法验证。

## 边界与稳定性读数

| 场景 | 会话侧读数 | 面板侧读数 |
|---|---|---|
| 0 轮空会话 | `userRows=0`，`scrollHeight=960` | 面板不存在，空态文案 `null`，无错误 |
| 1 轮 | `userRows=1` | 面板不存在，条目 0 |
| 6 轮 | `userRows=6`，`scrollHeight=2644` | 面板不存在，条目 0 |
| 20 轮 | `userRows=20`，`scrollHeight=6054` | 面板不存在，条目 0 |
| 超长单轮（4000+ 字） | `userRows=3`，`scrollHeight=3733` | 面板不存在，无额外错误 |
| 连续 reload ×3 | `userRows=6`、`scrollHeight=2644` 三次一致 | `styleTags=1`（无重复注入）、面板三次都不存在 |
| 冷重启（换新实例新端口） | 回放后 `userRows=6` | 面板仍不存在 |
| 分段滚动 11 步（0→1720） | `scrollTop` 每步精确命中目标 | 每步 `activeItemKeys=[]` |
| 快速来回滚动 10 次 | 全部执行，`scrollHeight` 预算内 | 无崩溃、`pageerror=[]` |
| localStorage | 无任何 `dsh-turn-index` 前缀键 | 与面板从未渲染一致 |

分段滚动每步 `scrollTop` 都精确落在目标值，说明聊天流滚动容器本身完全正常，排除了「读不到滚动容器」这一替代解释；面板缺席也因此不是滚动相关问题。

## 环境与限制

- 宿主 `@deepseek-ai/dsh 0.2.0-rc.2`，`--profile web --host 127.0.0.1 --port 0 --no-open`，四次自建实例（2122 / 14959 / 7924 / 9772）全部只用回环地址，未触碰用户正在使用的 GUI（1241 / 2538 / 4756）。
- `DSH_HOME` / `HOME` / `USERPROFILE` / `APPDATA` / `LOCALAPPDATA` / npm cache 全部指向本任务目录。
- 会话样本用官方 `@deepseek-ai/dsh-session-persistence-jsonl` 合成（手写文本 + `assistant/message` 事件），**不含真实模型生成过程**，不作为真实模型输出引用。
- 探针包 `dsh-probe-ti` 只装在隔离 profile 里，只读宿主 API，不改被测源码、不参与被测判定。

## 没测到的部分

- 0.1.0-rc.6 宿主上是否同样失效。README 只声明支持 rc.6，本轮只有 rc.2，未回测旧宿主 —— 因此本报告的主张严格限定为「在 0.2.0-rc.2 上不渲染」。
- 滚动联动高亮、点击跳转、折叠状态保留、加载更早轮次、轨迹视图兜底提示、双语文案切换、details 左移避让：这些都需要面板存在才有对象可观测。本轮一律记 `untested`，**没有**写成「功能坏了」。
- 真实模型轮次下的表现（同上，样本为合成会话）。

## 建议（给维护者）

1. `lib/client.js:292` 处改用 `s.byId` 与 `retainedBy.mainView` 推导当前会话 id，或用宿主声明的会话作用域（`session` scope）取当前绑定，不要依赖 `s.current`。
2. 给 `package.json` 加宿主版本约束（`engines` 或 peer 范围），让不兼容在安装阶段就可见，不要把失败藏到运行时。
3. 面板渲染失败时至少留一个可诊断信号（比如 `console.warn` 一次「找不到当前会话」），否则用户与维护者都只能看到「什么都没发生」。
