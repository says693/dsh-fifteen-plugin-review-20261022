# dedup-gh-issues-all.json 未收录说明

原始文件 16.6 MB，是悬赏站 issue 全量查重的原始响应。其中包含安全测评用的**虚构**测试凭据字符串
（如 `ghp_CANARY...`、`ghp_FAKESENTINEL...`、`sk-CANARY...`，均为非真实样例值），
触发 GitHub 的 secret 扫描而被拒绝入库。

查重结论本身已写入同目录上级的 `PROVENANCE.json` 与 `RESULTS.json`：
`dsh-turn-index` / `Simon314620` / `turn-index` 在悬赏站与评测站均无同范围重复，判定为首报。
若需原始响应，可按 `REPRODUCE.md` 的查重小节重新拉取。
