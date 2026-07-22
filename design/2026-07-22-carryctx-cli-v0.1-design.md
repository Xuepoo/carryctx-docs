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

The CLI is one product but is delivered as independently testable slices:

1. **Foundation:** package, error/output contracts, project discovery, configuration, SQLite connection, migrations, and `init`.
2. **Work model:** Agent, Task, dependency, Progress, Event, and atomic claim.
3. **Session model:** Session lifecycle, current-entity resolution, stale detection, and worktree binding.
4. **Continuity:** Git snapshot, Checkpoint, Resume, Context, and Status.
5. **Collaboration:** Worktree create, Decision, Scope conflicts, and Handoff.
6. **Operations:** Doctor, backup/migrate, skill installation, packaging, and acceptance tests.

Each slice produces a usable command set and receives its own implementation plan. Later slices build only on public interfaces established by earlier slices.

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

Each state-changing command maps to one use case with an explicit input and result type. The use case:

1. resolves the project and current entities;
2. validates domain preconditions;
3. opens a repository transaction;
4. writes the state change and its Event;
5. commits once; and
6. returns a presentation-neutral result.

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

Interfaces expose intent-oriented operations rather than generic SQL access. Examples include `claimTask`, `createCheckpoint`, `findCurrentSession`, `listReadyTasks`, `appendEvent`, `captureGitState`, and `loadEffectiveConfig`.

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

State transitions are centralized. Starting or claiming requires completed strong dependencies. Blocking and cancelling require a reason. Completing warns about open Todo or Blocker items and becomes an error only when strict completion is enabled. Reopening a completed or cancelled task returns it to `ready` unless unresolved dependencies require `planned`.

Claim uses one SQLite write transaction. It verifies the agent is active, dependencies are satisfied, project single-task policy is respected, and ownership remains unassigned at write time. Concurrent losers receive `TASK_ALREADY_CLAIMED` with exit code 3.

### 7.3 Dependency

Task dependencies are directed edges. Self-dependencies and any insertion that would make the target reachable from the prerequisite are rejected. A ready query includes only tasks in a startable state with all strong dependencies completed and no conflicting owner.

### 7.4 Session state

States are `active`, `paused`, `ended`, `stale`, and `abandoned`. Starting a session captures Agent, Task, worktree, branch, HEAD, current directory, provider, and timestamps. Ending a session never completes its Task. A session becomes stale when its last activity exceeds the effective `stale_after` duration; detection is deterministic at query or doctor time and can be persisted as an audited transition.

### 7.5 Progress and history

Progress types are Todo, Completed, Blocker, Risk, and Note. Items retain their creation Session, stable order, completion timestamp, and edit history through Events. Checkpoints, Events, Decisions, and Handoffs are immutable; corrections or superseding records preserve history.

## 8. SQLite design

Each project database contains these tables:

| Table | Responsibility |
| --- | --- |
| `schema_migrations` | Applied migration version, checksum, and timestamp |
| `projects` | Project identity, repository/common paths, branches, and schema metadata |
| `sequences` | Atomic display-ID counters scoped by project and entity kind |
| `agents` | Stable agent identity, provider, role, metadata, and active state |
| `tasks` | Task fields, status, priority, owner, parent, and lifecycle timestamps |
| `task_dependencies` | Directed strong or informational dependency edges |
| `task_scopes` | Repository-relative glob patterns for relevance and conflict warnings |
| `progress_items` | Ordered structured progress attached to a Task and source Session |
| `worktrees` | Git worktree path, branch, HEAD, bound Task, and observation time |
| `sessions` | Agent work session, Task/worktree binding, state, activity, and summary |
| `checkpoints` | Immutable semantic summary and captured Git snapshot |
| `decisions` | Immutable technical decision content and supersession relation |
| `decision_tasks` | Many-to-many Decision to Task relation |
| `decision_paths` | Decision to repository-relative path relation |
| `handoffs` | Immutable transfer summary, source/target, status, and Git snapshot |
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

Before upgrading an existing database, CarryCtx creates a consistent backup under `<git-common-dir>/carryctx/backups/`, applies migrations transactionally where SQLite permits, verifies the resulting version and foreign keys, and only then writes the migration Event. A checksum mismatch or newer unknown schema returns `MIGRATION_REQUIRED` or `UNSUPPORTED_OPERATION` without modifying state.

Restore never overwrites the active database in place without a pre-restore backup. Destructive doctor repairs require `--yes` in non-interactive mode.

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

Context uses the relevance order in `requirements.md` and enforces event/time limits before rendering. Compact mode excludes unrelated historical data. Text, Markdown, and JSON renderers consume the same presentation-neutral context model.

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

Errors use the documented error envelope and suggestions. Success data goes to stdout. Warnings, errors, and verbose diagnostics go to stderr. JSON mode emits no ANSI escapes, spinner frames, prompts, or non-JSON prose on the relevant stream.

Exit codes 0 through 12 match `cli-specification.md` and are centralized in one mapping. Domain errors carry stable codes; adapters translate Git, SQLite, configuration, validation, and filesystem failures at their boundaries.

Human output remains concise and action-oriented. Snapshot normalization replaces absolute fixture paths, timestamps, ULIDs, and platform separators without weakening assertions about public fields.

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

## 18. Packaging and operations

The npm package is named `carryctx`, starts at version `0.1.0`, and requires Bun 1.3.14 or newer. The executable uses `#!/usr/bin/env bun`. Exact dependencies and `bun.lock` are committed.

The package contains only `dist/`, migrations, the bundled Skill, templates, README, LICENSE, and required package metadata. A smoke test installs the packed tarball in a temporary directory, runs `carryctx --version`, initializes a temporary Git repository, and executes `carryctx status --json`.

CI runs formatting, typecheck, lint, documentation lint, unused-code checks, unit tests, integration tests, Actionlint, and package smoke tests. Release checks additionally require a clean worktree, consistent versions, and a changelog entry.

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
