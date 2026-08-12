# 任务管理 (Task)

创建与流转工作单元。

- `carryctx task create`: 创建新任务，支持 `--depends-on`。
- `carryctx task list`: 查看任务列表（支持状态、归属等过滤）。
- `carryctx task show <ID>`: 查看任务完整详情。
- `carryctx task edit <ID> --title/--priority/--description`: 编辑任务。
- `carryctx task claim <ID>`: 认领任务（转入 in_progress）。
- `carryctx task release <ID>`: 释放任务所有权。
- `carryctx task start <ID>`: 开始任务。
- `carryctx task block <ID> --reason`: 阻塞任务。
- `carryctx task unblock <ID>`: 解除阻塞。
- `carryctx task review <ID>`: 标记为待审核。
- `carryctx task complete <ID>`: 标记任务完成。
- `carryctx task cancel <ID> --reason`: 取消任务。
- `carryctx task reopen <ID>`: 重新打开已完成/取消的任务。
- `carryctx task depend <ID> --on <DEP_ID> [--kind strong|informational]`: 建立任务依赖。
- `carryctx task undepend <ID> --on <DEP_ID>`: 移除任务依赖。
