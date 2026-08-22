# CarryCtx 配置与存储规范

**文档路径：** `carryctx-docs/configuration.md`
**文档版本：** v0.1
**适用版本：** CarryCtx v0.1.x

---

## 1. 文档目的

本文档定义 CarryCtx 的：

- 配置文件位置
- XDG Base Directory 使用规范
- 项目级 `.carryctx/` 目录结构
- 配置优先级
- 配置合并规则
- 项目状态数据库位置
- 静态资源、缓存和日志存储位置
- 多 Git worktree 共享状态方式
- 配置文件格式与 Schema
- 环境变量映射规则
- 跨平台路径策略

CarryCtx 必须明确区分：

- Configuration
- Persistent State
- Application Data
- Cache
- Runtime Files
- Project Configuration
- Project Coordination State

这些数据不得全部混合保存在 `.carryctx/` 中。

---

# 2. XDG 路径规范

CarryCtx 在 Linux 和其他 Unix-like 系统上遵循 XDG Base Directory Specification。

正确的环境变量是：

```text
XDG_CONFIG_HOME
XDG_DATA_HOME
XDG_STATE_HOME
XDG_CACHE_HOME
XDG_RUNTIME_DIR
```

CarryCtx 不使用不存在的：

```text
XDG_CONFIG_DIR
```

---

## 2.1 全局配置目录

```text
${XDG_CONFIG_HOME:-$HOME/.config}/carryctx/
```

默认：

```text
~/.config/carryctx/
```

目录结构：

```text
~/.config/carryctx/
├── config.toml
└── profiles/
    ├── default.toml
    ├── strict.toml
    └── minimal.toml
```

用途：

- 用户级默认配置
- 默认 Agent 身份
- UI 和输出偏好
- 默认 Session 策略
- 默认 worktree 目录模板
- 默认 Context 输出限制
- 用户定义的配置 Profile

该目录中的配置适用于所有 CarryCtx 项目。

---

## 2.2 全局持久状态目录

```text
${XDG_STATE_HOME:-$HOME/.local/state}/carryctx/
```

默认：

```text
~/.local/state/carryctx/
```

目录结构：

```text
~/.local/state/carryctx/
├── registry.sqlite
├── logs/
├── history/
└── backups/
```

用途：

- 已发现项目注册表
- 最近访问的项目
- 全局 Agent 使用记录
- CarryCtx 自身日志
- 全局迁移记录
- 不属于单个 Git 项目的持久状态

`registry.sqlite` 只能作为项目索引，不能成为项目任务状态的唯一数据库。

项目的权威任务状态仍然存储在项目对应的 Git common directory 中。

---

## 2.3 全局数据目录

```text
${XDG_DATA_HOME:-$HOME/.local/share}/carryctx/
```

默认：

```text
~/.local/share/carryctx/
```

目录结构：

```text
~/.local/share/carryctx/
├── skills/
│   ├── carryctx/
│   └── providers/
├── templates/
├── schemas/
├── profiles/
└── assets/
```

用途：

- 已安装的 Agent Skill
- Context 模板
- Checkpoint 模板
- Handoff 模板
- JSON Schema
- TOML Schema
- 静态资源
- Provider-specific Skill 适配文件

这些文件属于持久应用数据，不应因为清理缓存而丢失。

---

## 2.4 缓存目录

```text
${XDG_CACHE_HOME:-$HOME/.cache}/carryctx/
```

默认：

```text
~/.cache/carryctx/
```

目录结构：

```text
~/.cache/carryctx/
├── context/
├── indexes/
├── git/
├── completions/
├── temporary/
└── update-metadata/
```

用途：

- 生成后的 Context 缓存
- 代码搜索索引
- Git 状态短期缓存
- Shell completion 缓存
- 临时导出文件
- 可重新下载或重新生成的资源
- 更新检查元数据

该目录可以被用户安全删除。

删除缓存不得导致：

- Task 丢失
- Session 丢失
- Checkpoint 丢失
- Decision 丢失
- Handoff 丢失
- 项目配置丢失

---

## 2.5 Runtime 目录

优先使用：

```text
$XDG_RUNTIME_DIR/carryctx/
```

