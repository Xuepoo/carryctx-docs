# Code Review Consolidation — 2026-08

Consolidated output of a four-pass automated code review of `carryctx-cli` at 0.5.8. All findings below were recorded as durable progress notes by reviewer agents in the shared state DB, then deduplicated, grouped into themes, and filed as GitHub issues.

## Methodology

Four reviewer agents ran in parallel over separate scopes, recording findings as `[P1|P2|P3] path:line | description | fix` progress notes (`PX-0074`–`PX-0179`) against review tasks CTX-0063…CTX-0066:

| Reviewer  | Scope                                                 | Task     | Raw notes |
| --------- | ----------------------------------------------------- | -------- | --------- |
| rev-cmds  | CLI command layer (`src/commands/*`, wiring)          | CTX-0063 | 47        |
| rev-ops   | Ops surfaces: MCP, worktree, team, concurrency        | CTX-0066 | 17        |
| rev-store | Storage & adapter layer (`src/adapter/*`, migrations) | CTX-0065 | 18        |
| rev-core  | Application & domain layer                            | CTX-0064 | 24        |
| **Total** |                                                       |          | **106**   |

Raw severity mix: **28 × P1 · 45 × P2 · 33 × P3**.

Cross-reviewer duplicates were merged (all file:line refs preserved):

- handoff status machine missing end-to-end: rev-ops PX-0090 ≈ rev-core PX-0158
- task claim check-then-act described independently by rev-ops (SQL + application refs kept together)
- swallowed event-append errors in handoff paths: rev-cmds PX-0109 ≈ rev-ops PX-0100
- `let _ = uow.commit()` in project.rs: rev-cmds PX-0127 ≈ rev-store PX-0143
- prune PRAGMA foreign_keys=OFF wrapper: rev-store PX-0141 ≈ rev-cmds PX-0128
- init --force cascade/orphan wipe: rev-store PX-0142 ≈ rev-core PX-0167
- team dry-run JSON silent exit: rev-cmds PX-0075 ≈ PX-0077
- prune_stale transaction handling: rev-store PX-0147 ≈ rev-ops PX-0098
- discarded `--reason` flags: rev-cmds PX-0086 + PX-0108 combined

**Result: 106 raw → 98 consolidated findings across 12 themed issues.**

## Severity Summary (after dedup)

| Severity                         | Raw     | Consolidated |
| -------------------------------- | ------- | ------------ |
| P1 (correctness/data-loss/panic) | 28      | 25           |
| P2 (robustness/integrity)        | 45      | 41           |
| P3 (polish/enhancement)          | 33      | 32           |
| **Total**                        | **106** | **98**       |

## Issues Filed

