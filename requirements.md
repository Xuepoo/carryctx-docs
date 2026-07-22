# CarryCtx 软件需求规格说明书

**项目名称：** CarryCtx
**英文定位：** Persistent project context for coding agents
**文档版本：** v0.1
**产品阶段：** Requirements Draft
**目标发布形式：** TypeScript CLI / npm Package
**默认命令：** `carryctx`

---

## 1. 文档目的

本文档用于定义 CarryCtx 的产品定位、核心概念、功能需求、非功能需求、数据模型、CLI 交互、核心工作流、产品边界及 MVP 验收标准。

本文档主要服务于：

* CarryCtx 的设计与开发
* CLI 命令设计
* SQLite 数据模型设计
* Agent Skill 编写
* 自动化测试与验收
* 后续版本规划
* 开源项目 README 和贡献者文档编写

本文档重点回答：

1. CarryCtx 要解决什么问题
2. CarryCtx 不解决什么问题
3. CarryCtx 需要保存哪些项目状态
4. Coding Agent 如何通过 CarryCtx 延续工作
5. 多个 Agent 如何共享项目进度
6. CLI 与 Agent Skill 如何协作
7. 第一版必须实现哪些能力

---

# 2. 项目背景

大型基础设施项目通常具有以下特点：

* 仓库规模较大
* 模块数量较多
* 任务持续时间较长
* 多个任务之间存在依赖关系
* 一个任务可能跨越多个 Agent Session
* 多个 Coding Agent 可能并行工作
* 项目可能使用多个 Git worktree
* 重要信息分散在代码、文档、Issue 和 Agent 对话中

项目开发过程中可能同时使用不同的 Coding Agent，例如：

* Claude Code
* OpenCode
* GitHub Copilot
* Kiro
* Antigravity
* Codex
* 其他可以执行 Shell 命令的 Agent

这些 Agent 通常拥有独立的会话上下文。

当 Agent 窗口关闭、上下文压缩、模型切换或任务交接后，新 Session 很难准确知道：

* 上一个 Session 正在处理什么
* 已经完成了哪些工作
* 哪些工作尚未完成
* 当前代码位于哪个 worktree
* 当前 branch 和 commit 是什么
* 工作区是否存在未提交修改
* 当前任务依赖哪些其他任务
* 其他相关任务完成到了什么程度
* 最近发生了哪些架构决策
* 是否有其他 Agent 正在修改相同区域
* 下一步应该继续执行什么

如果完全依赖 Markdown 文档维护这些信息，会产生以下问题：

* 文档数量不断增加
* Agent 不知道应该读哪些文档
* 文档内容容易过期
* 多个 Agent 可能同时修改状态文档
* 状态缺少统一格式
* 很难查询和聚合
* 很难区分当前状态和历史记录
* 很难自动检测 worktree、branch 和代码变更
* Agent 需要消耗大量上下文读取无关信息

CarryCtx 用于解决上述项目状态连续性问题。

---

# 3. 产品定位

CarryCtx 是一个面向 Coding Agent 的、本地优先的项目状态与连续性管理 CLI。

它通过保存结构化的：

* Task
* Agent
* Session
* Worktree
* Progress
* Remaining Work
* Checkpoint
* Dependency
* Blocker
* Decision
* Handoff
* Event
* Git State

使不同 Coding Agent 能够跨窗口、跨 Session、跨工具持续完成同一个大型项目。

CarryCtx 不负责调用或控制 Coding Agent。

它是 Coding Agent 与 Git 项目之间的一层状态管理工具。

可以将各组件的职责概括为：

```text
Git         = 代码历史
Worktree    = 隔离工作空间
CarryCtx    = 项目状态与连续性记忆
Agent Skill = Agent 使用 CarryCtx 的操作协议
Coding Agent = 分析、决策和编写代码
```

---

# 4. 一句话定义

> CarryCtx 是一个 local-first、agent-agnostic 的 CLI，用于在不同 Coding Agent、窗口、Session 和 Git worktree 之间持久化并恢复项目任务、进度、上下文和协作状态。

英文定义：

> A local-first project state and continuity manager for coding agents.

---

# 5. 核心产品目标

## 5.1 跨 Session 连续工作

同一个 Agent 开启新窗口后，应能够通过一个命令恢复上一次工作状态。

```bash
carryctx resume
```

该命令应回答：

* 当前项目是什么
* 当前 Agent 身份是什么
* 当前任务是什么
* 上一个 Session 做了什么
* 还有什么没有完成
* 当前有哪些 Blocker
* 当前 worktree、branch 和 commit 是什么
* 工作区是否存在未提交修改
* 相关任务最近发生了什么变化
* 推荐下一步是什么

---

## 5.2 跨 Agent 任务交接

任务状态不能绑定在某个特定 Agent 的聊天记录中。

一个任务可能经历：

```text
Claude Code Session 1
        ↓
Claude Code Session 2
        ↓
OpenCode Session 1
        ↓
GitHub Copilot Review
```

无论由哪个 Agent 接管，都应读取同一个结构化任务状态。

---

## 5.3 提供全局项目进度视图

Agent 和项目维护者应能够查看：

* 当前所有任务
* 哪些任务正在进行
* 哪些任务已经完成
* 哪些任务被阻塞
* 每个任务当前由谁处理
* 最近一次 Checkpoint
* 任务剩余事项
* 任务依赖和被依赖关系
* 可能发生修改冲突的任务
* 当前存在的 worktree

---

## 5.4 降低上下文获取成本

Agent 不应在每次启动时读取所有协作文档。

CarryCtx 应根据当前 Agent、Task、worktree 和依赖关系，生成与当前工作相关的最小上下文。

```bash
carryctx context
```

生成内容应优先包含：

1. 当前任务
2. 最新 Checkpoint
3. 剩余工作
4. Blocker
5. 直接依赖
6. 被当前任务阻塞的任务
7. 路径可能重叠的活跃任务
8. 相关技术决策
9. 当前 Git 状态
10. 最近相关事件

---

## 5.5 独立于具体 Agent 产品

CarryCtx 不得依赖 Claude Code、OpenCode 或其他 Agent 的私有内部状态。

任何能够执行 CLI 并读取文本或 JSON 的 Agent，都应能够使用 CarryCtx。

---

## 5.6 保持轻量

CarryCtx 是管理工具，不是大型 Agent 编排平台。

它不应要求：

