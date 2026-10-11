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

2026-10-11 北极熊日志补译：以已发布中文提交
`1a35f7339e88679083389ea1dbc4467beeb814d8` 为来源，补入 330 条有效英文记忆，
总计 18,779 条。该文章的 1,237 个唯一可翻译正文块和主标题均有匹配记忆；
解释正文无中文回退，代码、公式、技术标识符、链接和中文来源保持不变。

125 个旧不确定块人工补译，不恢复旧任务或重发旧请求。新任务只请求 205 个新块，
其中 173 个结果通过验证；32 个校验失败／计费不确定块同样人工补齐，不重试。
157 条人工结果标记为零 Usage；付费任务的失败及保守费用记录完整保留。
Owner 继续授权总额 3 美元：旧任务保守记账 2.423444 美元，本轮 0.120317 美元，
累计 2.543761 美元，没有追加预算。新任务绑定已提交来源，正在编辑的两处横滚
参数草稿没有发送给供应商，也没有进入翻译提交。

发布前执行全量记忆校验、62 项翻译／Markdown 回归，以及文章完整拼装、中文代码
保护和中文来源锚点校验。仍按内容 PR 和 Content-only Sync 发布；不构建网站镜像。
英文锚点沿用中文来源；HK/TW 保持既有确定性转换。回退使用新 Revert Commit，
不会删除本地付费记录；移除这些记忆后相关英文块安全回退中文。
