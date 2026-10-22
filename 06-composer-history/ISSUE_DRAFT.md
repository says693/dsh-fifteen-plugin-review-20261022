# 悬赏 intake 草稿 — dsh-composer-history 0.8.15

```json
{"project":"bench","target":"PerryLink/dsh-composer-history","form":"intake"}
```

## 结论先行（给主 agent）

**本轮实测未找到可提交的缺陷，`confirmed` 缺陷 0 条。本文件不是一份可投稿的 issue，只作留档。**

被测件为 npm 发布件 `dsh-composer-history@0.8.15`（PerryLink/dsh-composer-history，Apache-2.0）。本地重算 SHA1 = `9ae4a6636a733f7624c53354aa5173063ec8d814`、sha512 base64 = `L3/RiaCKvaUf8TETcDozTzId5nMjsCiM9vk9P66MhFedam0KCBUhx5Q99RbUVkDwUAxl/NPfLlYregkRaOO9cA==`，与 registry 的 `dist.shasum` / `dist.integrity` 逐字一致；LICENSE 为 Apache-2.0 全文（11357 字节，sha256 `c71d239d…`）。任务给的仓库固定提交 `1227c30175c6270b8576840c804f4c498abdff82` 仅作源码逻辑对照（发布件自带 `src/`，未克隆、未构建）。

宿主 `@deepseek-ai/dsh` 0.2.0-rc.2，Windows + Edge 155.0.4283.62（Playwright 1.64.1 headless），隔离 `DSH_HOME` 与 profile，端口 0 或固定 5470。

---

## 已验正常项（支撑「本轮无缺陷」的判断）

32 项检查：27 confirmed / 3 uncertain / 2 untested，confirmed 缺陷 **0**。逐项读数见 `RESULTS.json` 与 `evidence/`。

**方向键召回（重点 1）**

- 空草稿、光标在首行按 ↑ → 草稿变最新一条 `第三条历史：再检查一遍错别字`，caret 14（= 长度）。（`evidence/run4-05-recall-matrix.json`）
- 连续 ↑：第二条(13) → 第一条(13) → 停在第一条不再变化（caret 归 0、文本零变化、0 pageerror）。到边界不清空草稿，与 README 的 hold 语义一致。
- 半途草稿保护：草稿 `还没写完的草稿` 按 ↑ 被 stash 并填最新条（caret 14、`defaultPrevented=true`）；一路 ↓ 回到最新条再按 ↓，草稿**逐字还原**；Esc 同样还原。
- 多行草稿边界：两行文本光标在第二行（caret 6）按 ↑ → 文本不变、caret 3、`defaultPrevented=false`（放行）；移回首行（caret 0）按 ↑ → 召回。

**Ctrl+R 反搜（重点 2）**

- 只吞 `r` 键（capture 阶段 Control `false` / r `true`），页面不刷新；浮层 `__dsh-composer-history-search__` 出现，`3 entries`，首行 `aria-selected=true`，焦点进浮层 INPUT。
- 过滤 `第三` → 仅 1 行；`历史` → 3 行、`3 matches`、三个 `<mark>` 依次 `历史`/`历史`/`历史`。
- ↑↓ 两端成环：↓↓↓ 从第一行绕回第一行，再 ↑ 绕到第三行。
- Enter 取词：草稿变选中行、caret 14、浮层 `display:none` + rect 0×0 + `aria-expanded=false`。
- Esc 取消：浮层同上关闭，草稿保持原样。归因读数——capture `defaultPrevented=false`、浮层 root 监听读到 `true`、input 自身 `false`，说明拦截在 root 冒泡监听（`search-overlay.ts` 的 Escape 分支）。
- 大小写：默认 `searchCaseSensitive:false`，`hello` 与 `HELLO` 都 `2 matches`，mark 按原文大小写。
- 无匹配 `zzzz-no-match`：rows=[]、marks=[]、status 空串；Esc 后草稿仍为空。

**持久化与存储位置（重点 6）**

- 存储为**浏览器 localStorage 单键** `dsh.composer-history.v1`，`{"v":1,"entries":[…]}`；另有 `.insights.v1`。宿主磁盘无对应文件。
- 刷新后 entries 不变、仍可召回（`window.__reloadMark` 变 null 证明真刷新）。
- **固定端口冷重启后仍保留**：job_kill pid 20832 → `--port 5470` 重起 pid 33680 → 同一持久化浏览器 profile 打开，entries 仍 4 条、仍召回。

**异常与边界（重点 7）**

- 空历史：干净 profile 下按 ↑ 无副作用、`defaultPrevented=false`（README「no history → pass」）。
- 20014 字条目：连按 ↑ 三次 26→30→20014 字，settle 10/6/14ms，无截断；召回后继续打字变 20016，Esc 还原为 1。
- 损坏 payload（`{not json at all`）：召回仍工作（回落会话节点）、浮层仍列 3 条、0 pageerror；损坏串原样保留未被改写。

**与输入框原生命令不冲突（重点 5）**

- Shift+Enter → `甲\n乙`；Shift+↑ 与 Alt+↑ 均放行（文本不变、`defaultPrevented=false`）；Ctrl+↑ 按 `enableCtrlAlias` 默认等效裸 ↑。
- 粘贴三行后：末行 ↑ 放行（capture 与 bubble 两阶段都 false），首行 ↑ 召回整段 stash。

---

## 未验 / 不确定 / 不做

**未验（untested，2 项）**