* 启动后台 Agent 集群
* 配置复杂消息队列
* 部署 Kubernetes
* 托管模型
* 创建中心化 SaaS 账户
* 改造现有 Git 仓库结构

第一版本应能够以单个 npm CLI 包运行。

---

# 6. 非目标

CarryCtx 第一阶段不负责以下能力。

## 6.1 不负责 Agent Runtime

CarryCtx 不负责：

* 启动 Claude Code
* 启动 OpenCode
* 调用 LLM API
* 管理 Token
* 管理模型上下文窗口
* 控制 Agent 推理过程

---

## 6.2 不负责自动任务规划

CarryCtx 不负责自动：

* 将需求拆分成任务
* 选择最适合的模型
* 给 Agent 自动分配工作
* 根据模型能力调度任务
* 判断应该并行还是串行执行

这些工作可以由人类或其他 Agent 完成。

CarryCtx 只负责保存拆分后的结果和状态。

---

## 6.3 不负责完整项目管理

CarryCtx 不用于替代：

* Jira
* Linear
* GitHub Projects
* GitHub Issues
* Notion
* 企业级项目管理系统

CarryCtx 关注的是 Coding Agent 执行过程中的仓库状态和上下文连续性。

---

## 6.4 不负责自动解决 Git 冲突

CarryCtx 可以：

* 检测潜在路径冲突
* 提示两个任务可能修改相同文件
* 记录任务关联的 worktree
* 提示 branch 已经落后

但不承诺自动解决复杂 Git merge conflict。

---

## 6.5 不负责完整代码知识图谱

第一版本不实现：

* CodeQL 等价能力
* 全语言 AST 数据库
* 完整 Call Graph
* 自动语义理解
* 向量数据库
* 大规模代码 Embedding

后续可以通过 Adapter 集成这些能力。

---

# 7. 目标用户

## 7.1 项目维护者

项目维护者负责：

* 初始化 CarryCtx
* 定义项目配置
* 创建和维护任务
* 定义任务依赖
* 查看项目整体状态
* 处理长期未更新任务
* 查看 Agent Session
* 管理 worktree
* 定义验证命令
* 处理任务交接和异常状态

---

## 7.2 Coding Agent

Coding Agent 通过 CarryCtx：

* 注册或恢复身份
* 查询当前任务
* 启动 Session
* 获取项目上下文
* 更新工作进度
* 记录未完成事项
* 记录 Blocker
* 创建 Checkpoint
* 记录技术决策
* 结束 Session
* 创建 Handoff
* 完成任务

---

## 7.3 人类开发者

人类开发者也可以使用 CarryCtx。

CarryCtx 不应假设所有使用者都是 AI Agent。

人类开发者可以：

* 查看任务进度
* 接管 Agent 未完成的工作
* 创建 Checkpoint
* 修正错误状态
* 记录决策
* 查看相关 worktree
* 将工作重新交给 Agent

---

# 8. 核心概念

## 8.1 Project

Project 表示一个由 CarryCtx 管理的 Git 仓库。

一个 Project 应具有：

* 唯一 ID
* 项目名称
* Git repository root
* Git common directory
* 默认主分支
* 配置版本
* 数据库版本
* 创建时间
* 最近更新时间

---

## 8.2 Agent

Agent 表示一个逻辑执行者。

Agent 可以是：

* Claude Code
* OpenCode
* Copilot
* Kiro
* Codex
* Human
* 自定义 Agent

Agent 应具有稳定身份。

例如：

```text
claude-auth
opencode-renderer
copilot-review
human-xuepoo
```

Agent 身份不应与一次窗口或一次进程完全绑定。

---

## 8.3 Session

Session 表示一次具体的工作会话。

一次新的终端窗口、Agent 窗口或 Agent 运行实例，可以对应一个新的 Session。

Session 必须与 Agent 区分。

```text
Agent: claude-auth
  ├── Session 01
  ├── Session 02
  └── Session 03
```

Session 关闭后，Agent 身份和 Task 状态仍然存在。

---

## 8.4 Task

Task 是 CarryCtx 的核心工作单位。

Task 表示可以被执行、暂停、恢复、交接和完成的一项工作。

Task 应包含：

* 标题
* 详细描述
* 当前状态
* 优先级
* 当前 Owner
* 依赖任务
* 被依赖任务
* 已完成事项
* 剩余事项
* Blocker
* 相关路径
* 相关 worktree
* 最新 Checkpoint
* 相关决策
* 验证要求

---

## 8.5 Worktree

Worktree 表示与 Task 关联的 Git 工作目录。

推荐关系：

```text
一个 Task 对应零个或一个活跃 Worktree
一个 Worktree 对应一个主要 Task
```

CarryCtx 不强制所有 Task 都必须使用 worktree。

简单任务可以直接绑定当前工作目录。

---

## 8.6 Checkpoint

Checkpoint 表示某一时刻的任务工作快照。

Checkpoint 应同时包含：

### 自动采集信息

* 当前 worktree
* 当前 branch
* HEAD commit
* 工作区是否 dirty
* 修改文件列表
* 未跟踪文件列表
* Diff 统计
* 创建时间

### Agent 报告信息

* 已完成工作
* 剩余工作
* 当前 Blocker
* 已知风险
* 下一步建议
* 补充说明

Checkpoint 是 `resume` 的主要数据来源。

---

## 8.7 Progress Item

任务进度不应只使用百分比表示。

应使用结构化事项：

```text
Completed:
- Implemented LRU cache
- Added basic unit tests

Remaining:
- Add concurrent access tests
- Expose TTL configuration
- Update API documentation
```

Progress Item 至少具有：

* 内容
* 类型
* 状态
* 创建时间
* 完成时间
* 来源 Session
* 排序字段

类型可以包括：

* Todo
* Completed
* Blocker
* Risk
* Note

---

## 8.8 Dependency

Dependency 表示 Task 之间的依赖关系。

例如：

```text
TASK-142 depends on TASK-138
TASK-153 depends on TASK-142
```

CarryCtx 应能够查询：

* 当前任务依赖什么
* 哪些依赖已经完成
* 哪些依赖仍然阻塞
* 当前任务正在阻塞哪些其他任务

---

## 8.9 Decision

Decision 表示项目中的重要技术决策。

例如：

* 缓存使用 LRU
* 默认 TTL 为 15 分钟
* 数据库迁移使用向后兼容策略
* 不允许在核心模块引入某个依赖

Decision 不应只存在于 Agent 对话中。

---

## 8.10 Handoff

