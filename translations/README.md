# 英文翻译记忆

中文 Markdown 仍在 `content/`；英文 Markdown 不提交 Git。
`en-us/manifest.json` 与 `en-us/v1/<SHA256 前两位>.json` 保存可复用的语义块翻译。
网页服务器只导入 JSON 并在 PostgreSQL 中拼装英文，不调用付费 API。

在 `tungchiahui_web` 使用：

```bash
./site translate pending --content-root ../tungchiahui_content --dry-run
./site translate pending --content-root ../tungchiahui_content --execute --budget-usd 3 --key-file /private/deepseek-key.json
./site translate validate --content-root ../tungchiahui_content
./site translate status --content-root ../tungchiahui_content
```

Key 放在两个仓库之外，权限 0600；任务/费用/锁保存在开发机私有目录，不提交。
先提交中文变更，再显式运行翻译。未变块复用，未译/删除块显示当前中文。
JSON 修改经验证、Review、PR 合入 main 后只触发 Content Sync，不构建网站镜像。
删除有效记忆会让关联内容回退中文；回退采用新的 Revert Commit。
断点恢复使用同一 `--job-id`、源内容、Scope 和预算；不自动重复未知已付费请求。
具体架构/预算/发布与恢复详见 Web 仓库 ADR 0028 和 translation-operations.md。
