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

2026-10-11 标题补漏：手工审阅并补入 10 条标题记忆，覆盖 11 个文章主标题和 15 个目录字段，
总计 18,449 条有效记忆。新增条目以 `provider=manual`、`model=codex-semantic-review` 和零 Usage
标记来源，不能算成供应商付费结果；全部通过 Web 仓库原有 Hash/AST/技术标识符校验。
中文 Markdown、文件名、URL、锚点和作者正在编辑的北极熊草稿均未纳入本次提交。

缺失键与已终止标题任务的 10 个 `failedKeys` 完全对应。旧任务只持久化统一的校验/未知
计费错误，没有保存失败响应，因此不能再精确归因每次请求；本次没有恢复任务、重发请求或
追加预算。保留原失败和保守费用记录，详情及回归 Fixture 见 Web 仓库国际化补漏文档。
