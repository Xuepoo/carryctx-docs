# Bug Triage Report — stats, handoff, session lifecycle (2026-08-10)

## Tool versions

- `carryctx 0.4.6` (installed via cargo, `2026-08-07` release)
- Test project: scratch repo at `/tmp/opencode/cctx-test` (write tests) + `vectojs/vectojs` (read-only data inspection, 313 tasks / 36 sessions / 213 checkpoints)
- SQLite inspection via `sqlite3 .git/carryctx/state.sqlite`

## Summary

Five defects found, all reproducible. Root causes confirmed in source.

| #   | Severity | Command                      | Symptom                                                                                   | Root cause                                                                                                            |
| --- | -------- | ---------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| A   | high     | `stats`                      | Time Spent shows 4784h for an agent whose project is weeks old                            | `session end`/`abandon` never write `ended_at`; `stats` falls back to `Utc::now()` for every open session             |
| B   | high     | `stats`                      | Per-agent Checkpoints column is always 0 while the overview totals 213                    | Per-agent subquery joins `checkpoints.session_id = sessions.id`, but `session_id` is never populated (agent_id is)    |
| C   | medium   | `task start`                 | `task start` after `task claim` errors "Cannot transition from InProgress to start"       | `claim` already moves Ready → InProgress; the documented claim-then-start workflow can never succeed                  |
| D   | high     | `handoff create`             | `--target <name-or-role>` always fails `FOREIGN KEY constraint failed`                    | `target` is inserted verbatim as `to_agent_id`; names/roles are never resolved to an agent ULID (only raw ULIDs work) |
| E   | medium   | task transitions (text mode) | Output stream contains a `warning: ...` line appended after the JSON document, breaking ` | jq`/`json.load`                                                                                                       | `render_json_with_warnings` appends warnings to stdout after pretty-printed JSON in text mode |
| F | medium | `agent register --role` | `agents.role` stays NULL; role-based lookups always miss | The command stashes the role in `metadata_json` and passes `role: None` to the repo; the `role` column has no writer |

## Repro & evidence

### A — `session end` never sets `ended_at`; stats counts every session to now

```bash
cd /tmp/opencode/cctx-test   # scratch repo
carryctx agent register --name tester --provider test-cli
carryctx session start --agent tester
carryctx session end --agent tester
sqlite3 .git/carryctx/state.sqlite "SELECT state, ended_at FROM sessions;"
# ended | (NULL)     <-- ended_at never written
```

`src/adapter/sqlite_repos.rs` `update_state` (line ~770) updates only
`state, summary, updated_at`. `ended_at` has **no write site anywhere in the
codebase** — `grep -rn "ended_at" src/` matches only the stats reader.
Consequently `src/application/stats.rs` (line ~145) computes
`Utc::now() - started_at` for _every_ session, ended or not.

In `vectojs`: 33/36 sessions are `ended` with `ended_at` NULL and average
11.2 days since `last_activity_at`; `stats` bills each from its start to now →
opencode 4784h, omp 4180h. `mark_stale_sessions` (`src/application/session.rs`)
exists but is **dead code** — no command calls it.

### B — per-agent checkpoints always 0

```bash
sqlite3 .git/carryctx/state.sqlite \
  "SELECT COUNT(*) FROM checkpoints WHERE session_id IS NOT NULL;"  # 0 of 213
```

All checkpoints store `agent_id` (212/213) but never `session_id`. The stats
subquery `JOIN sessions cs ON c.session_id = cs.id` therefore matches nothing.

### C — claim-then-start can never succeed

```bash
carryctx task create --title t --agent tester --json
carryctx task claim CTX-0001 --agent tester
carryctx task start CTX-0001 --agent tester
# {"success":false,"error":{"message":"Cannot transition from InProgress to start."}}
```

`claim` transitions Ready → InProgress; `start` is only allowed from
Ready|Planned (`src/domain/task.rs:157`). The quick-reference in
`carryctx-core` documents claim and start as separate sequential steps.

### D — `handoff create --target <name>` fails FK

```bash
AID=$(sqlite3 .git/carryctx/state.sqlite "SELECT id FROM agents WHERE name='tester'")
carryctx handoff create --target tester --task CTX-0002 --summary s --agent tester
# {"error":{"code":"DATABASE_ERROR","message":"SQLite error: FOREIGN KEY constraint failed"}}
carryctx handoff create --target $AID --task CTX-0002 --summary s --agent tester
# success
```

`src/commands/handoff.rs:140` inserts `target.clone()` verbatim as
`to_agent_id` — the help text promises "agent ULID or role name", but there is
no resolution step. This blocks the primary cross-agent routing workflow.

### E — warning line appended after JSON

```bash
NT=$(carryctx task create --title w --agent tester --json | jq -r .data.id)
carryctx task claim $NT --agent tester --json >/dev/null
carryctx progress todo "open item" --task $NT --agent tester >/dev/null
carryctx task complete $NT --agent tester | jq .       # jq: parse error: Extra data
```

`src/output.rs:112-118` (text mode) does
`text.push_str("\nwarning: ...")` after pretty-printed JSON. `--json` mode is
correct (warnings embedded in the envelope).

## Recommended fixes (as implemented in 0.5.0)

1. **A**: `update_state` writes `ended_at = now` on terminal transitions
   (ended/abandoned/stale); `stats` uses `last_activity_at` as the end bound
   for open sessions; migration backfills `ended_at` from `last_activity_at`
   for terminal sessions.
2. **B**: per-agent checkpoint count via `checkpoints.agent_id`.
3. **C**: `task start` becomes idempotent on InProgress (no-op success).
4. **D**: `handoff create` resolves target by name → ULID → role before insert,
   with a clear error when unresolvable.
5. **E**: text-mode warnings go to stderr.

## Known gaps

- `carryctx handoff show`/`accept`/`close` exercised only partially (create/list
  covered; accept/close blocked by D until fixed).
- `sync push/pull`, `hooks install`, `mcp` server, `graph scan` not exercised
  (require remote config / service).
- Data points above are from a live project; absolute figures differ per repo,
  ratios hold.