用途：

- 进程锁
- 临时 Socket
- daemon PID
- 短生命周期 IPC 文件

如果 `XDG_RUNTIME_DIR` 不存在，则回退到：

```text
${XDG_CACHE_HOME:-$HOME/.cache}/carryctx/runtime/
```

Runtime 文件不得被视为持久状态。

---

# 3. 项目级 `.carryctx/` 目录

每个启用 CarryCtx 的项目在 Git repository root 中包含：

```text
<repository-root>/.carryctx/
```

推荐结构：

```text
.carryctx/
├── config.toml
├── config.local.toml
├── skills/
├── templates/
├── schemas/
└── README.md
```

---

## 3.1 `.carryctx/config.toml`

项目级主配置。

该文件：

- 应提交到 Git
- 应由所有 worktree 共享其逻辑配置
- 应参与 Code Review
- 可以覆盖全局配置
- 不得包含机器私有路径或 Secret
- 不得包含运行时 Task 状态

示例：

```toml
schema_version = 1

[project]
id = "carryctx"
name = "CarryCtx"
task_prefix = "CTX"

[git]
main_branch = "main"
worktree_root = "../.worktrees"
branch_template = "carryctx/{task_id}-{slug}"

[session]
stale_after = "2h"
single_active_session_per_agent = true

[task]
# Deprecated compatibility key. Multiple active tasks per agent are supported;
# this key is accepted for existing configurations but has no effect on claim,
# start, or assign. Capacity policy belongs to the commander or external harness.
single_active_task_per_agent = true
strict_completion = false

[context]
default_mode = "compact"
max_events = 10
lookback = "7d"
include_git_status = true

[checkpoint]
require_before_session_end = true
capture_diff_stats = true
capture_untracked_files = true

[output]
color = "auto"
unicode = true
# 文本输出完整记录（等价于全局 --verbose），默认 false（compact 单行摘要）
verbose = false
# 按命令裁剪输出字段（同时作用于 text 与 JSON 的 data）
[output.fields]
"handoff.list" = ["display_id", "status", "summary"]
```

---

## 3.2 `.carryctx/config.local.toml`

项目在当前机器或当前 worktree 上的本地覆盖配置。

该文件：

- 不提交到 Git
- 必须加入 `.gitignore`
- 覆盖 `.carryctx/config.toml`
- 可以包含绝对路径
- 可以包含当前机器的工具路径
- 可以指定当前开发者的默认 Agent
- 不得存储认证 Token

示例：

```toml
[git]
worktree_root = "/home/xuepoo/Develop/worktrees/carryctx"

[agent]
default_name = "claude-core"
default_provider = "claude-code"

[output]
color = "always"
```

项目初始化时应确保 `.gitignore` 包含：

```gitignore
.carryctx/config.local.toml
```

---

## 3.3 `.carryctx/skills/`

项目专用 Skill。

用途：

- 覆盖全局 CarryCtx Skill
- 描述项目特有工作协议
- 定义项目测试流程
- 定义项目架构约束
- 定义 Checkpoint 和 Handoff 要求

推荐：

```text
.carryctx/skills/
└── carryctx/
    ├── SKILL.md
    └── references/
```

项目 Skill 优先级高于全局 Skill。

---

## 3.4 `.carryctx/templates/`

项目专用模板，例如：

```text
.carryctx/templates/
├── checkpoint.md
├── handoff.md
├── decision.md
└── context.md
```

项目模板优先于全局模板。

---

## 3.5 `.carryctx/schemas/`

保存项目自定义扩展 Schema。

第一版本只预留目录，不要求实现用户自定义数据实体。

---

# 4. 项目运行状态目录

项目 Task、Session、Checkpoint 等运行状态不得保存在：

```text
.carryctx/state.sqlite
```

因为不同 Git worktree 中的 `.carryctx/` 是不同工作目录，无法自然共享同一个数据库。

项目权威状态存储在：

```text
$(git rev-parse --git-common-dir)/carryctx/
```

结构：

```text
<git-common-dir>/carryctx/
├── state.sqlite
├── state.sqlite-wal
├── state.sqlite-shm
├── backups/
├── migrations/
├── locks/
└── metadata.json
```

