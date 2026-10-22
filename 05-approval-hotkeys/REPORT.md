# dsh-approval-hotkeys 0.2.0 实测报告（DSH 0.2.0-rc.2 / Edge 155）

被测对象：`SiriLee/dsh-approval-hotkeys`，npm 发布件 `dsh-approval-hotkeys@0.2.0`，源码对照提交 `d6b16a1229a8dc23d83404e06173af067847c106`，MIT。
测试形态：官方 `@deepseek-ai/dsh` 0.2.0-rc.2 隔离宿主 + Edge headless（Playwright）+ 本机假 LLM 端点，制造**真实**的审批面板。
读数汇总：30 项检查中 23 项 confirmed、2 项 uncertain、5 项 untested；其中 2 项属缺陷。逐条读数与证据文件名见 `RESULTS.json`，复跑步骤见 `REPRODUCE.md`。

---

## 1. 发现与影响

### 1.1 手快连按 Enter 会批准还没看的下一项（主要发现）

插件对 Enter 没有任何节流，也没有「新面板出现需要重新确认」的门槛。当用户的按键间隔与宿主渲染下一条审批的延迟（实测 400–625ms）正好重叠时，多出来的那一次 Enter 会落在刚渲染出来、用户尚未阅读的新面板上。

读数（`evidence/run7-11-multi-enter.json`）：

| 场景 | 按键间隔 | Enter 次数 | 实际发生 click | 结束时剩余面板 |
|---|---|---|---|---|
| P1 单击 | — | 1 | 1（允许一次） | approval:3（625ms 后才出现） |
| P2 双击 | 313ms | 2 | 1 | approval:6（未被批） |
| P3 双击 | 140ms | 2 | 1 | approval:9（未被批） |
| P4 五连击 | 200ms | 5 | 2 | approval:13（approval:12 被批掉） |

会话日志侧证：`evidence/session-session-5cbc5e08-e4d0-4061-9f34-1f9cb540f175.jsonl` 中同一次会话的 `approval/decided` 序列包含连续的 `allowed-once`。

影响：审批被批准一次即不可撤销。用户的本意是「批准眼前这一条」，实际却授权了自己还没读过的下一条工具调用。风险随审批密度上升（一条模型回复里带多个越权调用时，宿主会逐个弹出面板）。

**绕过办法**：等面板渲染完再按，一次一条；批完看一眼再决定是否继续按。不要在面板刚消失、屏幕还空着的时候继续敲 Enter。

### 1.2 同一 tick 内对同一按钮发出两次 click（次要）

用合成事件在同一 tick 连发两个 Enter，记录到两条 `click:允许一次`（`evidence/run5-08-guards.json` F_doubleEnter_sameTick，两次 bubble 的 `defaultPrevented` 都是 true，按钮当时都还是 `disabled=false`）。

没有造成二次批准，原因是宿主的按钮在结算后立即 disabled、审批也随即 settled。也就是说**安全性依赖下游状态而不是插件自身幂等**：插件每次都照着「动作行最后一个按钮」点下去，没有先确认这个面板是否已经被自己点过。

影响：当前宿主上不产生可见后果，但它把「不会重复批准」这件事外包给了宿主实现。宿主若改变结算顺序（例如先移除 DOM 再置 disabled），同一路径就可能变成真实的二次批准。

### 1.3 定位与实现相符，主要行为都成立

正面读数（均为 confirmed，明细见 `RESULTS.json`）：

- 面板在场、焦点不在面板内时，Enter 点「允许一次」、Esc 点「拒绝」，间隔 1–2ms，面板 2ms 内消失（`run5-07-matrix.json` T2/T3、`run4-06-timing.json`）。
- 归因干净：同一组操作在无插件的对照 profile 上完全无反应（`run6-09-control.json`、`run8-12-attribution.json`），装插件后每次都前进（`run7-12-attribution.json` 2→3→4）。
- 无面板时 Enter/Esc 都不被拦（`defaultPrevented` 全为 false，`run5-08` A/G，`run7-12` 与 `run8-12` 四段事件逐条相同）。
- 组合键 Ctrl/Shift/Alt+Enter 不批准（`run5-07` T7、`run9-15` T4）。
- 焦点在 input 时 Enter/Esc 都不触发（`run5-08` B、`run9-16` A/B）；焦点在页面按钮时走浏览器原生激活，插件不代批（`run5-08` C）。
- IME 组字态（`isComposing=true`、`keyCode=229`）与 `repeat=true` 的 Enter 都被拦住（`run5-08` D/E）。
- 面板中途消失后按键无报错、无悬挂监听（`run5-07` T8、`run9-15` T6）。
- 宿主快捷键 Ctrl+Alt+N 在面板在场时照常换会话，插件未拦截（`run7-13` hostShortcut）。

### 1.4 一个前提被推翻：不存在「多面板焦点争夺」

任务书预设「多个待审批项并存时，插件要挑当前焦点项」。实测宿主是**串行**渲染的：任意时刻 `[data-approval-key]` 的数量都是 1（`run7-10` 采样 45 秒、`run4-05c` T3/T4 都是 count=1），下一条要等上一条答复完 400–625ms 才出现。因此「插件在多面板间跳焦点」这个检查项没有对象，焦点标记（`aria-selected` / `data-active`）也无从谈起——面板存在期间 `focusWithin` 恒为 false。