Handoff 表示任务从一个 Session 或 Agent 交接给另一个 Session 或 Agent。

Handoff 应包含：

* 来源 Agent
* 来源 Session
* 目标 Agent，可选
* 任务 ID
* 工作摘要
* 已完成事项
* 剩余事项
* 修改文件
* Commit
* 测试结果
* Blocker
* 风险
* 下一步建议

---

## 8.11 Event

Event 表示项目状态发生的一次变化。

例如：

```text
project.initialized
agent.registered
session.started
task.created
task.claimed
task.updated
checkpoint.created
decision.created
task.blocked
handoff.created
session.ended
task.completed
```

Event Log 用于审计和恢复状态变化历史。

---

# 9. 任务状态模型

CarryCtx 第一版采用以下任务状态：

```text
planned
ready
in_progress
blocked
review
completed
cancelled
```

## 9.1 planned

任务已经创建，但尚未满足执行条件。

可能原因：

* 描述不完整
* 依赖未确认
* 尚未准备开始

---

## 9.2 ready

任务已经具备执行条件，可以被 Agent 接管。

---

## 9.3 in_progress

任务已经被某个 Agent 接管并正在开发。

---

## 9.4 blocked

任务由于依赖、技术问题、权限或外部条件无法继续。

进入 blocked 状态时必须记录原因。

---

## 9.5 review

开发工作已经完成，等待：

* 人工检查
* Agent Review
* 测试
* Merge
* 验收

---

## 9.6 completed

任务已经完成并通过定义的验收条件。

---

## 9.7 cancelled

任务不再需要执行。

取消时应保留历史记录和取消原因。

---

# 10. Session 状态模型

Session 使用以下状态：

```text
active
paused
ended
stale
abandoned
```

## active

Session 当前正在工作。

## paused

Session 暂时停止，但预计继续。

## ended

Session 正常结束，并且已经创建最终 Checkpoint。

## stale

Session 长时间没有更新，CarryCtx 判断它可能已经失效。

## abandoned

Session 异常结束或被明确放弃。

---

# 11. 核心功能需求

# 11.1 项目初始化

## FR-PROJECT-001

用户应能够在 Git 仓库中初始化 CarryCtx：

```bash
carryctx init
```

初始化过程应：

1. 检查当前目录是否位于 Git 仓库中
2. 获取 repository root
3. 获取 Git common directory
4. 创建 CarryCtx 配置
5. 创建 SQLite 数据库
6. 创建数据库 Schema
7. 记录项目初始化事件
8. 输出下一步建议

---

## FR-PROJECT-002

默认配置目录：

```text
<repository-root>/.carryctx/
```

建议包含：

```text
.carryctx/
├── config.json
├── skill/
└── templates/
```

---

## FR-PROJECT-003

本地运行数据库应存储在 Git common directory 中：

```text
<git-common-dir>/carryctx/state.sqlite
```

这样多个 linked worktree 可以访问同一个数据库。

---

## FR-PROJECT-004

重复执行 `carryctx init` 不应破坏已有状态。

CLI 应提示项目已经初始化，并提供：

```bash
carryctx doctor
carryctx migrate
```

---

# 11.2 Agent 管理

## FR-AGENT-001

用户或 Agent 应能够注册逻辑身份：

```bash
carryctx agent register \
  --name claude-auth \
  --provider claude-code
```

---

## FR-AGENT-002

Agent 至少包含：

* ID
* Name
* Provider
* Role
* Metadata
* Created At
* Last Active At

---

## FR-AGENT-003

系统应允许同一个 Provider 注册多个 Agent：

```text
claude-auth
claude-database
claude-review
```

---

## FR-AGENT-004

CLI 应支持查询 Agent：

```bash
carryctx agent list
carryctx agent show claude-auth
carryctx agent current
```

---

## FR-AGENT-005

CLI 应允许通过环境变量设置当前 Agent：

```bash
CARRYCTX_AGENT=claude-auth
```

优先级建议：

1. 命令行 `--agent`
2. 环境变量 `CARRYCTX_AGENT`
3. 当前 Session 绑定
4. 项目默认 Agent
5. 交互式选择

---

# 11.3 Session 管理

## FR-SESSION-001

Agent 应能够启动 Session：

```bash
carryctx session start \
  --agent claude-auth \
  --provider claude-code
```

---

## FR-SESSION-002

启动 Session 时，CarryCtx 应自动采集：

* 当前目录
* 当前 worktree
* 当前 branch
* 当前 HEAD
* 当前 Agent
* 当前 Task
* 启动时间

---

## FR-SESSION-003

如果当前 worktree 已经绑定任务，Session 应自动关联该任务。

---

## FR-SESSION-004

如果当前 Agent 存在未结束 Session，CarryCtx 应提示：

* 恢复已有 Session
* 结束已有 Session
* 创建新 Session

---

## FR-SESSION-005

Agent 应能够结束 Session：

```bash
carryctx session end
```

结束 Session 前，CLI 应检查是否存在最新 Checkpoint。

如果没有，应提示创建 Checkpoint。

---

## FR-SESSION-006

支持非交互模式：

```bash
carryctx session end \
  --summary "Paused before concurrent tests" \
  --json
```

---

## FR-SESSION-007

Session 结束不能自动将 Task 标记为 completed。

Session 生命周期和 Task 生命周期必须分离。

---

# 11.4 Task 管理

## FR-TASK-001

用户应能够创建任务：

```bash
carryctx task create \
  --title "Implement authentication cache"
```

---

## FR-TASK-002

Task 应支持以下核心字段：

* ID
* Title
* Description
* Status
* Priority
* Owner Agent
* Parent Task
* Created At
* Updated At
* Started At
* Completed At

---

## FR-TASK-003

系统应自动生成稳定、可读的任务 ID。

示例：

```text
CTX-0001
CTX-0002
```

项目应允许配置任务 ID 前缀。

---

## FR-TASK-004

用户应能够查询任务：

```bash
carryctx task list
carryctx task show CTX-0001
```

过滤条件至少包括：

```bash
carryctx task list --status in_progress
carryctx task list --owner claude-auth
carryctx task list --ready
carryctx task list --blocked
carryctx task list --mine
```

---

## FR-TASK-005

Agent 应能够接管任务：

```bash
carryctx task claim CTX-0001
```

任务接管操作必须是原子的。

两个 Agent 不能同时成功接管同一个任务。

---

## FR-TASK-006

接管任务时，CarryCtx 应检查：