该目录能够被同一个 Git repository 的所有 linked worktree 访问。

---

## 4.1 项目状态数据库

默认：

```text
<git-common-dir>/carryctx/state.sqlite
```

保存：

- Project
- Agent
- Session
- Task
- Dependency
- Progress Item
- Worktree
- Checkpoint
- Decision
- Handoff
- Event
- Project-level metadata

该数据库是项目运行状态的 Source of Truth。

---

## 4.2 数据库配置

初始化数据库时执行：

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

关键写操作必须使用事务。

---

## 4.3 数据库备份

备份目录：

```text
<git-common-dir>/carryctx/backups/
```

命名格式：

```text
state-2026-07-22T183000Z-v1.sqlite
```

以下操作前必须自动备份：

- Schema migration
- `doctor --fix` 的破坏性修复
- Database restore
- 批量数据导入
- 不可逆数据转换

---

# 5. 配置文件格式

CarryCtx 统一使用：

```text
TOML
```

文件名：

```text
config.toml
config.local.toml
```

选择 TOML 的原因：

- 适合人工维护
- 支持注释
- 层级结构清晰
- 比 JSON 更适合配置文件
- 与 `justfile`、Cargo 等开发工具风格接近
- 不需要将运行时状态写入配置文件

运行时状态仍然使用 SQLite，不使用 TOML。

---

# 6. 配置加载优先级

CarryCtx 的有效配置按以下顺序合并，后者覆盖前者：

```text
1. CarryCtx 内置默认值
2. 全局配置
3. 全局 Profile
4. 项目配置
5. 项目本地配置
6. 显式指定的额外配置文件
7. 环境变量
8. CLI 参数
```

完整表示：

```text
Built-in Defaults
    ↓
$XDG_CONFIG_HOME/carryctx/config.toml
    ↓
$XDG_CONFIG_HOME/carryctx/profiles/<profile>.toml
    ↓
<repo>/.carryctx/config.toml
    ↓
<repo>/.carryctx/config.local.toml
    ↓
--config <file>
    ↓
CARRYCTX_* Environment Variables
    ↓
Command-line Flags
```

因此：

> 项目级配置默认覆盖全局配置。

CLI 参数始终具有最高优先级。

---

# 7. 配置合并规则

## 7.1 Scalar

字符串、数字、布尔值由高优先级配置覆盖：

```toml
[output]
color = "auto"
```

项目配置中的：

```toml
[output]
color = "never"
```

最终值为：

```text
never
```

---

## 7.2 Table

TOML Table 进行递归合并。

全局配置：

```toml
[context]
max_events = 20
lookback = "14d"
```

项目配置：

```toml
[context]
max_events = 10
```

最终：

```toml
[context]
max_events = 10
lookback = "14d"
```

---

## 7.3 Array

Array 默认整体替换，不进行隐式拼接。

全局：

```toml
[verification]
commands = ["bun test", "bun run typecheck"]
```

项目：

```toml
[verification]
commands = ["just check"]
```

最终：

```toml
commands = ["just check"]
```

显式追加应通过独立字段实现：

```toml
[verification]
commands_append = ["actionlint"]
```

---

## 7.4 Relative Path

相对路径相对于声明该值的配置文件所在目录解析。

例如项目配置：

```toml
[git]
worktree_root = "../.worktrees"
```

相对于：

```text
<repository-root>/.carryctx/
```

解析。

但 `git.worktree_root` 可以特别定义为相对于 repository root，以减少意外。

每个路径字段必须在 Schema 中明确其解析基准。

---

# 8. 配置环境变量

所有环境变量使用：

```text
CARRYCTX_
```

前缀。

嵌套配置使用双下划线：

```text
CARRYCTX_<SECTION>__<KEY>
```

示例：

```bash
export CARRYCTX_AGENT__DEFAULT_NAME=claude-core
export CARRYCTX_OUTPUT__COLOR=never
export CARRYCTX_CONTEXT__MAX_EVENTS=20
export CARRYCTX_SESSION__STALE_AFTER=4h
```

特殊环境变量：

