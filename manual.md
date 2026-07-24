# CarryCtx Official User Manual (v0.3.0)

This is the comprehensive, official documentation for CarryCtx CLI. It serves as the ultimate product user manual and is the basis for the `carryctx-website` content.

---

## 1. What is CarryCtx?

CarryCtx is an offline-first, Git-native, context-preservation engine built for Coding Agents (like Claude Code, Cursor, Aider) and Human developers. It runs entirely on your local machine using a high-performance Rust core and an embedded SQLite database (`.git/carryctx/state.sqlite`), ensuring zero latency and 100% privacy.

With CarryCtx, you can seamlessly pause your coding session, jump to another branch, hand over a task to an AI agent, and resume exactly where you left off, with full memory of your previous thoughts, code dependencies, and checkpoints.

---

## 2. Core Concepts

* **Project**: Mapped 1:1 with your local Git repository.
* **Agent**: The entity performing the work. Can be a human developer (`alice`) or an AI model (`claude-code`).
* **Task**: A tracked unit of work (e.g., `CTX-0001`). Tasks support hierarchies, dependencies, and scope binding.
* **Session**: A continuous block of active development attached to a Task.
* **Progress (todo/done/block/note)**: Micro-journaling primitives during a Session.
* **Checkpoint**: A persistent snapshot of your context, unfinished tasks, and Git diff, acting as a "save state" for AI memory.
* **Git Worktrees**: Isolated workspace directories for parallel task execution without switching branches in your main directory.
* **AST Graph**: A semantic dependency graph of your codebase parsed locally to give AI agents spatial awareness of your code.
* **Presets**: Shareable combinations of Rules, Personas, and Workflows stored in `.carryctx/`.

---

## 3. Quick Start

### 3.1 Initialization & Agent Registration
Initialize CarryCtx in any Git repository:
```bash
carryctx init
```

Register yourself or your AI agent:
```bash
carryctx agent register --name antigravity --provider gemini --role presets/personas/architect.md
```
*Note: As of v0.3.0, if no agent is specified and there is only one active agent in the database, CarryCtx intelligently infers the agent context.*

### 3.2 Managing Tasks
```bash
# Create a task
carryctx task create --title "Implement OAuth2 login" --priority high

# Claim the task
carryctx task claim CTX-0001
```

### 3.3 Seamless Sessions & Smart Inference (v0.3.0 Feature)
Start working on your task. CarryCtx intelligently infers your task context if you are inside a Git Worktree or have a single active task:
```bash
carryctx session start

# Record your thoughts as you code
carryctx progress todo "Add JWT token validation"
carryctx progress done "Created OAuth2 endpoints"
carryctx progress block "Waiting for Google Client ID from DevOps"
```

### 3.4 Checkpoints & Context Export
When you finish your shift, save your context:
```bash
carryctx checkpoint --done "Finished OAuth2 endpoints" --remaining "JWT token validation"
carryctx session end
```

To fetch the full context (designed for feeding into LLM prompts):
```bash
carryctx context
```

---

## 4. Comprehensive Command Reference

CarryCtx supports `--json` for machine-readable output and `--quiet` / `--verbose` for logging control across all commands.

### 4.1 Global Options
- `--agent <AGENT_ID>`: Explicitly define the operating agent, overriding environmental variables and auto-resolution.
- `--json`: Format output as a JSON envelope.
- `--format <text|markdown|json>`: Define the output format.

### 4.2 Project & Lifecycle Commands
- `carryctx init`: Initializes the `.git/carryctx/state.sqlite` database.
- `carryctx status`: Displays a health dashboard (active sessions, current tasks, agents, and worktrees).
- `carryctx project prune [--older-than <days>]`: Prunes old database backups to free up disk space.
- `carryctx project backup / restore`: Manages manual SQLite backups.
- `carryctx doctor`: Runs integrity checks on the database, Git hooks, and worktree bindings.

