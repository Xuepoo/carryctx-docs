# CarryCtx CLI v0.1 Design

**Status:** Approved direction

**Date:** 2026-07-22

**Goal:** Deliver a Bun-first, local-first CLI that preserves and restores coding-agent project state across sessions, agents, and Git worktrees.

## 1. Scope

The release implements the complete P0 workflow and the P1 capabilities needed to satisfy AC-001 through AC-012:

- Project initialization, configuration, registry, migration, and backup
- Agent registration and resolution
- Session lifecycle, reuse, and stale detection
- Task creation, querying, claiming, release, dependencies, scopes, and state transitions
- Structured progress items
- Worktree discovery, binding, creation, and consistency checks
- Immutable Git-aware checkpoints
- Deterministic resume, context, and status views
- Append-only events
- Decisions and handoffs
- Doctor diagnostics and safe repair boundaries
- Text, JSON, and Markdown output
- A generic Agent Skill bundled with the CLI package

P2 capabilities are not part of v0.1: MCP, remote synchronization, hosted services, a web dashboard, code knowledge graphs, GitHub synchronization, and multi-repository projects.

## 2. Specification resolutions

The following decisions resolve contradictions in the draft documents and are normative for v0.1:

1. Project configuration uses `.carryctx/config.toml`, not JSON.
2. Machine-local overrides use `.carryctx/config.local.toml` and are ignored by Git.
3. Authoritative project state uses `<git-common-dir>/carryctx/state.sqlite` so linked worktrees share one database.
4. Handoff and stale-session handling are v0.1 requirements because AC-006 and AC-012 require them.
5. Database migration and backup are v0.1 requirements because schema safety is part of initialization, doctor, and packaging.
6. Decision, task scope, conflict warnings, worktree creation, and Markdown context are included to complete the documented v0.1 CLI surface.
7. The generic Skill is shipped from `carryctx-cli/skills/carryctx/` in v0.1. The independent `carryctx-skills` repository remains inactive until a synchronization or publishing design is approved.
8. Bun compatibility is required; Node.js runtime compatibility is not promised for v0.1.
9. Linux is the primary supported platform and macOS is secondary. Windows-aware path abstractions are required, but Windows release support is deferred.
10. Provider names remain open strings. Domain statuses and error codes remain closed, validated sets.

## 3. Delivery decomposition

The CLI is one product but is delivered as small vertical units with executable exit tests:

| Unit | Owns | Depends on | Exit evidence |
| --- | --- | --- | --- |
| A. Toolchain | Package, strict TypeScript, quality tools, help/version, output envelope | None | `carryctx --version`, format, typecheck, lint, and unit-test commands pass |
| B. Project discovery | Git root/common-dir adapter, XDG paths, project resolution | A | Temporary normal and linked worktrees resolve the same common directory |
| C. Configuration | TOML parsing, strict schema, merge/source tracking, config queries and mutations | A, B | Precedence, unknown-key, source, and atomic-write tests pass |
| D. State store | SQLite connection, project and registry schemas, migrations, backup primitives, Event store | A, B | Empty/open/reopen/migrate/rollback/foreign-key tests pass |
| E. Initialization | `init`, project registration, `.carryctx` files, idempotency | B, C, D | AC-001 and repeated-init tests pass |
| F. Agent and Task | Agent commands, Task state machine, dependencies, atomic claim | C, D, E | AC-002, AC-008, cycle, transition, and claim-race tests pass |
| G. Progress | Progress lifecycle and Task projection | F | AC-003 and progress history tests pass |
| H. Worktree and Session | Worktree bind/create/query and Session lifecycle/stale recovery | B, D, F | AC-009 and AC-012 pass |
| I. Checkpoint | Git snapshot and checkpoint create/list/show/correct | G, H | AC-004 and immutable-correction tests pass |
| J. Continuity views | Resume, Context, Status and all renderers | F, G, H, I | AC-005, AC-007, and AC-010 pass |
| K. Collaboration | Scope/conflicts, Decision, Handoff and related projections | F, H, I | AC-006 and overlap/supersession tests pass |
| L. Operations | Doctor, project backup/migrate/restore, bundled Skill operations | C through K | AC-011, doctor recovery, package smoke, and full CI pass |

Each unit receives a separate task group in the implementation plan. A later unit consumes only exported domain types, repository contracts, and application results from earlier units; it does not reach into their adapter internals.

## 4. Architecture

```text
CLI entry and command definitions
            ↓
Application use cases and transaction boundaries
            ↓
Pure domain entities, state machines, and value rules
            ↓
Repository and service interfaces
            ↓
SQLite, Git, configuration, filesystem, XDG, clock, ID, and terminal adapters
```