* 任务是否存在
* 任务是否允许接管
* 任务是否已被其他 Agent 接管
* 必需依赖是否完成
* 当前 Agent 是否存在
* 当前 worktree 是否已有其他任务

---

## FR-TASK-007

用户应能够更新任务状态：

```bash
carryctx task start CTX-0001
carryctx task block CTX-0001 --reason "Waiting for API schema"
carryctx task review CTX-0001
carryctx task complete CTX-0001
carryctx task cancel CTX-0001 --reason "No longer required"
```

---

## FR-TASK-008

任务进入 completed 前，应检查是否存在未完成 Progress Item。

默认行为是警告，不强制阻止。

项目配置可以开启严格模式。

---

# 11.5 任务依赖

## FR-DEPENDENCY-001

用户应能够创建任务依赖：

```bash
carryctx task depend CTX-0002 --on CTX-0001
```

---

## FR-DEPENDENCY-002

CarryCtx 必须拒绝直接或间接循环依赖。

---

## FR-DEPENDENCY-003

`carryctx task show` 应显示：

* Depends On
* Blocking
* 已完成依赖
* 未完成依赖

---

## FR-DEPENDENCY-004

`carryctx task list --ready` 仅返回：

* 状态允许开始
* 所有强依赖已完成
* 当前未被其他 Agent 接管

的任务。

---

# 11.6 Worktree 管理

## FR-WORKTREE-001

CarryCtx 应能够绑定已有 worktree：

```bash
carryctx worktree bind CTX-0001
```

默认绑定当前目录对应的 worktree。

---

## FR-WORKTREE-002

CarryCtx 可以提供创建 worktree 的便捷命令：

```bash
carryctx worktree create CTX-0001
```

内部可调用原生 Git CLI。

---

## FR-WORKTREE-003

创建 worktree 时应允许配置：

* 目标路径
* Branch 名称
* Base Branch
* Base Commit

---

## FR-WORKTREE-004

默认 branch 模板建议为：

```text
carryctx/<task-id>-<slug>
```

例如：

```text
carryctx/ctx-0001-auth-cache
```

---

## FR-WORKTREE-005

CarryCtx 应能够列出所有已知 worktree：

```bash
carryctx worktree list
```

至少显示：

* Task
* Path
* Branch
* HEAD
* Dirty
* Agent
* Session
* Last Updated

---

## FR-WORKTREE-006

CarryCtx 应检测：

* 数据库记录存在但目录已删除
* Git worktree 存在但未被 CarryCtx 管理
* Branch 被删除
* Worktree HEAD 与记录不一致

这些检查由：

```bash
carryctx doctor
```

执行。

---

# 11.7 Progress 管理

## FR-PROGRESS-001

Agent 应能够添加已完成事项：

```bash
carryctx progress done "Implemented LRU cache"
```

---

## FR-PROGRESS-002

Agent 应能够添加剩余事项：

```bash
carryctx progress todo "Add concurrent access tests"
```

---

## FR-PROGRESS-003

Agent 应能够添加 Blocker：

```bash
carryctx progress block "Waiting for database migration API"
```

---

## FR-PROGRESS-004

Agent 应能够列出当前任务进度：

```bash
carryctx progress list
```

---

## FR-PROGRESS-005

Progress Item 应支持：

* 排序
* 完成
* 重新打开
* 删除
* 编辑

示例：

```bash
carryctx progress complete ITEM-12
carryctx progress reopen ITEM-12
carryctx progress edit ITEM-12
```

---

## FR-PROGRESS-006

百分比进度仅作为可选字段。

CarryCtx 的主要进度表达方式必须是 Completed、Remaining 和 Blocker。

---

# 11.8 Checkpoint

## FR-CHECKPOINT-001

Agent 应能够创建 Checkpoint：

```bash
carryctx checkpoint
```

---

## FR-CHECKPOINT-002

交互模式应询问：

1. 本次完成了什么
2. 还有什么未完成
3. 是否存在 Blocker
4. 是否存在风险
5. 下一步建议是什么

---

## FR-CHECKPOINT-003

非交互模式示例：

```bash
carryctx checkpoint \
  --done "Implemented cache invalidation" \
  --remaining "Add concurrent tests" \
  --next "Run auth package test suite"
```

---

## FR-CHECKPOINT-004

创建 Checkpoint 时应自动采集：

* Repository Root
* Worktree Path
* Branch
* HEAD Commit
* Dirty State
* Modified Files
* Staged Files
* Untracked Files
* Diff Statistics
* Timestamp
* Session
* Agent
* Task

---

## FR-CHECKPOINT-005

Checkpoint 创建后应成为不可变历史记录。

如果内容错误，应创建修正记录，而不是静默覆盖历史。

---

## FR-CHECKPOINT-006

系统应允许查询历史 Checkpoint：

```bash
carryctx checkpoint list
carryctx checkpoint show <checkpoint-id>
```

---

# 11.9 Resume

## FR-RESUME-001

`carryctx resume` 是 CarryCtx 的核心命令。

```bash
carryctx resume
```

---

## FR-RESUME-002

CarryCtx 应按以下顺序确定当前 Task：

1. 显式 `--task`
2. 当前 Session 绑定 Task
3. 当前 worktree 绑定 Task
4. 当前 Agent 唯一活跃 Task
5. 交互式选择

---

## FR-RESUME-003

Resume 输出必须包含：

### 当前身份

* Project
* Agent
* Session
* Task

### Git 状态

* Worktree
* Branch
* HEAD
* Dirty State
* Modified Files 数量
* Untracked Files 数量

### 任务状态

* Task Title
* Status
* Priority
* Owner
* 最新 Checkpoint
* Completed Items
* Remaining Items
* Blockers

### 关联状态

* 未完成依赖
* 最近完成依赖
* 被当前任务阻塞的任务
* 相关活跃任务
* 潜在路径冲突
* 最近相关决策

### 下一步

* 上次记录的 Next Action
* CarryCtx 根据状态生成的操作提示

---

## FR-RESUME-004

Resume 必须支持机器可读输出：

```bash
carryctx resume --json
```

---

## FR-RESUME-005

Resume 默认不得依赖 LLM。

输出应由结构化数据和确定性规则生成。

---

## FR-RESUME-006

如果当前 Git 状态与最新 Checkpoint 不一致，应明确提示：

```text
Warning: Worktree changed after the latest checkpoint.
```

---

# 11.10 Context 生成

## FR-CONTEXT-001

