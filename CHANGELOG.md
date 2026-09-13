# CarryCtx Changelog

## [0.8.0] - 2026-08-30

### Changed

- The v0.8 CLI is shipped as a native Rust 2024/Cargo binary. Its release workflow
  builds verified multi-platform assets for Cargo, GitHub Releases, and platform
  packages; npm remains an optional platform-wrapper distribution channel.
- Release validation now binds the tag and Cargo version to the built assets before
  package publication, including the platform npm packages.
- AUR publication is unavailable for this release because of an upstream outage;
  users should use Cargo or the GitHub Releases binaries until it resumes.

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
- Executable project policy trust gate (`feat(trust)`, CTX-0100, shipped in
  CLI but not yet released): repository-controlled _executable_ policy
  (today `[verification].commands`) no longer runs on its own. A command
  executes only when both the global `[security].allow_project_commands =
true` (default `false`, global config only) and a matching local trust
  entry (`trusted-projects.json` with a `sha256` policy fingerprint) allow
  it. New `carryctx trust grant|revoke|list|status`; denied execution fails
  closed with `TRUST_DENIED` (exit 9). Specified in
  `cli-specification.md` §26.1 and
  `design/2026-09-11-project-trust-executable-policy.md`.

## [0.11.4] - 2026-09-13

### Fixed

- **Resolve graph nodes by path/name in `graph edges` (`fix(graph)`, CTX-0168)**: `carryctx graph edges <TARGET>` accepts an exact node ULID, an exact node name, or an unambiguous name suffix (`ends_with`), matching `graph export --focus` order. Unknown references fail `RESOURCE_NOT_FOUND`; ambiguous references fail `VALIDATION_FAILED` with candidates. Fixes carryctx#191.
- **Expand brace-grouped `use` imports in Rust dependency extraction (`fix(graph)`, CTX-0169)**: `use crate::path::{a, b};` (nested groups, `self`/`Self`, globs, `as` aliases) expands into individual module paths; nothing containing `{`, `}`, or `*` is emitted. Fixes carryctx#192.

## [0.11.3] - 2026-09-11

### Added

- **Executable project policy trust gate (`feat(trust)`, CTX-0100)**: shipped in the CLI but released in 0.11.3 — `[verification].commands` execute only with global `[security].allow_project_commands = true` plus a matching local trust entry; new `trust grant|revoke|list|status`, `TRUST_DENIED` (exit 9).
- **Automatic `when_idle` worktree cleanup (CTX-0165)**: terminal task transitions and `session end` drain eligible cleanup requests without an explicit `cleanup run`; `delete_branch = "when_removed"` deletes only fully-merged branches.

## [0.11.2] - 2026-09-11

### Added

