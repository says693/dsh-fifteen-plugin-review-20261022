# dsh-composer-history 0.8.15 实测报告（技术版）

被测件：`dsh-composer-history@0.8.15`（PerryLink/dsh-composer-history，Apache-2.0，发布 2026-10-18T12:31:33Z，gitHead `2315642`）。
宿主：`@deepseek-ai/dsh` 0.2.0-rc.2，`--host 127.0.0.1 --port <0|5470> --no-open`，隔离 `DSH_HOME`/`HOME`/`APPDATA`/`LOCALAPPDATA`/npm cache。
浏览器：Edge 155.0.4283.62 + Playwright 1.64.1（headless，只放行本机地址）。

## 一、结论先行

**本轮没有找到可提交的功能缺陷。** 32 项检查里 27 项 confirmed 正常，3 项 uncertain，2 项 untested，confirmed 缺陷 0 条。

需要特别说明的是：探测过程中出现过三条「疑似缺陷」，全部在复测后被推翻，成因都在我的探针而不是插件：

| 早期怀疑 | 复测结论 | 推翻依据 |
|---|---|---|
| Esc 关不掉 Ctrl+R 搜索浮层 | 能关 | 浮层节点常驻 DOM，只切 `display`。改用 `getComputedStyle` + `getBoundingClientRect` 判可见性后，Esc 后 `display:none`、rect 0×0、`aria-expanded=false`、焦点回输入区 |
| 20000 字超长条目召回后草稿不出现 | 正常召回 | 之前只等 5000ms 且只看第一次采样。逐次按 ↑ 走到该条、按长度收敛判定后，20014 字一次到位，settle 14ms |
| 输入草稿后按 ↑ 未召回（草稿没被 stash） | 正常召回 | 之前在同一步里先重设 caret 再读，读到的是中间态。复测：↑ 得最新条（caret 14），↓ 逐字还原草稿 |

这三条如果照早期读数写进 issue，会是误报。

## 二、实际验到的行为

### 方向键召回与草稿/光标还原

- 空草稿、光标在首行按 ↑：草稿变最新一条，光标落在末尾（caret = 文本长度）。capture 阶段该 keydown `defaultPrevented=true`，即插件确实接管了事件。
- 连续按 ↑：逐条回溯到最老一条后**停住不动**（文本零变化、caret 归 0），不清空草稿、不抛错。这与 README 的 hold 语义一致。
- 半途草稿保护：草稿 `还没写完的草稿` 按 ↑ 被 stash 并填入最新条；一路 ↓ 回到最新条再按 ↓，草稿**逐字还原**。
- Esc 同样能还原 stash 的草稿。
- 边界判定按「逻辑行」：两行草稿里光标在第二行时 ↑ 完全放行（`defaultPrevented=false`，caret 由浏览器从 6 移到 3）；把光标移回首行才召回。

### Ctrl+R 反搜

- 浮层根类 `__dsh-composer-history-search__`，行 `...__row`，高亮用 `<mark>`，状态文案 `N entries` / `N matches`。
- 打开时只吞 `r` 键（capture 阶段 Control `defaultPrevented=false`、r `true`），页面不刷新。
- 过滤正确、匹配子串高亮、↑↓ 在两端成环、Enter 取词写入草稿且光标到末尾、Esc 取消且草稿不被改写、点击面板外同样关闭。
- 大小写：默认 `searchCaseSensitive: false`，小写 `hello` 与大写 `HELLO` 都命中两条（`Hello…` 与 `hello…`），高亮按原文大小写显示。
- 无匹配时列表为空、状态文案为空字符串、不崩、Esc 后草稿仍为空。

### 与输入框原生命令的冲突面

- Shift+Enter 正常插入换行（`甲\n乙`）。
- Shift+↑、Alt+↑ 一律放行（文本不变、`defaultPrevented=false`）。
- Ctrl+↑ 按 `enableCtrlAlias` 默认 true 等效裸 ↑（召回）。
- 粘贴三行文本后：光标在末行按 ↑ 放行（capture 与 bubble 两阶段都 `false`）；移到首行按 ↑ 才召回。

### 持久化与存储位置

