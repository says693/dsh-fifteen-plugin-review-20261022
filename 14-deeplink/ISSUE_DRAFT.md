<!-- intake:v1 {"project":"bench","target":"qyw233/dsh-deeplink","form":"intake"} -->
# [intake 输入] dsh-deeplink 0.6.1：两条深链在本版宿主上都不生效——宿主 `?token=` 鉴权会先吃掉整个 query，绕过鉴权后插件调的 `sessions.open` / `workspaces.connectWorkspace` 也不存在

我用 `?session=<id>` 与 `?workspace=<id>` 想让页面直达指定会话/工程，结果页面**每次都停在默认会话、地址栏也不变**，控制台里能查到两条不同的报错。设置卡（深链卡片）也没出现。

## 复现

环境：`@deepseek-ai/dsh 0.2.0-rc.2`，Windows 11，Edge 155.0.4283.62 headless（Playwright）。插件以固定提交 `fffa63a8906c962625d79ab62469e2ce03aee9b9`（package.json 0.6.1，MIT）本地目录安装到派生 profile `dl`（`dsh --profile dl --from-default-profile web --dump-config` 后 `dsh plugin --profile dl add file:<dir>`）。

宿主启动：`node <dsh>/lib/bin.js --profile dl --host 127.0.0.1 --port 0 --no-open`，隔离 `DSH_HOME`/`HOME`/`APPDATA`/`npm_config_cache`。

会话夹具：官方 `dsh-session-persistence-jsonl` 写入三个合成会话（`sess-dl-alpha-one`/`-two` 属 alpha 工程 `7529e9de-4bfc-4528-92e9-9ea05c755f53`，`sess-dl-beta-one` 属 beta 工程 `ee73cda6-f6fe-44b6-b772-dabdbafe22e8`）。对照 profile `ctl`（只 dsh-base + dsh-web-app，不装插件）点开 `sess-dl-alpha-two` 后 `[data-chat-flow]` 12 个子元素、`[data-chat-flow-kind=user]` 3 个，确认夹具本身能渲染。

**阻断 1：宿主鉴权重定向吞掉深链参数**

1. 浏览器新开 context，直接访问宿主打印的 `http://127.0.0.1:<port>/?token=<t>&session=sess-dl-alpha-two`。
2. 观察 document 响应链：`303 http://127.0.0.1:<port>/?token=<t>&session=sess-dl-alpha-two` → `200 http://127.0.0.1:<port>/`。
3. 页内 init script 记录的首帧 `location.search` 是空串；`session` 参数在插件执行前已经不存在。

7 种 URL 形状（token-only / token+session / session+token / token+workspace / token+foo / 无 token / 无 query）表现一致：带 token 一律 303 到 `/`，不带 token 一律 401。

宿主源码 `dsh-client-connection/lib/index.js` 的 `authorizeIndex()`：

```js
if (req.method === "GET" && url.pathname === "/" && tokens.length === 1 && ...) {
  res.writeHead(303, { "location": "./", ... });   // 相对 ./ ，query 不保留
}
```

**阻断 2：插件调用的服务方法在本版不存在**

4. 在同一 context 内先访问一次带 token 的 URL（让宿主签发 cookie），再导航到 `http://127.0.0.1:<port>/?session=sess-dl-alpha-two`。
5. 控制台出现：
   `[client-store] subscriber failed: TypeError: sessions.open is not a function  at onSessionList (...)`
6. 导航到 `/?workspace=7529e9de-4bfc-4528-92e9-9ea05c755f53`：
   `[workspace-controller] subscriber failed: TypeError: workspaces.connectWorkspace is not a function  at onWorkspaceList (...)`
7. 两种情况下当前会话都不变（仍是宿主的默认 `session-e9ea570e-d88f-4e06-ada9-d954cdb0874b`），`[data-chat-flow]` 子元素数 0，URL 未被改写成 `?session=<目标>`。

宿主服务面核对：`dsh-api-session-controller/lib/client.js` 的 `sessions` 服务（`rootCtx.reflect.provide("sessions", this, void 0)`）暴露 `retain`/`using`/`search`/`refresh`/`list`，无 `open`；`list` 快照是 `{ids, byId, phase, projectionsBySession}`，无 `current`（插件 L107/L121 读的 `.current` 恒 `undefined`）。`dsh-api-workspace-controller` 的 `workspaces` 服务暴露 `create/rename/delete/archiveSession/pinSession/...`，无 `connectWorkspace`（后者在 `dsh-client-ui-workspace` 的 `UiWorkspaceService` 上）。

**阻断 3：设置卡不渲染**

8. 打开「插件」页 → 「插件配置」区：能看到「已安装 1 `@dsh-community/dsh-deeplink`」且启用为开，但只有 5 个官方开关；`地址栏跟随` 文案在 600/760/1440 宽与深浅两色下都不出现，`.dshdl-switch` 计数 0。
9. 插件注入的样式节点存在（`style[data-plugin-css="@dsh-community/dsh-deeplink"]`，857 字符），说明浏览器半确实执行了。插件 L156 注册的槽是 `settings.plugin.item`，本版插件页（`dsh-client-ui-settings-plugins` / `-agent-loop` / `-subagent` / `-web-search`）消费的是 `plugins.item` 与 `settings.plugins.tab`。

