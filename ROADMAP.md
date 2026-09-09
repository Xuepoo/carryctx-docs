# CarryCtx Roadmap

CarryCtx is a local-first lifecycle and control layer for coding agents and
human developers: durable project state, agent and team lifecycle, and
portable offline state. It is not an agent harness, not a cloud sync service,
and not a TODO list. The core binary never initiates network connections
(see `architecture/zero-network-policy.md`); moving state between machines is
local `export` / `import` composed with user-chosen transport
(see `architecture/state-transport-boundary.md`).

## Where we are (v0.9.0)

- Runtime truth: a Rust CLI over a SQLite project state at
  `<git-common-dir>/carryctx/state.sqlite`, shared by linked worktrees.
- Workspace: Cargo workspace 4+1 (`core` / `sqlite` / `vcs` / `pack` / `cli` + root facade) with zero CLI contract change (002 Complete).
- Three separated layers: SQLite persistence, ctxpack-dir interchange
  (`manifest.json` plus JSONL via `carryctx export` / `carryctx import`),
  and external transport (Git, SSH, NAS, Syncthing, rclone).
- Shipped surface: tasks and dependencies, sessions, checkpoints, teams,
  worktrees, handoffs, decisions, context graph, append-only event audit,
  presets, MCP stdio server, local-only `sync push` / `pull`, and ctxpack
  dir v1 (replace-import with conflict refusal; merge deferred).
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
3. **ctxpack v1 hardening.** Export profiles and privacy review (hostname,
   paths, agent names, task text), machine-local field audit, and
   diff/inspection UX. Run real multi-machine flows first; decide
   per-entity merge semantics from observed conflicts before designing merge.
4. **Single user-manual source of truth.** `manual/` is normative; the
   website manual generates or syncs from it (website-repo work). The
   remaining docs-side step is pointing the lifecycle manual at
   `export` / `import`.

## Explicitly later

- Semantic merge, three-way merge, and DAG (`parents` and `sequences` are
  already reserved in the v1 manifest), plus `snapshot log` / `diff` and
  `conflict list` / `show` / `resolve`.
- Multi-repository projects — only if the design passes the boundary test,
  and still with no network in Core.

## Non-goals (binding)

- No network code in the core binary: no remote sync adapters, no GitHub
  Issue or Pull Request synchronization, no cloud sync service, and no
  preset registry or marketplace in Core (preset fetching is a user-run
  `git clone` plus local `preset install`).
- No second user-manual source of truth.
