<!-- intake:v1 {"project":"bench","target":"SiriLee/dsh-approval-hotkeys","form":"intake"} -->
# [intake 输入] dsh-approval-hotkeys 0.2.0：连按 Enter 会批准尚未阅读的下一条审批；同一 tick 内对同一按钮点两次

一条模型回复里带多个待审批工具调用时，宿主逐个弹出审批面板（上一个答复后 400–625ms 才出现下一个）。在 dsh 0.2.0-rc.2 上装 dsh-approval-hotkeys@0.2.0 后连按 Enter，多出来的那一次按键会落在刚渲染、用户还没读的新面板上，把下一条审批直接批准掉；另外，同一 tick 内连发两个 Enter 会对同一个「允许一次」按钮发出两次 click 调用，目前靠宿主按钮 disabled 兜底才没造成二次批准。

## 复现

1. 建隔离 profile：`dsh --profile ah --from-default-profile web`，再 `dsh plugin --profile ah add dsh-approval-hotkeys@0.2.0`。
2. 起本机假 LLM 端点（返回 Anthropic Messages SSE），只在用户消息含 `TRIGGER` 时返回工具调用；工具名 `pwsh`，input 带 `sandbox_permissions: "danger-full-access"` 与非空 `justification`。用 `FAKE_TOOL_COUNT=3` 让一条回复里放三个待审批调用。全程 `DEEPSEEK_API_KEY` 用本地假值，不连真实模型。
3. 起隔离宿主：`--profile ah --host 127.0.0.1 --port 0 --no-open`，`DEEPSEEK_BASE_URL` 指假端点，`DSH_PERMISSION_MODE=workspace-write`。
4. Playwright + Edge headless 打开宿主 URL，在 composer 输入含 `TRIGGER` 的消息并发送。等 `[data-approval-key]` 出现（实测 250–560ms）。
5. **连按场景**：焦点停在 body（不点面板），以 200ms 间隔连按 5 次 Enter。
   - 观察点：document 上 `click` 事件里「允许一次」被点了几次；结束时还剩哪个 `[data-approval-key]`。
   - 实测：第 1 次批准 approval:11，第 2–4 次落在面板空窗（无 click），第 5 次批准了刚出现的 approval:12，最后剩 approval:13 —— 共 2 次批准。
   - 同样结构以 313ms / 140ms 间隔双击时只批准 1 次（多出来的那次落在空窗里），所以是否误批取决于按键间隔与新面板出现时刻是否重叠。
6. **同 tick 双 Enter 场景**：面板在场时用 `document.dispatchEvent(new KeyboardEvent('keydown', {key:'Enter', bubbles:true}))` 在同一 tick 连发两次。
   - 实测：记录到两条 `click:允许一次`（此时按钮 `disabled=false`），两次 keydown 的 bubble 段 `defaultPrevented` 均为 true。

对照（无插件 profile `ah0`，同一套脚本）：同样条件下 Enter 与 Esc 都不产生任何 click，面板一直挂在那里等人工应答。

## 测试环境

- 宿主 `@deepseek-ai/dsh` 0.2.0-rc.2；`--host 127.0.0.1 --port 0 --no-open`。
- 浏览器 Edge 155.0.4283.62，headless，Playwright 驱动，独立 context，只放行 127.0.0.1。
- Windows 11 build 26200，Node v24.19.0。
- 被测件：npm 发布件 `dsh-approval-hotkeys@0.2.0`（integrity `sha512-rJNu0SAET1ltdCT6MjWtlpLbjfzv9skPFc/PaSyjfTduI7Ul2Lsyym8ThUVChs7m2aFVCdhj48SfJZozNoEdZg==`，tarball SHA1 本地复算 `cf55c7e8cb043b1957ba39af6888401985e12020`，MIT）；源码对照提交 `d6b16a1229a8dc23d83404e06173af067847c106`。
- DSH_HOME / HOME / USERPROFILE / APPDATA / LOCALAPPDATA / npm cache 全部指向任务目录内的隔离位置；未读写日常配置、真实会话或真实密钥。

## 证据

