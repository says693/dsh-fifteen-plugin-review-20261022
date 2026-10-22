# dsh-deeplink 0.6.1（提交 fffa63a）实测报告

> 对象：`qyw233/dsh-deeplink`，固定提交 `fffa63a8906c962625d79ab62469e2ce03aee9b9`，package.json `0.6.1`，MIT。
> **未测发布包**：npm registry / 腾讯镜像均 404，npm search 无同名包，因此只能以本地目录方式安装固定提交，未做 published tarball 的 shasum/integrity 核对。
> 宿主 `@deepseek-ai/dsh 0.2.0-rc.2`；Edge 155.0.4283.62 headless（Playwright，独立 context，仅放行本机地址）。

## 一、先说结论

插件被宿主正常加载（`__DSH_BOOT__` 里可见、注入的 style 节点存在 857 字符），参数解析、优先级、无效 id 回落这些**逻辑层都按源码设计工作**。但在这版宿主上，**两条深链都走不到终点**，而且是两个彼此独立的阻断：

| # | 阻断 | 位置 | 属谁 |
|---|---|---|---|
| B1 | 根路径请求带 `?token=` 时，宿主 303 重定向到 `./`，**整个 query 被丢弃** | 宿主 `dsh-client-connection/lib/index.js` 的 `authorizeIndex()` | 宿主行为（不是插件缺陷，但决定插件能不能用） |
| B2 | 即便绕过 B1 让插件读到参数，它调的 `sessions.open(...)` / `workspaces.connectWorkspace(...)` 在本版服务面上**不存在** | 插件 `lib/client.js` L120 / L136 | 插件与本版宿主不兼容 |

另外，README 承诺的**设置卡（设置 → 插件 → 插件配置 → 深链）在本版不渲染**：插件注册到 `settings.plugin.item`，而本版插件页消费的槽是 `plugins.item`（配合 `settings.plugins.tab`）。

## 二、逐条发现

### F1（confirmed）宿主鉴权重定向会吞掉深链参数

对 7 种 URL 形状做同一套观测（记录首个 init script 看到的 `location.search`、document 响应链、history 调用）：

| 形状 | 结果 |
|---|---|
| `/?token=<t>` | 303 → `200 /` |
| `/?token=<t>&session=sess-dl-alpha-two` | 303 → `200 /` |
| `/?session=...&token=<t>` | 303 → `200 /` |
| `/?token=<t>&workspace=<id>` | 303 → `200 /` |
| `/?token=<t>&foo=bar` | 303 → `200 /` |
| `/?session=...`（无 token） | `401`，无应用 |
| `/`（无 query） | `401`，无应用 |

所有 303 的落地 URL 都是 `/`，init script 首帧 `location.search` 为空串——说明**参数在插件运行前就没了**，与插件无关。宿主源码里 `authorizeIndex()` 在「GET + pathname `/` + 恰好一个 token 且匹配」时 `res.writeHead(303, { location: './' })`，Location 是相对 `./` 而非原 URL，query 不保留。

影响：README「Usage」表格和插件自己给模型的 prompt 里示范的都是 `http://127.0.0.1:3080/?session=...`；只要用户手上那条 URL 还带 `?token=`（首次打开必然带），深链就失效。要绕开，必须先让浏览器拿到 cookie（即先访问一次带 token 的 URL），再用**不带 token** 的 `/?session=...` 导航。

### F2（confirmed）`sessions.open` 在本版宿主上不存在

铸好 cookie 后导航 `/?session=sess-dl-alpha-two`，插件确实读到了参数，随后控制台报：

```
[client-store] subscriber failed: TypeError: sessions.open is not a function
    at onSessionList (.../plugins/??@deepseek-ai/...)
```

核对宿主 `dsh-api-session-controller/lib/client.js`：`sessions` 服务（`rootCtx.reflect.provide("sessions", this, void 0)`）暴露的是 `retain` / `using` / `search` / `refresh` / `list` 等，**没有 `open`**；`list` 快照的字段是 `{ids, byId, phase, projectionsBySession}`，**也没有 `current`**（插件 L107/L121 读的 `sessions.list.getSnapshot().current` 因此恒为 `undefined`）。

同理 `workspaces` 服务（`dsh-api-workspace-controller`，`super(ctx, "workspaces")`）暴露 `create/rename/delete/archiveSession/pinSession/...` 与 `list`，**没有 `connectWorkspace`**；`connectWorkspace` 在 `dsh-client-ui-workspace` 的 `UiWorkspaceService` 上，插件取的不是这个对象。

结果是：`?session=` 与 `?workspace=` 都在同一处断掉，页面停在默认会话（`session-e9ea570e-…`），`[data-chat-flow]` 子元素数 0。

### F3（confirmed）设置卡不渲染

插件页显示「已安装 1 `@dsh-community/dsh-deeplink`」且启用开关为开；注入的样式节点 `style[data-plugin-css="@dsh-community/dsh-deeplink"]` 长度 857，证明浏览器半执行了。但「插件配置」区只有官方 5 个开关，`地址栏跟随` 文案在 600 / 760 / 1440 宽、浅色与深色下都不出现，`.dshdl-switch` 计数恒为 0。

原因是槽名对不上：插件 L156 注册 `settings.plugin.item`，本版官方插件页（`dsh-client-ui-settings-plugins`、`dsh-client-ui-settings-agent-loop` 等）消费的是 `plugins.item` 与 `settings.plugins.tab`。宿主侧 `dsh-settings` 服务里也找不到插件调用的 `settings.register(namespace, schema)` 方法。