| #                                                     | Title                                                                                                                               | Sev | Findings |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | --- | -------- |
| [#94](https://github.com/Xuepoo/carryctx/issues/94)   | Reachable panics: `unreachable!()` on dry-run JSON path and unguarded multibyte byte-slices                                         | P1  | 6        |
| [#95](https://github.com/Xuepoo/carryctx/issues/95)   | `--dry-run` contract violations: graph writes despite dry-run; JSON error paths exit silently                                       | P1  | 5        |
| [#96](https://github.com/Xuepoo/carryctx/issues/96)   | Output-envelope violations: markdown renderers print `Error:` to stdout with exit 0; fake envelopes; hooks never match tasks        | P1  | 16       |
| [#97](https://github.com/Xuepoo/carryctx/issues/97)   | config get/set broken TOML handling: dotted keys into wrong table, stringified values, line-prefix get                              | P1  | 4        |
| [#98](https://github.com/Xuepoo/carryctx/issues/98)   | State-machine & CAS races: task claim last-writer-wins, handoff lifecycle unguarded, dependency-gate bypasses                       | P1  | 6        |
| [#99](https://github.com/Xuepoo/carryctx/issues/99)   | Swallowed errors break audit atomicity: `let _ =` on appends/commits; stats failures as confident zeros                             | P2  | 10       |
| [#100](https://github.com/Xuepoo/carryctx/issues/100) | Data-loss risks in prune/archive/init: swallowed unlink/copy errors, NO-ACTION FKs never unlinked, init --force cascade             | P1  | 7        |
| [#101](https://github.com/Xuepoo/carryctx/issues/101) | SQL robustness: unescaped LIKE, interpolated VACUUM INTO/ATTACH, error-swallowing probes, manual transactions                       | P2  | 8        |
| [#102](https://github.com/Xuepoo/carryctx/issues/102) | MCP server robustness: EPIPE panics, unbounded stdin reads, child without timeout, responses to notifications                       | P2  | 4        |
| [#103](https://github.com/Xuepoo/carryctx/issues/103) | Locking/concurrency/portability: metaless lock dir bricks commands, /proc-only PID liveness, non-atomic hook install, 2.5s lock cap | P2  | 6        |
| [#104](https://github.com/Xuepoo/carryctx/issues/104) | Identity/session integrity: foreign session takeover, duplicate agent names, deactivated agents resolve, cwd-prefix mismatch        | P2  | 5        |
| [#105](https://github.com/Xuepoo/carryctx/issues/105) | Enhancements batch: pagination, duplication, query performance, ignored flags, small correctness nits                               | P3  | 29       |

## Full Finding List by Theme

### #94 — Reachable panics (P1)

- `src/commands/task.rs:302` — `_ => unreachable!()` on dry-run JSON read-only arms; `--json --dry-run task list` panics SIGABRT.
- `src/commands/handoff.rs:313` — `&summary[..40]` sliced after byte-length check; CJK/emoji summaries abort exit 134.
- `src/commands/progress.rs:243` — `&p.content[..40]` same defect, verified live panic.
- `src/commands/decision.rs:188-190` — `&d.title[..40]` same defect on free-text titles.
- `src/commands/worktree.rs:151` — `w.task_id[..8]` raw-byte slice in markdown preview.
- `src/commands/event.rs:110` — `actor_agent_id[..8]` raw-byte slice; actors can be unresolved multibyte names.

### #95 — `--dry-run` contract violations (P1)

- `src/commands/graph.rs:161-315` — `handle_graph` never checks `ctx.dry_run`; `--dry-run graph add-node` still writes.
- `src/commands/team.rs:192-231` — `.map_err(|e| e.exit_code)?` discards error: `--json --dry-run team member add <bad>` exits 7 with empty stdout+stderr.
- `src/commands/worktree.rs:48` — check_dry_run before runtime open for JSON mode too: stderr text only, no stdout envelope.
- `src/commands/{session.rs:105,handoff.rs:93,decision.rs:69,checkpoint.rs:75,config.rs:78,progress.rs:85}` — check_dry_run before runtime open, no is_json guard anywhere.

### #96 — Output-envelope violations (P1)

- Markdown renderer set (all print `Error: {e}` to stdout, return Ok/exit 0): `task.rs:395-410`, `worktree.rs:141-158`, `session.rs:237-253`, `handoff.rs:307-332`, `search.rs:105`, `progress.rs:254`, `decision.rs:204`.
- `src/commands/hooks.rs:52-56,68-74` — templates grep `"displayId"` but context JSON emits snake_case `display_id`: hooks never match a task.
- `src/commands/project.rs:110-117` — register/unregister fake success envelopes (`needs_init`/`unregistered` as success).
- `src/commands/preset.rs:139-145` — `preset list --json` hardcodes `data:[]`.
- `src/commands/preset.rs:62-120,152-162` — hand-built JSON via println!: errors to stdout, unescaped interpolation, `{:?}` not legal JSON.
- `src/commands/preset.rs:118,178-186` — exit codes mismapped regardless of cause.
- `src/commands/checkpoint.rs:145-147` — show maps not-found to bare exit 7, no envelope/message.
- `src/commands/checkpoint.rs:99-101,296-299` + `search.rs:66-74` — ExitCode-only plumbing discards messages (silent exits).
- `src/commands/handoff.rs:205-207,425-427,489-491,553-555` — update_status/commit failures mapped to bare `.exit_code`.
- `src/main.rs:337,348-350` — wiring boundaries discard CarryCtxError messages.
- `src/main.rs:226-244` — outside-repo text-mode failures print nothing yet exit non-zero.
- `src/commands/hooks.rs:96` — handle_hooks ignores `is_json`; space-containing label.

### #97 — config TOML handling (P1)

- `src/commands/config.rs:126-176` — set appends at EOF so dotted keys land in the last table (verified `task.strict_completion` under `[verification]`); bools stringified; no re-parse validation.
- `src/commands/config.rs:96-112` — get uses line-prefix matching: `project.name` empty, `task` collides with task_prefix.
- `src/commands/config.rs:139-141` — scope-less set/unset silent exit 2; parsed `--local` ignored.
- `src/commands/config.rs:92-94,105` — `read_to_string().unwrap_or_default()` hides unreadable files as empty.

### #98 — State-machine & CAS races (P1)

- `src/adapter/sqlite_repos.rs:573` + `src/application/task.rs:404` — claim is check-then-act with unconditional UPDATE; last-writer-wins double-claim.
- `src/adapter/sqlite_repos.rs:3070` — handoff update_status lacks compare-and-set guard.
- `src/commands/handoff.rs:454,508,559` + `src/application/collaboration.rs:586-674` — accept/reject/close never check current status; transition_handoff dead code reimplemented inline.
- `src/domain/task.rs:214` — Complete ignores `strong_dependencies_complete`.
- `src/application/task.rs:104` + `src/main.rs:596` — create `--status` bypasses dependency gating incl. completed-at-birth tasks.

### #99 — Swallowed errors / audit atomicity (P2)

- `src/commands/handoff.rs:437-443,498-503,~459,~513,543-560` — `let _ = event_repo.append(...)`; Close appends no event at all.
- `src/commands/graph.rs:190-215,238-262` — graph mutations skip UoW + audit events entirely.
- `src/commands/project.rs:125,156` — `let _ = uow.commit()` reports success without persisting.
- `src/application/project_mgmt.rs:47-56` — backup audit event impossible (`project_id=""` on read-only conn, wrapped in let _).
- `src/application/stats.rs:54-100` — queries swallow errors → confident all-zero output.
- `src/application/stats.rs:146-157` — duration parse failure falls back to `Utc::now()`, billing wall-clock days.
- `src/application/collaboration.rs:402`, `worktree.rs:31`, `handoff.rs:344-355,461,514` — `.ok().flatten()` turns DB failures into not-found; more swallowed event appends.
- `src/adapter/sqlite.rs:587` — backup retry loop discards up to 99 VACUUM INTO errors.

### #100 — Data-loss risks in prune/archive/init (P1)

- `src/application/project_mgmt.rs:92-98` — prune swallows unlink errors before hard deletes.
- `src/application/project_mgmt.rs:110-146` — archive copy steps discard errors then rows are hard-deleted: permanent loss on silent failure.
- `migrations/project/0005+0006` + `project_mgmt.rs:95` — `handoffs.task_id` / `checkpoint_corrections.checkpoint_id` NO-ACTION FKs never unlinked/archived: FK=ON delete fails, FK=OFF orphans.
- `src/commands/project.rs:144-163` (+147-149,159-161,71) — prune wraps op in PRAGMA foreign_keys=OFF with both PRAGMAs error-swallowed.
- `src/application/init.rs:149-155` — `--force` INSERT OR REPLACE cascades away tasks/agents/progress/teams; mints new project_id orphaning old rows.

### #101 — SQL robustness (P2)

- `src/adapter/sqlite_repos.rs:2753-2759` — decision search LIKE without `%`/`_` escaping or ESCAPE clause.
- `src/adapter/sqlite.rs:571` + `project_mgmt.rs:111` — VACUUM INTO / ATTACH interpolate paths instead of binding.
- `src/adapter/sqlite.rs:202-209,243-250` — sqlite_master probes `unwrap_or(false)` misread DB errors as fresh DB.
- `src/domain/search.rs:101-112` + `commands/search.rs:81` — unterminated quotes surface raw FTS5 syntax errors; empty queries error too.
- `src/adapter/sqlite_repos.rs:1986,2010-2065` — prune_stale manual BEGIN/COMMIT via execute_batch breaks under UoW; scan outside tx (TOCTOU).
- `src/adapter/sqlite.rs:299-301` — rebuild heuristic hardcodes version 12||13 for FK-off.
- `migrations/project/0001_foundation.sql:47-53` — CREATE INDEX without IF NOT EXISTS.

### #102 — MCP server robustness (P2)

- `src/application/mcp.rs:64-65,90-91,182-183,243-244,283-284,300-301,316-317` — writeln!/flush unwrap panics on EPIPE.
- `src/application/mcp.rs:46` — unbounded lines() read allows OOM.
- `src/application/mcp.rs:203` — spawned child waited with no timeout freezes single-threaded server.
- `src/application/mcp.rs:316` — responses emitted for id-less notifications violate JSON-RPC.

### #103 — Locking/concurrency/portability (P2)

- `src/adapter/filesystem.rs:109` — crash leaves metaless admission-lock dir; bricks all mutating commands.
- `src/adapter/filesystem.rs:204` — PID liveness /proc-only (non-portable) + Linux PID reuse.
- `src/commands/hooks.rs:163` — hook written non-atomically though write_atomic exists.
- `src/commands/hooks.rs:173` — chmod failure swallowed.
- `src/application/worktree.rs:191` — worktree.create journal has no recovery consumer; no rollback on bind failure.
- `src/main.rs:416` — admission lock fixed 2.5s cap serializes parallel subagents.

### #104 — Identity/session integrity (P2)

- `src/application/session.rs:157-240` — end/pause/resume never verify session ownership.
- `src/application/agent.rs:31-77` — no agent-name uniqueness; resolver picks arbitrary row (add UNIQUE index).
- `src/application/runtime.rs:270-278` — deactivated agents still resolve despite message claiming otherwise.
- `src/application/runtime.rs:220` — worktree fallback `cwd.starts_with` prefix match resolves wrong task (`/repo/f` vs `/repo/foo`).
- `src/main.rs:418` — HOSTNAME fallback literal `'unknown'`.

### #105 — Enhancements batch (P3)

Pagination/lifecycle: `application/event.rs:33-49` inclusive occurred_at cursor duplicates pages (→ keyset `(occurred_at,id)`); `collaboration.rs:387-421` supersede lacks self/already-superseded guards; `application/task.rs:101` vs `sqlite_repos.rs:628` drifted incomplete-strong-deps definitions (cancelled counted differently); `progress.rs:216-258` reorder no membership/validation; `task.rs:293-369` edit_task mutates terminal tasks freely; `team.rs:186-190` resolve_context_team internal-id-only lookup; `init.rs:216-222` init writes EMPTY config.toml via `unwrap_or_default`.

Domain/dead code: `domain/task.rs:236-241` Reopen dead deps arm; `task.rs:487-493` transition_task(Claim) owner trap; `domain/ids.rs:19` validate_task_prefix dead code; `collaboration.rs:210-255` classify_overlap misses leading-`**` patterns.

Identity/actors: `task.rs:185,349,501` actor identity drift raw-name vs ULID in events; `task.rs:60-66` title length caps.

Duplication/perf: `session.rs:131` hardcoded default agent + 4× require-agent dup; `handoff.rs:347-369,389-411,461-480,517-535` 4× resolve block dup; `main.rs:247-261` double DB open pre-dispatch vs handler; `sqlite_repos.rs:520-559,2327+` unbounded lists without LIMIT/index; `sqlite_repos.rs:786-846` team status N+1; `sqlite_repos.rs:1211` correlated latest-checkpoint subquery.

CLI surface: `session.rs:36-38` --reuse ignored; `session.rs:47-49` + `handoff.rs:476` reject/abandon `--reason` discarded; `status.rs:44` _args flags ignored; `main.rs:87` --config-compat unreachable; `main.rs:596-609` parse_task_status case-sensitive vs lowercased priority; `graph.rs:226,244,264,281,303` command labels with spaces vs dotted rule; `context.rs:267` --output write failure swallowed; `context.rs:113-150` query failures rendered as empty context; `tests/concurrency_test.rs:7` concurrency coverage only parallel task-create.

## Proposed Fix Waves

Grouped by file clusters so each wave touches a coherent layer:

- **Wave 1 — storage + MCP/locking files** (highest data-risk first): `src/adapter/sqlite_repos.rs`, `src/adapter/sqlite.rs`, `src/adapter/filesystem.rs`, `src/application/project_mgmt.rs`, `src/application/init.rs`, `migrations/project/*`, `src/application/mcp.rs`, `src/commands/hooks.rs` (atomic write/chmod parts), `src/main.rs` (lock acquisition, hostname). Covers issues #98, #99 (storage parts), #100, #101, #102, #103. Rationale: CAS guards, FK-safe prune/archive, and crash-safe locking unblock trustworthy multi-agent execution; everything else can be rebased safely on top.
- **Wave 2 — CLI surface + application logic**: `src/commands/*` (rendering, envelopes, dry-run helper, config), `src/application/{task,collaboration,session,agent,runtime,stats,event,progress,team}.rs`, `src/domain/{task,ids,search}.rs`. Covers issues #94, #95, #96, #97, #99 (command/app parts), #104, #105. Suggested order inside the wave: shared markdown/error rendering helpers first (kills most of #96/#94 mechanically), then config (#97), then state-machine/CAS plumbing through application use cases (#98 remainder).

## Notes & Deviations

- Raw note count was 106, not ~92 as estimated pre-dump; all counts above use the actual DB contents.
- One item in the grouping plan ("event filters .ok()-swallowed returning ALL events") had no corresponding note in the DB; nearest recorded behavior is PX-0117 (context queries swallow failures → empty context), filed under #105.
- Report filename follows this directory's `YYYY-MM-DD-*` convention.