### 4.1 CLI layer

The CLI layer uses Citty to parse commands and flags. It resolves the requested output mode, invokes one application use case, and renders a success or error result. It contains no SQL, Git operations, or domain state transitions.

Global flags are parsed once and passed as an immutable invocation context. `--json` implies JSON format, disables prompts and decorations, and preserves stdout for the success envelope. `--non-interactive`, CI, `CARRYCTX_NON_INTERACTIVE`, or non-TTY stdin prohibit prompts.

### 4.2 Application layer

Each state-changing command maps to one use case with an explicit input and result type. SQLite-only changes resolve inputs, validate preconditions, begin an immediate transaction when contention matters, write the state and Event, and commit once.

Commands that also mutate Git, TOML, registry, backup, or filesystem state use a recoverable operation protocol because those resources cannot share a SQLite transaction:

1. preflight all domain and external constraints without mutation;
2. write an `operations` row with a stable operation ID, type, intended effect, and `prepared` state;
3. apply the external effect idempotently using an atomic temporary-file rename or deterministic Git target;
4. in one SQLite transaction, persist the domain projection, append its Event, and mark the operation `completed`;
5. update the non-authoritative global registry last;
6. on failure, mark the operation `failed` when possible and return a recovery suggestion; and
7. on the next matching command or `doctor`, inspect prepared/failed operations and either finalize an already-observed effect or offer a safe retry.

`init` bootstraps this protocol by creating configuration and database files at temporary sibling paths, validating both, renaming the database first and configuration second, then registering the project last. Repeated `init` reconciles any partial bootstrap instead of replacing valid state. Git worktree creation never automatically deletes a created worktree as compensation; a failed finalization is recorded and recovered by `doctor`.

Dry-run uses the same validation and planning path but does not open a write transaction or modify Git, configuration, or files.

### 4.3 Domain layer

The domain layer is pure TypeScript. It owns:

- task and session state machines;
- dependency-cycle detection;
- claim and completion invariants;
- progress semantics;
- current-entity resolution rules;
- context relevance ordering;
- scope-overlap classification;
- stable domain errors.

It imports no Bun APIs and performs no I/O, making every rule independently unit-testable.

### 4.4 Repository and service interfaces

Interfaces expose intent-oriented operations rather than generic SQL access. The initial contracts are:

```typescript
interface UnitOfWork {
  run<T>(mode: "deferred" | "immediate", work: (repos: Repositories) => T): T;
}

interface ProjectRepository { get(): Project; updateSchemaVersion(version: number): void; }
interface AgentRepository { create(input: NewAgent): Agent; find(ref: AgentRef): Agent | undefined; list(filter: AgentFilter): Agent[]; }
interface TaskRepository { create(input: NewTask): Task; transition(input: TaskTransition): Task; claim(input: ClaimTask): ClaimResult; list(filter: TaskFilter): Task[]; }
interface DependencyRepository { add(edge: Dependency): void; remove(edge: Dependency): void; wouldCreateCycle(edge: Dependency): boolean; }
interface ProgressRepository { create(input: NewProgressItem): ProgressItem; transition(input: ProgressTransition): ProgressItem; list(taskId: string): ProgressItem[]; }
interface SessionRepository { create(input: NewSession): Session; transition(input: SessionTransition): Session; findCurrent(input: SessionResolution): Session[]; touch(id: string, at: string): void; }
interface WorktreeRepository { bind(input: WorktreeBinding): Worktree; list(): Worktree[]; }
interface CheckpointRepository { create(input: NewCheckpoint): Checkpoint; addCorrection(input: CheckpointCorrection): void; list(taskId: string): Checkpoint[]; }
interface CollaborationRepository { addScope(input: TaskScope): void; createDecision(input: NewDecision): Decision; createHandoff(input: NewHandoff): Handoff; appendHandoffTransition(input: HandoffTransition): void; }
interface EventRepository { append(event: NewEvent): Event; list(filter: EventFilter): Event[]; }
interface OperationRepository { prepare(input: NewOperation): Operation; complete(id: string): void; fail(id: string, code: string): void; listRecoverable(): Operation[]; }
interface GitService { discover(path: string): GitProject; capture(path: string): GitSnapshot; listWorktrees(project: GitProject): GitWorktree[]; createWorktree(input: CreateGitWorktree): GitWorktree; }
interface ConfigService { load(input: ConfigLoadInput): EffectiveConfig; planMutation(input: ConfigMutation): FileMutation; applyAtomic(mutation: FileMutation): void; }
interface BackupService { create(input: BackupInput): Backup; verify(path: string): BackupVerification; restore(input: RestoreInput): void; }
```

