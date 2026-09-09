# ctxpack Export/Import Design (Interchange Format v1)

**Status:** Draft for review, 2026-09-09. Phase 0 + Phase 1 only. Merge/three-way/DAG explicitly deferred.

**Date:** 2026-09-09

**Commander:** cmd-export (`carryctx-cli`)

**Scope:** offline-first portable project state. CarryCtx owns serialization, validation, replace-import, and conflict refusal. Transport (scp/ssh/NAS/Syncthing/rclone/git) belongs to the user. No network code enters the binary (see `architecture/zero-network-policy.md`).

**Goal:** let a user move a project's agent/task/context state between machines with `export` + `import`, using only local operations the user can pipe through any transport they choose.

## 0. Binding constraints

1. **Persistence vs interchange are separate.** SQLite (`<git-common-dir>/carryctx/state.sqlite`) stays the internal runtime format. `ctxpack-dir` (v1) is the interchange format. Never zip the raw state dir as the protocol: it carries `locks/`, `journals/`, `backups/`, `*-wal`/`*-shm`, and absolute paths.
2. **Fail closed.** Import into an initialized project without an explicit `--mode` refuses with `STATE_CONFLICT`. `--merge` reports `UNSUPPORTED_OPERATION` until the merge milestone lands. Destructive paths require `--yes` (+ `--non-interactive` in CI) and take a verified pre-import backup first, reusing the `restore`/`sync pull` journal pattern.
3. **Append-only audit is never rewritten.** Import appends `project.exported` / `project.imported` events in the same transaction as the state change. Historical event payloads (mixed snake/camelCase) are preserved byte-for-byte.
4. **Display IDs are not identity.** Internal ULIDs are stable entity identity (`src/domain/ids.rs`). `CTX-xxxx`/`DEC-`/`PX-`/`HO-` come from the per-project `sequences` table and must be reconciled as `max+1` on import, never trusted from the bundle.

## 1. Non-goals for v1

- No `merge`, no `conflict list/show/resolve`, no `snapshot log/diff`, no three-way merge, no export DAG. Manifest reserves `parents: []` and `sequences: {}` for that future; v1 readers ignore them.
- No `--allow-non-git`. Import requires a Git repository (same as every other command via `git.discover`).
- No native single-file compression. v1 ships `--format dir` only (git-diff friendly, zero new dependencies). Single-file transport is a documented external recipe: `tar -cf - <dir> | zstd | ssh ...`, `age`, etc. Native `.ctxpack` (tar+zstd in-binary) is a later option pending `cargo deny`/`audit` review of `tar`+`flate2`.
- No `sync` replacement. `sync push/pull` (whole-DB LWW + `SYNC_PROJECT_MISMATCH`) stays for same-project-id fast sync. Export/import is the portable, re-anchorable path.

## 2. Interchange layout v1 (`--format dir`)

```text
<export-dir>/
├── manifest.json
├── project.json
├── agents.jsonl
├── tasks.jsonl
├── task_dependencies.jsonl
├── progress_items.jsonl
├── sessions.jsonl
├── worktrees.jsonl
├── checkpoints.jsonl
├── checkpoint_corrections.jsonl
├── scopes.jsonl
├── decisions.jsonl
├── handoffs.jsonl
├── teams.jsonl
├── team_members.jsonl
├── graph_nodes.jsonl
├── graph_edges.jsonl
├── events.jsonl
└── sequences.jsonl
```

- One JSON object per line in `*.jsonl`, keys in `snake_case`. `events.jsonl` preserves stored payloads verbatim.
- Excluded from export: `locks/`, `journals/`, `backups/`, WAL/SHM sidecars, XDG cache/runtime, absolute-path-anchored II runtime files. Paths export as stored; re-anchoring happens at import (Section 4).
- `manifest.json` (v1):