- `historyScope: 'workspace'`（需改 profile Config 并重启）：未测「同工作区其他会话并入召回」。
- 同 origin 下不同工作区的召回隔离：只验到「不同 origin 之间不共享」，未验同 origin 跨工作区。

**不确定（uncertain，3 项）**

- `I2` 焦点不在输入区时按 Ctrl+R 是否被吞：capture 阶段读到 r 键 `defaultPrevented=false`（未吞），但该次试验开始时浮层已处于打开态，无法把「浮层出现」归因到这次按键。若投稿需先关浮层重测。
- `J1` 取词后焦点归属：Enter 后 activeElement 是 composer DIV、随后可正常打字，但未覆盖「取词后立刻连按 ↑」的组合。
- `D1` 跨工作区隔离的归因：现象成立（B 工作区召回不到 A 的条目），但成因是 localStorage 按 origin 隔离，**不能**据此说插件实现了工作区级过滤。

**不做**

- 桌面 Electron 端、真实 IME 会话、真实模型往返（宿主未配 `DEEPSEEK_API_KEY`，模型调用一律「处理失败」，但 user 消息正常入库，不影响历史链路）。
- 不投稿、不上传、不建公开仓库、不改插件源码。

---

## 三条被推翻的早期怀疑（留档，避免后续误报）

| 早期怀疑 | 复测结论 | 成因 |
|---|---|---|
| Esc 关不掉 Ctrl+R 浮层 | 能关（`display:none`、rect 0×0、`aria-expanded=false`） | 浮层节点常驻 DOM，只切 `display`；我最初用 `querySelector` 非空当可见判据 |
| 20014 字条目召回后草稿不出现 | 正常召回，一次到位 | 只等 5000ms 且只看第一次采样 |
| 草稿存在时按 ↑ 未召回 | 正常召回，↓ 可逐字还原 | 同一步里先重设 caret 再读，读到中间态 |

## 安装面记录（非插件缺陷）

1. `dsh plugin --profile chist add <tgz>` 首次生成的 profile，`dsh.profile.bundles` 只有 `dsh-base` + 插件两行，**缺 `@deepseek-ai/dsh-web-app`**，宿主启动即退（EXIT 1、stdout/stderr 均 0 字节、无监听端口）；补成三行后正常。官方包目录里 `dsh-web-app` 存在，属 profile 初始化行为。
2. PowerShell `Set-Content -Encoding utf8` 回写 profile `package.json` 会带 UTF-8 BOM，`readProfileManifest` 的 `JSON.parse` 直接失败；改用 node 剥 BOM 后写回。属测评工具链侧。

## 建议

本轮没有需要维护者修复的问题。若后续要扩大覆盖，优先补：`historyScope:'workspace'` 的同 origin 双会话对照、取词后紧接 ↑ 的组合、以及真实 IME 输入法下的 `isComposing` 抑制。

## 证据（上传证据仓后把相对文件名替换为链接）

```text
evidence/step06-file-hashes.json                      9 个关键文件 sha256
evidence/step01-npm-view.json                         registry 元数据（shasum/integrity/engines）
evidence/step03-plugin-add.log                        安装输出（EXIT 0）
evidence/step04-dump-config.txt                       dump-config 里出现 composer-history 行
evidence/step05-profile-before-bundles-fix.json       缺 dsh-web-app 的 profile 原样
evidence/run4-05-recall-matrix.json                   B1/B2/B4/B5/F2/F3 完整矩阵
evidence/run4-06-search-reload.json                   B3 复测 + C 系列 + H3 + G2
evidence/run4-07-edge-cases.json                      H1 空历史 / H4 损坏存储
evidence/run5-14-esc-display.json                     Esc/Enter/点击外 三种关闭路径
evidence/run5-13-esc-attribution.json                 Esc 拦截归因（capture/root/input）
evidence/run5-15-case-longentry.json                  大小写 + 无匹配 + 20k 条目
evidence/run5-16-long-entry-walk.json                 20014 字条目逐次召回
evidence/run5-11-coldrestart-workspace-b.json         冷重启 + 跨工作区
evidence/run4-10-cross-session.json                   侧栏响应式折叠与跨会话历史
evidence/lingshulab-reviews-all.json                  评测站全量 172 条
evidence/lingshulab-plugins-all.json                  评测站插件目录全量 133 条
evidence/lingshulab-dedupe.json                       查重结论
evidence/run4-15-ctrlr-open.png                       浮层打开截图
evidence/run5-30-cold-restart.png                     冷重启后召回截图
evidence/run6-31-workspace-b.png                      工作区 B 截图
```

## 查重

- GitHub `FuRongJun-1999/dsh-bounty/issues`（2026-10-22T09:14+08:00）：`dsh-composer-history` 命中 1 条（#1317，pi2dsh v0.27.0 安全测评，正文不含本插件，是按分词命中的无关条目）；`PerryLink` 命中 12 条，全是 PerryLink 名下其他插件（dsh-session-sync / dsh-permission-rules / dsh-approval-gate / dsh-doublecheck / dsh-memento ×2 / dsh-host-rsi / 上下文税横评 / dsh-memory-vault / dsh-mirror / dsh-core-memory / only-memory）；`composer` 命中 6 条，均非本插件。**同范围重复 0**。
- `review.lingshulab.com`（2026-10-22T09:14:56+08:00）：`?q=` 服务端不过滤，改为拉全量 —— `/api/reviews` 4 页 172 条、`/api/plugins` 3 页 133 条，本地逐条扫描。评论命中 0；插件目录里 PerryLink 名下只有 `dsh-memento` 一个，`PerryLink/dsh-composer-history` 未被收录。**无同范围重复**。