CarryCtx 应能够生成当前任务上下文：

```bash
carryctx context
```

---

## FR-CONTEXT-002

支持输出格式：

```bash
carryctx context --format text
carryctx context --format markdown
carryctx context --format json
```

---

## FR-CONTEXT-003

上下文应按照相关性排序：

1. 当前 Task
2. 最新 Checkpoint
3. Remaining Work
4. Blocker
5. 直接依赖
6. 直接被依赖任务
7. 路径重叠任务
8. 相关 Decision
9. 最近相关 Event
10. 全局项目摘要

---

## FR-CONTEXT-004

上下文生成应支持限制：

```bash
carryctx context --max-events 10
carryctx context --since 7d
carryctx context --compact
carryctx context --full
```

---

## FR-CONTEXT-005

`--compact` 应适合作为 Agent 每次启动时默认加载的上下文。

---

## FR-CONTEXT-006

CarryCtx 不应默认将所有任务和全部历史事件输出给 Agent。

---

# 11.11 全局状态

## FR-STATUS-001

用户应能够查看项目状态：

```bash
carryctx status
```

---

## FR-STATUS-002

默认状态输出应包含：

* 项目名称
* 当前 branch 和 worktree
* 当前 Agent
* 当前 Task
* Active Sessions
* Ready Tasks 数量
* In Progress Tasks 数量
* Blocked Tasks 数量
* Review Tasks 数量
* Completed Tasks 数量
* 最近活动
* 潜在冲突

---

## FR-STATUS-003

支持详细模式：

```bash
carryctx status --all
```

---

## FR-STATUS-004

支持只查看当前 Agent：

```bash
carryctx status --mine
```

---

## FR-STATUS-005

支持机器可读输出：

```bash
carryctx status --json
```

---

# 11.12 Decision 管理

## FR-DECISION-001

Agent 应能够记录技术决策：

```bash
carryctx decision add \
  --title "Use LRU for authentication cache"
```

---

## FR-DECISION-002

Decision 应支持关联：

* Task
* Path
* Module
* Agent
* Session

---

## FR-DECISION-003

Decision 至少包含：

* Title
* Context
* Decision
* Consequences
* Related Tasks
* Related Paths
* Created By
* Created At

---

## FR-DECISION-004

用户应能够查询 Decision：

```bash
carryctx decision list
carryctx decision show DEC-001
carryctx decision search "cache"
```

---

# 11.13 Handoff

## FR-HANDOFF-001

Agent 应能够创建 Handoff：

```bash
carryctx handoff create
```

---

## FR-HANDOFF-002

CLI 应自动从 Task、Checkpoint 和 Git 获取：

* 当前 Task
* 当前 Agent
* 当前 Session
* Worktree
* Branch
* HEAD
* Changed Files
* 最新 Checkpoint
* Remaining Work
* Blocker
* 验证结果

---

## FR-HANDOFF-003

Agent 应补充：

* 交接摘要
* 重要实现细节
* 已知风险
* 推荐下一步
* 目标 Agent，可选

---

## FR-HANDOFF-004

其他 Agent 应能够查看并接管 Handoff：

```bash
carryctx handoff list
carryctx handoff show HANDOFF-001
carryctx handoff accept HANDOFF-001
```

---

# 11.14 路径范围与冲突感知

## FR-SCOPE-001

Task 应能够声明相关路径：

```bash
carryctx task scope add CTX-0001 "packages/auth/**"
```

---

## FR-SCOPE-002

路径范围第一阶段主要用于：

* 上下文相关性计算
* 潜在冲突提示
* 相关 Decision 过滤
* 相关 Task 查询

---

## FR-SCOPE-003

当两个 active Task 的路径范围可能重叠时，CarryCtx 应发出警告。

---

## FR-SCOPE-004

MVP 中路径范围默认为软约束。

CarryCtx 提示冲突，但不强制禁止文件修改。

---

## FR-SCOPE-005

后续版本可以增加 TTL Lease 和严格写入保护。

---

# 11.15 Event Log

## FR-EVENT-001

所有关键状态变化必须写入 Event Log。

---

## FR-EVENT-002

Event 至少包含：

* ID
* Event Type
* Actor
* Agent
* Session
* Task
* Payload
* Created At

---

## FR-EVENT-003

Event 应采用 append-only 设计。

---

## FR-EVENT-004

用户应能够查询事件：

```bash
carryctx event list
carryctx event list --task CTX-0001
carryctx event list --agent claude-auth
carryctx event list --since 24h
```

---

# 11.16 Doctor 与状态修复

## FR-DOCTOR-001

CarryCtx 应提供诊断命令：

```bash
carryctx doctor
```

---

## FR-DOCTOR-002

Doctor 至少检查：

* 当前目录是否位于 Git 仓库
* CarryCtx 是否已初始化
* 配置文件是否有效
* 数据库是否可读取
* Schema 是否匹配
* Git common directory 是否可访问
* Worktree 记录是否有效
* Branch 是否存在
* Session 是否长期未更新
* Task Owner 是否有效
* Dependency 是否循环
* Checkpoint Git 状态是否可解析

---

## FR-DOCTOR-003

安全问题可以自动修复：

```bash
carryctx doctor --fix
```

破坏性修复必须请求确认。

---

# 12. CLI 命令结构

建议使用以下一级命令：

```text
carryctx init
carryctx status
carryctx resume
carryctx context
carryctx checkpoint
carryctx doctor

carryctx project
carryctx agent
carryctx session
carryctx task
carryctx progress
carryctx worktree
carryctx decision
carryctx handoff
carryctx event
carryctx config
```

建议命令树：

```text
carryctx
├── init
├── status
├── resume
├── context
├── checkpoint
├── doctor
├── project
│   ├── show
│   └── migrate
├── agent
│   ├── register
│   ├── list
│   ├── show
│   └── current
├── session
│   ├── start
│   ├── show
│   ├── list
│   ├── pause
│   └── end
├── task
│   ├── create
│   ├── list
│   ├── show
│   ├── claim
│   ├── release
│   ├── start
│   ├── block
│   ├── review
│   ├── complete
│   ├── cancel
│   ├── depend
│   └── scope
├── progress
│   ├── todo
│   ├── done
│   ├── block
│   ├── list
│   ├── complete
│   ├── reopen
│   └── edit
├── worktree
│   ├── create
│   ├── bind
│   ├── list
│   ├── show
│   ├── unbind
│   └── remove
├── decision
│   ├── add
│   ├── list
│   ├── show
│   └── search
├── handoff
│   ├── create
│   ├── list
│   ├── show
│   └── accept
├── event
│   └── list
└── config
    ├── get
    ├── set
    └── list
```