```json
{
  "format": "carryctx-pack-dir",
  "format_version": 1,
  "carryctx_version": "0.8.1",
  "schema_version": 17,
  "project_id": "01KY6ZK0TMQM5ANGZ97T68C71G",
  "export_id": "01M22...",
  "created_at": "2026-09-09T06:00:00Z",
  "parents": [],
  "sequences": { "task": 111, "decision": 14, "progress": 203, "handoff": 7 },
  "source": {
    "git_branch": "main",
    "git_commit": "21951eb",
    "hostname": "dev-a"
  },
  "counts": { "tasks": 104, "events": 5120 }
}
```

`format_version` is independent of `carryctx_version`. Import migrates `format_version < current` forward; unknown future `format_version` refuses with `UNSUPPORTED_OPERATION`.

## 3. Export contract

```bash
carryctx export --pack-format dir -o ./ctxpack-dir/ [--dry-run]
carryctx export --pack-format dir --stdout  # v1: returns UNSUPPORTED_OPERATION; use external tar pipe
```

> 实现注（2026-09-09 review）：pack 格式开关命名为 `--pack-format` 而非
> `--format`，避免与控制输出信封渲染的全局 `--format text|json|markdown`
> 冲突（`graph export` 因同类冲突使用 `--type`，见 `src/main.rs`）。
> `--stdout` 在 v1 明确返回 `UNSUPPORTED_OPERATION`（exit 10），不静默降级。

- Default exports the whole project. `--task <id>` (optional v1) exports that task's closure for scoping tests; full-project remains the primary contract.
- `--dry-run`: validate + print plan (entity counts, target path), write nothing, exit 0.
- Success envelope `export.create` with `data: {manifest, counts, path}`; stdout carries only the envelope (or the tar stream with `--stdout`, in which case the envelope goes to stderr per stream rules).
- Appends `project.exported` audit event (export_id, counts, format_version) in the source project.

## 4. Import contract

```bash
carryctx import ./ctxpack-dir/ [--mode replace] [--dry-run] [--yes]
```

State machine: `validate bundle -> require git repo -> state exists? -> fresh INIT path | initialized REFUSE-or-REPLACE path`.

- **Fresh repo** (git repo, no `state.sqlite`): equivalent to `init` reusing the bundle's `project_id`/`task_prefix`/`name` (same rule as cloning with `.carryctx/config.toml`), then load, then re-anchor `projects.repository_root/git_common_dir` to the target, then `project.imported`. If `.carryctx/config.toml` exists with a _different_ `project.id`, refuse with `STATE_CONFLICT` (never silently fork identity).
- **Initialized repo**: bare `carryctx import <dir>` refuses (`STATE_CONFLICT`, exit 3) with a hint to pass `--mode replace`. `--mode replace --yes` takes a verified pre-import backup (`backups/pre_import_*`), stages a candidate DB built from the bundle, validates it with the same gate as `sync pull` (`integrity_check`, FK check, exactly one well-formed project row, migration compat), then atomically swaps via the restore journal pattern. `--mode merge` returns `UNSUPPORTED_OPERATION` in v1.
- **Re-anchor policy (normative):** `projects.repository_root/git_common_dir` are rewritten to the target. `worktrees.*` rows whose `normalized_path` does not exist at the target are dropped from the live table with a warning (visible in `doctor` as pruned, audited as `worktree.pruned`); `sessions.working_directory` values are kept as history but never trusted for liveness. Absolute paths are never compared for equality across machines.
- `--dry-run --format json`: full validation + diff summary (`would_replace`, bundle vs local `project_id`, counts, dropped worktrees), writes nothing, `operation.applied: false`.

## 5. Errors and exit codes (public API)