Application use cases own composition and transaction boundaries. Repository implementations own SQL mapping. Services own external effects. Commands own parsing and rendering only.

Command handlers never receive a database connection. SQLite-specific row types do not cross the adapter boundary.

## 5. Source structure

```text
carryctx-cli/
├── src/
│   ├── cli.ts
│   ├── commands/
│   ├── application/
│   ├── domain/
│   ├── repositories/
│   ├── adapters/
│   │   ├── config/
│   │   ├── filesystem/
│   │   ├── git/
│   │   ├── sqlite/
│   │   ├── terminal/
│   │   └── xdg/
│   ├── schemas/
│   ├── output/
│   ├── errors/
│   └── utils/
├── migrations/
├── skills/carryctx/
├── templates/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── fixtures/
│   └── helpers/
├── docs/
├── scripts/
└── .github/workflows/
```

Files are grouped by responsibility and kept small enough that a domain rule, use case, or adapter can be understood without reading unrelated commands.

## 6. Project and configuration resolution

Project discovery follows this order:

1. `--project`
2. `CARRYCTX_PROJECT`
3. `git rev-parse --show-toplevel` from the current directory
4. the global registry mapping for the current directory
5. `PROJECT_NOT_FOUND`

After locating the repository, Git commands resolve the repository root, absolute Git common directory, current worktree root, branch, and HEAD. A repository without `.carryctx/config.toml` is discoverable but not initialized.

Configuration layers merge from lowest to highest priority:

1. built-in defaults;
2. global configuration;
3. global profile;
4. project configuration;
5. project-local configuration;
6. explicit `--config` file;
7. environment variables;
8. CLI flags.

Tables merge recursively, arrays replace by default, and scalar values override. Every layer is parsed as TOML and validated with a strict Zod schema before use. Unknown keys are errors unless compatibility mode explicitly downgrades them to warnings. Relative paths retain source metadata so the configured resolution base is deterministic.

`CARRYCTX_CONFIG` supplies the explicit extra configuration file when `--config` is absent. `--config-compat <error|warn>` controls unknown-key handling and defaults to `error`. Nested environment values use `CARRYCTX_<SECTION>__<KEY>` and are schema-coerced rather than accepted as arbitrary strings.

`config set` and `config unset` require exactly one of `--global`, `--project`, or `--local`; there is no implicit write scope. Mutations preserve unrelated TOML fields and comments where the TOML library supports it, validate the complete result before replacement, and use the recoverable atomic-file protocol.

The global registry at `${XDG_STATE_HOME:-$HOME/.local/state}/carryctx/registry.sqlite` is a non-authoritative discovery index with its own migrations. It contains `projects(id, repository_root, git_common_dir, config_path, last_seen_at)` and `path_mappings(path_prefix, project_id)`. Paths and project IDs are unique, longest matching prefixes win, and stale entries are ignored when the repository or configuration no longer exists. Registry writes use SQLite transactions and the same busy timeout; failure produces a warning because the project database remains authoritative.

## 7. Domain model and invariants

### 7.1 IDs

- Task IDs use the configured uppercase prefix and an atomic per-project sequence: `CTX-0001`.
- Progress IDs use `ITEM-` plus an atomic sequence.
- Decision and handoff display IDs use `DEC-` and `HANDOFF-` sequences.
- Internal record IDs and immutable history IDs use ULIDs.
- Agent names are stable, user-selected identifiers unique within a project.

Sequences are allocated within the same transaction as record creation, so failed writes do not expose partially created entities.

### 7.2 Task state

States are `planned`, `ready`, `in_progress`, `blocked`, `review`, `completed`, and `cancelled`.

The closed transition matrix is:

| Command | Allowed source | Target | Ownership rule |
| --- | --- | --- | --- |
| `task claim` | `ready` | `in_progress` | Must be unowned; assigns current/explicit active Agent |
| `task start` | `ready` | `in_progress` | Requires existing current/explicit owner; otherwise use claim |
| `task release` | `in_progress`, `blocked` | `ready` if dependencies complete, otherwise `planned` | Current owner or explicit override; clears owner and requires no active Session |
| `task block` | `ready`, `in_progress`, `review` | `blocked` | Preserves owner and requires a non-empty reason |
| `task unblock` | `blocked` | `in_progress` when owned, otherwise `ready` or `planned` by dependency state | Preserves owner |
| `task review` | `in_progress` | `review` | Preserves owner |
| `task complete` | `review`, `in_progress` | `completed` | Preserves final owner; open work warns or fails under strict completion |
| `task cancel` | Any nonterminal state | `cancelled` | Clears owner and requires a non-empty reason |
| `task reopen` | `completed`, `cancelled` | `ready` if dependencies complete, otherwise `planned` | Clears owner |

