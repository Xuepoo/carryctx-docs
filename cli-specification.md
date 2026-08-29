# CarryCtx CLI 命令规范

**文档路径：** `carryctx-docs/cli-specification.md`
**文档版本：** v0.6.0
**适用版本：** CarryCtx v0.6.x

---

## 1. 设计目标

CarryCtx CLI 应同时服务于：

- Coding Agent
- 人类开发者
- Shell Script
- CI
- Agent Skill
- 后续 MCP Adapter

CLI 必须具备：

- 稳定的命令名称
- 确定性的行为
- 机器可读 JSON
- 明确的 Exit Code
- 非交互模式
- 可审计的写操作
- 可预测的配置解析
- 清晰的错误恢复建议

---

# 2. 基础调用形式

```text
carryctx [global-options] <command> [subcommand] [arguments]
```

示例：

```bash
carryctx status
carryctx task list --status in_progress
carryctx checkpoint --done "Implemented state store"
carryctx resume --json
```

---

# 3. 全局参数

```text
--project <path>
--config <path>
--profile <name>
--agent <agent-id>     # 别名: --owner
--session <session-id>
--task <task-id>
--format <text|json|markdown>
--json
--no-color
--quiet
--verbose
--yes
--dry-run
--non-interactive
--config-compat <error|warn>
--version
--help
```

---

## 3.1 `--project`

显式指定项目路径：

```bash
carryctx --project ~/Develop/carryctx status
```

该路径可以是：

- Repository root
- Repository 子目录
- Linked worktree
- `.git` 指向的 worktree

---

## 3.2 `--config`

加载额外配置文件：

```bash
carryctx --config ./ci/carryctx.toml status
```

显式配置覆盖全局和项目配置，但低于环境变量与 CLI 参数。

---

## 3.3 `--json`

等价于：

```bash
--format json
```

JSON 模式下：

- 不输出 ANSI Color
- 不输出 Spinner
- 不输出交互提示
- `stdout` 只输出结果 JSON
- 错误 JSON 输出到 `stderr`
- 必须设置非零 Exit Code

---

## 3.4 `--non-interactive`

禁止所有交互式提示。

缺少必要信息时直接返回错误。

以下环境默认启用非交互模式：

```text
CI=true
CARRYCTX_NON_INTERACTIVE=true
stdin is not a TTY
```

---

## 3.5 `--dry-run`

对于支持的写操作，仅输出计划变更，不写入：

- SQLite
- Git
- 配置文件
- 文件系统

例如：

```bash
carryctx task claim CTX-0012 --dry-run
```

---

# 4. 项目解析顺序

CarryCtx 按以下顺序确定当前项目：

1. `--project`
2. `CARRYCTX_PROJECT`
3. 从当前目录执行 `git rev-parse --show-toplevel`
4. 全局 registry 中当前目录的项目映射
5. 返回 `PROJECT_NOT_FOUND`

CarryCtx 不应因为当前目录没有 `.carryctx/` 就立即失败。

应先确定 Git repository root，再检查是否初始化。

---

# 5. 当前实体解析

## 5.1 当前 Agent

顺序：

1. `--agent`
2. `CARRYCTX_AGENT`
3. 当前 Session 的 Agent
4. 项目本地配置
5. 全局配置
6. 唯一活跃 Agent
7. 交互式选择
8. 返回 `AGENT_NOT_RESOLVED`

---

## 5.2 当前 Task

顺序：

1. `--task`
2. `CARRYCTX_TASK`
3. 当前 Session
4. 当前 worktree
5. 当前 Agent 恰好唯一的 active Task；如果有多个则进入交互式选择
6. 交互式选择
7. 返回 `TASK_NOT_RESOLVED`

---

## 5.3 当前 Session

顺序：

1. `--session`
2. `CARRYCTX_SESSION`
3. 当前终端环境绑定
4. 当前 Agent 在当前 worktree 的唯一 active Session
5. 返回未解析状态

查询命令可以在没有 Session 时运行。

需要 Session 的写命令必须明确报错。

---

# 6. 一级命令

```text
carryctx init
carryctx status
carryctx resume
carryctx context
carryctx checkpoint
carryctx doctor
carryctx config
carryctx project
carryctx agent
carryctx session
carryctx task
carryctx team
carryctx progress
carryctx worktree
carryctx decision
carryctx handoff
carryctx event
carryctx search
carryctx skill
carryctx graph
carryctx mcp
```

---

# 7. `carryctx init`

初始化当前项目：

```bash
carryctx init
```

选项：

```text
--name <name>
--task-prefix <prefix>
--main-branch <branch>
--force
--minimal
--install-skill
```

`--force` 只能重建或修复项目声明文件和缺失 Schema，不得覆盖可读取的状态数据库、修改 Project ID 或删除 Task。交互模式必须确认；非交互模式必须同时使用：

```bash
carryctx init --force --yes --non-interactive
```

行为：

1. 定位 Git repository
2. 创建 `.carryctx/config.toml`
3. 创建 Git common state directory
4. 初始化 SQLite
5. 运行数据库迁移
6. 注册项目
7. 写入 `project.initialized` Event
8. 可选安装 Skill

重复执行必须保持幂等。

---

# 8. `carryctx status`

显示项目当前状态：

```bash
carryctx status
```

选项：

```text
--mine
--all
--compact
--sessions
--tasks
--worktrees
```