---

# 13. 输出规范

## 13.1 人类可读输出

默认输出应：

* 简洁
* 层次明确
* 不输出无关字段
* 支持终端颜色
* 在非 TTY 环境禁用颜色
* 错误信息包含解决建议

---

## 13.2 JSON 输出

所有主要查询命令必须支持：

```bash
--json
```

所有 JSON 输出应包含：

```json
{
  "schemaVersion": 1,
  "success": true,
  "data": {}
}
```

错误输出：

```json
{
  "schemaVersion": 1,
  "success": false,
  "error": {
    "code": "TASK_ALREADY_CLAIMED",
    "message": "Task CTX-0001 is already claimed.",
    "details": {}
  }
}
```

---

## 13.3 Exit Code

建议定义：

```text
0  成功
1  一般错误
2  参数错误
3  状态冲突
4  Git 错误
5  数据库错误
6  配置错误
7  未找到资源
```

具体值应在 CLI 规范中固定。

---

# 14. 数据存储需求

## 14.1 配置数据

适合提交到 Git 的配置保存在：

```text
.carryctx/config.json
```

例如：

```json
{
  "schemaVersion": 1,
  "project": {
    "name": "VectoJS",
    "taskPrefix": "VCT"
  },
  "git": {
    "mainBranch": "main",
    "worktreeDirectory": "../.worktrees"
  },
  "session": {
    "staleAfterMinutes": 120
  }
}
```

---

## 14.2 运行状态

运行状态保存在：

```text
<git-common-dir>/carryctx/state.sqlite
```

原因：

* 多个 worktree 共享
* 不污染业务代码目录
* 不参与 branch merge
* 查询效率高
* 支持事务

---

## 14.3 SQLite 配置

建议启用：

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
```

---

## 14.4 数据库迁移

每个数据库必须保存 Schema Version。

CLI 升级后应通过：

```bash
carryctx project migrate
```

执行迁移。

迁移前应自动创建备份。

---

## 14.5 数据备份

CarryCtx 应支持：

```bash
carryctx project backup
carryctx project restore <backup>
```

MVP 可以先在迁移和修复前自动备份。

---

# 15. 概念数据模型

核心实体：

```text
Project
Agent
Session
Task
TaskDependency
TaskScope
ProgressItem
Worktree
Checkpoint
Decision
Handoff
Event
```

关系：

```text
Project 1 ── N Agent
Project 1 ── N Task
Project 1 ── N Worktree

Agent 1 ── N Session
Agent 0 ── N Task

Task 1 ── N Session
Task 0 ── 1 Worktree
Task N ── N TaskDependency
Task 1 ── N ProgressItem
Task 1 ── N Checkpoint
Task N ── N Decision
Task 1 ── N Handoff

Session 1 ── N Checkpoint
Session 1 ── N Event
```

---

# 16. Context 相关性规则

CarryCtx 生成上下文时，应使用确定性规则计算相关信息。

相关性从高到低：

1. 当前 Task
2. 当前 Task 最新 Checkpoint
3. 当前 Task 未完成事项
4. 当前 Task Blocker
5. 当前 Task 直接依赖
6. 直接依赖最近状态变化
7. 被当前 Task 阻塞的任务
8. Path Scope 重叠任务
9. 当前 Agent 的其他活跃任务
10. 与相关路径绑定的 Decision
11. 最近项目级 Decision
12. 最近项目事件

默认不包含：

* 已完成很久且无依赖关系的任务
* 与当前路径无关的 Checkpoint
* 全部历史 Session
* 所有 Agent 的完整事件记录

---

# 17. Agent Skill 需求

CarryCtx 应提供一个通用 Agent Skill。

建议目录：

```text
skills/
└── carryctx/
    ├── SKILL.md
    ├── references/
    │   ├── session-lifecycle.md
    │   ├── task-workflow.md
    │   ├── checkpoint-policy.md
    │   ├── handoff-policy.md
    │   └── troubleshooting.md
    └── scripts/
        ├── start-session.sh
        ├── resume-task.sh
        └── checkpoint.sh
```

---

## 17.1 Skill 启动协议

每个 Agent 新窗口启动时：

1. 检查当前目录是否启用了 CarryCtx
2. 执行 `carryctx session start`
3. 执行 `carryctx resume --json`
4. 阅读当前 Task、Checkpoint 和 Remaining Work
5. 检查当前 Git worktree 状态
6. 开始工作前确认 Task 和 worktree 一致

---

## 17.2 Skill 工作协议

Agent 工作期间应：

* 在完成重要阶段后创建 Checkpoint
* 发现新工作时添加 Progress Todo
* 完成工作后更新 Progress
* 遇到阻塞时记录 Blocker
* 作出重要架构决策时记录 Decision
* 不将聊天上下文视为唯一状态来源

---

## 17.3 Skill 结束协议

Agent 暂停或关闭窗口前应：

1. 更新 Completed Items
2. 更新 Remaining Items
3. 记录 Blocker
4. 创建 Checkpoint
5. 必要时创建 Handoff
6. 执行 `carryctx session end`

---

## 17.4 Skill 约束

Skill 不能：

* 直接修改 SQLite
* 直接伪造 CarryCtx 状态文件
* 通过编辑生成文档替代 CLI
* 在任务完成前自动标记 completed
* 忽略 Git 工作区未提交状态

---

# 18. 核心用户流程

# 18.1 第一次初始化

```bash
cd project
carryctx init

carryctx agent register \
  --name claude-core \
  --provider claude-code

carryctx task create \
  --title "Implement state persistence"

carryctx task claim CTX-0001
carryctx worktree bind CTX-0001
carryctx session start
```

---

# 18.2 Agent 工作并关闭窗口

```bash
carryctx progress done "Created SQLite schema"
carryctx progress todo "Implement migrations"

carryctx checkpoint \
  --done "Created initial SQLite schema" \
  --remaining "Implement migration runner" \
  --next "Add migration integration tests"

carryctx session end
```

---

# 18.3 新窗口恢复

```bash
cd project-worktree
carryctx session start
carryctx resume
```

期望输出：

```text
Project: CarryCtx
Agent: claude-core
Task: CTX-0001 — Implement state persistence
Status: in_progress

Last checkpoint:
  Created initial SQLite schema

