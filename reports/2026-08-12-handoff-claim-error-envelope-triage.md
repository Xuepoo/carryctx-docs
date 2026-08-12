# Bug Triage Report — handoff claim-task & not-found error envelopes (2026-08-12)

## Tool versions

- `carryctx 0.5.5` (tag `v0.5.5`) — bugs measured
- `carryctx 0.5.6` (built from `main` @ `93f78b5`, tag `v0.5.6`) — fixes verified
- Test project: disposable Git repo under `/tmp/opencode/` (init with `--task-prefix CTX`, agents `tester`/`acceptor`/`target` registered)

## Summary

Both open issues from the 0.5.5 audit were confirmed and fixed in 0.5.6
(PR [#78](https://github.com/Xuepoo/carryctx/pull/78)).

| #   | Severity | Command                                             | Symptom                                                                       | Root cause                                                                            |
| --- | -------- | --------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| 75  | low      | `handoff accept --claim-task`                       | Flag accepted but no-op: task never claimed, owner stayed `null`              | `claim_task` destructured as `claim_task: _` and never wired to the application layer |
| 76  | medium   | `handoff show/accept/reject/close`, `progress show` | Not-found printed nothing and exited without the standard JSON error envelope | `ok_or(ExitCode::ResourceNotFound)?` bypassed `render_and_print_entity`'s error path  |

## Repro & evidence

### 75 — `handoff accept --claim-task` never claimed the task

```bash
$ CARRYCTX_AGENT=tester carryctx task create --title "Claim probe" --json
{ ..., "display_id": "CTX-0001", "owner_agent_id": null, ... }
$ CARRYCTX_AGENT=tester carryctx handoff create --target target --task CTX-0001 --summary "claim probe" --json
$ CARRYCTX_AGENT=acceptor carryctx handoff accept HO-0001 --claim-task --json
{ ..., "success": true, "data": { "status": "open", ... } }   # 0.5.5: handoff accepted, task untouched
$ CARRYCTX_AGENT=tester carryctx task show CTX-0001 --json
{ ..., "status": "ready", "owner_agent_id": null }            # 0.5.5: nobody owns it
```

Fixed by resolving the accepting agent and calling the application-layer
`claim_task` inside the accept transaction. Verified on 0.5.6:

```bash
$ CARRYCTX_AGENT=acceptor carryctx handoff accept HO-0001 --claim-task --json
accept success: True | handoff status: open
$ CARRYCTX_AGENT=tester carryctx task show CTX-0001 --json
task status: in_progress | owner: 01KZV7PCR7Z10TG7Q41A613GYY   # owned by "acceptor"
```

If the task cannot be claimed (already owned, wrong status, incomplete
dependencies), the whole accept now fails with a standard error envelope and
the transaction rolls back — the handoff stays pending instead of silently
dropping the flag. Regression test: `tests/handoff_test.rs`
`test_handoff_accept_claim_task_claims_the_task`.

### 76 — not-found skipped the standard error envelope

```bash
$ carryctx handoff show HO-9999 --json; echo $?
# 0.5.5: nothing on stdout or stderr, exit 1 (bare ExitCode::ResourceNotFound)
```

Fixed by routing `CarryCtxError::resource_not_found` through
`render_and_print_entity`. Verified on 0.5.6:

```bash
$ carryctx handoff show HO-9999 --json > stdout.txt 2> stderr.txt; echo $?
7
$ wc -c < stdout.txt
0
$ jq '{success, command, code: .error.code}' stderr.txt
{ "success": false, "command": "handoff.show", "code": "RESOURCE_NOT_FOUND" }
$ carryctx progress show PG-9999 --json 2>&1 >/dev/null | jq -c '{success, command, code: .error.code}'
{ "success": false, "command": "progress.show", "code": "RESOURCE_NOT_FOUND" }
```

stdout stays empty, the envelope lands on stderr, and the exit code is the
standard 7 (ResourceNotFound) — identical contract to `task show`. Regression
tests: `tests/handoff_test.rs` (`test_handoff_show_missing_returns_standard_error_envelope`,
`test_handoff_accept_missing_returns_standard_error_envelope`) and
`tests/progress_test.rs` (`test_progress_show_missing_returns_standard_error_envelope`).

## Verification

- `cargo test` — 39 lib + 21 integration (incl. 4 new regression tests) all pass
- `cargo fmt --check`, `cargo clippy --workspace -- -D warnings` — clean
- GitHub Actions on PR #78: Analyze (rust), CodeQL, Quality, Test, Actionlint all pass
- Release `v0.5.6`: GitHub Release (13 assets), crates.io `0.5.6`, Homebrew, Scoop, npm — all published; AUR git repo updated but the AUR web/package database is still not processing uploads since the 2026-08-01 upstream outage (page 404, RPC still reports `0.4.4-1`) — known outage, no action taken