这条不影响上述缺陷的成立，但改变了它的表述：不是「跳到了错误的面板」，而是「多出来的那一次按键被刚出现的面板接住了」。

---

## 2. 测试步骤（可复跑）

1. 建隔离 profile：`dsh --profile ah --from-default-profile web`，再 `dsh plugin --profile ah add dsh-approval-hotkeys@0.2.0`；另建 `ah0` 作无插件对照。
2. 起本机假 LLM（`fake-llm.mjs`，绑 127.0.0.1:0，返回 Anthropic Messages SSE），只有用户消息含 `TRIGGER` 才回一次带 `justification` 的越权 `pwsh` 调用。
3. 起隔离宿主（`host.mjs`）：`DSH_HOME/HOME/USERPROFILE/APPDATA/LOCALAPPDATA/npm cache` 全部指向任务目录，参数 `--host 127.0.0.1 --port 0 --no-open`，`DEEPSEEK_BASE_URL` 指假端点，`DSH_PERMISSION_MODE=workspace-write`。
4. Playwright + Edge headless 打开宿主 URL，`context.route` 只放行 127.0.0.1。
5. 在 composer 里输入含 `TRIGGER` 的消息并发送 → 宿主自己产生真实审批面板（`[data-approval-key]` 出现，耗时 250–560ms）。
6. 逐场景按键，采集：document 上 capture/bubble 两段 keydown 的 `defaultPrevented`、`click` 事件的按钮文案与 disabled、面板 key 序列、`activeElement`、页面错误与控制台。
7. 有/无插件各跑一遍同样脚本，用对照确认行为归属。

命令细节与前置条件见 `REPRODUCE.md`。

---

## 3. 环境

| 项 | 值 |
|---|---|
| 宿主 | `@deepseek-ai/dsh` 0.2.0-rc.2 |
| 浏览器 | Edge 155.0.4283.62（headless，Playwright 驱动） |
| 驱动 | Playwright（只读引用既有安装） |
| 运行时 | Node v24.19.0 / Windows 11 build 26200 |
| 模型 | 本机假端点，未调用真实模型、未读真实密钥 |
| 隔离 | 全部配置与会话落在任务目录内 |
| 宿主实例 | run2–run9 共 8 个，含 2 个无插件对照（run6、run8） |

---

## 4. 限制与未测

- **未测提问面板与计划审查面板**（`[data-question-key]` / `[data-plan-review-key]`）。本次所有运行只产生审批面板；bundle 里能搜到这两个选择器，但那是代码事实，不是行为读数。不能把审批面板的结论外推过去。
- **未测真实中文输入法**。只用了合成 `KeyboardEvent`（`isComposing=true` / `keyCode=229`）。合成事件能证明代码路径按标志位拦住，不能证明真实输入法下的时序。
- **未测真实长按的 OS 级 repeat**。`keyboard.down` 保持 1.5 秒只产生 1 次 click，Playwright 不产生 repeat 事件；`repeat=true` 是合成的。
- **未测运行时降级分支**。本宿主始终能报出 `pendingInteraction`，插件的「退回 DOM 第一个可见面板」这条路从未被走到。
- **Esc「暂停键盘审阅」不确定**。`run5-07` T3 里 Esc 直接拒绝并关面板；`run7-13` 的同类步骤里 Esc 却没有产生 click、随后的 Enter 才批准。两次差异在焦点落点（BODY vs DIV）。README.zh.md 第 68 行明确写「无面板时 Esc 不拦截（不做暂停/停止）」，即产品没有这个功能，故既不能判缺陷，也不能判通过。
- **下一面板延迟的成因不确定**。400–625ms 这段延迟由宿主产生还是被插件影响，本次无法区分。
- **审批面板在场时 composer 不可聚焦**（`focus()` 返回 null，可编辑元素只剩侧栏那个 4px 宽搜索框）。所以「正在打字时面板弹出」这个场景在本宿主里需要焦点先被挪走才可能出现——这是宿主结构，不是插件缺陷，但会限制相关结论的适用范围。
- 未用真实账号、未连真实模型服务，因此「多条越权调用的真实密度」是假端点造出来的，实际用户遇到的频率取决于自己的使用方式。

---

## 5. 给维护者的建议

1. **给 Enter 加一层「新面板确认」**：记录上一次已应答的面板 key（`data-approval-key`）与时间戳，新 key 出现后的极短窗口（例如 150–250ms）内忽略 Enter，或要求第二次确认。这一条能直接消掉 1.1。
2. **让点击自身幂等**：在点击前把面板 key 记入一个「已应答」集合，同一 key 只点一次；不要依赖按钮 `disabled` 的结算时序。这一条能消掉 1.2。
3. 若采纳 1、2，建议同时在 README 的「守卫」一节补一行说明，因为当前文档只描述了「不拦输入框、不拦组合键、不拦 IME、无面板不拦」，未涉及「连按」。
4. 提问面板与计划审查面板的行为目前只有代码与文档，缺运行时读数；建议作者补一个能构造这两类面板的自检脚本，外部复测才有对象。