`--worktrees` adds a Git worktrees table to the Markdown report. JSON output
always embeds the full `worktrees` array regardless of this flag. The former
`--since <duration>` filter was removed in 0.6.0: it had no effect on the
report and silently ignored its argument.

任务计数与上限（0.6.0 起）：JSON 输出在既有键之外新增 `totalTasks`——项目任务
总数由精确 COUNT(*) 查询得出，不受 `task list` 默认上限（200）影响；`tasks`
数组仍是有上限的一页数据。Markdown 报告的 Total Tasks 行同样使用该精确计数。
`carryctx doctor` 的 orphaned / in-progress 诊断也改为全量查询统计，超过上限
的任务同样计入，不再因分页截断而漏报。

默认输出：

- Project
- Current Agent
- Current Session
- Current Task
- Git State
- Active Sessions
- Task 状态计数
- 最近活动
- Blocker
- Scope 冲突

`--mine` 仅显示当前 Agent 相关内容。

`--all` 显示全部活跃任务和 Session。

---

# 9. `carryctx resume`

恢复当前工作上下文：

```bash
carryctx resume
```

选项：

```text
--task <task-id>
--session <session-id>
--compact
--full
--start-session
--include-diff
--max-events <number>
```

默认流程：

1. 解析项目
2. 解析 Agent
3. 解析 Task
4. 查询最新 Checkpoint
5. 读取 Progress Item
6. 读取依赖
7. 读取相关任务
8. 获取当前 Git 状态
9. 比较 Checkpoint 与当前 Git 状态
10. 生成下一步提示

`resume` 不得自动：

- 修改任务
- 创建 Checkpoint
- 提交 Git
- 更改 Task Owner

使用：

```bash
carryctx resume --start-session
```

时可以在不存在 Session 的情况下创建 Session。

---

# 10. `carryctx context`

生成 Agent 上下文：

```bash
carryctx context
```

选项：

```text
--compact
--full
--task <task-id>
--include-decisions
--include-events
--include-related-tasks
--max-events <number>
--since <duration>
--output <path>
```

格式：

```bash
carryctx context --format text
carryctx context --format markdown
carryctx context --format json
```

`--compact` 是 Agent Skill 的默认模式。

---

# 11. `carryctx checkpoint`

创建 Checkpoint：

```bash
carryctx checkpoint
```

查询与修正：

```text
carryctx checkpoint list
carryctx checkpoint show <checkpoint-id>
carryctx checkpoint correct <checkpoint-id>
```

选项：

```text
--done <text>
--remaining <text>
--blocker <text>
--risk <text>
--next <text>
--note <text>
--task <task-id>
--session <session-id>
--no-git
--include-diff
```

同一个参数可以重复：

```bash
carryctx checkpoint \
  --done "Implemented parser" \
  --done "Added parser tests" \
  --remaining "Add invalid input cases"
```

自动采集：

- Branch
- HEAD
- Dirty State
- Staged Files
- Modified Files
- Untracked Files
- Diff Stats
- VCS Backend（`vcs_backend`: `"git"` 或 `"jj"`）
- Changed Files（`changed_files`：跨两种后端都准确的合并文件列表）

当检测到 [Jujutsu (jj) colocated 仓库](../plans/2026-07-25-jujutsu-compatibility.md)（`.git/` 旁存在 `.jj/`）时，`vcs_backend` 为 `"jj"`，且 `staged_files`/`modified_files`/`untracked_files` 始终为空数组 —— jj 的自动工作副本快照机制会让这个三分法失去意义（只读命令也会写入 Git index）。此时应使用 `changed_files`，它在两种后端下都是准确的“变更文件”合并列表。`dirty` 与 Diff Stats 在两种后端下都保持准确。

Checkpoint 创建后不可直接修改。

修正命令：

```bash
carryctx checkpoint correct <checkpoint-id>
```

修正会创建新的 Correction Event。

---

# 12. `carryctx project`

```text
carryctx project show
carryctx project list
carryctx project register
carryctx project unregister
carryctx project migrate
carryctx project backup
carryctx project restore
carryctx project export
carryctx project import
```

## `project show`

显示：

- Project ID
- Repository Root
- Git Common Dir
- Database Path
- Main Branch
- Schema Version
- Config Sources

## `project migrate`

```bash
carryctx project migrate
```

必须：

- 检查目标版本
- 创建备份
- 事务执行
- 写入 Migration Event

---

# 13. `carryctx config`

```text
carryctx config list
carryctx config get
carryctx config set
carryctx config unset
carryctx config validate
carryctx config sources
carryctx config path
```

示例：

```bash
carryctx config get session.stale_after
carryctx config set --cfg-project task.list_limit 250
carryctx config set --global output.color auto
carryctx config sources
```

0.6.0 起 `config get` / `config set` / `config unset` 基于 `toml_edit`
实现类型化往返（typed round-trip）：

- 点号 key 按段定位到正确的 TOML table（`task.list_limit` 写入
  `[task]` 表），不再追加到文件末尾或落入错误的最后一个表。
- 值按 TOML 类型解析并写回：`true`/`250` 写为真正的 bool/integer，
  不再一律字符串化；写入前先序列化-重解析校验，失败即报错不落盘。
- `config get` 查询类型化配置树：未知 key 返回 `value: null` 且
  退出码 0；不再做行前缀文本匹配。
- 缺少写入作用域时返回明确的错误与建议（必须且只能指定一个）。
- `--local` 被显式拒绝，返回 `UNSUPPORTED_OPERATION`：配置加载器
  尚未读取 `.carryctx/local.toml`。

