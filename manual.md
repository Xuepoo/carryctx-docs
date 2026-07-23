# CarryCtx 详细使用手册 (Manual)

本文档是 CarryCtx CLI 的官方使用说明书，全面介绍了 CarryCtx 的核心概念、标准工作流以及所有子命令的详细用法。本文档可作为后续 `carryctx-website` 官方文档站点的内容基础。

---

## 1. 核心概念

CarryCtx 旨在为 Coding Agent（AI 编程助手）和人类开发者提供一个持久化的**本地上下文存储层**，使开发过程具备“记忆”、“可随时中断与恢复”以及“支持多 Agent 协作”的能力。

核心实体包括：
- **Project (项目)**: 对应一个 Git 仓库及其根目录。所有 CarryCtx 数据存储在 `.git/carryctx/state.sqlite` 中。
- **Agent (代理)**: 参与项目的开发实体。可以是人类，也可以是不同供应商的 AI（如 Claude-Code, Aider）。
- **Task (任务)**: 开发工作的基础单元（如一个需求、一个 Bug 修复）。任务可以具有依赖关系（Dependencies）。
- **Session (会话)**: 针对某个 Task 进行的一段连续开发时间。一次工作流通常由开启会话开始，以结束会话（或提交 Checkpoint）告终。
- **Progress (进度)**: 在会话期间记录的碎片化脑图，包括待办 (todo)、完成 (done)、阻塞 (block) 和笔记 (note)。
- **Checkpoint (快照)**: 对当前工作进度的完整定格，包含所有未提交的文件 Diff、完成的步骤和后续计划，可方便其他 Agent（或明天的自己）一键恢复大脑上下文。

---

## 2. 快速开始工作流

### 2.1 初始化与注册
在任何一个新的代码仓库中，首先需要初始化 CarryCtx 数据库并注册你的 Agent 身份。

```bash
# 初始化当前目录为 CarryCtx 项目
carryctx init

# 注册一个新的 Agent（比如人类开发者或某个 AI）
carryctx agent register --name my-agent --provider user
# 或者注册一个 Claude Agent
carryctx agent register --name claude-code --provider anthropic

# 设置当前环境的 Agent ID (推荐配置在 bashrc 中或由 Agent 自动注入)
export CARRYCTX_AGENT=my-agent
```

### 2.2 创建任务与开启开发
当有一个新需求时，先创建 Task 并认领。

```bash
# 创建一个新的任务，得到显示 ID (如 CTX-0001)
carryctx task create --title "实现用户登录功能"

# 如果有前置任务，可以指定依赖
carryctx task create --title "实现重置密码" --depends-on CTX-0001

# 认领你要处理的任务（将你设为 Owner 并标记为进行中）
carryctx task claim CTX-0001

# 开启一段开发会话 (Session)
carryctx session start --task CTX-0001
```

### 2.3 记录进度与打快照
在敲击代码的过程中，随时将思路持久化：

```bash
# 记录思维碎片
carryctx progress todo "需要添加密码的 bcrypt hash 处理"
carryctx progress done "已经建好 users 数据库表"
carryctx progress block "等待前端提供具体参数格式"
```

当你要下班、被其他事情打断、或者需要清除过长上下文时，保存快照：

```bash
# 生成包含未提交代码 Diff、进度汇总的上下文快照
carryctx checkpoint \
  --done "完成了基础表结构和登录接口" \
  --remaining "还要做密码加密和 Token 签发" \
  --blocker "无"
  
# 结束本次会话
carryctx session end
```

### 2.4 上下文恢复
当你明天重新打开终端，或者换了一个 Agent 接手：

```bash
# 一键查看当前的全面状态（包含上个快照的记录、未完成的进度等）
carryctx status

# 恢复上下文（将最近的 Checkpoint、未完成进展输出给大模型读取）
carryctx resume
```

---

## 3. CLI 子命令详尽参考

所有命令支持 `--json` 输出机器可读格式，以及 `--non-interactive` 静默模式。

### 3.1 核心状态流转 (`init`, `status`, `resume`, `checkpoint`)
- **`carryctx init`**
  - **功能**: 初始化 CarryCtx。在 `.git/carryctx/` 下创建 SQLite 数据库。
  - **用法**: `carryctx init`

- **`carryctx status`**
  - **功能**: 输出当前项目维度的健康状态，包括活跃 Session、进行中的任务、挂起的阻塞点等。
  - **用法**: `carryctx status`