- 历史存在**浏览器 localStorage 单键** `dsh.composer-history.v1`，形状 `{"v":1,"entries":[...]}`；另有 `dsh.composer-history.insights.v1`。宿主磁盘上没有对应历史文件。
- 页面刷新后 entries 不变、仍可召回。
- **宿主冷重启（固定端口、同 origin）后仍保留**：kill 掉 pid 20832 后以 `--port 5470` 重启（pid 33680），同一持久化浏览器 profile 打开，entries 仍 4 条、仍能召回同一条文本。
- 反例（环境限制，不是缺陷）：`--port 0` 会让每次启动换 origin，localStorage 按 origin 隔离，此时读不到旧存储。这一条已在 RESULTS 里写明，不作为缺陷。

### 工作区隔离

- 换 cwd 到 `fixtures/workspace-b` 的宿主（origin 8975）用独立浏览器 profile 打开：没有任何 composer-history 键、按 ↑ 无反应；在该工作区发一条消息后 entries 恰为 1 条，连按 5 次 ↑ 只召回这一条。
- **但归因要谨慎**：该隔离来自「不同 origin 的 localStorage 不共享」，不能单独证明插件做了工作区级过滤 —— 插件存储本身是单键、不带工作区维度。同 origin 下不同工作区是否隔离，本轮未测（记 untested）。

### 异常与边界

- 空历史：干净 profile 下按 ↑ 无副作用、事件不被拦截（与 README「no history → pass」一致）。
- 20014 字超长条目：整条召回、无截断、settle 14ms，召回后输入区仍可继续打字。
- 损坏 payload（`{not json at all`）：召回仍工作（回落到会话节点），浮层仍能列出 3 条，0 pageerror；损坏字符串原样留在存储里未被改写（源码只承诺读时回落）。

## 三、安装面与环境记录（不是插件缺陷）

1. `dsh plugin --profile chist add <tgz>` 首次初始化出的 profile，`dsh.profile.bundles` 只有 `dsh-base` 与插件两行，**缺 `@deepseek-ai/dsh-web-app`**，宿主启动即退（EXIT 1、stdout 与 stderr 均 0 字节、无监听端口）。手工补成三行后正常。官方包目录里 `dsh-web-app` 是存在的，属 profile 初始化行为。
2. 用 PowerShell `Set-Content -Encoding utf8` 回写 profile `package.json` 会写入 UTF-8 BOM，`readProfileManifest` 的 `JSON.parse` 直接失败（`Unexpected token '锘?, ...`）。改用 node 剥 BOM 后写回即可。这条属测评工具链侧。

## 四、限制

- 宿主未配置 `DEEPSEEK_API_KEY`，模型调用一律 `处理失败`；但 user 消息仍正常入库，历史与召回链路不受影响。所有「历史条目」都来自真实发出的用户消息或直接注入的存储 payload，**没有真实模型往返**。
- `historyScope: 'workspace'` 未实测（需改 profile 配置并重启）。
- 跨会话作用域：宿主侧栏在 1440 宽下只渲染「默认工作区」一行，需先缩到 1100 宽才展开会话行；真正切换会话的取证因此受限，只完成了「同一 origin 内持久化历史跨会话可见（符合 README 声明）」这一条。
- 未测桌面 Electron 端、未测 IME 真实输入法会话（只用过 `isComposing` 语义判断的源码对照）。
- 未测 `recallWithDraft: 'gate'`、`edgeMode: 'visual'`、自定义 `upKey`/`downKey`/`searchKeys` 等非默认配置。

## 五、证据索引

`evidence/` 下按 `stepNN-*`（发布件取证与安装）与 `pNN-run.txt` / `runN-NN-*.json` / `runN-NN-*.png`（探针读数与截图）分组。关键几份：

- `evidence/step06-file-hashes.json` — 9 个关键文件 sha256
- `evidence/run4-05-recall-matrix.json` — B1/B2/B4/B5/F2/F3 的完整矩阵
- `evidence/run4-06-search-reload.json` — B3 复测、C 系列、H3、G2
- `evidence/run5-14-esc-display.json` — Esc/Enter/点击外 三种关闭路径的 display 读数
- `evidence/run5-16-long-entry-walk.json` — 20014 字条目逐次召回
- `evidence/run5-11-coldrestart-workspace-b.json` — 冷重启 + 跨工作区
- `evidence/lingshulab-reviews-all.json` / `lingshulab-plugins-all.json` / `lingshulab-dedupe.json` — 评测站全量查重