- `run7-11-multi-enter.json` — 四组连按（1/2/2/5 次）的 keydown、click、面板 key 时间线；P4 的两次 click 与 approval:12 出现时刻可逐条对照。
- `session-session-5cbc5e08-e4d0-4061-9f34-1f9cb540f175.jsonl` — 同一次会话的 `approval/asked` 与 `approval/decided` 记录，可核对 outcome 序列。
- `run5-08-guards.json` — F_doubleEnter_sameTick：同一 tick 两个 Enter → 两条 `click:允许一次`；同文件还有 input 焦点、IME 组字、repeat、组合键四组守卫读数。
- `run5-07-matrix.json` — T2/T3 单次 Enter/Esc 的正常路径与 `defaultPrevented` 读数；T7 组合键、T8 面板消失。
- `run7-12-attribution.json` / `run8-12-attribution.json` — 装插件与不装插件的同条件对照（approval 前进 vs 停在原地）。
- `run9-16-focus-burst.json` — 250ms 间隔三连击只批准 1 次的另一组读数。
- `run9-16-C-burst.png`、`run5-08-F-double-enter.png` — 对应截图。
- `run7-10-multi.json` — 采样 45 秒只见 1 个 `[data-approval-key]`，证明面板是串行渲染。

## 已验

- 面板在场、焦点在 body 时，Enter 点击「允许一次」（间隔 1–2ms，面板 2ms 内消失），Esc 点击「拒绝」。
- 无插件对照下同样条件 Enter/Esc 都不产生 click，归因成立。
- 无面板时 Enter/Esc 的 `defaultPrevented` 全为 false，插件不拦键。
- 组合键 Ctrl/Shift/Alt+Enter 不批准；焦点在 input 时 Enter/Esc 都不触发；焦点在页面按钮时走浏览器原生激活。
- IME `isComposing=true`/`keyCode=229` 与 `repeat=true` 的 Enter 都被拦住。
- 面板中途消失后按键无报错、无悬挂监听；宿主快捷键 Ctrl+Alt+N 在面板在场时照常换会话。
- 连按 Enter 在特定时间窗内会批准尚未阅读的下一条（200ms×5 组实测 2 次批准）。
- 同一 tick 双 Enter 产生两条 click 调用（未造成二次批准，靠宿主 disabled 兜底）。
- 两站查重：评测站全量 172 条评测 + 133 个插件、悬赏仓四个关键词，均无同范围重复，属首报。

## 未验

- 提问面板 `[data-question-key]` 与计划审查面板 `[data-plan-review-key]`：本次只造出审批面板，这两类面板从未在页面上出现，不能把审批面板的结论外推。
- 真实中文输入法：只用合成 `KeyboardEvent` 模拟 `isComposing`，没有用真实 IME 输入并确认候选词。
- 真实长按产生的 OS 级 repeat：Playwright 的 `keyboard.down` 不产生 repeat 事件。
- 插件「运行时读不到 `pendingInteraction` 时退回 DOM 第一个可见面板」的降级分支：本宿主始终能报出待应答请求，这条路径没被走到。

## 不确定

- Esc 是否构成「暂停键盘审阅」：`run5-07` T3 里 Esc 直接拒绝并关面板；`run7-13` 的同类步骤里 Esc 的 `defaultPrevented=false`、没有 click，随后的 Enter 才批准。两次差异在焦点落点（BODY vs DIV），无法判定是没生效还是被暂停。README.zh.md 第 68 行写「无面板时 Esc 不拦截（不做暂停/停止）」，说明产品本身没有这个功能。
- 下一条审批面板 400–625ms 的延迟是宿主产生还是被插件影响，未定位成因。

## 不做

- 不修改插件源码，不自建兜底实现替主流程通过。
- 不投稿、不上传、不新建公开仓库，本文件仅为本地草稿。
- 不读真实密钥、不连真实模型、不使用真实第三方账号。
- 不把「无面板时 Enter 被 `preventDefault`」算作插件拦键——无插件对照下同样为 true，那是宿主 composer 自己的行为。

## 建议

1. **给 Enter 加「新面板确认」**：记住上一次已应答的面板 key 与时间戳；新 key 出现后的极短窗口（150–250ms）内忽略 Enter，或要求确认。这能直接消掉连按误批。
2. **让点击自身幂等**：点击前把面板 key 记入「已应答」集合，同一 key 只点一次；不要依赖按钮 `disabled` 的结算时序兜底。
3. README 的「守卫（刻意不做的事）」一节目前列了输入框、按钮焦点、组合键与连发、IME、无面板、运行时未确认六条，但没有提到「连按 Enter」；若采纳 1 和 2，建议把这一条也补进去，让使用者知道面板切换期间该停手。
4. 提问面板与计划审查面板目前只有文档与选择器字符串，缺运行时读数；建议补一个能构造这两类面板的自检脚本，外部复测才有对象。