## 测试环境

- 宿主：`@deepseek-ai/dsh 0.2.0-rc.2`（`C:\Users\32621\AppData\Roaming\npm\node_modules\@deepseek-ai\dsh\lib\bin.js`），`--host 127.0.0.1 --port 0 --no-open`
- 浏览器：Edge 155.0.4283.62 headless，Playwright 独立 context，`context.route` 只放行 `127.0.0.1/localhost/[::1]`
- 插件：固定提交 `fffa63a8906c962625d79ab62469e2ce03aee9b9`，本地目录安装（`file:` 依赖），**未测发布包**
- 隔离：`DSH_HOME`/`HOME`/`USERPROFILE`/`APPDATA`/`LOCALAPPDATA`/`XDG_*`/`npm_config_cache` 全部指向任务目录；未访问用户在用 GUI（1241/2538/4756），未读真实 token/cookie/会话
- 合成会话：官方 `@deepseek-ai/dsh-session-persistence-jsonl`（zstd），envelope `{seq,time}`、turn/step 从 1 起、`turn/end` 带 `data.reason:{kind:'completed'}`；不调用模型

## 证据

- `evidence/dl-09-url-shapes.json` — 7 种 URL 形状的 303/401、首帧 `location.search`、history 调用
- `evidence/dl-10-authed-deeplink.json` — 铸 cookie 后 9 个深链用例的 selectedKey / chatFlowKids / console
- `evidence/dl-08-trace.json` — init script 前后打点：`urlAtScriptStart` 已是 `/`，history 只有一次 `replaceState("/")`
- `evidence/dl-12-anomalies.json` — 空参数 / 重复参数 / 非法编码 / 2000 字符 / jump 开关
- `evidence/dl-13-follow-precise.json` — `/?session=…&keepme=1` 被改写成 `/?keepme=1`
- `evidence/dl-11-follow-settings-viewports.json` — 设置页无深链卡、600/760/1440 与深色读数
- `evidence/ctl-05-rows.json` — 对照 profile（不装插件）三个合成会话正常渲染
- 截图：`evidence/dl-D-D1-session.png`、`dl-D-D3-workspace-alpha.png`、`dl-C7-settings-plugins.png`、`dl-E-E2-bad-encoding.png`、`ctl-09-all-rows.png`

## 已验 / 未验 / 不确定 / 不做

**已验（confirmed）**

- 宿主 `?token=` 鉴权会把根路径请求 303 到 `/` 并丢弃全部 query，深链参数在插件运行前就没了（7/7 形状一致）。
- 绕过鉴权后，`?session=` 触发 `TypeError: sessions.open is not a function`，`?workspace=` 触发 `TypeError: workspaces.connectWorkspace is not a function`；两种情况下当前会话都不变。
- 插件确实被加载（`__DSH_BOOT__` 含该包，注入 style 857 字符），但「插件配置」里的深链卡片不渲染（槽名 `settings.plugin.item` vs `plugins.item`）。
- 无效 id（不存在/格式错/超长/非法编码）回落干净：无 pageerror、无白屏，console.warn 文案可检索。
- 空参数视为无参数、重复参数取第一个、两参数同时出现时 session 优先、`jump=0` 时确实不处理深链、`syncUrl` 会消费 `session` 且保留未知参数。

**未验（untested）**

- npm 发布包（无发布件，只测固定提交的本地目录）。
- 节点半的 model prompt section 是否真的进入 prompt、模型是否按其贴链接（不调用模型）。
- 旧宿主（0.1.x）上的行为——本机只有 0.2.0-rc.2，因此不能断言作者写错。

**不确定（uncertain）**

- 地址栏跟随（`follow`）：跟随依赖 current 变化，而 current 在所有用例里都不变，无法判断是「跟着没变化」还是「跟随逻辑没触发」。
- 无效 id 时界面上零提示是否就是作者要的「静默回落」——README 这么写，但对新手不友好。

**不做**

- 不投稿、不上传、不改插件源码、不改技能包、不读真实密钥、不访问用户 GUI。

## 建议

1. 文档层面先止血：README 与 model prompt 里示范的 `/?session=...` 形态在带 token 的 URL 上必然失效，请改成「先在浏览器里打开一次带 token 的 GUI，再换成不带 token 的深链」，或提供一条能保留 query 的落地路径。
2. 把 `sessions.open(id)` 换成当前服务面的等价调用（`sessions.using(id, {source}, ref => …)`，或走 `dsh-client-ui-workspace` 的 `UiWorkspaceService.openSession`）；`workspaces.connectWorkspace` 换成该服务上的方法；`sessions.list` 快照没有 `current`，跟随逻辑需要换数据源。
3. 槽名对齐：`settings.plugin.item` → `plugins.item`（配合 `settings.plugins.tab`）；`settings.register(namespace, schema)` 在本版 `settings` 服务上不存在，需换现有方法。
4. 无效 id 建议除 console.warn 外给一个非阻断 toast：现在用户点了链接什么都没发生，无从自查。