### 4.3 Task Commands
- `carryctx task create --title <TITLE> [--depends-on <ID>]`: Creates a task.
- `carryctx task claim <TASK_ID>`: Assigns the task to the current agent and transitions it to `in_progress`.
- `carryctx task list [--status <STATUS>] [--mine]`: Lists tasks.
- `carryctx task start / pause / complete / cancel / review / block`: Transitions the task state.
- `carryctx task deps add / remove / tree`: Manages task dependencies.

### 4.4 Session & Progress Commands
- `carryctx session start [--task <TASK_ID>]`: Starts a session. Smart inference automatically attaches the task if omitted.
- `carryctx session pause / resume / end`: Manages session lifecycles.
- `carryctx progress todo <TEXT>`: Adds a pending item to the current session.
- `carryctx progress done <TEXT>`: Marks an item as completed.
- `carryctx progress block <TEXT>`: Logs an active blocker.
- `carryctx progress note <TEXT>`: Logs architectural thoughts or debugging notes.

### 4.5 Agent Commands
- `carryctx agent register --name <NAME>`: Registers a new agent.
- `carryctx agent current`: Displays the auto-resolved active agent.
- `carryctx agent list / show / deactivate`: Manages agent lifecycles.

### 4.6 Checkpoint & Context Commands
- `carryctx checkpoint --done <TEXT> --remaining <TEXT> [--blocker <TEXT>]`: Snapshots the task state, active progress, and current Git diff.
- `carryctx context [--task <TASK_ID>]`: Compiles a detailed markdown context (perfect for pasting into LLM chats) containing dependencies, session history, and recent progress.
- `carryctx resume`: Similar to `context`, but tailored specifically for resuming work after an interruption.

### 4.7 Git Worktree Commands
Isolated workspaces are first-class citizens in CarryCtx.
- `carryctx worktree create <BRANCH> [--task <TASK_ID>]`: Creates a Git worktree linked to a specific task. Running `carryctx` inside this worktree automatically infers the bounded task.
- `carryctx worktree list`: Lists active worktrees and their bounded tasks.
- `carryctx worktree remove <BRANCH>`: Safely cleans up the worktree.

### 4.8 AST Code Graph Commands
CarryCtx can locally parse your codebase into an Abstract Syntax Tree (AST) to understand imports, exports, and function calls.
- `carryctx graph scan`: Scans the current Git repository and updates the AST database.
- `carryctx graph query --pattern <GLOB>`: Queries the graph for specific symbols or files.
- `carryctx graph explain <FILE_OR_SYMBOL> [--depth <N>]`: Generates a semantic explanation of how a file or function fits into the codebase.
- `carryctx graph export [--format <mermaid|dot|json>]`: Exports the codebase dependency graph for visualization.

### 4.9 Presets & Rules (The \`.carryctx/\` Ecosystem)
Share best practices, personas, and workflows across your team.
- `carryctx preset list`: Lists available presets in `.carryctx/`.
- `carryctx preset show <NAME>`: Previews a preset.
- `carryctx preset apply <NAME>`: Activates a workflow or rule preset.

### 4.10 Analytics
- `carryctx stats [--format markdown|json|csv] [--output <FILE>]`: Computes rich project analytics, including Total Agent Hours, Task Completion Rates, and Codebase Graph Complexity.

### 4.11 MCP Integration
- `carryctx mcp`: Launches the Model Context Protocol (MCP) `stdio` server. This allows compatible clients (like Claude Desktop or Cursor) to seamlessly discover and execute CarryCtx tools natively via JSON-RPC.

---

## 5. Best Practices for Coding Agents

1. **Auto-Resolution**: Take advantage of v0.3.0's auto-resolution. If you create a worktree for a task, simply `cd` into it; you no longer need to pass `--task` or `--agent` to subsequent commands.
2. **Commit Hooks**: Run `carryctx hooks install` to automatically trigger checkpoints upon Git commits.
3. **Atomic Operations**: Rely on the JSON envelope (`--json`). CarryCtx guarantees SQLite ACID transactions. If a command exits with code `0`, it succeeded; otherwise, read the `error` object.
4. **Rich Personas**: When initializing an agent, assign a Persona preset (e.g., `presets/personas/architect.md`) so the AI naturally adopts the defined rigor.