Creation defaults to `planned`; `--status ready` is allowed only when strong dependencies are already complete. Adding an incomplete strong dependency to a ready unowned Task moves it to `planned`; removing/completing the final blocker makes it eligible for explicit `task unblock` or ready projection but does not silently start it.

Claim uses `BEGIN IMMEDIATE` with SQLite `busy_timeout = 5000`. Once the write lock is held, the use case re-reads the Task, Agent policy, dependencies, current Agent workload, and current worktree binding. A current worktree bound to another nonterminal Task returns `WORKTREE_TASK_CONFLICT`. The owner/status change uses a conditional update requiring `owner_agent_id IS NULL AND status = 'ready'`; zero changed rows are re-read and mapped to `TASK_ALREADY_CLAIMED` or `INVALID_TASK_TRANSITION`. Lock acquisition is retried with bounded jitter until the busy timeout; only true lock exhaustion maps to database exit code 5.

### 7.3 Dependency

Task dependencies are directed edges with kind `strong` or `informational`; `task depend` defaults to `strong` and accepts `--kind`. Self-dependencies and any insertion of either kind that would make the target reachable from the prerequisite are rejected. Only strong dependencies gate claim, start, and ready queries. A ready query includes only `ready`, unowned Tasks whose strong dependencies are completed.

### 7.4 Session state

States are `active`, `paused`, `ended`, `stale`, and `abandoned`. Starting captures Agent, Task, worktree, branch, HEAD, current directory, provider, and timestamps. Ending never completes its Task.

`session start` with no matching active Session creates one. If a matching active Session exists, interactive mode offers reuse, end, or new; non-interactive mode returns `SESSION_ALREADY_ACTIVE`. `--reuse` touches and returns the unique matching Session. `--new` atomically pauses matching active Sessions before creating a new one when single-active policy is enabled. Multiple ambiguous matches require an explicit Session ID.

`last_activity_at` changes only after a successful state-changing use case linked to the Session, `session resume`, checkpoint creation, or explicit reuse; read-only queries do not keep Sessions alive. A query or doctor check whose clock exceeds `stale_after` persists `active → stale` with `session.stale` once. `session resume <id>` permits `paused|stale → active` after enforcing active-session policy. `session end <id>` permits `active|paused|stale → ended`; `session abandon <id>` permits `active|paused|stale → abandoned`. Both recovery paths satisfy AC-012 and create an Event.

### 7.5 Progress and history

Progress `type` is `todo`, `completed`, `blocker`, `risk`, or `note`; lifecycle `status` is independently `open`, `completed`, or `removed`. `progress complete` and `reopen` change lifecycle status, while `progress edit`, `reorder`, and `remove` append Events containing before/after values. Removed items are hidden by default but retained for audit.

Checkpoint base rows and Git snapshots are immutable. `checkpoint correct` appends a `checkpoint_corrections` row that may replace semantic summary fields for Resume while never replacing the captured Git snapshot. Resume applies the latest correction and labels it as corrected.

Decision base rows are immutable. Supersession is stored in `decision_supersessions` and the effective status is derived. Handoff base content is immutable; accept, reject, and close append `handoff_transitions`, and current status is derived from the latest transition. Events record every correction, supersession, and transition.

## 8. SQLite design

Each project database contains these tables:

| Table | Responsibility |
| --- | --- |
| `schema_migrations` | Applied migration version, checksum, and timestamp |
| `projects` | Project identity, repository/common paths, branches, and schema metadata |
| `sequences` | Atomic display-ID counters scoped by project and entity kind |
| `operations` | Recoverable multi-resource operation intent, state, and failure code |
| `agents` | Stable agent identity, provider, role, metadata, and active state |
| `tasks` | Task fields, status, priority, owner, parent, and lifecycle timestamps |
| `task_dependencies` | Directed strong or informational dependency edges |
| `task_scopes` | Repository-relative glob patterns for relevance and conflict warnings |
| `progress_items` | Ordered structured progress attached to a Task and source Session |
| `worktrees` | Git worktree path, branch, HEAD, bound Task, and observation time |
| `sessions` | Agent work session, Task/worktree binding, state, activity, and summary |
| `checkpoints` | Immutable semantic summary and captured Git snapshot |
| `checkpoint_corrections` | Immutable semantic corrections applied by Resume |
| `decisions` | Immutable technical decision content and supersession relation |
| `decision_tasks` | Many-to-many Decision to Task relation |
| `decision_paths` | Decision to repository-relative path relation |
| `decision_modules` | Decision to logical module-name relation |
| `decision_supersessions` | Append-only old-to-new Decision relation |
| `handoffs` | Immutable transfer summary, source/target, status, and Git snapshot |
| `handoff_transitions` | Append-only accept/reject/close status history |
| `events` | Append-only audit record with actor relations and JSON payload |