- **`carryctx resume`**
  - **功能**: 输出上文恢复指南。读取最近一次 Checkpoint 以及 Progress，为 AI 提供续写上下文。
  - **用法**: `carryctx resume [--task <task-id>]`

- **`carryctx checkpoint`**
  - **功能**: 创建进度快照。它会自动抓取 Git 状态（staged、modified、untracked）与你的输入结合。
  - **用法**: `carryctx checkpoint --done "..." --remaining "..." [--blocker "..."] [--note "..."]`

- **`carryctx context`** *(为大模型专门设计)*
  - **功能**: 将当前任务相关的完整上下文结构化导出，便于拼接到 Prompt 中。
  - **用法**: `carryctx context [--task <task-id>]`

### 3.2 任务管理 (`task`)
管理需求、Bug 与开发任务，支持依赖图。
- **`task create`**: 创建新任务。`--title <标题> [--description <描述>] [--depends-on <task-id>]`
- **`task list`**: 列出任务。`[--status <状态>] [--assignee <agent-id>]`
- **`task claim`**: 认领任务并更新状态为 `in_progress`。
- **`task start`**: 开始任务（不更改所有者）。
- **`task review`**: 提交任务审查。
- **`task block`**: 将任务挂起/阻塞。
- **`task complete`**: 完成任务。
- **`task cancel`**: 取消任务。
- **`task deps`**: 管理依赖（`add`, `remove`, `tree` 查看依赖树）。
- **`task scope`**: 限定当前任务影响的文件范围。

### 3.3 会话与进度追踪 (`session`, `progress`)
- **`session start`**: 开始一段编码时间。`[--task <task-id>]`
- **`session pause`**: 暂停会话（如去吃午饭）。
- **`session end`**: 结束会话。通常配合 `checkpoint` 使用。
- **`session list`**: 查看近期会话历史。

- **`progress todo`**: 记录下一步要做的待办事项。`"内容" [--task <id>]`
- **`progress done`**: 记录刚完成的小步骤。
- **`progress block`**: 记录当前的阻碍。
- **`progress note`**: 记录参考笔记或发现。

### 3.4 多代理协作与高级功能 (`agent`, `worktree`, `handoff`, `decision`)
- **`agent`**
  - `agent register --name <名称> --provider <引擎>`: 注册代理。
  - `agent list`: 查看项目中的所有协作者。
  - `agent update`: 更新代理状态（如停用）。

- **`worktree`** (基于 Git Worktree 的任务并行开发)
  - `worktree create <分支名> [--task <task-id>]`: 为特定任务快速创建一个独立的代码工作区。
  - `worktree list`: 查看绑定的工作区。
  - `worktree remove <id>`: 清理工作区。

- **`handoff`** (接力与交接班)
  - `handoff create --target <agent-id> --message "..."`: 向特定 Agent 发送交接班请求。
  - `handoff read/claim`: 读取或认领交接请求。

- **`decision`** (架构级决策记录 / ADR)
  - `decision record --title "..." --content "..."`: 记录项目为什么做出某种技术选择。

### 3.5 诊断与系统 (`doctor`, `config`, `event`, `project`)
- **`doctor`**: 诊断 CarryCtx 数据库的完整性和一致性。
- **`config`**: 读取/修改配置（如修改前缀规则、提交格式）。`config get <key>`, `config set <key> <value>`。
- **`event list`**: 基于 Event Sourcing 模型，列出项目中发生的所有底层事件追踪。
- **`project show/migrate`**: 显示当前项目元数据或运行数据库迁移。

---

## 4. 给 Agent 编写者的建议 (Agent Guidelines)

如果你在开发一个新的 CLI Agent 并希望深度集成 CarryCtx：

1. **统一身份**: 在 Agent 启动时，主动探测环境变量 `CARRYCTX_AGENT`，若不存在，则可利用 `carryctx agent register` 进行自我注册并持久化该 ULID。
2. **善用 JSON**: 所有命令调用附带 `--json`，并通过 exit code 判断操作是否被业务逻辑拒绝（如 `exit code 3` 为 State Conflict）。
3. **安全边界**: CarryCtx 提供的是本地控制面缓存，请避免在 `carryctx progress note` 等命令中写入海量数据（如超大的 Log），它适用于人类可读的高密度总结。
4. **中断友好**: 任何时刻如果大模型生成超时或需要强制退出，务必拦截 SIGINT 信号并调用 `carryctx checkpoint` 以保存遗留现场。

> 提示：本手册的所有命令都可以通过附加 `--help` 参数查看更详细的标志位说明，例如 `carryctx task create --help`。