| Condition                                                  | code                                         | exit |
| ---------------------------------------------------------- | -------------------------------------------- | ---- |
| Not a git repo                                             | `GIT_ERROR`                                  | 4    |
| Bundle manifest missing/invalid, checksum/count mismatch   | `VALIDATION_FAILED`                          | 8    |
| `format_version` newer than reader                         | `UNSUPPORTED_OPERATION`                      | 10   |
| Target initialized, no `--mode`                            | `STATE_CONFLICT`                             | 3    |
| Bundle project_id vs local config mismatch on fresh import | `STATE_CONFLICT`                             | 3    |
| `--mode merge` (v1)                                        | `UNSUPPORTED_OPERATION`                      | 10   |
| Candidate/backup validation failure                        | `DATABASE_ERROR` / `BACKUP_INTEGRITY_FAILED` | 5    |
| Destructive import without `--yes` (TTY)                   | prompt; non-TTY refuses                      | 3    |

JSON errors go to stderr; `--dry-run` never writes SQLite, git, or config.

## 6. Acceptance criteria (Phase 1)

- AC1: `export --format dir` on `carryctx-cli` round-trips into a fresh `git init` clone: `import` succeeds, `task list` counts match, `event list` count matches, `doctor` clean except expected worktree-prune warnings.
- AC2: Re-import over an initialized project without `--mode` fails `STATE_CONFLICT`; with `--mode replace --yes` succeeds and leaves a `pre_import_*` backup plus `project.imported` event.
- AC3: Tampered manifest (edited count, unknown `format_version`) fails closed with the table above; no partial state written.
- AC4: `--dry-run` (both directions) writes nothing; `--stdout` pipe `export | import` works across two temp clones.
- AC5: `cargo fmt --check`, `clippy -- -D warnings`, `cargo test`, `markdownlint` pass; docs (`cli-specification.md` stability table, `configuration.md` if keys change) updated in the same change.

## 7. Test plan (incl. self-hosted git)

- Unit: manifest validate/reject matrix, sequences reconcile (`max+1`), re-anchor pure functions, error mapping.
- Integration (temp git repos, no network): fresh-import, replace-refuse, replace-apply, tamper-matrix, dry-run-cleanliness, stdout-pipe round-trip.
- Self-hosted git end-to-end via podman (user-provided): bring up a local git host (e.g. Gitea/Forgejo container), `git clone` on two client mounts, carry `.carryctx/config.toml` through git, carry state through `export --stdout | ssh/import` or a shared volume standing in for NAS/Syncthing; assert AC1–AC4 across hosts. (`carryctx-cli/src/commands/sync.rs:11`, `:17`). Container definitions live under workspace `recording/` (never in product repos) and are documented in the verification report under `reports/`.

## 9. Review and 0.8.2 release gate (2026-09-09 addendum)

- Code review is mandatory before any `0.8.2` tag: storage integrity (parameter binding, transaction + audit atomicity, FK/WAL handling), CLI contract (envelope, exit codes, stdout/stderr, `--dry-run` cleanliness), and error-message safety (no SQL/secrets/absolute user paths in output).
- Gates: `cargo fmt --check`, `RUSTFLAGS="-D warnings" cargo clippy --workspace -- -D warnings`, `cargo test`, `just markdownlint`, `just package-smoke`, plus the AC1–AC5 matrix in Section 6 and the podman Gitea e2e in Section 7.
- Versioning: `Cargo.toml` + `CHANGELOG.md` + docs stability table move together; tag `v0.8.2` only on a clean worktree after review findings are closed or explicitly deferred as new tasks.

## 8. Work breakdown (commander dispatch)

- T1 (docs, done here): this design + `cli-specification.md` delta.
- T2 (export): `export --format dir`, manifest builder, `project.exported` event, `--dry-run/--stdout`.
- T3 (import fresh + replace): bundle validator, candidate-build + restore-journal swap, re-anchor + worktree prune, `--dry-run` diff.
- T4 (tests): unit + integration matrix + podman Gitea e2e harness in `recording/`.
- T5 (docs sync): `cli-specification.md`, `README`, skills preset notes; `reports/` verification record.
- Dependencies: T3 on T2's reader/validator; T4 spans T2–T3; T5 last. `--merge`/DAG reserved for a follow-up design.
