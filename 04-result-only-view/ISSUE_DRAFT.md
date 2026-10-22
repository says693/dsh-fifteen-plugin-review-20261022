# [intake 输入] dsh-result-only-view 1.7.0 实测：主路径全部按文档工作，未发现可提交缺陷（三项未测）

> **给主 agent 的结论先行**：29 项检查里 **confirmed 25 项、uncertain 1 项、untested 3 项，确认缺陷 0 条**。本文件不是一份可提交的 issue，而是把「一轮完整实测跑到哪、哪些没跑到、为什么」按 intake 格式留档。**如果主 agent 要投稿，本文件没有可发的缺陷条目**，只适合作为「正面复核记录」。

被测件为 npm 发布件 `dsh-result-only-view@1.7.0`（作者 YEYEYEYESHIFU，MIT）。仓库固定提交 `47cfe87a7d183be2506fcca8ff9ed2e5f6069b2c` 的 package.json 是 **1.6.4**，与发布件**不是同一版本**：两者 `lib/index.js`、`cordis.patch.yml`、`LICENSE` 哈希相同，`lib/client.js`（59,767 vs 53,761 字节）、`package.json`、`README.md`、`CHANGELOG.md` 不同。仓库提交**只用于源码差异核对，没有安装运行**。宿主 DeepSeek-Harness 0.2.0-rc.2，Windows + Edge 无头（Playwright），隔离 `DSH_HOME` 与 profile，端口 0。

---

## 一、confirmed 项（支撑「无缺陷」判断，逐条读数在 RESULTS.json）

**开关与即时性**

- 开关在输入框左侧工具行（`conversation.input.left`），文案「◉ 只看结果」/「○ 只看结果」，rect (655,936,79,22)，`borderRadius=11px`。
- 点击即时生效：点前 `styleTags=0 / thinkHidden=0`；同一 tick 内（点击自身耗时 1.9ms）`styleTags=1 / thinkHidden=1`；下一 rAF `aria-pressed` 翻成 true；50/150/400ms 采样稳定。
- 状态三者同步：`aria-pressed`、`localStorage["dsh.result-only-view.enabled"]`（"0"/"1"）、注入的 `style[data-plugin="dsh-result-only-view"]` 数量（0/1）始终同向。

**隐藏与保留**

- 思考行 `[data-variant="think"]`：开 → `display:none` 高 0；关 → `flex` 高 24px。
- `turn-process` 与 thinking-only 的 `assistant-step`：开 → `none`；真正带正文的 `assistant-step` 保持 `block` 且高 549px。
- 回答零丢失：开 386 字 / 关 385 字，两边都含尾标记、表头「状态」、代码行 `function greet`；开态可见文本里不含思考正文。

**还原**

- 关闭后样式标签消失（0 个），所有被查元素 `display` 逐项回到初始值，`inline style` 全为 `null`（插件不写内联样式）。
- 三态（关/开/再关）的 flow 项与 think 行数组逐项比较 `identical=true`。

**持久化**

- 关 → reload → 仍 `pressed=false / styleTags=0 / think 可见`。
- 开 → reload → 仍 `pressed=true / label=◉ 只看结果 / styleTags=1 / thinkDisplayed=0`。
- 切会话：关态 one→multi 仍 false（3 个 think 全可见）；开态 multi→one 仍 true（think 全隐藏）。

**长内容**

- 2852 字正文 + 代码块 + 4 行表格完整；`pre` `overflowX=auto`、`maxHeight=none`、`scrollHeight=clientHeight=159`；`table` 742×172、父容器 `overflow=auto`、`maxHeight=none`；尾标记与最后一段都在。

**多轮与痕迹行**

- 3 轮会话有 3 条 `turn-tail`，各有独立「已处理 1 步 · 5s ▸」按钮（`aria-expanded=false`），点第 1 条只展开第 1 个 think（`thinkVisible` 0→1），再点回到 0。
- 痕迹行步数 1 与该回合 `turn-process` 计数 1 一致（不是 0 步）。

**两个异常**

- 展开中切换：先展开 think（`data-expanded=true`、高 96）→ 开（`none`、高 0，`data-expanded` 仍 true）→ 关（`flex`、高 96、`hasBody=true`）；全页孤儿扫描 `orphanCount=0`。
- 20 次 ×30ms 快速切换：一次停在 `false / styleTagCount=0 / thinkDisplayed=1`；另一次停在 `true / styleTagCount=1 / CSS 7610 字符`（只有一个 style 标签，不累积）；`errors=[]`、`consoleLog=[]`。

**安装面与无报错**

- `dsh plugin --profile web add <tgz>` EXIT=0（有 peerDependencies WARN 不阻断）；profile `dsh.profile.bundles` 含 `dsh-result-only-view`；运行页有 1 个插件样式标签、开关按钮、`/plugins` 资源列表含 `dsh-result-only-view/client.js`。
- 除一次测试侧 BOM 事故（见第三节）外，全程无 pageerror / console error。

---

## 二、uncertain 与 untested（不可写成缺陷）