Remaining:
  - Implement migration runner

Next:
  Add migration integration tests

Git:
  Branch: carryctx/ctx-0001-state-persistence
  HEAD: 32ac891
  Worktree: dirty
  Modified files: 2
```

---

# 18.4 不同 Agent 接管任务

原 Agent：

```bash
carryctx handoff create \
  --target opencode-core
```

新 Agent：

```bash
carryctx agent register \
  --name opencode-core \
  --provider opencode

carryctx handoff accept HANDOFF-001
carryctx session start
carryctx resume
```

---

# 18.5 查看其他任务进度

```bash
carryctx status --all
```

输出示例：

```text
In Progress

CTX-0001  State persistence
Owner: claude-core
Remaining: 2
Last checkpoint: 18 minutes ago

CTX-0002  Worktree integration
Owner: opencode-git
Remaining: 4
Blocked by: CTX-0001

Blocked

CTX-0003  Context generator
Reason: Waiting for task schema
Blocked by: CTX-0001
```

---

# 19. 非功能需求

# 19.1 性能

## NFR-PERF-001

不需要扫描 Git 历史的本地状态查询，目标响应时间应小于 100ms。

## NFR-PERF-002

包含 Git status 的常规查询，目标响应时间应小于 1 秒。

## NFR-PERF-003

CarryCtx 不应在每次命令执行时扫描整个仓库文件内容。

---

# 19.2 可靠性

## NFR-REL-001

Task claim、Session start 等关键写操作必须使用数据库事务。

## NFR-REL-002

CLI 异常退出不能导致数据库处于部分更新状态。

## NFR-REL-003

迁移和破坏性修复前必须创建备份。

---

# 19.3 可恢复性

## NFR-REC-001

Agent 窗口关闭后，Task 和 Checkpoint 必须保留。

## NFR-REC-002

CarryCtx 必须能够识别 stale Session。

## NFR-REC-003

数据库记录与 Git worktree 不一致时，应提供诊断和修复建议。

---

# 19.4 可移植性

## NFR-PORT-001

第一阶段优先支持：

* Linux
* macOS

## NFR-PORT-002

架构和路径处理必须为 Windows 支持保留兼容性。

## NFR-PORT-003

不得依赖 Bash 才能执行核心功能。

---

# 19.5 隐私

## NFR-PRIV-001

CarryCtx 默认不访问网络。

## NFR-PRIV-002

CarryCtx 默认不上传：

* 代码
* Git Diff
* Task
* Checkpoint
* Agent 信息
* 项目路径

## NFR-PRIV-003

未来增加远程同步时，必须明确启用。

---

# 19.6 Agent 无关性

## NFR-AGENT-001

所有核心能力必须通过 CLI 提供。

## NFR-AGENT-002

不能要求某个 Agent 支持特定私有 Plugin API。

## NFR-AGENT-003

核心命令必须支持 JSON 输出。

---

# 19.7 可测试性

## NFR-TEST-001

业务逻辑应与终端 UI、Git Adapter 和 SQLite Adapter 分离。

## NFR-TEST-002

每个核心状态转换应有单元测试。

## NFR-TEST-003

应使用临时 Git 仓库进行集成测试。

---

# 20. TypeScript 技术要求

CarryCtx 计划使用 TypeScript 开发并发布到 npm。

推荐包结构：

```text
carryctx/
├── package.json
├── tsconfig.json
├── src/
│   ├── cli/
│   ├── commands/
│   ├── domain/
│   ├── services/
│   ├── repositories/
│   ├── adapters/
│   │   ├── git/
│   │   ├── sqlite/
│   │   └── filesystem/
│   ├── output/
│   └── errors/
├── migrations/
├── skills/
├── tests/
└── docs/
```

---

## 20.1 npm 配置

```json
{
  "name": "carryctx",
  "type": "module",
  "bin": {
    "carryctx": "./dist/cli.js"
  }
}
```

---

## 20.2 架构原则

CLI 应采用分层架构：

```text
CLI Command Layer
        ↓
Application Service
        ↓
Domain Model
        ↓
Repository / Adapter
        ↓
SQLite / Git / File System
```

业务逻辑不得直接散落在 CLI 参数处理代码中。

---

## 20.3 Git 操作

第一版本优先调用系统 Git CLI，而不是完全依赖 libgit2。

例如：

```text
git rev-parse
git status
git worktree list
git branch
git diff
```

这样能够尽可能保持与用户现有 Git 配置一致。

---

## 20.4 SQLite Driver

SQLite Driver 应支持：

* Transaction
* WAL
* Migration
* Prepared Statement
* Foreign Key
* Backup

具体 Driver 在技术选型阶段确定。

---

# 21. MVP 范围

CarryCtx v0.1 必须完成以下闭环。

## P0：必须实现

1. `carryctx init`
2. Agent 注册与查询
3. Session start/end
4. Task 创建、查询、claim 和状态更新
5. Task dependency
6. Worktree bind
7. Progress Item
8. Checkpoint
9. Resume
10. Status
11. Context
12. Event Log
13. SQLite 持久化
14. Git 状态自动采集
15. `--json`
16. 通用 Agent Skill
17. Doctor 基础检查

---

## P1：建议实现

1. Worktree 自动创建
2. Handoff
3. Decision
4. Task path scope
5. 潜在冲突检测
6. Session stale 检测
7. 数据库备份
8. Schema migration
9. Shell completion
10. Markdown context 输出

---

## P2：后续版本

1. TTL Path Lease
2. Git Hook 集成
3. MCP Server
4. Remote Sync Adapter
5. PostgreSQL 或 Dolt 后端
6. Web Dashboard
7. LSP / SCIP 集成
8. Code impact analysis
9. GitHub Issue 同步
10. GitHub Pull Request 同步
11. 多仓库 Project
12. Task Template

---

# 22. MVP 验收标准

## AC-001 项目初始化

在一个 Git 仓库中执行：

```bash
carryctx init
```

应成功创建配置和状态数据库。

---

## AC-002 创建并接管任务

执行：

```bash
carryctx task create --title "Test task"
carryctx task claim CTX-0001
```

当前 Agent 应成为 Task Owner。

---

## AC-003 记录进度

执行：

```bash
carryctx progress done "Completed A"
carryctx progress todo "Complete B"
```

任务查询中应正确显示 Completed 和 Remaining。

---

## AC-004 创建 Checkpoint

执行：

```bash
carryctx checkpoint \
  --done "Completed A" \
  --remaining "Complete B"