```text
--global
--cfg-project
--local        # 仅接受为显式拒绝的 UNSUPPORTED_OPERATION
```

v0.1 不提供隐式默认写入作用域。

---

# 14. `carryctx agent`

```text
carryctx agent register
carryctx agent list
carryctx agent show
carryctx agent current
carryctx agent rename
carryctx agent deactivate
```

Since v0.6.0, `agent register` accepts an optional execution kind:

```bash
carryctx agent register --name planner --kind commander --role planning
carryctx agent register --name worker-1 --kind subagent --role implementation
```

`--kind` accepts only `commander` or `subagent`. Omitting it leaves `kind` as
`null`, preserving pre-Team behavior. Registering a duplicate agent name is
rejected; it does not create a second Agent. `role` is descriptive metadata
and does not grant permissions or select execution policy.

0.6.0 起 Agent 身份完整性约束：

- 名称唯一性由迁移 `0015_agent_name_unique` 的
  `UNIQUE(project_id, name)` 索引强制；重复注册返回 `STATE_CONFLICT`
  并给出可操作建议（改名、rename 或 reactivate）。
- `agent deactivate` 的状态写入与其审计事件在同一事务中提交并持久化；
  deactivated 状态在 `agent show` 与所有解析路径中立即可见。
- Deactivated agent 无法在任何位置被解析或行动：任务创建/认领、session
  操作、`search --assignee`、`event list --agent` 等按名称过滤的路径
  统一返回 `PERMISSION_SCOPE`（"is deactivated and cannot act"）。

注册：

```bash
carryctx agent register \
  --name claude-core \
  --provider claude-code \
  --role engineer
```

Provider 为开放字符串，不使用封闭 Enum。

内置建议值：

```text
claude-code
opencode
github-copilot
kiro
codex
antigravity
human
custom
```

---

# 15. `carryctx session`

```text
carryctx session start
carryctx session list
carryctx session show
carryctx session current
carryctx session pause
carryctx session resume
carryctx session end
carryctx session abandon
```

启动：

```bash
carryctx session start \
  --agent claude-core \
  --task CTX-0001
```

选项：

```text
--reuse
--new
--provider <provider>
--metadata <key=value>
```

0.6.0 起 `--reuse` 生效：存在同 Agent 的 active session 时直接返回该
session（不新建、不改状态）；不存在时正常创建新 session
（reuse-if-active-else-new）。省略该参数时保持历史行为（总是创建新
session，旧 active session 按实现收尾/共存）。`session abandon` 与其
`--reason` 参数的行为在本版本未改变：reason 目前仍被解析但不持久化。

结束：

```bash
carryctx session end \
  --summary "Paused before migration tests"
```

没有最新 Checkpoint 时：

- TTY：提示创建
- Non-interactive：返回 Warning 或根据 strict 配置失败

---

# 16. `carryctx task`

```text
carryctx task create
carryctx task list
carryctx task show
carryctx task edit
carryctx task claim
carryctx task release
carryctx task start
carryctx task block
carryctx task unblock
carryctx task review
carryctx task complete
carryctx task cancel
carryctx task reopen
carryctx task depend
carryctx task undepend
carryctx task scope
```

---

## 16.1 创建任务

```bash
carryctx task create \
  --title "Implement SQLite migrations" \
  --description "Create transactional migration runner" \
  --priority high
```

选项：

```text
--parent <task-id>
--depends-on <task-id>
--scope <glob>
--assignee <agent-id>
--status <status>
--team <team-id-or-name>
--required-role <role>
```

`--team` associates a new Task with an existing Team in the current project.
`--required-role` stores an advisory role label for commander or harness
queries; it does not prevent claiming, starting, or assigning a Task. Both
fields are serialized as `team_id` and `required_role` in JSON. `task edit`
also accepts `--required-role`.

0.6.0 起创建约束：

- `--status` 只接受 `planned` 或 `ready`（白名单）。`ready` 还要求强依赖
  已全部完成；其余状态必须通过 claim/start/block/complete/cancel 等生命周期
  转换到达，不能在出生时伪造（例如直接铸造 completed 任务）。
- `--title` 最长 200 字符，`--description` 最长 8000 字符，超长返回
  `VALIDATION_FAILED`。
- `task list` 默认上限 200 条（可用 `[task].list_limit` 配置或
  `task list --limit` 覆盖）；列表按 `created_at DESC` 排序并由迁移
  `0016_task_list_index` 提供索引支撑。

---

## 16.2 Task 状态

```text
planned
ready
in_progress
blocked
review
completed
cancelled
```

非法状态转换必须返回：

```text
INVALID_TASK_TRANSITION
```

0.6.0 起 Complete 与 Claim/Start/Unblock 一样受强依赖门控：存在未完成的
strong prerequisite 时 `task complete` 返回
`DEPENDENCY_INCOMPLETE`，不允许带着 open blocker 收尾（单一领域谓词
`prerequisite_settled` 统一判定，completed/cancelled 均视为已结算，
创建与转换的门控不会漂移）。

## Team Coordination (v0.6.0)

Team coordination is a durable, project-scoped record set for a commander or
external execution harness. It is not a worker pool, scheduler, session pool,
or concurrency policy. Team, Agent, and Task references are resolved against
the current project; a record from another project cannot be used even when
its ID is known.

### Commands

