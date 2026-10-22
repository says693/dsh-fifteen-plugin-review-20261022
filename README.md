# DSH 第三方插件评测证据（2026-10-22 批次，14 个）

本仓库是 14 个 DSH 第三方插件在 **DeepSeek-Harness 0.2.0-rc.2** 上的真实前端行为评测证据。
每个目录含该插件的技术报告、评测站稿件、证据清单与原始读数（JSON / 截图 / 宿主日志）。

## 测试环境

- 宿主：`@deepseek-ai/dsh` 0.2.0-rc.2（隔离 `DSH_HOME` 与 profile，`--host 127.0.0.1 --port 0 --no-open`）
- 浏览器：Edge 155.0.4283.62 + Playwright 1.64.1 headless
- 会话数据：官方 `dsh-session-persistence-jsonl` 合成会话，不调用模型、不产生费用
- 凭据：全程未读取任何真实 API Key / token / cookie；文中出现的 `CANARY` / `FAKESENTINEL` / `PROBE-NOT-A-REAL-KEY` 均为虚构测试值
- 脱敏：URL 中的 token 已替换为 `REDACTED`，用户目录已替换为 `C:\Users\USER`

## 目录

| # | 插件 | 版本 | 仓库 | 一句话结论 |
|---|---|---|---|---|
| [01-navbar](01-navbar/) | `@vlln/dsh-navbar` | 0.4.0 | [vlln/dsh-navbar](https://github.com/vlln/dsh-navbar) | 导航条不出现：取数锚点 data-time-hover-root 已不存在；滚动高亮整段卡住；超长预览未截断 |
| [02-minimap](02-minimap/) | `dsh-conversation-minimap` | 1.3.4 | [Braidy-Wu/dsh-conversation-minimap](https://github.com/Braidy-Wu/dsh-conversation-minimap) | 阻断：完全不渲染（读 sessions.list.current，字段已移除）；连点描边残留；深色对比度 2.08:1 |
| [03-turn-index](03-turn-index/) | `dsh-turn-index` | 0.1.1 | [Simon314620/dsh-turn-index](https://github.com/Simon314620/dsh-turn-index) | 面板从不出现（useSessions(s=>s.current) 恒 undefined）；无宿主版本约束 |
| [04-result-only-view](04-result-only-view/) | `dsh-result-only-view` | 1.7.0 | [YEYEYEYESHIFU/dsh-result-only-view](https://github.com/YEYEYEYESHIFU/dsh-result-only-view) | 主路径全部按文档工作，未发现可提交缺陷（3 项未测） |
| [05-approval-hotkeys](05-approval-hotkeys/) | `dsh-approval-hotkeys` | 0.2.0 | [SiriLee/dsh-approval-hotkeys](https://github.com/SiriLee/dsh-approval-hotkeys) | 连按 Enter 会批准尚未阅读的下一条；同一 tick 双 Enter 对同一按钮点两次 |
| [06-composer-history](06-composer-history/) | `dsh-composer-history` | 0.8.15 | [PerryLink/dsh-composer-history](https://github.com/PerryLink/dsh-composer-history) | 方向键召回/Ctrl+R 反搜/冷重启保留均按文档工作，未发现可提交缺陷 |
| [07-session-pin](07-session-pin/) | `dsh-session-pin` | 0.7.19 | [PerryLink/dsh-session-pin](https://github.com/PerryLink/dsh-session-pin) | 导航条点击会关掉「已置顶」面板，看板/标签/已存视图在鼠标路径下不可用；另 7 条不一致 |
| [08-session-hotkeys](08-session-hotkeys/) | `dsh-session-hotkeys` | 1.9.5 | [YEYEYEYESHIFU/dsh-session-hotkeys](https://github.com/YEYEYEYESHIFU/dsh-session-hotkeys) | 18 个默认键位三方一致、改绑/恢复/冷重启保留均可复现；两处提示语与交互可改进 |
| [09-milestone](09-milestone/) | `dsh-milestone` | 0.7.2 | [SnowCrescenter-tech/dsh-milestone](https://github.com/SnowCrescenter-tech/dsh-milestone) | 时间轴整条不渲染（useSession(s=>s.queue) 抛 TypeError 被兼容边界停用）；安装期核心包被遮蔽 |
| [10-task-status](10-task-status/) | `@vlln/dsh-task-status` | 0.3.1 | [vlln/dsh-task-status](https://github.com/vlln/dsh-task-status) | 状态条从不渲染；/tasks 漏掉有主任务；/output 恒 404 且破坏官方 read 契约 |
| [11-settings-ui](11-settings-ui/) | `@weibaohui/dsh-settings-ui` | 0.2.10 | [weibaohui/dsh-settings-ui](https://github.com/weibaohui/dsh-settings-ui) | 自定义宽高输入框无法清空重输；只有下界没有上界；缺可访问名称；发布件无 LICENSE |
| [12-sticky-disclosure](12-sticky-disclosure/) | `dsh-sticky-disclosure` | 1.0.1 | [Han-1413141/dsh-sticky-disclosure](https://github.com/Han-1413141/dsh-sticky-disclosure) | 常驻计数与一键收起按文档工作，未复现出可提交缺陷（5 项未验） |
| [13-cost-meter](13-cost-meter/) | `@steven-wu/dsh-cost-meter` | 0.1.0-rc.3 | [Sttrevens/dsh-cost-meter](https://github.com/Sttrevens/dsh-cost-meter) | 徽标在宿主 0.2.0-rc.2 上一个都不出现；折叠与落盘正确，客户端视图被丢弃 |
| [14-deeplink](14-deeplink/) | `@dsh-community/dsh-deeplink` | 0.6.1 | [qyw233/dsh-deeplink](https://github.com/qyw233/dsh-deeplink) | 宿主 ?token= 鉴权重定向吞掉整个 query；sessions.open / workspaces.connectWorkspace 不存在 |

## 四态边界

每条结论标注其一：**已验**（有直接读数）、**未验**（环境或前置条件不具备，附原因）、**不确定**（有读数但归因不能定论）、**不做**（付费模型/真实密钥/真实用户数据类调用）。
报告中不含总分、星级或排名；结论限定在实测版本上。

## 复现

各目录 `REPRODUCE.md` 给出可复跑链路；多数任务的关键前提是：先 `--from-default-profile web` 派生 profile 再 add 插件（直接 add 会缺 `dsh-web-app`，宿主静默失败）。