```

Checkpoint 应记录当前 branch、HEAD、worktree 和 dirty state。

---

## AC-005 新 Session 恢复

结束 Session 后重新打开终端并执行：

```bash
carryctx session start
carryctx resume
```

应恢复：

* 当前 Task
* 最新 Checkpoint
* 已完成事项
* 剩余事项
* 当前 Git 状态

---

## AC-006 跨 Agent 接管

Agent A 创建 Handoff 后，Agent B 应能够读取任务状态并继续工作。

---

## AC-007 全局状态

存在多个 Task 和 Session 时：

```bash
carryctx status --all
```

应正确显示每个任务的状态、Owner 和最新进度。

---

## AC-008 依赖状态

当 Task B 依赖未完成的 Task A 时，Task B 不应出现在 ready 列表中。

---

## AC-009 多 worktree 共享状态

在两个 linked worktree 中执行 CarryCtx，应读取同一个项目数据库。

---

## AC-010 JSON 输出

以下命令应提供稳定 JSON：

```bash
carryctx status --json
carryctx resume --json
carryctx task list --json
carryctx context --json
```

---

## AC-011 离线运行

关闭网络后，所有 MVP 核心命令仍应正常工作。

---

## AC-012 异常恢复

模拟 Session 未正常结束后，CarryCtx 应能够识别 stale Session，并允许恢复或终止。

---

# 23. 建议开发阶段

## 阶段一：基础设施

* npm 工程
* CLI Framework
* 配置加载
* Git 仓库发现
* Git common directory
* SQLite
* Migration
* Error Model
* JSON 输出

---

## 阶段二：Task 与 Session

* Agent
* Session
* Task
* Task Dependency
* Progress Item
* Event Log

---

## 阶段三：连续性核心

* Checkpoint
* Resume
* Context
* Status
* Git 状态采集

这是 CarryCtx 最关键的阶段。

---

## 阶段四：Worktree 与协作

* Worktree bind/create
* Handoff
* Decision
* Path Scope
* Conflict Warning

---

## 阶段五：Agent Skill

* 通用 SKILL.md
* Claude Code 使用说明
* OpenCode 使用说明
* Generic Shell Agent 使用说明
* 测试跨 Session 恢复流程

---

# 24. 成功指标

CarryCtx 第一阶段的成功不以功能数量衡量。

核心成功指标是：

## 24.1 恢复效率

一个新 Agent Session 在执行 `carryctx resume` 后，能够在较短时间内理解并继续当前任务，而不需要完整阅读旧对话。

---

## 24.2 状态完整性

任务的重要状态不再只存在于 Agent 对话中。

---

## 24.3 信息准确性

CarryCtx 展示的：

* Task
* Worktree
* Branch
* Commit
* Progress
* Dependency

与实际项目状态一致。

---

## 24.4 跨 Agent 可用性

同一个 Task 可以在 Claude Code、OpenCode 等不同工具之间交接。

---

## 24.5 低使用成本

Agent 的常见操作不应需要大量命令。

核心使用流程应围绕：

```bash
carryctx resume
carryctx checkpoint
carryctx status
carryctx context
```

---

# 25. 风险

## 25.1 Agent 不主动更新状态

如果 Agent 不创建 Checkpoint，CarryCtx 只能读取 Git 状态，无法准确知道语义进度。

解决方向：

* Skill 强制规定检查点流程
* Session end 时提示
* Git commit 后提示创建 Checkpoint
* 支持从 Git Commit 生成 Checkpoint 草稿

---

## 25.2 状态与代码不一致

Checkpoint 创建后代码可能继续变化。

解决方向：

* 保存 Checkpoint HEAD
* Resume 时比较当前 HEAD
* 检测 dirty state 差异
* 明确标记 Checkpoint 是否 stale

---

## 25.3 任务粒度不合理

任务过大会导致 Remaining Work 过多，任务过小会增加管理成本。

CarryCtx 不负责自动解决任务拆分，但可以在文档中提供建议。

---

## 25.4 多机器状态同步

SQLite 适合本地多 worktree，但不适合直接跨机器共享。

第一版本明确定位 local-first。

后续通过 Storage Adapter 增加远程同步。

---

## 25.5 CLI 复杂度膨胀

如果添加过多命令，CarryCtx 可能逐渐变成项目管理平台。

必须始终围绕以下核心价值评估新功能：

> 这个功能是否能够改善 Coding Agent 的项目状态连续性和协作感知？

---

# 26. 待决策事项

以下事项在进入详细设计前需要进一步确定：

1. Task ID 默认前缀是否使用 `CTX`
2. 配置文件使用 JSON、JSONC 还是 TOML
3. SQLite Driver 选择
4. CLI Framework 选择
5. 是否在 v0.1 实现自动 worktree create
6. 是否在 v0.1 实现 Handoff
7. 是否默认允许一个 Agent 同时拥有多个 in-progress Task
8. Session stale 默认时间
9. Checkpoint 是否允许编辑
10. 是否需要将部分状态导出为可提交的 JSONL
11. 是否需要生成自动 Markdown 状态报告
12. Skill 的安装方式
13. 是否在 v0.1 支持 MCP
14. Windows 支持优先级
15. npm Package 使用无作用域 `carryctx` 还是 scoped package

---

# 27. 最终产品原则

CarryCtx 应遵循以下原则：

1. **Task 比 Session 更持久**
2. **结构化状态比聊天记录更可靠**
3. **Git 状态自动采集，语义状态由 Agent 报告**
4. **Markdown 是输出视图，不是唯一数据库**
5. **CLI 是 Source of Truth**
6. **Skill 是操作协议**
7. **默认本地运行**
8. **不绑定特定 Agent**
9. **不承担 Agent 编排**
10. **优先打磨 Resume、Checkpoint、Context 和 Status**

---

# 28. 产品摘要

CarryCtx 是一个使用 TypeScript 开发并通过 npm 发布的本地 CLI。

它为大型 Git 项目提供结构化的 Agent 工作状态，使多个 Coding Agent 以及同一 Agent 的不同窗口能够持续理解：

* 自己正在处理什么
* 上次完成了什么
* 还有什么没有完成
* 当前 Git 工作区是什么状态
* 其他相关任务进展如何
* 当前有哪些依赖和阻塞
* 下一步应该继续做什么

CarryCtx 不负责运行或编排 Agent，而是作为所有 Coding Agent 共享的项目记忆和协作状态层。

