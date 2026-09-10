# CarryCtx TODO (short/mid-term)

Only genuinely outstanding work. Shipped items live in `CHANGELOG.md`,
verification evidence in `reports/`, design records in `design/`, and
long-term invariants in `architecture/`. `ROADMAP.md` sets direction;
`architecture/state-transport-boundary.md` decides what may enter Core.

## Contract stability

- [ ] Implement the version-metadata CI check specified in
      `architecture/state-transport-boundary.md` §7 (CLI-repo work): extract the
      CLI, ctxpack-format, DB-schema, and skill-surface values from their sources
      of truth, compare the exact JSON shape, and fail on drift. The
      `use-carryctx` surface is aligned as of 0.10.0 (skill frontmatter
      `1.2.0`, `min_carryctx 0.10.0`, README `v0.10.0`; merge-milestone
      contract values: ctxpack `format_version` `2`, DB schema `18`).

## Lifecycle hooks

- [ ] Land design CTX-0010 (owner: doc-2, in progress), then implement trust,
      timeout, output limits, reentrancy guards, and failure policy.

## ctxpack hardening (merge semantics shipped in 0.10.0)

- [ ] Export profiles and privacy review: hostname, paths, agent names, and
      task text in the manifest source block and JSONL.
- [ ] Machine-local field audit: worktree paths,
      `sessions.working_directory`, and re-anchor plus prune warnings.
- [ ] Diff and inspection UX for bundles before import (including
      `snapshot log` / `snapshot diff`).
- [ ] Redacted publication flow on the dedicated public
      `refs/heads/carryctx-snapshots` ref (DEC-0052, issue #138).

## Docs single source of truth

- [ ] Website manual generates or syncs from `manual/` (website-repo work);
      the lifecycle manual now points at `export` / `import`.

## Carried-over correctness items (verified still open 2026-09-09: no callers in `carryctx-cli/src`)

- [ ] Wire `touch_activity`: `sessions.last_activity_at` never advances
      because nothing calls it. Decide call sites (progress note, checkpoint,
      resume, periodic) and whether `stats` shows tracked time versus span.
- [ ] `mark_stale_sessions` is dead code (`application/session.rs`): never
      invoked, so stale sessions are never auto-marked. Call it on `resume` and
      `session start` with the config's `stale_after`.
- [ ] Release workflow `Build (${{ matrix.target }})` renders as `skipping`
      on PRs — the matrix template does not resolve for PR checks. Fix the
      workflow so the build gate actually reports (CLI-repo workflow).

## Retired from this list

- P1 preset schema, instruction precedence, `preset` commands, and the
  supply-chain lockfile: shipped.
- P2 MCP server and IDE adapters (`carryctx mcp` over stdio): shipped.
- Phase 2 context-graph schema and queries (`graph add-node` / `link` /
  `edges` / `extract-deps` / `scan` / `export`, plus `graph_nodes` /
  `graph_edges` in Core and in ctxpack): shipped. Any remaining graph work
  is hardening, not greenfield design.
- Phase 3 subagent shared state (`team`, `handoff`, task dependencies, ready
  queue): shipped as Core coordination records.
- Phase 4 marketplace and native-integration vision, plus the remote-sync
  and GitHub-sync roadmap items: removed as Core scope — inconsistent with
  the zero-network invariant (see the ADR §6 non-goals).
