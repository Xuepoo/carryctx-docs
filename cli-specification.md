# CarryCtx CLI 命令规范

**文档路径：** `carryctx-docs/cli-specification.md`
**文档版本：** v0.1
**适用版本：** CarryCtx v0.1.x

---

## 1. 设计目标

CarryCtx CLI 应同时服务于：

* Coding Agent
* 人类开发者
* Shell Script
* CI
* Agent Skill
* 后续 MCP Adapter

CLI 必须具备：

* 稳定的命令名称
* 确定性的行为
* 机器可读 JSON
* 明确的 Exit Code
* 非交互模式
* 可审计的写操作
* 可预测的配置解析
* 清晰的错误恢复建议

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
--agent <agent-id>
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

* Repository root
* Repository 子目录
* Linked worktree
* `.git` 指向的 worktree

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

* 不输出 ANSI Color
* 不输出 Spinner
* 不输出交互提示
* `stdout` 只输出结果 JSON
* 错误 JSON 输出到 `stderr`
* 必须设置非零 Exit Code

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

* SQLite
* Git
* 配置文件
* 文件系统

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
5. 当前 Agent 的唯一 active Task
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
carryctx progress
carryctx worktree
carryctx decision
carryctx handoff
carryctx event
carryctx skill
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
--since <duration>
```

默认输出：

* Project
* Current Agent
* Current Session
* Current Task
* Git State
* Active Sessions
* Task 状态计数
* 最近活动
* Blocker
* Scope 冲突

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

* 修改任务
* 创建 Checkpoint
* 提交 Git
* 更改 Task Owner

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

* Branch
* HEAD
* Dirty State
* Staged Files
* Modified Files
* Untracked Files
* Diff Stats

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

* Project ID
* Repository Root
* Git Common Dir
* Database Path
* Main Branch
* Schema Version
* Config Sources

## `project migrate`

```bash
carryctx project migrate
```

必须：

* 检查目标版本
* 创建备份
* 事务执行
* 写入 Migration Event

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
carryctx config set --project session.stale_after 4h
carryctx config set --global output.color auto
carryctx config sources
```

`config set` 和 `config unset` 必须且只能显式指定一个写入作用域：

```text
--global
--project
--local
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

结束：

```bash
carryctx session end \
  --summary "Paused before migration tests"
```

没有最新 Checkpoint 时：

* TTY：提示创建
* Non-interactive：返回 Warning 或根据 strict 配置失败

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
--owner <agent-id>
--status <status>
```

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

---

## 16.3 Claim

```bash
carryctx task claim CTX-0001
```

Claim 必须在一个 SQLite Transaction 中完成。

检查：

* Task 存在
* Task 可接管
* 强依赖已完成
* 当前未被其他 Agent 接管
* Agent active
* 单 Agent Task 限制
* Worktree 冲突

---

## 16.4 Dependency

```bash
carryctx task depend CTX-0002 --on CTX-0001
carryctx task undepend CTX-0002 --on CTX-0001
```

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

---

# 17. `carryctx progress`

```text
carryctx progress todo
carryctx progress done
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
carryctx progress done "Created migration table"
carryctx progress block "Waiting for config schema"
```

Progress Item 默认关联当前 Task 和 Session。

---

# 18. `carryctx worktree`

```text
carryctx worktree create
carryctx worktree bind
carryctx worktree list
carryctx worktree show
carryctx worktree status
carryctx worktree unbind
carryctx worktree remove
carryctx worktree prune
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

CarryCtx 使用系统 Git CLI 执行 worktree 操作。

删除前必须检查：

* Dirty State
* Untracked Files
* Active Session
* 未完成 Task
* 未合并 Commit

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

* 验证目标 Agent
* 更新 Handoff 状态
* 可选转移 Task Owner
* 创建新 Session
* 写入 Event

是否转移 Owner 必须通过：

```text
--claim-task
```

显式指定。

---

# 21. `carryctx event`

```text
carryctx event list
carryctx event show
carryctx event tail
```

过滤：

```text
--task
--agent
--session
--type
--since
--until
--limit
```

示例：

```bash
carryctx event list \
  --task CTX-0001 \
  --since 24h
```

Event 不提供普通删除命令。

---

# 22. `carryctx doctor`

```bash
carryctx doctor
carryctx doctor --fix
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

破坏性修复要求：

```text
--yes
```

---

# 23. `carryctx skill`

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

# 24. JSON 输出规范

成功：

```json
{
  "schemaVersion": 1,
  "command": "task.list",
  "success": true,
  "data": {
    "tasks": []
  },
  "warnings": [],
  "meta": {
    "projectId": "carryctx",
    "timestamp": "2026-07-22T18:30:00Z"
  }
}
```

失败：

```json
{
  "schemaVersion": 1,
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

# 25. Exit Code

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

# 26. 输出流

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

---

# 27. 命令稳定性等级

命令在文档中标记：

```text
stable
experimental
internal
deprecated
```

v0.1 中：

```text
stable:
  init
  status
  resume
  context
  checkpoint
  project show/list/register/unregister/migrate/backup/restore
  task
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
  worktree bind/list/show/status/unbind

deferred:
  project export/import
  worktree remove/prune
  skill update/export
  shell completion
```

Experimental 命令仍必须遵守 JSON Schema 和错误模型。