Foreign keys enforce ownership and lifecycle relations. Check constraints enforce closed status sets and non-empty required content. Unique indexes protect Agent names, display IDs, dependency edges, task scopes, and active worktree bindings. Query indexes cover task status/owner, session state/activity, progress task/order, checkpoint task/time, event relations/time, and handoff status/target.

Structured metadata and immutable Git file lists use canonical JSON text after Zod validation. Fields that participate in filtering, joins, state transitions, or ordering remain typed columns rather than JSON.

Opening a connection executes:

```sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
```

Every external value uses parameter binding. Dynamic identifiers are selected only from code-owned allowlists. User-facing errors map SQLite failures to domain codes without exposing SQL or internal paths unnecessarily.

## 9. Migrations and backup

Migration files are immutable, ordered SQL assets bundled in the npm package. Each file has a version and checksum. Initialization applies all migrations in order and records them in `schema_migrations`.

Before upgrading an existing database, CarryCtx acquires an owner-recorded migration lock, checkpoints the WAL, and creates a consistent backup under `<git-common-dir>/carryctx/backups/` using SQLite's online backup API or `VACUUM INTO` through the same live connection. Raw copying of the main database while WAL is active is forbidden. The backup is opened read-only and must pass `PRAGMA integrity_check` and contain the expected schema version before migration begins.

Every v0.1 migration must be transactional. Migration SQL that SQLite cannot run inside a transaction is rejected during build-time migration tests and must be redesigned as a create/copy/swap sequence within one transaction. The runner uses `BEGIN EXCLUSIVE`, applies pending files, validates `foreign_key_check`, records versions/checksums and `project.migrated`, then commits. An interruption before commit rolls back to the pre-migration schema. A checksum mismatch or newer unknown schema returns `MIGRATION_REQUIRED` or `UNSUPPORTED_OPERATION` without modifying state.

Restore acquires the same exclusive lock, closes other application connections, creates and verifies a pre-restore backup, validates the requested backup with `integrity_check`, restores into a temporary sibling database, reopens and validates it, then atomically swaps files. An operation record allows `doctor` to complete or roll back an interrupted swap. Restore never overwrites the only valid database copy. Destructive doctor repairs require `--yes` in non-interactive mode.

## 10. Git and worktree adapter

The adapter invokes the installed Git CLI with argument arrays, never shell-interpolated commands. It provides:

- repository and Git-common discovery;
- worktree enumeration using porcelain output;
- current branch and HEAD;
- staged, modified, deleted, renamed, and untracked files;
- diff statistics without storing full diffs by default;
- worktree creation and safe preflight checks.

Checkpoint snapshots normalize repository-relative paths. `--include-diff` is explicit and its output is never persisted unless a documented field requires it. Worktree removal is not automated in v0.1 when the directory is dirty, contains untracked files, has active sessions, belongs to an incomplete task, or includes unmerged commits.

## 11. Current entity resolution

Agent, Task, and Session resolution follows the CLI specification. Resolution returns either one value, an explicitly representable absent result for read-only commands, or a typed ambiguity/error. Non-interactive invocations never guess among multiple candidates.

Terminal environment binding for the current Session uses an explicit `CARRYCTX_SESSION` value in v0.1. The CLI may print an export command after starting a Session; it does not modify the parent shell environment.

## 12. Resume, context, and status

Resume is read-only unless `--start-session` is supplied. It combines the resolved entities, latest checkpoint, structured progress, dependencies, relevant decisions/events, and a fresh Git snapshot. Deterministic rules generate warnings and next actions. A changed HEAD, dirty state, or file set after the latest checkpoint marks the checkpoint comparison as stale.

Context uses this single canonical relevance order: current Task; effective latest Checkpoint; current Git snapshot; open Blockers; open Todo items; incomplete strong dependencies and their latest state change; Tasks directly blocked by the current Task; active Tasks with overlapping scopes; Decisions linked to the current Task, path, or module; the current Agent's other active Tasks; recent Task-related Events; and finally recent project Decisions and Events. Items sort first by this group order, then `updated_at` descending, then stable ID ascending. `--since` and `--max-events` are applied before rendering; compact mode omits the final two project-wide groups. Text, Markdown, and JSON renderers consume the same presentation-neutral context model.

Status summarizes task counts, active/stale sessions, the current worktree, recent activity, blockers, and scope conflicts. `--mine` adds an Agent filter; `--all` expands active entity details without emitting the entire historical database.

## 13. Scope conflicts