```text
carryctx team create --name <NAME> [--commander <AGENT_REF>]
carryctx team member add <TEAM_REF> --agent <AGENT_REF> [--role <ROLE>]
carryctx team member remove <TEAM_REF> --agent <AGENT_REF>
carryctx team commander set <TEAM_REF> --agent <AGENT_REF>
carryctx team commander set <TEAM_REF> --clear
carryctx team status [<TEAM_REF>]
carryctx team context [<TEAM_REF>] [--agent-for <AGENT_REF>] [--task <TASK_REF>]
carryctx task team set <TASK_REF> --team <TEAM_REF|none>
carryctx task team unset <TASK_REF>
```

`team create --commander` atomically creates the Team, adds the commander as
its first member, and sets the commander. `team commander set` requires the
target Agent to already be a member; `--clear` temporarily leaves the Team
without a commander. Removing the current commander is rejected until the
commander is cleared or replaced. Duplicate membership is rejected. There is
no Team delete command in v0.6.0; an empty Team remains available for history.

Team and task mutations are transactional and audited with these event types:
`team.created`, `team.member_added`, `team.member_removed`,
`team.commander_changed`, and `task.team_changed`. A failed mutation leaves
the state and its audit event unchanged. `task team set --team none` and
`task team unset` both clear `team_id`; responses include the prior value as
`previous_team_id`.

### Read-only projections and sessions

`team status` and `team context` only read persisted project records. They do
not create, end, or reset a Session, change membership, change task ownership,
or associate a Task with a Team. `team status` reports Team metadata, members,
member kinds and roles, active session IDs, associated tasks, and counts. Its
`active_task_count` is descriptive only; it is not a capacity or liveness
signal.

`team context` rebuilds a structured projection from durable records. The
commander view may include members, associated tasks, dependencies, scopes,
progress, blockers, scope conflicts, conflicts, latest checkpoints, decisions,
handoffs, and recent events. A member view is task-relevant by default and can
be narrowed with `--task`; `--agent-for` selects the member view. Without a
Team reference, the command resolves a single Team from `--task`,
`--agent-for`, or the project only when exactly one Team exists. An explicitly
provided `--session` is still validated; an unknown session returns an error,
but a valid session is never changed by these read-only commands.

### JSON contract

Use `--json` for the stable machine-readable envelope. Team projections and
mutation data use `snake_case`, including `schema_version`, `project_id`,
`commander_agent_id`, `active_session_id`, `active_task_count`, `team_id`,
`required_role`, and `previous_team_id`. The command names are:
`team.create`, `team.member_add`, `team.member_remove`, `team.commander_set`,
`team.status`, `team.context`, `task.team_set`, and `task.team_unset`.

The status form with no Team reference returns `data.teams`. A single-Team
status returns `data.team`, `data.members`, and `data.counts`. Context returns
`data.team`, `data.view`, the projection arrays, and
`data.rebuild.source = "durable_records"`. Mutation responses include changed
`team`, `member`, `commander`, or `task` data and an `operation.applied`
boolean; `--dry-run --json` sets it to `false` and writes nothing.

### Team errors and exit codes

| Condition                                                                          | JSON error code                                                              | Exit code |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- | --------: |
| Team, Agent, Task, or member reference is absent in the current project            | `RESOURCE_NOT_FOUND` (explicit Team projection lookup uses `TEAM_NOT_FOUND`) |         7 |
| Commander is not a Team member, duplicate membership, or current commander removal | `STATE_CONFLICT`                                                             |         3 |
| Empty Team name                                                                    | `VALIDATION_FAILED`                                                          |         8 |
| Unknown `--kind` value                                                             | `VALIDATION_FAILED`                                                          |         8 |
| Database or migration backup failure                                               | `DATABASE_ERROR`, `BACKUP_FAILED`, or `BACKUP_INTEGRITY_FAILED`              |         5 |

JSON successes go to stdout and JSON errors go to stderr, with the standard
`schema_version`, `command`, `success`, `data`/`error`, `warnings`, and `meta`
envelope. Team mutation audit events carry the invocation Agent when one is
resolved; Team membership does not imply a current Session.

### Migration and backup behavior

Team fields are introduced by project migrations `0012_agent_teams` and
`0013_agent_kind_constraint`. When pending migrations are applied to an
already-versioned database, CarryCtx first creates and integrity-checks a
timestamped `state_*.sqlite` backup under the state database's `backups/`
directory. If the database has no migration history or is at version zero,
that pre-migration backup is not required. A backup creation or integrity
failure aborts the migration. Migrations 0012 and 0013 rebuild affected SQLite
tables as part of their schema changes; this does not change public Team
command semantics.

0.6.0 hardening 将最新 schema 版本推进到 **16**，新增三个迁移：

- `0014_cascade_task_refs`：重建 `handoffs` 与
  `checkpoint_corrections`，把 `task_id` / `checkpoint_id` 外键从
  NO ACTION 改为 ON DELETE CASCADE，使 prune 能在 FK=ON 下安全删除
  父行，不再依赖关闭外键强制或留下孤儿行。
- `0015_agent_name_unique`：幂等重建 `UNIQUE(project_id, name)`
  agent 名称唯一索引（修复历史表重建中可能丢失的约束）。
- `0016_task_list_index`：新增 `tasks(project_id, created_at DESC)`
  索引，支撑默认任务列表排序。

---

## 16.3 Claim

```bash
carryctx task claim CTX-0001
```

Claim 必须在一个 SQLite Transaction 中完成。

检查：

- Task 存在
- Task 可接管
- 强依赖已完成
- 当前未被其他 Agent 接管
- Agent active
- Worktree 冲突

