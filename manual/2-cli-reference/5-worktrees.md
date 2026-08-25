# Git 工作区 (Worktree)

隔离的并行开发空间。

- `carryctx worktree create <BRANCH>`: 结合任务快速创建独立工作区。
- `carryctx worktree list`: 查看工作区映射。
- `carryctx worktree remove <REF>`: 删除工作区并移除注册；`<REF>` 接受 CTX
  编号、Worktree ULID 或路径，脏工作区需加 `--force`。
