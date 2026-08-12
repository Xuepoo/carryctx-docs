# 会话与进度

微观层面追踪你的思路。

- `carryctx session start`: 开启会话 (自动推断活跃任务)。
- `carryctx session pause`: 暂停会话计时。
- `carryctx session resume`: 恢复暂停的会话。
- `carryctx session end`: 结束会话。
- `carryctx progress todo <TEXT>`: 记录待办事项。
- `carryctx progress block <TEXT>`: 记录阻塞点。
- `carryctx progress risk <TEXT>`: 记录风险。
- `carryctx progress note <TEXT>`: 记录一般性备注或观察。
- `carryctx progress list`: 列出任务下的进度条目。
- `carryctx progress show <REF>`: 查看进度条目详情。
- `carryctx progress edit <REF> --content <TEXT>`: 修改条目内容。
- `carryctx progress complete <REF>`: 标记条目为已完成。
- `carryctx progress reopen <REF>`: 重新打开已完成的条目。
- `carryctx progress remove <REF>`: 永久删除条目。
- `carryctx progress reorder --task <ID> --order <REF...>`: 调整条目顺序。