一个 Agent 可以同时拥有多个 active/in-progress Task。`task claim`、`task start`
和 `task assign` 不会因为当前 Agent 已持有其他任务而失败。历史配置键
`task.single_active_task_per_agent` 仍可被接受以保持配置兼容，但它是 deprecated
且 advisory，不是 enforced cap，也不改变上述命令行为。现有配置无需立即迁移；
容量限制或批处理策略由外部 commander 或执行 harness 决定，CarryCtx 不新增框架
级别的 enforcement。

---

## 16.4 Dependency

```bash
carryctx task depend CTX-0002 --on CTX-0001
carryctx task undepend CTX-0002 --on CTX-0001
```

`depend` 支持：

```text
--kind <strong|informational>
```

默认使用 `strong`。

添加依赖前必须检测环。

---

## 16.5 Scope

```bash
carryctx task scope add CTX-0001 "src/storage/**"
carryctx task scope remove CTX-0001 "src/storage/**"
carryctx task scope list CTX-0001
carryctx task scope conflicts CTX-0001
```

v0.1 Scope 为软约束。

## 16.6 编辑任务

```bash
carryctx task edit CTX-0001 --title "New title"
carryctx task edit CTX-0001 --priority high
carryctx task edit CTX-0001 --description "Revised requirements"
carryctx task edit CTX-0001 --title "Corrected title" --force
```

`task edit` 只修改传入的参数，支持：

```text
--title <text>
--priority <low|normal|high|urgent>
--description <text>    # 0.5.4 起
```

owner 与 status 不使用 edit 修改，分别走 claim/release/start 等转换命令。
普通（非 terminal）编辑必须在一个 SQLite Transaction 中完成，并追加
`task.edited` 审计事件（payload 携带 before/after 的 title、priority、description、
required_role）。`--force` 不适用于 planned/ready/in_progress/blocked/review 任务；
这些任务继续使用不带 `--force` 的普通编辑路径。

对于 completed/cancelled 任务，普通编辑仍返回 `STATE_CONFLICT`。必须显式传入
`--force` 才能执行 correction；调用者必须是当前有效（active）的任务 owner，或
terminal transition 审计事件中的有效 actor。历史事件中以 agent name 保存的
actor 会按当前 active agent 解析；未知或已停用 actor 不得授权 correction。
Correction 与字段变更在同一 SQLite Transaction 中完成，并追加
`task.corrected` 审计事件，payload 携带 before/after 字段及 `forced: true`。
失败的授权检查不修改任务或事件，JSON/text 输出继续使用标准 entity envelope。

0.6.0 起编辑约束：

- 处于 terminal 状态（completed/cancelled）的任务被冻结；不带 `--force` 的
  `task edit` 返回 `STATE_CONFLICT`，带 `--force` 才进入 `task.corrected` correction 路径。
- `--title` / `--description` 同样受 200/8000 长度上限约束。
- 传入空字符串可清空可选字段（description、required_role）；省略参数
  保持原值。

---

# 17. `carryctx progress`

```text
carryctx progress todo
carryctx progress block
carryctx progress risk
carryctx progress note
carryctx progress list
carryctx progress show
carryctx progress edit
carryctx progress complete
carryctx progress reopen
carryctx progress remove
carryctx progress reorder
```

示例：

```bash
carryctx progress todo "Implement WAL initialization"
carryctx progress note "Created migration table"
carryctx progress block "Waiting for config schema"
```

Progress Item 默认关联当前 Task 和 Session。

---

# 18. `carryctx worktree`

Cleanup records returned by `worktree cleanup list`, `show`, and `run` include
the existing `state` field and the structured lifecycle fields `status`,
`reason`, `attempt_count`, and `blocked_reason`. Cleanup audit events use the
same fields in their payloads, so pending, blocked, completed, and failed
attempts are observable without parsing human output. `doctor` reports pending,
blocked, and failed cleanup requests as warning findings.

```text
carryctx worktree create
carryctx worktree bind
carryctx worktree list
carryctx worktree show
carryctx worktree status
carryctx worktree unbind
carryctx worktree remove
```

创建：

```bash
carryctx worktree create CTX-0001
```

选项：

```text
--path <path>
--branch <branch>
--base <branch-or-commit>
--checkout
```

CarryCtx 使用系统 Git CLI 执行 worktree 操作。当检测到 [Jujutsu (jj) colocated 仓库](../plans/2026-07-25-jujutsu-compatibility.md)（`.git/` 旁存在 `.jj/`）时，`worktree create` 拒绝执行并返回 `VALIDATION_FAILED`：jj 的二级工作区（`jj workspace add`）没有独立的 `.git/` 目录，`carryctx` 的状态命令无法在其内部读取仓库状态；而 `git worktree add` 创建的目录 jj 也无法识别为工作区。用户应直接运行 `jj workspace add <path>`，进入该目录后仅使用 `jj` 命令；如需将其纳入 CarryCtx 状态追踪，应在主 colocated 仓库内运行 `carryctx worktree bind`。

## `worktree remove`（Unreleased）

```bash
carryctx worktree remove <REF> [--force]
```

`<REF>` 按以下形式解析：

```text
CTX display-id（被绑定 Task 的 CTX 编号，如 CTX-0002）
Worktree ULID
Worktree 路径
```

行为（均已对照真实 CLI 验证）：

