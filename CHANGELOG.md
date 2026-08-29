# CarryCtx Changelog

## [Unreleased]

Post-v0.7.0 fixes from the issue #106 campaign, verified against a release
build of the fix branch (`carryctx/ctx-0083` @ `91a5275`). All changes keep
the JSON envelope contract and single-agent workflows unchanged.

### Added

- `worktree remove <REF>`: deletes a bound Git worktree **and** removes its
  registration row. `<REF>` accepts the bound task's CTX display-id, the
  worktree ULID, or its path. Dirty worktrees are refused with
  `STATE_CONFLICT` (exit 3) unless `--force` is passed; when the directory
  is already gone only the registration is cleaned up
  (`data.git_removed=false`). The removal appends a `worktree.removed`
  audit event in the same transaction. `worktree unbind` help now reads
  "Detach a worktree from its task without deleting anything".
- MCP server implements `ping`, answering id-bearing requests with an empty
  object result (`{"id":<id>,"jsonrpc":"2.0","result":{}}`); notifications
  (requests without id) receive no response.

### Changed

- Terminal tasks remain immutable by default, while `task edit --force` allows
  corrections only for an active authenticated agent who is either the task
  owner or an agent recorded on the terminal transition. Authorized
  corrections are recorded as `task.corrected`; `--force` on non-terminal
  tasks remains rejected.
- **Behavior change**: `CARRYCTX_AGENT` no longer implicitly scopes
  `event list`. Without an explicit `--agent` flag the full project event
  stream is returned; identity resolution and event attribution via the
  environment variable are unaffected.
- `event list --cursor` tokens are opaque: the `(occurred_at, id)` keyset
  tuple is base64url-encoded and suffixed with an 8-hex checksum. Tampered
  or malformed tokens fail with `VALIDATION_FAILED` instead of silently
  returning wrong pages. Legacy plaintext `(occurred_at|id)` tokens remain
  readable only when they contain no `.` (they are rewritten to the opaque
  format on the next emitted `next_cursor`); genuine pre-opaque cursors
  carry fractional-second timestamps and therefore must be discarded after
  upgrading — restart pagination from page one.
- Search hits of kind `task` populate top-level `display_id` with the
  task's CTX id (previously always `null`; `task_display_id` unchanged).
  Progress/decision hits keep their own ids; checkpoint hits stay `null`.
- `doctor` exit codes reflect finding severity: exit `1` is reserved for
  `error`/`critical` findings or infrastructure failure;
  warning/info-only reports exit `0` (rendered report unchanged, `all_ok`
  still reports `false`). Scripts can distinguish "nothing broken" from
  "stale worktree registration" without parsing output.

## [0.8.0] - 2026-08-30

### Changed

- Release readiness aligns version metadata, distribution checks, package smoke
  validation, stale URLs, and the authoritative shared-state wording.

## [0.7.0] - 2026-08-24

### Upgrade notes

- **Schema migration to version 16 is required** and runs automatically on the
  first command against an existing project database (migrations
  `0014_cascade_task_refs`, `0015_agent_name_unique`,
  `0016_task_list_index`). Migrations 0014 rebuilds `handoffs` /
  `checkpoint_corrections`; if it fails, the migration aborts before any data
  changes — back up `<git-common-dir>/carryctx/state.sqlite` before upgrading.
- The `status --since <duration>` flag was removed; use `--worktrees` for the
  new Markdown worktree table.
- `config set`/`unset` scope flags renamed in behavior: `--local` is now
  explicitly rejected (`UNSUPPORTED_OPERATION`); write to the project-shared
  file with `--cfg-project`.

### Overview

Defect-fix campaign closing issues #94–#105: zero reachable panics, honest
output envelopes, typed config round-trips, CAS-guarded state machines,
data-safe maintenance paths, and multi-agent identity integrity. All changes
keep the single-agent workflows and the JSON envelope contract
(`schema_version`/`command`/`success`/`data`/`error`/`meta`) unchanged.

### Panics & CLI surface