**uncertain — 冷重启后状态保留**
宿主用 `--port 0`，五次启动 origin 分别是 `9798 / 4133 / 11114 / 5315 / 12495`；`localStorage` 按 origin 隔离，新 origin 读到 `dsh.result-only-view.enabled = null`（源码缺省视为开）。本测法下「冷重启仍保留」原理上不可能成立，无法归因到插件。同一 origin 内 reload 读写正常。**未用固定端口复测。**

**untested — 工具调用卡是否隐藏**
三次造含工具调用的合成会话都被宿主格式校验拒绝：`step/end leaves unresolved tool call call-1` → `tool/result requires toolCallId matching its tool source` → `tool/call call-1 has no advertised tool lifecycle`。同因三次，按纪律停。这是夹具问题，不是插件问题。

**untested — 白名单卡片（`ask_user_question` / `cordis_run`）**
本轮会话里没有这两类卡片，无观测对象。README 承诺不被隐藏，但未验不等于成立。

**untested — 0 轮会话**
空会话（只有 `session/title`）即使登记进 `workspace.json` 的 `sessionIds`，侧栏也不显示，点不进去。

---

## 三、测试链路自伤两例（复查者必读，勿误判为插件缺陷）

1. **`button[aria-pressed]` 有 4 个**：自动换行 / 好的回答 / 有问题的回答 / 只看结果。取第一个会点到「自动换行」，表现为「点了开关但思考行没变」。正确做法是按文案匹配「只看结果」，并断言祖先槽含 `conversation.input.left`。
2. **PowerShell `Set-Content -Encoding utf8` 会写 UTF-8 BOM**：改写 `isolated-home/dsh/storages/workspace.json` 后宿主解析失败，页面显示「暂无会话」、侧栏 0 行、控制台报 `[session-controller] control stream failed: … Service "sessionController" is unavailable`。改用 Node `fs.writeFileSync(..., 'utf8')`（首字节 `{`）立即恢复。
3. 另外：宿主**不热扫** `sessions` 目录，新合成会话必须把 id 追加进 `workspace.json` 的 `sessionIds` 再重启才可见。

---

## 四、环境

DSH 0.2.0-rc.2 / Node v24.19.0 / npm 11.17.0 / pnpm 11.7.0 / Edge 无头（Playwright）/ Windows x64 / 1440×1000 / 隔离 `DSH_HOME`、`HOME`、`APPDATA`、npm cache / `--host 127.0.0.1 --port 0 --no-open` / 网络只放行本机。用户 GUI（1241 / 2538 / 4756）未访问。

自建宿主 5 个（pid 13324 / 37028 / 3552 / 3520 / 24896）已全部停止，对应端口 9798 / 4133 / 11114 / 5315 / 12495 均无监听。

---

## 五、证据

投稿时把下面的相对文件名替换为链接：

```text
evidence/run1-04-inventory.json                四个 aria-pressed 控件的清点，定位被测开关
evidence/run1-05-core.json                     开关即时性 / 隐藏与保留 / 三态还原 / 长内容
evidence/run1-05-core.stdout.txt               同上的一次运行完整输出
evidence/run1-06-persist-anomaly.json          localStorage 写入、展开中切换、20 次快速切换
evidence/run1-07-followup.json                 刷新保留、切会话、痕迹行展开与计数
evidence/run1-07-trace-click.png               痕迹行点击展开后的画面
evidence/run3-10-tool.json / run5-12-tool.json 工具卡夹具三次失败的宿主错误原文
evidence/diff-01-1.7.0-vs-47cfe87.txt          发布件 1.7.0 与仓库 1.6.4 逐文件哈希
evidence/diff-02-keys-and-features.txt         两版共有的实现键与 1.7.0 变更点
evidence/install-02-add-tarball.txt            安装面（tgz 方式）输出与退出码
evidence/dedup-01-lingshulab.txt               评测站全量 172 条扫描，命中 0
evidence/dedup-02-bounty-github.txt            悬赏站 5 个查询，同插件命中 0
```

---

## 六、查重

- `review.lingshulab.com`：`/api/reviews` 全量 4 页共 **172** 条（50/50/50/22）本地扫描，按 `result-only-view` / `YEYEYEYESHIFU` / `result only` / `只看结果` 匹配 → **命中 0**。
- `github.com/FuRongJun-1999/dsh-bounty/issues`：5 个查询，`YEYEYEYESHIFU` **0 条**；`dsh-result-only-view` 11 条、`result-only-view` 11 条、`只看结果` 482 条、`result only` 75 条，逐条比对标题后**同插件重复 0**（都是别的插件被分词命中）。
- 查询时间 `2026-10-22T01:17:35Z`。两站均查清，**可写首报**。

---

## 七、建议（不构成缺陷）

无。若维护者愿意收改进建议，只有一条可选：README 里「默认开启」这一点对使用者是隐性的——首次打开插件时开关就是「◉」态，用户可能以为是自己之前设置的。在设置行的提示文案里点明「默认开启」会更清楚。这只是文案建议，本轮没有把它当缺陷。