- 工作区包含已修改或未跟踪文件时拒绝执行：`STATE_CONFLICT`（退出码 3），
  错误信息提示追加 `--force`。
- 工作区干净或显式传入 `--force` 时，删除 Git 工作区并从注册表中整体移除
  该行。
- 目录已不存在时仅做注册清理：成功信封中 `data.git_removed=false`。
- 成功路径在同一事务内追加 `worktree.removed` 审计事件。
- 在检测到 `.git/` 旁存在 `.jj/` 的 colocated 仓库时，禁止删除仍存在的 Git
  worktree，即使传入 `--force` 也返回 `VALIDATION_FAILED`，并保留注册记录；这避免
  `git worktree remove` 绕过 jj 的 workspace 状态。目录已经不存在时仍可安全地清理
  CarryCtx 注册记录。

`worktree unbind` 仅解除 Task 绑定关系，不删除任何文件或注册记录
（"detach without deleting anything"）。

---

# 19. `carryctx decision`

```text
carryctx decision add
carryctx decision list
carryctx decision show
carryctx decision search
carryctx decision supersede
```

Decision 不允许直接删除。

`decision list` 可选 `--task <ref>`（0.5.5 起）只返回该任务下的决策；
ref 必须可解析，否则返回 `RESOURCE_NOT_FOUND`。不带 `--task` 时列出全部。

错误 Decision 使用：

```bash
carryctx decision supersede DEC-001 --by DEC-014
```

保留历史关系。

---

# 20. `carryctx handoff`

```text
carryctx handoff create
carryctx handoff list
carryctx handoff show
carryctx handoff accept
carryctx handoff reject
carryctx handoff close
```

创建：

```bash
carryctx handoff create \
  --target opencode-core \
  --summary "Migration runner implemented"
```

Accept 操作应：

- 验证目标 Agent
- 更新 Handoff 状态
- 可选转移 Task Owner
- 创建新 Session
- 写入 Event

0.6.0 起 Handoff 生命周期由状态机强制（Open→Accepted/Rejected→Closed）：

- 合法转换：Open→Accepted、Open→Rejected、Open/Accepted/Rejected→Closed。
  不存在任何回到 Open 的路径，也不接受 accept 已关闭的 handoff、
  accept 后再 reject 等跳变；违规返回 `STATE_CONFLICT`。
- 存储层 UPDATE 带 compare-and-set 守卫（只从允许的源状态更新），
  并发双 accept 只有一个成功，另一个得到 `STATE_CONFLICT`。
- 命令信封返回转换后的最新行（post-transition row），而非旧状态。
- 每次转换在同一个事务中原子追加审计事件：`handoff.accepted`、
  `handoff.rejected`、`handoff.closed`；事件写入失败则整个变更回滚，
  审计与状态不会分叉。

是否转移 Owner 必须通过：

```text
--claim-task
```

显式指定。

---

# 21. `carryctx event`

```text
carryctx event list
carryctx event show <event-id>
```

过滤：

```text
--task
--agent
--session
--event-type
--since
--until
--limit
--cursor <token>
```

示例：

```bash
carryctx event list \
  --task CTX-0001 \
  --since 24h

carryctx event list \
  --limit 100 \
  --cursor "MjAyNi0wOC0yNVQwODo0NzoyNi41MTE4NDEyNTQrMDA6MDB8MDFNMFcxTVBTRzhRV01DMkE1VFZENEExV1Y.a46140c7"
```

分页与默认上限（0.6.0 起）：

- `--limit` 省略时每页最多返回 200 条事件。
- 事件列表按 `(occurred_at, id)` keyset 排序，保证同一时间戳批次不重复、
  不遗漏。
- `next_cursor`：当返回的是一整页（其后可能还有数据）时，输出 opaque 游标
  token。token 形如 `<base64url("occurred_at|id")>.<8 位十六进制校验和>`，
  把它作为 `--cursor <token>` 原样传回即可获取下一页；游标严格按元组推进，
  翻页不重复、不遗漏。
- 游标校验（Unreleased）：任何包含 `.` 的 token 一律按 opaque 格式校验，
  载荷被篡改或校验和不匹配时返回 `VALIDATION_FAILED`（错误信封走 stderr，
  退出码 8），不会静默忽略。旧版明文 `(occurred_at|id)` token 仅在不包含
  `.` 时继续被接受，并在下一次输出 `next_cursor` 时自动改写为 opaque 格式；
  注意真实历史 token 的时间戳带小数秒（必然含 `.`），升级后会按 opaque
  校验拒绝——应丢弃旧 token 从头翻页。
- 兼容性：不传 `--cursor` 时首页行为与旧版一致，仅 `next_cursor` 从固定
  `null` 变为在存在后续页时填充真实 token。
- 引用解析失败即大声报错：`--agent` 指向不存在或已 deactivated 的 agent 时，
  分别返回 `RESOURCE_NOT_FOUND` / `PERMISSION_SCOPE`（"is deactivated and
  cannot act"，错误信封走 stderr），与 `search --assignee` 完全一致；不会
  静默放宽为全量事件流。`--task` 同样大声报错。

Agent 过滤语义（Unreleased 行为变更）：环境变量 `CARRYCTX_AGENT`
不再隐式过滤 `event list` 结果——省略 `--agent` 时始终返回项目全量事件流；
只有显式传入 `--agent` 才按 agent 过滤。该变量的身份解析与事件归因
（写入事件的 actor）不受影响。

Event 不提供普通删除命令。

---

# 22. `carryctx search`