后果：两个开关（jump / follow）虽然读 localStorage 的逻辑是对的（F5），但**用户没有界面去改**，只能手动写 `localStorage['dsh-deeplink.jump']`。

### F4（confirmed）无效 id 的回落干净，但只在控制台可见

不存在会话 / 不存在工程 / `/../../etc/passwd` / `not-a-uuid` / 2000 字符 / `%E0%A4%A` 六类输入，全部：无 pageerror、无白屏、`bodyLen` 正常，且给出可检索的 warn：

```
[dsh-deeplink] 会话 'sess-nope' 不在会话列表中，忽略深链参数
[dsh-deeplink] 工程 '00000000-0000-4000-8000-000000000000' 不在工程列表中，忽略深链参数
```

这是源码 L114 / L130 的显式分支，行为与 README「静默回落，不报错、不影响页面」一致。界面上没有任何提示（截图与正常打开肉眼无差别）——按 README 的措辞算符合设计，但对新手不友好，记 uncertain 的那一半（U2）说的就是这个。

### F5（confirmed）开关读取与 URL 改写逻辑本身是对的

- `dsh-deeplink.jump=0` 时，同一 `?session=` URL 不再产生任何 dsh-deeplink 输出（默认值为 1 时会报 F2 的 TypeError）——说明 `readFlag(JUMP_KEY, true)` 与一次性读取生效。
- `/?session=sess-dl-alpha-one&keepme=1` 被 `history.replaceState` 改成 `/?keepme=1`：`session` 被消费、未知参数 `keepme` 保留，与 L89-L98 的 `params.delete('session')/delete('workspace')/delete('new')` 一致。

### F6（confirmed）参数解析与优先级正确

`?session=` 空值 → 视为无参数（无 warn、无报错）；`?session=A&session=B` → `URLSearchParams.get` 取第一个；`?session=…&workspace=…` 与顺序反转 → 都走 session 分支（日志里出现的是 `sessions.open` 的 TypeError 而不是 `connectWorkspace` 的），确认 session 优先。

### F7（observed，uncertain）地址栏跟随无法得到干净判定

跟随逻辑依赖「current 会话发生变化」，而当前宿主上 current 永远不变（所有跳转都失败），所以：手动点侧栏会话行时 history 调用数为 0，刷新后 URL 仍是 `/?keepme=1`。既可能是「跟着没变化」，也可能是「跟随逻辑没触发」，两者都可能，不给成因结论。

### F8（confirmed）UI 面本身没错位

600 / 760 / 1440 三档宽度、浅色与深色：「插件」页无横向溢出（`scrollWidth == clientWidth`），深色下 `bg rgb(21,21,23)` / `fg rgb(249,250,251)`，官方开关 `rect 36×20` 位置正常（x 随宽度自适应 540/694/1304）。插件自己的控件不可见，所以「插件在窄窗下会不会破」无法评。

## 三、未测项

1. **model prompt section**：节点半用 `ctx.systemPrompt.section` 注册了 `plugin:dsh-deeplink`（order −97），本任务不调模型，无法验证该段是否真的进 prompt、模型是否按它贴链接。源码里该段示范的 URL 形态正是 F1 会被吞掉 query 的形态。
2. **旧宿主兼容性**：本机只有 0.2.0-rc.2，无法回测 0.1.x，因此 F2 只能判「与本版不兼容」，不能断言作者写错。
3. **npm 发布包**：无发布件，未做 tarball 完整性核对。

## 四、环境与复现

见 `REPRODUCE.md`。要点：先 `dsh --profile dl --from-default-profile web --dump-config` 派生 web profile 再 `add`（直接 add 新 profile 会缺 `@deepseek-ai/dsh-web-app`，boot 后 EXIT 1 且 stdout/stderr 全空）；宿主 `--host 127.0.0.1 --port 0 --no-open`；合成会话用官方 `dsh-session-persistence-jsonl`，envelope `{seq,time}`、turn/step 从 1 起、`turn/end` 带 `data.reason:{kind:'completed'}`；先跑不装插件的 `ctl` 对照 profile 确认会话能渲染（`chatFlowKids` 12 / `userRows` 3），再测插件。

隔离：`DSH_HOME`/`HOME`/`USERPROFILE`/`APPDATA`/`LOCALAPPDATA`/`XDG_*`/`npm_config_cache` 全部指向任务目录；未访问用户在用 GUI 的 1241 / 2538 / 4756，未读 token/cookie/真实会话。自建宿主 pid 29192（ctl）、1824（dl）已按 `evidence/*-host.local.json` 停止。

## 五、给维护者的建议（按性价比排序）

1. **先解决 F1 的可用性前提**：在 README 与 model prompt 里明确「深链必须在已鉴权（有 cookie）的浏览器里用」，或把文档里的示例改成「先打开一次带 token 的 GUI，再把地址栏换成 `/?session=...`」。否则用户按文档操作 100% 失败且得不到任何提示。
2. **把 `sessions.open` 换成当前服务面的等价调用**（`sessions.using(id, {source}, ref => …)` 或直接走 `dsh-client-ui-workspace` 的 `UiWorkspaceService.openSession`），`workspaces.connectWorkspace` 换成该服务上的同名方法；`current` 不能从 `sessions.list` 快照读，需要换数据源。
3. **槽名对齐**：`settings.plugin.item` → `plugins.item`（配合 `settings.plugins.tab`），并把 `settings.register` 换成当前 `settings` 服务的现有方法。
4. 无效 id 建议除 console.warn 外给一个非阻断 toast——现在用户点了链接什么都没发生，无法自查。
