# 项目与生命周期

CarryCtx v0.10.0 是面向 Agent 与人类协作者的、local-first 的**全项目生命周期持久化与控制层**。
它把项目契约、任务关系、协作身份、工作会话、Git 工作区、进度、交接和审计记录保存在一个可恢复的项目状态中，使工作能够跨 Agent、窗口、Session
和 worktree 延续。

CarryCtx 不负责调度项目流程或执行 Agent。外部 harness 负责过程调度、Agent 执行、模型选择和验证；CarryCtx 负责持久化状态、提供确定性查询与安全的状态转换。
v0.10.x 不提供通用的 Completion Gates 或已发布的 Automation Engine。
`task.strict_completion` 和 evidence checkpoint 是可选的任务/检查点策略，不是独立的自动化编排系统。

## 生命周期链

```text
init / project contract
  → planning / dependencies
  → team roles
  → sessions / worktrees
  → progress / checkpoints
  → handoffs / review
  → cleanup outbox / reconciliation / policies
  → audit / analytics
  → release evidence
  → state exchange / merge (export · import · snapshot ref)
```

1. **初始化与项目契约**：运行 `carryctx init` 创建项目声明配置、SQLite 状态和
   `project.initialized` 事件；使用 `status`、`doctor` 和 `project show` 检查项目健康、配置与迁移状态。
2. **规划与依赖**：用 `task create` 建立工作单元，用 `task depend` 声明强依赖或信息依赖；
   `task list --ready` 只呈现满足强依赖条件的候选任务。任务状态转换由 CarryCtx 校验，
   但如何拆分和安排任务由人类或外部 harness 决定。详见 [`tasks`](2-tasks.md)。
3. **团队角色**：用 `agent register` 保存稳定的 Agent 或人类身份，用 `team` 保存 commander、
   成员和描述性角色关系。Team 是项目范围内的持久协调记录，不是 worker pool、scheduler、
   Session pool 或并发策略。详见 [`teams`](../../cli-specification.md#team-coordination-v060)。
4. **Session 与 worktree**：通过 `session start/pause/resume/end` 记录工作会话；需要隔离时用
   `worktree bind/create/list` 关联 Git 工作区。Session 结束不会自动完成任务。详见
   [`sessions`](3-sessions-and-progress.md) 与 [`worktrees`](5-worktrees.md)。
5. **进度与检查点**：用 `progress todo/block/risk/note` 记录可恢复的工作事实，用 `checkpoint`
   保存完成项、剩余项、风险、阻塞、下一步和 Git 状态；`resume` 与 `context` 从这些记录重建
   下一步工作。可按项目策略启用 `[task] strict_completion = true`，并在需要时要求 evidence
   checkpoint，但这些仍是持久化检查与提示，不是自动验证器。详见 [`checkpoints`](4-checkpoints.md)
   和 [`configuration`](../../configuration.md#31-carryctxconfigtoml)。
6. **交接与 Review**：用 `handoff create/list/show/accept` 在 Agent、Session 或人类协作者之间
   转移上下文；用 `task review` 表示等待 Review 或验收，完成或取消必须通过明确的任务转换。
   Terminal 任务默认冻结；确需改正 completed/cancelled 任务时，使用 `task edit --force`，这会
   记录 `task.corrected` 审计事件。详见 [`handoffs`](7-handoffs.md) 与 [`tasks`](2-tasks.md)。
7. **清理、outbox 与策略**：任务结束后的 worktree 清理是持久化、可重试的 cleanup 请求，而不是
   无条件立即删除。`carryctx worktree cleanup list/show/run` 查看、重试和 reconciliation；
   `doctor` 报告 pending、blocked 或 failed 请求。清理策略可配置 clean worktree、无 active
   session、分支删除方式等安全条件，失败不会静默视为成功。详见
   [`configuration` 的清理策略](../../configuration.md#8-worktree-生命周期清理) 与
   [`worktree` 规范](../../cli-specification.md#18-carryctx-worktree)。
8. **审计与分析**：关键状态变化在同一事务中写入 append-only Event Log；使用 `event list/show`
   追踪责任、时间和状态转换，使用 `stats` 汇总 Agent 工时、任务完成率和图谱复杂度。详见
   [`analytics`](../3-ecosystem/3-analytics.md) 与 [`event` 规范](../../cli-specification.md#21-carryctx-event)。
9. **发布证据**：发布前由外部 harness 执行测试、Lint、构建和其他验证，并将结果、commit、已知缺口
   和相关 checkpoint/handoff 作为 release evidence 保存。CarryCtx 可保存这些进度与检查点并提供
   审计查询，但不替代验证工具或发布流程。
10. **状态交换与合并**（merge milestone，0.10.0 起）：用
    `carryctx export --pack-format dir -o <dir>` 生成 ctxpack（v2：`parents`
    DAG、`tombstones`、`redacted` 标记；v1 仍可读一个 release cycle），用
    `carryctx import <dir>` 做 fresh/replace 导入，用
    `carryctx import <dir> --mode merge`（可配 `--base`、`--require-base`、
    `--strict-edits`）做三方合并；阻断冲突以 `MERGE_CONFLICTS`（exit 3）落到
    `<git-common-dir>/carryctx/merges/<id>/`，由
    `conflict list/show/resolve/apply/abort` 处理。`export --snapshot` /
    `import --from-git` 用本地 Git ref 离线携带快照 DAG；二进制从不
    push/fetch，Git 传输由用户完成。完整命令与退出码见
    [`export/import` 规范 §12.1](../../cli-specification.md) 与
    [`conflict` 规范 §12.2](../../cli-specification.md)。

## 状态边界

项目权威状态位于：

```text
<git-common-dir>/carryctx/state.sqlite
```

所有 linked worktree 共享该数据库；`.carryctx/` 只保存项目声明配置和扩展，不是运行时数据库。
CarryCtx Core 不发起网络连接。`carryctx sync push/pull` 若使用，仅是针对本地文件系统路径的
whole-file、last-writer-wins 复制，不是云同步或冲突合并。详见 [`存储与配置`](../../configuration.md)
与 [`Zero Network Policy`](../../architecture/zero-network-policy.md)。

项目状态通过 ctxpack 目录在机器之间交换（`export`/`import`），它是互操作契约而不是运行时数据库。
merge milestone 的合并 staging 位于 `<git-common-dir>/carryctx/merges/`，快照 DAG
由本地 Git ref 承载；这些机器本地目录从不自动上传或同步。详见
[`export/import` 规范 §12.1](../../cli-specification.md)。

检测到 Git 与 `.jj/` 并存的 colocated 仓库时，CarryCtx 对不安全的 Git worktree 操作 fail closed：
`worktree create` 拒绝执行，仍存在的 Git worktree 也不能通过 `worktree remove` 删除。请直接使用
`jj workspace add`；若要纳入 CarryCtx 追踪，在主仓库中使用 `worktree bind`。jj 下 checkpoint
应使用 `changed_files`，不要把 staged/modified/untracked 三分法当作可靠语义。详见
[`jj 兼容性说明`](../../plans/2026-07-25-jujutsu-compatibility.md)。

常用入口：

```bash
carryctx status
carryctx resume
carryctx context
carryctx doctor
carryctx export --pack-format dir -o ./ctxpack-dir/    # 状态交换
carryctx import ./ctxpack-dir/ --mode merge             # 三方合并
```
