# dsh-session-hotkeys 1.9.5 实测报告

- 被测件：**npm 发布件 `dsh-session-hotkeys@1.9.5`**（不是仓库 HEAD）
- 宿主：官方 `@deepseek-ai/dsh` 0.2.0-rc.2，隔离 `DSH_HOME`，`--host 127.0.0.1 --port 0 --no-open`
- 浏览器：Edge + Playwright 1.64.1，headless 独立 context，只放行本机地址
- 作者：YEYEYEYESHIFU（npm 用户 `yeyeyeyeshifu` / Captain Americano），MIT

## 一、版本与对象核对（先说清测的是哪一个）

任务给的仓库固定提交 `d1f38ebc5128a7b8fada47f0472207c2d5260393` **只用于差异核对，不是被测件**。仓库 HEAD 是 1.9.2，npm 上是 1.9.5，两者内容不同，本报告所有结论只针对 1.9.5 发布件。

发布件哈希已本地复算并与 registry 声明逐字节对上：

| 项 | registry 声明 | 本地复算 | 一致 |
|---|---|---|---|
| shasum (SHA1) | `1d7500442ea03345b3d234db46fb703a34169c58` | 同值 | 是 |
| integrity (SHA512) | `sha512-e9CkjPW+KWM1l066wY/McsgCQEV9+0bmm9oboe6xb3B1wGSCQsXZkTwj5dIE3QHrgssRfJtyYqcqlKFGQnd91Q==` | 同值 | 是 |

包内 7 个文件，`lib/index.js` 只有一个空 `apply()`（宿主侧零行为），全部逻辑在 `lib/client.js`（152795 字节，2624 行）。`cordis.patch.yml` 以 `insert: id=session-hotkeys` 挂载，`dsh.client` 声明 `platform=web`、`immediately=true`、`inject` 为 ui-layout 与 ui-sidebar。

## 二、发现与影响

本轮**没有发现阻断级缺陷**。功能面基本按 README 工作。下面是值得记录的观察项与不确定项，按对用户的实际影响排序。

### 1. `Alt+N` 在「当前是空白会话」时提示成功但没有新会话（观察项，非插件缺陷）

**现象**：全新 profile 里只有一个空白会话（未发过消息，标题「新会话」）时，连按三次 `Alt+N`，每次 toast 都报「新建会话」，但侧栏行数恒为 1（1→1→1），当前会话也没变。

**归因（已定位，不是插件的错）**：抓包显示 `POST /api/session/create` 的请求体是

```json
{"method":"session/create","payload":{"args":{"request":{"workspaceId":"194af2d3-…","sessionId":"session-ca05d14f-…"}}}}
```

即调用链把**当前会话 id** 当 `sessionId` 传了进去，宿主于是复用该会话并返回同一个 id（`result.value.sessionId` 与当前会话相同）。三项对照确认这不是插件独有：点宿主自带的「新会话 Ctrl + Alt + N」按钮 → 1→1；按宿主原生 `Ctrl+Alt+N` → 1→1；把当前会话变成有内容后再按 `Alt+N` → 1→2，此时请求体只带 `workspaceId`。

**真正的问题在文案**：`client.js:1164-1174` 的 `newSession()` 只在 `start()` 抛异常时给 warn，否则一律 `showToast(newSessionOk)`。所以 toast 只证明「调用没抛异常」，不证明「会话真的建了」。用户看到「新建会话」却什么也没发生，会以为按键坏了。

**影响**：低（功能本身没错），但对新手的观感影响不小。**建议**：成功分支改为等宿主返回后校验 sessionId 是否为新值，或把文案弱化成「已请求新建会话」。

### 2. 重命名后侧栏标题未立即更新（不确定）

`Alt+Shift+R` 走 `window.prompt`（`client.js:1242`），accept「周记·第一篇」后 toast 报「已重命名为 · 周记·第一篇」，但当次读到的侧栏行文本仍是「新会话」。之后发送第一条消息，该行显示为「周记·第一篇 刚刚」。标题最终生效，但生效时机与 UI 更新之间存在差异，**没有定位到确切原因**，按 `uncertain` 记录，不写成缺陷。

### 3. 归档确认卡片会主动抢焦点（行为记录）

按 `Alt+Shift+A` 后 `.shk-confirm` 可见（382×196），焦点自动落到 danger 按钮 `shk-confirm-btn-danger`（「归档」），而不是安全的「取消」。`Esc` 取消已验证可用（卡片关闭，`workspace.json` 的 `archivedSessionIds` 仍为 `[]`，没有误归档）。卡片把默认焦点放在破坏性按钮上，属于可讨论的交互选择：好处是「Enter 确认」路径顺畅，风险是连按两次 `Alt+Shift+A` 加一次 Enter 就容易误归档。**建议**：默认焦点放「取消」，或给归档按钮加短暂延时保护。

### 4. 测试方法学上的两处坑（供复跑者注意，不是插件问题）