- Removed all six reachable panic sites (`task` dry-run `unreachable!()`;
  unguarded multibyte byte-slices in handoff/progress/decision/worktree/event
  renderers). Read-only subcommands now render normally under `--dry-run`;
  truncation is char-boundary-safe via a shared `truncate_chars` helper.
- `--dry-run --json` returns a proper envelope with `operation.applied=false`
  for every mutating command (graph add-node/link/extract-deps/scan included);
  read-only commands are unaffected.
- Markdown list renderers route failures through the standard error path
  (stderr + real exit code) instead of printing `Error:` to stdout with exit 0.
- `project register/unregister` return `UNSUPPORTED_OPERATION` instead of
  fabricating success; `preset` output goes through the standard renderer;
  `checkpoint show` not-found renders `RESOURCE_NOT_FOUND`.
- Installed git hooks grep snake_case `"display_id"` so auto-checkpoint can
  actually match a task; hook install is atomic (tmp+rename) and chmod
  failures fail loudly. Envelope labels normalized to dotted form
  (`hooks.status`, `graph.edges`, ...).

### Config

- `config get/set/unset` rewritten on `toml_edit`: dotted keys land in the
  correct table, values keep real TOML types, writes are round-trip validated.
- Unknown get key → `value: null`, exit 0; missing write scope → clear error
  with suggestions; `--local` rejected as `UNSUPPORTED_OPERATION`.

### Concurrency & lifecycle

- Task claim uses a compare-and-set UPDATE (`status='ready' AND owner IS
NULL`) — concurrent claims have exactly one winner (`TASK_ALREADY_CLAIMED`).
- Handoff lifecycle is enforced end-to-end: Open→Accepted/Rejected→Closed,
  storage-level CAS guard, post-transition row returned, `handoff.closed`
  event added, audit events atomic with mutations in the same transaction.
- `task complete` is gated on strong dependencies like claim/start (single
  shared domain predicate); task creation whitelists `planned|ready`;
  terminal tasks are frozen for edit; title/description capped at 200/8000.

### Data safety

- Prune/archive propagate every unlink/copy error and abort before deletes;
  archive copies run on a dedicated connection with bound ATTACH/VACUUM INTO.
- Migration 0014 rebuilds `handoffs` / `checkpoint_corrections` with ON DELETE
  CASCADE so pruning works under FK enforcement (schema version now **16**;
  0015 re-asserts agent-name uniqueness, 0016 adds the task-list index).
- `init --force` uses upsert semantics that never delete the project row and
  reuses the persisted project id when `.carryctx/config.toml` is missing.

### Robustness

- MCP stdio server: bounded frame size (1 MiB), clean EPIPE shutdown, child
  process timeout, no responses to JSON-RPC notifications.
- Admission lock: metaless lock dirs self-heal after a grace period, PID
  liveness uses portable `kill(pid, 0)`, hostname from `gethostname(2)`
  (with `/etc/hostname` fallback), retry budget raised to ~20 s exponential
  backoff for parallel subagents.
- Worktree create journals are reconciled on startup; bind failure rolls back
  the created worktree.
- SQL hardening: LIKE wildcards escaped, FTS5 queries quote-safe with empty
  short-circuit, sqlite_master probes map errors instead of defaulting,
  `prune_stale` joins the caller's transaction with an in-transaction TOCTOU
  re-check.

### Identity & performance

- Sessions enforce owner-only end/pause/resume; duplicate agent names are
  rejected with actionable errors; deactivated agents cannot resolve or act
  anywhere (including `search --assignee` and `event list --agent` filters).
- Worktrees match by path components, not string prefix.
- Event listing defaults to 200 per page ordered by the `(occurred_at, id)`
  keyset; task lists default to `[task].list_limit` (200) backed by migration
  0016; N+1 team-status and checkpoint lookups removed.
- `session start --reuse` honors reuse-if-active-else-new; `status --since`
  removed, `--worktrees` added (Markdown worktree table).

### Known intentional gaps

- Some command files outside the six threaded in this wave still use bare
  exit-code mapping and double DB opens.
- Worktree-recovery journal reconciliation window is millisecond-scale by
  design (documented in wave-2 report).
- Non-Unix platforms fall back to fail-safe lock behavior (holders are never
  stolen based on PID alone).