```text
CARRYCTX_CONFIG
CARRYCTX_PROJECT
CARRYCTX_AGENT
CARRYCTX_SESSION
CARRYCTX_TASK
CARRYCTX_PROFILE
CARRYCTX_NO_COLOR
CARRYCTX_LOG
```

优先级：

```text
CLI flag > environment variable > config file
```

---

# 9. 配置校验

CarryCtx 启动时应：

1. 解析 TOML
2. 验证 `schema_version`
3. 验证字段类型
4. 验证 Enum
5. 验证持续时间格式
6. 验证路径格式
7. 检测未知字段
8. 检测冲突配置
9. 输出配置来源

未知字段默认视为错误：

```text
Unknown configuration key: session.stale_minutes
Did you mean: session.stale_after?
```

可以通过兼容模式降级为 Warning：

```bash
carryctx --config-compat warn
```

---

# 10. 配置查询命令

```bash
carryctx config list
carryctx config get session.stale_after
carryctx config set --project output.color never
carryctx config unset --project output.color
carryctx config sources
carryctx config validate
carryctx config path
```

指定作用域：

```bash
carryctx config set --global output.color never
carryctx config set --project task.strict_completion true
carryctx config set --local agent.default_name claude-core
```

写操作必须且只能显式指定一个作用域：

```text
--global
--project
--local
```

避免意外修改错误层级。

---

# 11. 配置来源调试

执行：

```bash
carryctx config get context.max_events --explain
```

输出：

```text
Value: 10

Sources:
  Built-in default: 20
  Global config:    15
  Project config:   10
  Environment:      not set
  CLI:              not set

Effective source:
  /project/.carryctx/config.toml
```

这项功能对于大型项目排查配置问题非常重要。

---

# 12. 跨平台策略

## Linux

完整遵循 XDG。

## macOS

优先尊重显式设置的 XDG 环境变量。

未设置时，可使用平台目录 Adapter：

```text
~/Library/Application Support/carryctx
~/Library/Caches/carryctx
```

## Windows

使用 Known Folders：

```text
%APPDATA%\carryctx
%LOCALAPPDATA%\carryctx
```

内部代码不得直接拼接：

```text
~/.config
~/.cache
```

必须通过统一的 `PathResolver` 解析。

---

# 13. 权威数据边界

| 数据           | 默认位置                         | 是否权威 |     是否可删除 |
| -------------- | -------------------------------- | -------: | -------------: |
| 全局配置       | XDG Config                       |       是 |             否 |
| 项目配置       | `.carryctx/config.toml`          |       是 |             否 |
| 项目本地配置   | `.carryctx/config.local.toml`    |       是 | 可重建但不建议 |
| 项目 State DB  | Git common dir                   |       是 |             否 |
| 全局项目注册表 | XDG State                        |       否 |       可以重建 |
| Skill          | XDG Data / `.carryctx/skills`    |       是 |     视来源而定 |
| Template       | XDG Data / `.carryctx/templates` |       是 |     视来源而定 |
| Context Cache  | XDG Cache                        |       否 |             是 |
| Code Index     | XDG Cache                        |       否 |             是 |
| Runtime Lock   | XDG Runtime                      |       否 |             是 |

---

# 14. 项目初始化结果

执行：

```bash
carryctx init
```

应创建：

```text
<repo>/.carryctx/config.toml
<repo>/.carryctx/README.md
<git-common-dir>/carryctx/state.sqlite
```

并更新：

```text
.gitignore
```

加入：

```gitignore
.carryctx/config.local.toml
```

初始化不得自动将：

```text
.carryctx/
```

整体加入 `.gitignore`。

项目配置、Skill 和模板应允许提交到 Git。

---

# 15. 最终规范

CarryCtx 的存储原则是：

```text
全局偏好       → XDG_CONFIG_HOME
持久应用数据   → XDG_DATA_HOME
应用运行历史   → XDG_STATE_HOME
可重建内容     → XDG_CACHE_HOME
短期进程文件   → XDG_RUNTIME_DIR
项目声明配置   → <repo>/.carryctx/
项目协作状态   → <git-common-dir>/carryctx/state.sqlite
```

`.carryctx/` 是项目的声明式配置和扩展目录，不是运行时数据库目录。