Task scopes are repository-relative Picomatch globs. Conflict detection is conservative: exact paths, matching literal prefixes, or glob patterns that can match a shared candidate region produce a warning. Because proving arbitrary glob intersection is expensive, uncertain active-scope intersections may be labeled `possible` rather than silently ignored. Scope is advisory in v0.1 and never prevents file writes.

## 14. Output and errors

All successful JSON output has this envelope:

```json
{
  "schemaVersion": 1,
  "command": "task.list",
  "success": true,
  "data": {},
  "warnings": [],
  "meta": {
    "projectId": "carryctx",
    "timestamp": "2026-07-22T18:30:00Z"
  }
}
```

Errors use the documented error envelope and suggestions. In text mode, success data goes to stdout and warnings/errors/verbose diagnostics go to stderr. In JSON mode, a successful command writes exactly one success envelope to stdout, includes warnings in `warnings` and optional diagnostics in `meta.diagnostics`, and writes nothing to stderr. A failed JSON command writes nothing to stdout and exactly one error envelope to stderr. JSON mode emits no ANSI escapes, spinner frames, prompts, or non-JSON prose.

Exit codes 0 through 12 match `cli-specification.md` and are centralized in one mapping. Domain errors carry stable codes; adapters translate Git, SQLite, configuration, validation, and filesystem failures at their boundaries.

Every command result has a Zod schema exported from `src/schemas/output/`; the renderer cannot serialize unvalidated data. Query-list results use plural top-level keys, entity results use the singular entity name, mutations include the affected entity plus `operation`, and dry-run mutations return only `operation` with `applied: false`. Common entity projections have compact and full schemas so list output cannot accidentally leak internal columns.

Human output remains concise and action-oriented. Snapshot normalization replaces absolute fixture paths, timestamps, ULIDs, and platform separators without weakening assertions about public fields. Markdown is supported only by `context` in v0.1; another command receiving `--format markdown` returns `UNSUPPORTED_OPERATION` with exit code 10.

## 15. Command behavior boundaries

- `init` is idempotent and never destroys existing state without `--force` plus confirmation.
- Query commands do not require an active Session unless the query itself is Session-specific.
- State-changing commands that semantically belong to a Session fail clearly when no Session can be resolved.
- `resume` never changes ownership, Task state, Git state, or checkpoint history.
- `session end` never completes a Task.
- `handoff accept` transfers ownership only with explicit `--claim-task`.
- Checkpoints and Events have no ordinary update or delete command.
- Decisions are superseded, not deleted.
- `doctor --fix` applies only explicitly classified safe repairs without confirmation.
- Network access is absent from all core commands.

### 15.1 Closed v0.1 command surface

The following commands are required in v0.1; no unlisted subcommand is implied:

| Command | Included subcommands or forms |
| --- | --- |
| Root | `--help`, `--version` |
| `init` | default create/reconcile form |
| `status` | default, `--mine`, `--all` |
| `resume` | default and `--start-session` |
| `context` | compact/full text, JSON, and Markdown |
| `checkpoint` | create, `list`, `show`, `correct` |
| `project` | `show`, `list`, `register`, `unregister`, `migrate`, `backup`, `restore` |
| `config` | `list`, `get`, `set`, `unset`, `validate`, `sources`, `path` |
| `agent` | `register`, `list`, `show`, `current`, `rename`, `deactivate` |
| `session` | `start`, `list`, `show`, `current`, `pause`, `resume`, `end`, `abandon` |
| `task` | `create`, `list`, `show`, `edit`, `claim`, `release`, `start`, `block`, `unblock`, `review`, `complete`, `cancel`, `reopen`, `depend`, `undepend`, `scope add/remove/list/conflicts` |
| `progress` | `todo`, `done`, `block`, `risk`, `note`, `list`, `show`, `edit`, `complete`, `reopen`, `remove`, `reorder` |
| `worktree` | `create`, `bind`, `list`, `show`, `status`, `unbind` |
| `decision` | `add`, `list`, `show`, `search`, `supersede` |
| `handoff` | `create`, `list`, `show`, `accept`, `reject`, `close` |
| `event` | `list`, `show`, `tail` |
| `doctor` | inspect and `--fix` |
| `skill` | `install`, `list`, `path`, `doctor` |

Deferred beyond v0.1 are `project export/import`, `worktree remove/prune`, `skill update/export`, shell completion, and every P2 feature. They may appear in future-facing documentation only when labeled deferred.

### 15.2 Command traceability and JSON data contracts

All listed fields are required unless the schema explicitly marks the underlying concept nullable. Filters never alter the envelope shape.

