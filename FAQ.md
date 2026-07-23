# Multi-Agent Workflow FAQ & Simulation

随着 CarryCtx 的引入，大型代码库不再只能依赖单一的大模型代理 (Agent) 单打独斗。你可以组建一个“AI 开发团队”，让擅长不同领域的多个 Agent 协同工作。

本文将模拟一个大型项目中多 Agent 的协作全景，介绍新功能开发的完整工作流，并解答在这个过程中可能遇到的常见问题。

---

## 🏗 场景模拟：大型电商系统的多 Agent 协同

假设我们正在维护一个大型的全栈电商项目，并且我们有以下“员工”：
- **Agent A (PM / 架构师)**：例如高级推理模型（Claude 3.5 Sonnet / O1），负责需求拆解和任务分配。
- **Agent B (后端开发)**：擅长 Rust/Go 的模型，负责实现 API 和数据库迁移。
- **Agent C (前端开发)**：擅长 React/Tailwind 的模型，负责 UI 渲染。
- **Agent D (测试 / QA)**：负责补充单元测试和端到端测试。

### 🚀 新功能开发工作流：添加“商品秒杀”功能

当用户提出：“请为系统添加一个商品秒杀模块”时，标准的协作流如下：

#### 1. 需求拆解与派发 (Agent A - 架构师)
Agent A 启动并拉起 CarryCtx：
```bash
carryctx agent current --name "Architect-Claude"
carryctx session start
```
它分析需求后，在 CarryCtx 中拆解并创建任务树，并设定依赖关系：
```bash
# 创建总任务
carryctx task create --title "商品秒杀功能模块" --id CTX-100

# 创建后端任务，归属总任务
carryctx task create --title "实现秒杀扣减库存 API" --id CTX-101 --parent CTX-100

# 创建前端任务，依赖后端完成
carryctx task create --title "开发秒杀倒计时与抢购 UI" --id CTX-102 --parent CTX-100 --depends-on CTX-101
```

#### 2. 后端开发介入 (Agent B - 后端)
Agent B 定期运行 `carryctx task list --status ready` 发现 `CTX-101` 就绪。
```bash
carryctx agent current --name "Backend-GPT4"
carryctx task claim CTX-101

# 为防止污染主分支，为该任务创建隔离的 git worktree
carryctx worktree create --task CTX-101
cd .worktrees/CTX-101
carryctx session start
```
开发期间，Agent B 不断记录关键技术决策：
```bash
carryctx progress note "使用 Redis Lua 脚本保证库存扣减的原子性"
carryctx progress done "Redis lua 脚本编写完成"
```
完成并提交代码后（如果安装了 hook 会自动 checkpoint）：
```bash
git commit -m "feat: 实现秒杀扣减 API"
carryctx task complete CTX-101
carryctx session end
```

#### 3. 前端开发介入 (Agent C - 前端)
由于后端完成，`CTX-102` 的状态自动从 `blocked` 变更为 `ready`。Agent C 认领该任务：
```bash
carryctx agent current --name "Frontend-Gemini"
carryctx task claim CTX-102
carryctx worktree create --task CTX-102
cd .worktrees/CTX-102
carryctx session start
carryctx resume
```
**关键点**：`carryctx resume` 时，Agent C 能够看到依赖任务 `CTX-101` 留下的 checkpoint 和笔记，瞬间知道了“后端是用 Redis Lua 脚本处理的，接口路由是 `/api/flash-sale`”。
随后 Agent C 完成前端 UI 开发并结束任务。

---

## ❓ 常见问题与排错 (FAQ)

### 1. 多个 Agent 会不会同时认领同一个任务导致代码冲突？
**不会。**
CarryCtx 采用底层 SQLite 事务锁。当 `Agent B` 运行 `carryctx task claim CTX-101` 时，数据库会写入其独占所有权 (`owner_agent_id`)。此时如果 `Agent C` 尝试 claim，会被明确拒绝。
这就保证了同一时刻，一个任务节点绝对只被一个 Agent 独占。

### 2. 前端 Agent (C) 如何知道后端 Agent (B) 写了什么接口？
在没有 CarryCtx 时，Agent 之间上下文完全断裂。
在 CarryCtx 体系下，由于前后端任务存在强依赖 (`--depends-on`)，当前端运行 `carryctx resume` 时，CarryCtx 引擎会自动拉取上游依赖任务的 **Checkpoints (快照)**、**Progress (进度日志)** 以及关联的 **Git Commits** 交给前端 Agent。
后端留下的关键 `carryctx progress note` 就是最好的通信桥梁。

### 3. 如果某个 Agent 开发中途崩溃或陷入死循环怎么办？
大型模型有时会陷入幻觉或超时。此时：
1. 它占用的 Session 处于僵死状态。
2. 它占用的 Task 无法被其他人接手。
**解决办法**：
人工或者主管 Agent (Agent A) 可以定期运行 `carryctx doctor` 检查健康度。
若发现僵死任务，可强行剥夺权限：
```bash
carryctx session abandon <session_id>
carryctx task unclaim CTX-101 --force
```
随后将该任务重新放回池中，等待其他 Agent 接手。

### 4. 为什么强烈建议使用 `carryctx worktree create`？
大型项目中，如果所有 Agent 都在同一个根目录（同一个 branch）下工作，文件频繁被不同的 Agent 篡改，会导致底层 git 状态极度混乱。
`worktree` 命令通过 Git Worktree 机制，为每个独立任务分配一个与主目录平行的物理文件夹，并自动关联分支。这样不仅文件系统物理隔离，且各 Agent 之间的 Git Index 互不干扰，互不引发冲突。

### 5. 如果一个 Agent 认为任务做完了，但测试不通过怎么办？
当 `Agent D (QA)` 介入测试任务时，如果发现缺陷：
```bash
carryctx task block CTX-101 --reason "压力测试下 Redis 脚本偶发死锁"
```
此时 `CTX-101` 被打回阻塞状态。后端 Agent (B) 查询时，发现自己的模块状态异常，就会重新 Claim，运行 `carryctx resume` 看到打回理由，并接着最后一次 Checkpoint 继续修复 bug。

### 6. CarryCtx 数据库体积会无限膨胀吗？
不会。CarryCtx 设计为状态缓存，而非永久归档。进度条目 (`progress`) 和快照 (`checkpoints`) 主要保留给当前版本。随着任务 `complete`，它的中间繁杂细节后续可以通过归档命令定期清理，保证 `.git/carryctx/state.sqlite` 始终轻量高效。
