# CarryCtx Roadmap

CarryCtx is a local-first lifecycle and control layer for coding agents and
human developers: durable project state, agent and team lifecycle, and
portable offline state. It is not an agent harness, not a cloud sync service,
and not a TODO list. The core binary never initiates network connections
(see `architecture/zero-network-policy.md`); moving state between machines is
local `export` / `import` composed with user-chosen transport
(see `architecture/state-transport-boundary.md`).

## Where we are (v0.11.4)

- Runtime truth: a Rust CLI over a SQLite project state at
  `<git-common-dir>/carryctx/state.sqlite`, shared by linked worktrees.
- Workspace: Cargo workspace 4+1 (`core` / `sqlite` / `vcs` / `pack` / `cli` + root facade) with zero CLI contract change (002 Complete).
- Three separated layers: SQLite persistence, ctxpack-dir interchange
  (`manifest.json` plus JSONL via `carryctx export` / `carryctx import`),
  and external transport (Git, SSH, NAS, Syncthing, rclone).
- Shipped surface: tasks and dependencies, sessions, checkpoints, teams,
  worktrees, handoffs, decisions, context graph, append-only event audit,
  presets, MCP stdio server, local-only `sync push` / `pull`, and ctxpack
  dir v1 (replace-import with conflict refusal).
- Merge milestone, shipped in 0.10.0: ctxpack v2 (`parents` DAG,
  `tombstones`, `redacted` flag), schema 18, `import --mode merge` with
  conflict staging and `conflict list/show/resolve/apply/abort`, local
  snapshot Git refs (`export --snapshot`, `import --from-git`), and
  two-parent merge snapshot commits after `--mode merge`/`conflict apply`.
  The local-only unredacted ref guard is in place (DEC-0052).
- Release 0.11.0 ships the public redacted publication flow
  (`export --publication` writes `manifest.redacted: true` to the dedicated
  `refs/heads/carryctx-snapshots` ref) and the blank-session-ref foreign-key
  fix (CTX-0155, CTX-0153).
- Release 0.11.1 hardens that publication flow (CTX-0159): in addition to
  secret redaction, `export --publication` now neutralizes host-identifying
  paths in every published row, in `project.json`
  (`repository_root`/`git_common_dir`), and in `manifest.source` — user-home
  prefixes collapse to `~/` with the tail preserved, host roots
  (`/mnt/**`, `/media/**`, `/run/media/**`, `/private/var/**`,
  `/var/folders/**`) collapse to `***REDACTED-PATH***`, while URLs, Git
  SHA-1s, benign slugs, and multibyte text are left intact. The unredacted
  local `refs/carryctx/local` snapshots keep the real paths, and redacted
  bundles stay refused as merge sources.
- Release 0.11.2 fixes fresh-clone restore into an empty migrated database
  (CTX-0162, #184): a database with schema but zero project rows is now
  classified as empty and initialized from the bundle on both the directory
  and `--from-git` paths instead of failing `DATABASE_ERROR` (exit 5), and
  text-mode `stats` hints at the in-repo publication restore path when local
  state is empty (#183).
- Release 0.11.3 ships the executable project policy trust gate (CTX-0100:
  `[verification].commands` run only with global
  `[security].allow_project_commands = true` plus a matching local trust
  entry, new `trust grant|revoke|list|status`, `TRUST_DENIED` exit 9) and
  automatic `when_idle` worktree cleanup draining (CTX-0165, #188-#190).
- Release 0.11.4 fixes the context-graph UX gaps (CTX-0168/CTX-0169, #191,
  #192): `graph edges` resolves nodes by ULID, exact name, or unambiguous
  name suffix, and Rust `extract-deps`/`scan` expand brace-grouped `use`
  imports into individual dependencies instead of emitting brace paths.
- History lives in `CHANGELOG.md`, verification evidence in `reports/`,
  and design records in `design/`.

## Direction

Stability over breadth. Near-term investment goes to protocol stability, the
hooks extension boundary, cross-repo consistency, and ctxpack hardening —
not to new command breadth. Every proposed Core capability must pass the
state-transport boundary test before it is accepted.

## Near milestones

1. **Contract stability.** Pin the four contract versions (CLI, ctxpack
   format, DB schema, skill surface) as machine-readable metadata with CI
   comparison. The shape and check procedure are specified in
   `architecture/state-transport-boundary.md` §7; the check itself is
   implemented as CLI-repo work.
2. **Lifecycle hooks.** Land the hooks design (CTX-0010: CarryCtx lifecycle
   layer versus Git hooks, with trust, timeout, reentrancy, and failure
   policy), then implement. Backup, sync, notification, and CI compositions
   build on hooks — without network code in Core.
3. **ctxpack hardening.** Export profiles and privacy review (hostname,
   paths, agent names, task text), machine-local field audit, and
   diff/inspection UX. The semantic merge milestone shipped in 0.10.0, the
   public redacted publication flow shipped in 0.11.0 and its host-path
   redaction hardening shipped in 0.11.1; the remaining work is
   `snapshot log` / `snapshot diff`.
4. **Single user-manual source of truth.** `manual/` is normative and now
   documents the `export` / `import` lifecycle; the website manual generates
   or syncs from it (website-repo work).

## Explicitly later

- `snapshot log` / `snapshot diff` UX (the distinct public
  `refs/heads/carryctx-snapshots` ref itself shipped in 0.11.0; DEC-0052).
- Multi-repository projects — only if the design passes the boundary test,
  and still with no network in Core.

## Non-goals (binding)

- No network code in the core binary: no remote sync adapters, no GitHub
  Issue or Pull Request synchronization, no cloud sync service, and no
  preset registry or marketplace in Core (preset fetching is a user-run
  `git clone` plus local `preset install`).
- No second user-manual source of truth.