| Command family | Application result in `data` | Write Event types | Principal errors | Acceptance |
| --- | --- | --- | --- | --- |
| Root | `version: {name, version, runtime}` or help text outside JSON | None | `INVALID_ARGUMENTS` | Package smoke |
| `init` | `project`, `paths`, `created[]`, `reconciled[]`, `operation` | `project.initialized`, `project.reconciled` | `PROJECT_NOT_FOUND`, `PROJECT_ALREADY_INITIALIZED`, `CONFIG_INVALID`, `GIT_ERROR`, `DATABASE_ERROR` | AC-001 |
| `status` | `project`, `current`, `git`, `counts`, `sessions[]`, `tasks[]`, `activity[]`, `conflicts[]` | `session.stale` only when detected | Project/config/database resolution errors | AC-007, AC-010 |
| `resume` | `project`, `agent`, `session`, `task`, `git`, `checkpoint`, `progress`, `dependencies`, `related`, `nextActions[]` | `session.started` only with `--start-session`; `session.stale` when detected | Agent/Task ambiguity, not-found, project errors | AC-005, AC-010 |
| `context` | `mode`, `task`, `checkpoint`, `sections[]`, `generatedAt` | None | Task ambiguity/not-found, invalid duration/limit | AC-005, AC-010 |
| `checkpoint` | Create/correct: `checkpoint`, `operation`; list: `checkpoints[]`; show: `checkpoint` | `checkpoint.created`, `checkpoint.corrected` | Session/Task not resolved, Git error, not found | AC-004 |
| `project` | Singular: `project`; list: `projects[]`; backup/restore/migrate: `project`, `backup` or `migration`, `operation` | `project.registered`, `project.unregistered`, `project.migrated`, `project.backed_up`, `project.restored` | Migration required/unsupported, integrity, permission, not found | AC-001, AC-009 |
| `config` | Get: `key`, `value`, `source`; list: `values`; sources: `sources[]`; mutation: `mutation`, `operation`; validate: `valid`, `issues[]`; path: `paths` | `config.changed` for project-visible changes | Invalid scope/key/value/TOML, permission | Configuration tests |
| `agent` | Singular/mutation: `agent`, `operation`; list: `agents[]`; current: `agent` | `agent.registered`, `agent.renamed`, `agent.deactivated` | Already exists, not found, ambiguity, active ownership conflict | AC-002, AC-006 |
| `session` | Singular/mutation: `session`, `operation`; list: `sessions[]`; current: `session` | `session.started`, `session.paused`, `session.resumed`, `session.ended`, `session.stale`, `session.abandoned` | Already active, ambiguity, invalid transition, checkpoint required | AC-005, AC-012 |
| `task` | Singular/mutation: `task`, `dependencies`, `scopes`, `operation`; list: `tasks[]`; scope conflicts: `task`, `conflicts[]` | `task.created`, `task.updated`, `task.claimed`, `task.released`, `task.transitioned`, `task.dependency_added/removed`, `task.scope_added/removed` | Not found, already claimed, dependency cycle/incomplete, invalid transition, worktree conflict | AC-002, AC-008 |
| `progress` | Singular/mutation: `item`, `operation`; list: `items[]` | `progress.created`, `progress.edited`, `progress.completed`, `progress.reopened`, `progress.removed`, `progress.reordered` | Task/Session not resolved, item not found, invalid transition/order | AC-003 |
| `worktree` | Singular/mutation/status: `worktree`, `git`, `operation`; list: `worktrees[]` | `worktree.created`, `worktree.bound`, `worktree.unbound` | Git error, path exists/not found, task conflict, dirty preflight | AC-009 |
| `decision` | Singular/mutation: `decision`, `operation`; list/search: `decisions[]` | `decision.created`, `decision.superseded` | Not found, invalid relation, already superseded | Collaboration tests |
| `handoff` | Singular/mutation: `handoff`, `status`, `operation`; list: `handoffs[]` | `handoff.created`, `handoff.accepted`, `handoff.rejected`, `handoff.closed` | Not found, invalid transition/target, task claim conflict | AC-006 |
| `event` | Show: `event`; list/tail: `events[]`, `cursor` | None | Invalid filter/cursor, not found | Audit tests |
| `doctor` | `checks[]`, `summary`, `operations[]`, optional `repairs[]` | `doctor.repaired` plus recovered domain Event | Confirmation required, unsafe/unsupported repair, project/database/Git errors | AC-012 |
| `skill` | Install: `skill`, `paths`, `operation`; list: `skills[]`; path: `paths`; doctor: `checks[]` | `skill.installed` for project installs | Resource/target not found, permission, invalid bundled manifest | Package smoke |