跨 Task、Progress、Checkpoint、Decision 的全文搜索。基于 SQLite FTS5，按 `bm25()` 相关度排序。

```bash
carryctx search "markdown worker protocol"
```

参数：

````text
<query>       (位置参数，必填)
--type        task | progress | checkpoint | decision
--status      按拥有该记录的 Task 的状态过滤
--assignee    按拥有该记录的 Task 的 owner agent 过滤（名称或 ULID）
--limit       最大返回数量（默认 20）

`--assignee` 命名上有意区别于全局 `--agent`/`CARRYCTX_AGENT`（别名为 `--owner`）：两者同名会导致 clap 把全局身份参数的值泄漏进子命令的局部参数，`event list --agent` 曾经踩过这个坑（见 CHANGELOG 0.2.1），`search` 直接用不同名字规避。

每条结果（`SearchHit`）：

```json
{
  "kind": "checkpoint",
  "id": "01J...",
  "display_id": null,
  "task_id": "01J...",
  "task_display_id": "CTX-0001",
  "task_status": "in_progress",
  "branch": "feature/markdown-worker",
  "snippet": "PR #263 merged - [markdown] worker-owned source",
  "score": -3.2,
  "created_at": "2026-07-28T19:00:00Z"
}
````

`branch` 解析顺序：Checkpoint 命中优先使用该 Checkpoint 自身记录的 `branch`（创建时的真实分支），否则回退到该 Task 当前的 worktree 绑定分支；其余三种命中类型直接使用 worktree 绑定分支。两者都缺失时为 `null`。

顶层 `display_id`（Unreleased）：`kind=task` 的命中现在填充为该 Task 自己的
CTX 编号（此前恒为 `null`，只能读 `task_display_id`）；其余命中类型不变——
progress / decision 使用各自编号（`PX-*` / `DEC-*`），checkpoint 为 `null`。
`task_display_id` 字段对所有类型保持原样。

索引维护：`tasks`、`progress_items`、`checkpoints`、`decisions` 各有一张 FTS5 虚表，通过触发器随对应主表的增删改同步；升级到包含此功能的版本时，迁移会对已有数据一次性回填索引。

---

# 23. `carryctx doctor`

```bash
carryctx doctor
carryctx doctor --fix
carryctx doctor --prune-stale-worktrees
carryctx doctor --json
```

检查分类：

```text
configuration
database
git
worktree
task
session
checkpoint
filesystem
skill
```

每个结果包含：

```json
{
  "check": "worktree.path.exists",
  "status": "error",
  "message": "Registered worktree path does not exist.",
  "repairable": true
}
```

`worktrees.stale` reports registered worktrees whose directories are missing. The
check is read-only by default. `--prune-stale-worktrees` explicitly removes only
the stale registrations from SQLite; it never deletes filesystem paths and emits
a `worktree.pruned` audit event for each removed registration.

退出码反映发现严重级别（Unreleased）：

- 存在 `error` 或 `critical` 级检查、或基础设施本身不可用（如数据库无法打开）
  时，退出码为 `1`。
- 仅有 `warning` / `info` 级发现时，退出码为 `0`——此时 `all_ok` 仍为
  `false`，报告渲染不变。脚本因此可以区分"无实质故障"与"真坏了"
  （例如陈旧的 worktree 注册不应让 CI 失败）。

破坏性修复要求：

```text
--yes
```

---

# 24. `carryctx graph`

上下文依赖图（AST/文件级）的维护与导出：

```text
carryctx graph edges <node>
carryctx graph add-node --node-type <type> --name <name>
carryctx graph link <source> <target> <relation>
carryctx graph extract-deps <path>
carryctx graph scan
carryctx graph export <format>
```

写操作（`add-node` / `link` / `extract-deps` / `scan`）支持 `--dry-run`
（0.7.0 起）：

- 文本模式：计划以 `[dry-run] Would …` 形式输出到 stderr，stdout 为空，
  不落库。
- `--format json --dry-run`：stdout 输出标准成功信封，`data` 固定为
  `{"operation":{"applied":false}}`；stderr 输出同一行 `[dry-run]` 计划。

```json
{
  "schema_version": 1,
  "command": "graph.scan",
  "success": true,
  "data": { "operation": { "applied": false } },
  "meta": { "timestamp": "2026-08-25T09:01:26.543318320+00:00" }
}
```

---

# 25. `carryctx mcp`

以 stdio 传输启动 Model Context Protocol 服务器（JSON-RPC 2.0），供
Cursor、Claude Desktop 等客户端原生调用 CarryCtx 工具集：

```bash
carryctx mcp
```

协议行为：

- `initialize` 返回协议版本 `2024-11-05` 与工具能力声明。
- `ping`（Unreleased）：携带 id 的请求返回空对象结果
  `{"id":<id>,"jsonrpc":"2.0","result":{}}`；通知（无 id）不产生任何响应。
- 未知方法返回 `-32601 Method not found` 错误对象，请求 id 原样回显。

---

# 26. `carryctx skill`

```text
carryctx skill install
carryctx skill update
carryctx skill list
carryctx skill path
carryctx skill export
carryctx skill doctor
```

默认安装位置：

```text
$XDG_DATA_HOME/carryctx/skills/
```

项目级安装：

```bash
carryctx skill install --project
```

目标：

```text
.carryctx/skills/
```

---

# 27. JSON 输出规范

## 27.0 字段命名约定

已发布 CLI 的 JSON 成功/错误 Envelope 使用 snake_case：
`schema_version`、`command`、`success`、`data`、`warnings`、`meta`、`error`。
由 Rust 实体记录直接序列化的字段也使用 snake_case，例如
`display_id`、`owner_agent_id`、`project_id`、`occurred_at`。这些名称是当前
`schema_version = 1` 的公共接口；消费者不得根据旧示例把它们转换为 camelCase。

部分命令的 `data` 是历史上手工构造的投影，而不是实体记录本身，因此可能仍
包含既有 camelCase 键（例如 `status` 的 `projectId`、`projectName` 和
`activeSessions`）。这些命令的现有键保持不变，不能从本节的实体命名约定推断
出一次性的全局重命名。

Event 外层记录遵循实体命名约定，但 `payload` 是 append-only 的历史 JSON，
其内部键不属于统一重编码的实体字段。已发布事件中两种形式都存在，例如
`session_id`、`task_id` 与 `handoffId`、`beforeStatus`、`ownerAgentId`。
现有 payload 不回写、不补发别名，也不在读取时改名。新事件 payload 应使用
snake_case；若将来需要迁移旧 payload，必须单独提出版本化、兼容性和回填方案，
不得借此文档决定 broad schema migration。此决定保持所有现有事件消费者兼容。

成功：

```json
{
  "schema_version": 1,
  "command": "task.list",
  "success": true,
  "data": {
    "tasks": []
  },
  "warnings": [],
  "meta": {
    "timestamp": "2026-07-22T18:30:00Z"
  }
}
```

失败：

```json
{
  "schema_version": 1,
  "command": "task.claim",
  "success": false,
  "error": {
    "code": "TASK_ALREADY_CLAIMED",
    "message": "Task CTX-0001 is already claimed.",
    "details": {
      "owner": "opencode-core"
    },
    "suggestions": [
      "Run carryctx task show CTX-0001.",
      "Ask the current owner to release the task."
    ]
  }
}
```

---

## 27.1 文本输出（compact）

`--format text`（默认）为 Agent 上下文做精简：实体类命令只输出单行摘要，
不输出完整记录（ULID、时间戳、空字段等一律省略）。

```text
$ carryctx task create --title "fix: selection drift"
Task created: CTX-0321

