# 快照与上下文

为 AI 提供记忆的基础。

- `carryctx checkpoint`: 保存包含 Git Diff 与当前进度的快照。
- `carryctx context`: 导出当前上下文（非常适合放入 LLM Prompt 中）。
- `carryctx resume`: 恢复中断的上下文指南。