Error envelope `details` is validated per error code. Known state conflicts use exit code 3, Git failures 4, SQLite failures 5, configuration failures 6, missing resources 7, input/schema validation 8, permission/scope failures 9, deferred operations 10, migration requirements 11, and interrupts 12.

## 16. Security and privacy

- No telemetry or update request is performed by default.
- Secrets, tokens, complete environment dumps, full Git diffs, and source contents are not persisted by default.
- Configuration rejects secret-looking unsupported keys rather than encouraging credential storage.
- Database and backup files are created with owner-only permissions where supported.
- Paths are normalized, but human and JSON output exposes only paths required by the command contract.
- Output-to-file operations write through a temporary sibling and atomic rename where supported.

## 17. Testing strategy

### 17.1 Unit tests

Unit tests cover state transitions, current-entity resolution, dependency cycles, ready filtering, configuration merge and validation, duration parsing, ID formatting, context ranking, scope overlap, output envelopes, and error-to-exit mapping.

### 17.2 SQLite integration tests

Integration tests use isolated temporary databases and cover migration from an empty database, idempotent opening, PRAGMAs, foreign keys, rollback on failed multi-write use cases, atomic concurrent claim, parameter safety with adversarial text, indexes used by critical queries, backup, restore, and schema checksum errors.

### 17.3 Git integration tests

Temporary repositories and linked worktrees cover repository discovery from subdirectories, Git-common state sharing, detached HEAD, clean and dirty snapshots, staged/unstaged/untracked files, worktree binding, worktree creation, and changed-state checkpoint warnings.

### 17.4 CLI contract tests

Subprocess tests cover help/version, text/JSON/Markdown output, stdout/stderr separation, non-interactive failures, dry-run immutability, documented exit codes, and normalized snapshots.

### 17.5 Acceptance tests

An end-to-end suite maps one or more tests to every AC-001 through AC-012. The final report records the exact command and evidence for each criterion, plus `just ci` and package tarball smoke-test results.

### 17.6 Performance and platform tests

A seeded benchmark fixture contains 1,000 Tasks, 10,000 Events, 100 Sessions, and 20 worktrees. After one warm-up run, state-only `task list --json` and `status --json` each run 20 times and must have a local median below 100 ms; Git-aware `status --json` and `resume --json` each run 10 times and must have a median below 1 second. CI records measurements as evidence and uses a separate non-flaky regression ceiling of twice the target; release reporting states both target and observed values.

Linux runs the full matrix. macOS CI runs unit tests, project/config path tests, temporary Git repository tests, and package smoke. Path tests inject Linux XDG, macOS platform-directory, and Windows Known Folder adapters on every platform, including separator normalization, drive-root preservation, and case-handling contracts. Windows execution remains unsupported in v0.1, but adapter tests prevent hard-coded Unix home/cache paths.

## 18. Packaging and operations

The npm package is named `@xuepoo/carryctx`, starts at version `0.1.0`, and requires Bun 1.3.14 or newer. The installed executable remains `carryctx` and uses `#!/usr/bin/env bun`. Exact dependencies and `bun.lock` are committed.

The package contains only `dist/`, migrations, the bundled Skill, templates, README, LICENSE, and required package metadata. A smoke test installs the packed tarball in a temporary directory, runs `carryctx --version`, initializes a temporary Git repository, and executes `carryctx status --json`.

Local and GitHub CI definitions run formatting, typecheck, lint, documentation lint, unused-code checks, unit tests, integration tests, Actionlint, and package smoke tests. Dependabot configuration tracks npm and GitHub Actions dependencies. Remote repository rules and automatic npm publication are a later delivery task and do not block local CLI development or v0.1 behavior implementation. No workflow reads or prints registry credentials during development.

## 19. Documentation and repository ownership

`carryctx-docs` owns the cross-repository product contract, architecture, plans, roadmap, workflow, and verification evidence. `carryctx-cli` owns executable code, migrations, bundled runtime assets, and CLI-specific contributor instructions. Plugins, standalone skills, and website repositories remain minimal until their respective roadmap phases.

Any implementation-discovered contradiction is resolved by updating the affected source document in the same workstream. Passing tests never silently redefine the written public contract.

## 20. Completion criteria

CLI v0.1 is complete only when:

1. every included command has documented text and JSON behavior;
2. AC-001 through AC-012 have direct end-to-end evidence;
3. database migration, backup, transactions, and concurrent claim tests pass;
4. linked worktrees demonstrably share one project database;
5. `just ci` passes from a clean dependency install;
6. the packed npm tarball passes the smoke workflow;
7. package contents exclude source, tests, coverage, `.carryctx`, and state databases;
8. documentation contains no unresolved v0.1 contract contradictions; and
9. the final verification report records commands, versions, results, and known platform limits.