- **不能用冒泡阶段的 `document` 监听器读 `defaultPrevented`**：插件在捕获阶段（`window` + `document` 双注册）先 `preventDefault` 再 `stopPropagation`，挂在冒泡阶段的监听器根本收不到事件，会得出「这些键没被接管」的假读数。可靠做法是看 `target.dispatchEvent(ev)` 的**返回值**（返回 `false` 即有人 `preventDefault`），或直接用真实键盘输入。本轮先用同步合成事件测出 5 个键「未被接管」，改用 `page.keyboard.press` 后全部正常生效（`Alt+Shift+U` 打开已归档页、`Alt+Shift+1` 固定槽位、`Ctrl+Alt+1` 跳转固定槽位、`` Alt+` `` 进导航模式、`Alt+↓` 切下一个），说明是测量假象。
- **`--port 0` 会换 origin，而 localStorage 按 origin 隔离**。要验「冷重启后改绑是否保留」，必须固定端口 + `launchPersistentContext` 固定 userDataDir，否则测的是浏览器存储隔离而不是插件持久化。本轮冷重启测试固定用 23456。

## 三、已验证的读数（摘要）

完整列表见 `RESULTS.json`，证据在 `evidence/`。

**键位清单一致性（B1）**：面板「按键」页读到 18 条 keyval，依次为 ``Alt+` ``、`Alt+1-9`、`Alt+↑`、`Alt+↓`、`Alt+N`、`Alt+P`、`Alt+B`、`Alt+W`、`Alt+M`、`Alt+Enter`、`Esc`、`Alt+Shift+Enter`、`Alt+Shift+F`、`Alt+Shift+A`、`Alt+Shift+U`、`Alt+Shift+R`、`Alt+Shift+1-9`、`Ctrl+Alt+1-9`，与 README 键位表和源码 `WIN_DEFAULTS`（`client.js:99-115`）逐项一致。

**位置切换（C1/C2）**：两个会话时 `Alt+2` 使 `aria-selected` 行由 `session-b82a64e0…` 变为 `session-ca05d14f…`，toast「已切换到会话 2 · 周记·第一篇」；`Alt+1` 提示「已在会话 1」；`Alt+3`/`Alt+5` 提示「第 N 个位置没有会话」且不改当前会话。`Alt+↓` 真实键盘下 changed=true。

**改绑与旧键（D1/D2/D4）**：录制 `Ctrl+Shift+3` 后 `localStorage["dsh.session-hotkeys.keys"]` 的 `switchDigit` 变为 `{ctrl:true,alt:false,shift:true,meta:false,digit:true,code:null}`，面板显示 `Ctrl+Shift+1-9`；`Ctrl+Shift+2` 生效（changed=true），`Alt+1` 失效（changed=false，toast 退化为「已聚焦聊天区」）；「恢复默认」后 18 条全部回到 `Alt+1-9` 等默认值，toast「已恢复 Windows 默认键位」。

**持久化（G1/G2）**：`page.reload` 后改绑仍在且新键可用、旧键仍失效；冷重启（同 origin 23456）后同样保留，新键仍被接管、旧键仍失效。

**输入抑制（E1–E4）**：输入框聚焦时 `Alt+2` 仍能切换（README 卖点成立）；裸字母 `a b m n p w r` 不触发任何动作，字符正常进入输入框；IME 组合态（`compositionstart` + `isComposing=true` / `keyCode=229`）下的 Esc 没有把焦点抢走；`Esc` 两段式（「已聚焦聊天区」→「焦点已还原」）成立。

**异常（H1/H2）**：200ms 间隔连按 6 次位置键，无 `pageerror`、最终状态与最后一次按键一致；只剩 1 个会话时 `Alt+9` 只提示「第 9 个位置没有会话」。

**侧栏折叠（C5）**：`Alt+B` 使 aside 宽度 280 → 56 → 280，按钮 aria-label 在「收起侧边栏」与「打开侧边栏」间切换。

## 四、未测与不确定

| 项 | 状态 | 原因 |
|---|---|---|
| 归档 Enter 确认（真正归档） | untested | 那次读数落在宿主引导遮罩仍在的会话里，焦点不在卡片上，未取得 `archivedSessionIds` 的落盘对照；Esc 取消已验 |
| 改键冲突拒绝（`recConflict`） | untested | 本轮改绑用的 `Ctrl+Shift+数字` 不与现有绑定冲突，没有单独构造冲突组合 |
| 输入框光标位置保持 | untested | 只记录了文本未破坏，没记 `selectionStart/selectionEnd` |
| Windows 上 Alt 裸键的 `defaultPrevented` | untested | 只验证了 `Alt+数字` 未被浏览器抢走，没单独取 Alt 单键的读数 |
| 宿主弹窗与插件 Esc 竞争 | uncertain | 未构造宿主原生 dialog/popover 与插件 Esc 并存的场景 |
| 长按 `event.repeat` | uncertain | 未用真实长按构造 repeat 事件 |
| macOS 预设（`⌃⇧1-9` 等） | untested | 宿主为 Windows，`detectMac()` 走 WIN_DEFAULTS 分支 |
| `Alt+Shift+Enter`（非默认方式发送） | untested | 隔离环境无模型密钥，没有运行中的 agent |

## 五、环境与限制

- 隔离：`DSH_HOME`、`HOME`、`USERPROFILE`、`APPDATA`、`LOCALAPPDATA`、`XDG_*`、`npm_config_cache` 全部指向任务目录；浏览器 `context.route` 只放行 `127.0.0.1` / `localhost` / `[::1]`。
- 用户正在使用的 GUI（4756）全程未访问、未读 token。全程未读、未输入任何 API Key；宿主首次启动的「添加一个 API Key 开始使用」引导用「稍后配置」跳过。
- 隔离 profile 里的会话全部由宿主自身在隔离 `DSH_HOME` 内创建，没有读取或复制任何用户数据。
- `dsh plugin --profile <new> add` 建出的 profile 只含 `@deepseek-ai/dsh-base`，web 宿主需要 `@deepseek-ai/dsh-web-app`，因此先用 `dsh --profile shk --from-default-profile web --dump-config` 初始化 profile。这是 CLI 的 profile 初始化行为，不是插件缺陷。
- 自己启动的宿主进程（2280 / 23456 各一次，冷重启共 3 个 PID）已全部停止，两个端口无监听；无遗留 `--profile shk` 的 node 进程。

## 六、复跑

见 `REPRODUCE.md`。所有原始命令输出、截图与 JSON 读数在 `evidence/`，编号与 `RESULTS.json` 的 `evidence` 字段一一对应。