$ carryctx agent current --agent opencode
opencode

$ carryctx handoff list
HO-0003    open         → 01KY7HA5  Bug B: per-character positioned carriers ...
```

已覆盖的命令族：`task.*`、`agent.*`、`checkpoint.*`、`handoff.*`、`session.*`、
`progress.*`、`decision.*`、`worktree.*`、`event.*`、`search`、`status`。
未覆盖的命令保持原有 pretty JSON 文本输出。

## 27.2 完整文本输出

需要完整字段时使用以下任一方式（JSON 输出不受影响，始终为完整 Envelope）：

- 全局 `--verbose` 标志：`carryctx task show CTX-0321 --verbose`
- 配置文件：`.carryctx/config.toml` 中设置 `[output] verbose = true`

## 27.3 字段投影

可用 `--fields`（全局参数）或配置文件按命令过滤输出字段，只保留需要的高关注字段：

```bash
carryctx handoff list --json --fields display_id,status,summary,target_agent_id
```

```toml
# .carryctx/config.toml
[output.fields]
"handoff.list" = ["display_id", "status", "summary", "target_agent_id"]
"task.list"    = ["display_id", "title", "status"]
```

CLI `--fields` 优先级高于配置文件。投影同时作用于 text 与 JSON 输出；
JSON Envelope 结构（`schema_version`/`command`/`success`/`data`/`meta`）保持不变，
仅 `data` 内的记录字段被裁剪。

---

# 28. Exit Code

```text
0   Success
1   General Error
2   Invalid Arguments
3   State Conflict
4   Git Error
5   Database Error
6   Configuration Error
7   Resource Not Found
8   Validation Failed
9   Permission or Scope Error
10  Unsupported Operation
11  Migration Required
12  Interrupted
```

Exit Code 必须视为公共 API。

发布稳定版本后不得随意修改。

---

# 29. 输出流

```text
stdout → 正常结果
stderr → Warning、错误、诊断、Verbose Log
```

JSON 模式下：

```text
成功：stdout → 单个成功 JSON；stderr → 空
失败：stdout → 空；stderr → 单个错误 JSON
```

JSON Warning 放在成功 Envelope 的 `warnings` 中，Verbose 诊断放在 `meta.diagnostics` 中，不额外输出非 JSON 文本。

秘密、Token 和完整环境变量不得输出到日志。

管道提前关闭（如 `carryctx checkpoint list | head -3`）时，进程以退出码 **141**
（128 + SIGPIPE，Unix 惯例）静默终止，不输出 panic 信息。

---

# 30. 命令稳定性等级

命令在文档中标记：

```text
stable
experimental
internal
deprecated
```

v0.6.0 中：

```text
stable:
  init
  status
  resume
  context
  checkpoint
  project show/list/register/unregister/migrate/backup/restore
  task
  team create/member/commander/status/context
  progress
  session
  agent
  config
  event
  doctor

experimental:
  handoff
  decision
  task scope
  skill install/list/path/doctor
  worktree create
  worktree bind/list/show/status/unbind/remove
  graph add-node/link/extract-deps/scan/export/edges
  mcp

deferred:
  project export/import
  worktree prune
  event tail
  skill update/export
  shell completion
```

Experimental 命令仍必须遵守 JSON Schema 和错误模型。
