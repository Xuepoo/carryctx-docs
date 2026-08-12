# 交接 (Handoff)

在 Agent 之间转移未完成的工作。

- `carryctx handoff create --target <AGENT|ULID|ROLE> [--task <ID>] [--summary <TEXT>]`: 创建交接请求。
- `carryctx handoff list [--status <status>] [--all] [--for-agent <AGENT>]`: 列出交接（默认仅未处理项）。
- `carryctx handoff show <REF>`: 查看交接详情。
- `carryctx handoff accept <REF> [--claim-task]`: 接受交接；`--claim-task` 同时认领关联任务（转入 in_progress）。
- `carryctx handoff reject <REF> [--reason <TEXT>]`: 拒绝交接。
- `carryctx handoff close <REF>`: 关闭不再相关的交接。

交接是多人协作的核心流程：创建后目标 Agent 通过 `handoff list` 发现待处理请求，`handoff accept` 后任务所有权可在同一事务内转移给接受方。