- **Empty-state restore hint on `stats` (#183)**: when the local project
  carries no CarryCtx state but the repository exposes an in-repo
  publication ref (`refs/heads/carryctx-snapshots`, usually visible as
  `origin/carryctx-snapshots` after a clone), text-mode `carryctx stats`
  prints a hint pointing at the restore path (`git fetch` +
  `carryctx init --non-interactive` +
  `carryctx import --from-git <ref> --mode replace --yes`). Hint only:
  nothing is fetched or imported automatically.

### Fixed

- **Fresh-clone restore into an empty migrated database (`fix(import)`, CTX-0162)**: a database created by an earlier runtime open (schema migrated, zero project rows) is now classified as empty and initialized from the bundle, so the documented `carryctx import --from-git origin/carryctx-snapshots --mode replace --yes` succeeds instead of failing `DATABASE_ERROR: Database must contain exactly one project row` (exit 5). Bare and `--from-git` imports share the same path, and `--dry-run` reports `would_replace: false` for the empty case. Databases carrying a project row stay fail-closed: bare import refuses with `STATE_CONFLICT`, replace still requires `--mode replace --yes`, and a multi-row database remains malformed. The `--mode merge` refusal on an uninitialized target now points at the exact `carryctx import --from-git <ref>` restore command. Fixes #184.

## [0.11.1] - 2026-09-11

### Added

- **Host paths and usernames redacted from publication artifacts (`feat(publication)`, CTX-0159)**: `export --publication` now neutralizes host-identifying paths in every published row, in `project.json` (`repository_root`/`git_common_dir`), and in `manifest.source` metadata in addition to the existing secret redaction. User-home prefixes (`/home/<user>/`, `/Users/<user>/`, `C:\Users\<user>\`) collapse to `~/` with the tail preserved so the username never appears, while the host roots `/mnt/**`, `/media/**`, `/run/media/**`, `/private/var/**`, and `/var/folders/**` collapse wholly to `***REDACTED-PATH***` (a host-root tail cannot be proven free of a username). URLs, Git SHA-1s, benign slugs, and multibyte text are not mangled. Application stays confined to the publication path: unredacted `refs/carryctx/local` snapshots keep the real paths, `manifest.redacted: true` remains the fail-closed marker, and redacted bundles stay refused as merge sources. Covered by unit tests (Unix/macOS/Windows homes, host roots, benign URLs/spaces, multibyte, composition) and a publication E2E that seeds host paths in a task description and progress body and asserts they are absent from the public ref while the local snapshot keeps them.

## [0.11.0] - 2026-09-11

### Added

- **Redacted publication ref writer (`feat(vcs)`, CTX-0155)**: `export --publication` writes the redacted publication artifact (`manifest.redacted: true`) and commits it to the dedicated public ref `refs/heads/carryctx-snapshots` (DEC-0052, #138) with the same Git-plumbing compare-and-swap as local snapshots — one commit per publication, no index/worktree/network mutation, and CarryCtx never pushes it. The redaction pass runs on every snapshot row before the bundle is validated: secret-shaped field names (`*_KEY`/`*_TOKEN`/`*_SECRET`/`*_PASSWORD`, `GH_PAT`, `CLOUDFLARE_*`, `AWS_*`), `NAME=value` free-text pairs, and 40+-character token-like runs become `***REDACTED***` (git SHA-1s and lowercase slugs stay readable) while row counts and the bundle file set are preserved. The local-only `refs/carryctx/local` ref and `snapshot_state` are untouched, an unredacted `--snapshot` export still refuses the public ref with `INVALID_ARGUMENTS`, the target cannot be redirected with `--snapshot-ref`, and redacted bundles import fresh but stay refused as merge sources. Covered by guard tests that scan the public tree for raw secrets and verify `git push --all` cannot move the local ref.

### Fixed

- **Blank session refs crash handoff/decision with a foreign-key error (`fix(state)`, CTX-0153)**: an empty or whitespace `--session`/`CARRYCTX_SESSION` was neither normalized to "no session" nor rejected, so `""` reached the `handoffs.session_id` and `decisions.session_id` foreign keys and failed with `DATABASE_ERROR: FOREIGN KEY constraint failed`. Blank refs are now normalized to `NULL` when the invocation context is built, and `handoff create`/`decision add` resolve non-blank refs through the shared CTX-0148 policy at the point of use so an invalid ref fails closed instead of violating the constraint. Fixes #160.

## [0.10.0] - 2026-09-11

The merge milestone: semantic three-way merge of project state across clones,
conflict staging and resolution, and local Git snapshot refs.

### Added

- **Merge-prep storage schema (`feat(storage)`, CTX-0140)**: schema 0018 adds `tombstones` (a side table of hard-delete records keyed by `(project_id, table_name, row_id)`, with `deleted_at`/`deleted_by`/`reason`) and `snapshot_state` (local-only merge-base bookkeeping; never exported). Every hard-delete path now appends tombstones in the same transaction as the delete. Ordinary reads and command output are unchanged; `doctor` reports the tombstone count.
- **ctxpack format v2 (`feat(pack)`, CTX-0139)**: the interchange manifest gains `parents` (ordered export-id DAG, `[]` for a first export), `redacted` (publication-artifact marker, default `false`), and optional per-table `watermarks`; the directory layout adds the `tombstones.jsonl` side table with count coverage on both write and validate paths. `export` emits v2 once the local schema carries the tombstone side table (schema 0018) and keeps emitting v1 before that. v1 bundles stay readable for one release cycle through an explicit in-memory v1->v2 migrator; a v1 bundle shipping tombstone rows fails count validation instead of dropping them.
- **Pure three-way merge engine (`feat(pack)`, CTX-0141)**: deterministic base/ours/theirs row merge — row-level last-writer-wins by `updated_at`, monotonic-fact NULL-union, a terminal-wins task status lattice (`completed` vs `cancelled` is the only blocking pair), tombstone-aware delete rules, display-id renumbering and agent-name aliasing with reference remapping, sequence floors that never rewind, and append-only tables merged by id union. `merge(A,B) == merge(B,A)` and `merge(M,M) == M` are enforced by a fixture matrix.
- **`import --mode merge` (`feat(import)`, CTX-0142)**: an initialized project can merge a ctxpack (directory or `--from-git`) three-way from the export DAG instead of replacing state. Base resolution prefers `--base`, then the newest common ancestor in the export-id DAG, then a degraded base-less 2-way merge reporting `base: null` / `degraded: true`; `--require-base` refuses with `VALIDATION_FAILED` (exit 8). Blocking conflicts stage a session under `<git-common-dir>/carryctx/merges/<merge_id>/` and exit `MERGE_CONFLICTS` (3) with the live database untouched; `--strict-edits` promotes row edits to conflicts. Clean merges take a verified pre-merge backup and atomically swap via the restore journal, appending `project.merged` plus `merge.auto_resolved` / `merge.display_id_renumbered` events.
- **`conflict` command family (`feat(conflict)`, CTX-0143)**: `conflict list/show/resolve/apply/abort` inspects and settles a staged merge session. `resolve --ours|--theirs [--set field=value]` records a choice without touching the live DB; `apply` materializes every resolution and performs the atomic swap with exactly-once `project.merged` / `merge.conflict_resolved` events; `abort` deletes the session with no database change. `doctor` reports active and stale merge sessions.
- **Local Git snapshot ref (`feat(vcs)`, CTX-0144)**: `export --snapshot` commits the validated bundle to a local-only Git ref (one commit per snapshot; `hash-object`/`mktree`/`commit-tree`/`update-ref` compare-and-swap, no index, worktree, or network mutation) with `CarryCtx-Export-Id`/`CarryCtx-Parents`/`CarryCtx-Source` trailers, and records the tip export id in `manifest.parents` plus `snapshot_state`. `import --from-git <ref>` materializes a ref tip into a temporary bundle and reuses the normal validate/import/merge path, with ref-history ancestor base resolution. The ref is unredacted and local-only (`refs/carryctx/local`); the public redacted publication ref `refs/heads/carryctx-snapshots` is reserved (DEC-0052). See `cli-specification.md` §12.1.1.
- **Two-parent merge snapshot commits (`feat(merge)`, CTX-0145)**: after a clean `import --mode merge` or `conflict apply`, passing `--snapshot-ref` (default `refs/carryctx/local`; a custom ref needs the equals form) writes the merged state as a Git commit with two parents — `[local ref tip, incoming snapshot commit]` — carrying `CarryCtx-Export-Id` / `CarryCtx-Parents` / `CarryCtx-Source` trailers so the next merge resolves the correct common ancestor. The database swap remains the commit point; a failed snapshot write reports `GIT_ERROR` (exit 4) without rolling back the already-audited merge.

### Fixed

- **Direct-lock `--session` refs canonicalize or fail closed (`fix(cli)`, CTX-0151)**: `import --mode merge` and `conflict apply` resolve `--session` through the shared CTX-0148 policy against a read-only connection before any write. Fixes #156.
- **Git hooks derive the commit task from the commit, not an ambient session (`fix(hooks)`, CTX-0150)**: `prepare-commit-msg` now resolves its `[CTX-XXXX]` prefix from the task-bound branch name, then an explicit `--task`/`CARRYCTX_TASK`, then the worktree binding, and only then the single task-carrying active session; `post-commit` checkpoints the commit's own task (#149).
- **Unredacted snapshot refs stay local-only (`fix(vcs)`, CTX-0144)**: the `export --snapshot` default ref is the non-branch local namespace `refs/carryctx/local`, and `--snapshot-ref` accepts only `refs/carryctx/...`. `refs/heads/carryctx-snapshots` is reserved for redacted publication (DEC-0052, #138) and is refused for unredacted export, so a plain `git push` (even `--all`) cannot move unredacted state to a remote and CarryCtx never pushes it.
- **Short session refs resolve or fail closed (`fix(session)`, CTX-0148)**: `--session`/`CARRYCTX_SESSION` and the positional session refs now accept a unique ULID prefix (case-insensitive) and canonicalize it to the full ULID before any consumer sees it. Fixes #137.
- **Fresh-clone import failed on real snapshots (`fix(ctxpack)`, CTX-0137)**: `import` now loads `worktrees` first and nulls `sessions.worktree_id`/`checkpoints.worktree_id` links to worktrees not live at the target (pruned or absent from the bundle) with a warning while keeping the history rows; every other FK stays strictly enforced and export stays lossless.

## [0.9.1] - 2026-09-10

### Changed

- **Thin workspace facade `refactor(src)` (CTX-0128)**: root `src/` is now a thin facade re-exporting `crates/carryctx-cli`; only `main.rs` + `commands/` remain in `src/` until the follow-on migration.

### Fixed

- **`sync` requires explicit `--remote` `fix(sync)` (CTX-0129)**: `sync push/pull --remote` no longer defaults to `/tmp/carryctx-remote`; the snapshot path must be given explicitly.
- **Hardcoded-path sweep verdict NONE beyond `sync` (CTX-0130)**: audit of hardcoded `/tmp` snapshot defaults found no remaining instances beyond `sync --remote`.

## [0.9.0] - 2026-09-09

### Changed

- **Workspace 4+1 crate layout**: `carryctx` monolith split into `crates/{core,sqlite,vcs,pack,cli}` plus root `carryctx` facade. Architecture is now physically enforced: `core <- sqlite|vcs|pack <- cli` with zero CLI contract change. `core` remains pure (no `rusqlite`/`Git`/`clap`/network); see `engineering-standards.md` §5.2 and `design/002-workspace-crates.md`.

### Added

- **Contract version endpoint**: `carryctx version --json` emits `contract_versions{cli,ctxpack_format,db_schema,skill_surface}` with machine-readable `version --check` drift gate (ADR §7, `architecture/state-transport-boundary.md`).
- **Thin-shim hook dispatch**: `carryctx hooks dispatch git.{post-commit,prepare-commit-msg}` shims Git hooks to Rust dispatch with composition support; legacy hooks preserved via `compose`/`legacy` status.

## [0.8.2] - 2026-09-09

### Added

- **Portable export/import (ctxpack dir v1)**: Added `carryctx export --pack-format dir -o <dir>` and `carryctx import <dir> [--mode replace]`, an offline-first, transport-agnostic interchange format (`manifest.json` + `project.json` + per-table `*.jsonl`, `format_version` independent of CLI version). Export appends a `project.exported` event; fresh import reuses the bundle identity and re-anchors absolute paths; re-import without `--mode` refuses with `STATE_CONFLICT`; `--mode merge` reports `UNSUPPORTED_OPERATION` (merge/DAG deferred). Both support `--dry-run` with `operation.applied: false`. `--pack-format` is used instead of `--format` so the global output-envelope flag keeps its meaning. See `design/2026-09-09-ctxpack-export-import.md`.

## [0.8.1] - 2026-09-04

### Added

- **Task scope commands**: Added `task scope add`, `task scope remove`, `task scope list`, and `task scope conflicts`, backed by the existing transactional scope repository and collaboration conflict matcher. Scope changes are audited and expose compact text plus machine-readable JSON output.

### Changed

- **Compact context output**: `context --compact` now reduces historical progress, projects resume-critical task and event fields, truncates oversized values, and applies the configured event and lookback limits while preserving the full mode and JSON envelope.

### Fixed

- **`worktree create` honors `--project` from any cwd** (#119).
- **Stale sessions no longer block worktree cleanup forever** (#118).
- **jj-colocated lifecycle cleanup is fail-closed**, including forced removal; already-missing directories retain idempotent cleanup.
- **Rust unused-variable alerts**: preserved audit actor, session, and timestamp bindings in SQLite audit writes while removing the CodeQL-reported unused-variable findings.

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
