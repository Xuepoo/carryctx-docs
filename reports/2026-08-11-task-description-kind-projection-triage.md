# Bug Triage Report — task description, depend kind, compact projection (2026-08-11)

## Tool versions

- `carryctx 0.5.3` (installed via cargo, `2026-08-10` release) — bugs measured
- `carryctx 0.5.4` (built from `main` @ `36d7b1f`, tag `v0.5.4`) — fixes verified
- Test projects: `vectojs/vectojs` (real dogfooding workload, 320+ tasks) and the carryctx-cli workspace dogfood project

## Summary

All three open issues from the 0.5.3 audit were confirmed and fixed in 0.5.4
(PR [#72](https://github.com/Xuepoo/carryctx/pull/72)).

| # | Severity | Command | Symptom | Root cause |
| --- | -------- | ------- | ------- | ---------- |
| 70 | high | `task create` / `task edit` | `--description` accepted then stored `NULL`; no CLI path could ever set it | `description` destructured from CLI args and never forwarded; `task edit` had no `--description` flag; repo `edit` never wrote the column |
| 69 | high | `task depend --kind` | Invalid kind exited 2 with no output — no message, no JSON envelope; help advertised invalid values (`blocks`, `relates_to`) | `.map_err(\|e\| e.exit_code)?` discarded the constructed error before rendering; help text wrong |
| 68 | medium | `task show` / `task list` | Compact text ignored `--fields`/`[output.fields]`: `depends_on`/`blocks` unreachable in text, dropped fields printed empty brackets/padded columns | `task_summary`/`tasks_summary` render a hardcoded template regardless of the projection applied to the value |

## Repro & evidence

### 70 — description parsed then discarded

```bash
$ carryctx task create --agent opencode --title "probe" --description "probe body" --json \
    | jq -r '.data | {display_id, description}'
{ "display_id": "CTX-0322", "description": null }     # 0.5.3
$ carryctx task edit CTX-0321 --agent opencode --description "x"
error: unexpected argument '--description' found
```

`src/commands/task.rs` destructured `description: _` and `create_task` never
received it; `NewTask`/SQL already had the column. Fixed by threading
`description` through `create_task` → `NewTask` and adding `--description` to
`task edit` (`edit_task` → repo `edit` now writes the column; the
`task.edited` audit payload carries before/after).

Verified on 0.5.4 in the dogfood project: `task edit CTX-0024 --description …`
round-trips through the DB, `task create --description "…"` returns
`"description": "…"` in the envelope and on re-read.

### 69 — parse error constructed then thrown away

```bash
$ carryctx task depend CTX-0320 --on CTX-0321 --agent opencode --kind blocks
$ echo $?   # 2, no output, no edge     (0.5.3)
```

`parse_dependency_kind` builds a proper `INVALID_ARGUMENTS` error, but the
call site kept only `e.exit_code`. The same pattern existed for
`--status`/`--priority` parses in task create/list/edit. All four sites now
render through `render_and_print_entity`: text mode prints
`Unknown dependency kind: blocks` (JSON envelope on stderr), `--json` emits
the standard `INVALID_ARGUMENTS` envelope, exit 2; stdout stays clean for
`| jq` consumers. The `--kind` help text now documents
`strong or informational (default: strong)`; the `info` alias still works.

### 68 — projection ignored in compact text

```bash
$ carryctx task show CTX-0320 --fields display_id,status,title,depends_on,blocks   # 0.5.3
CTX-0320 [in_progress] fix(text): per-character positioned carriers… — owner 01KY7HA5
$ carryctx task show CTX-0320 --fields display_id                                  # 0.5.3
CTX-0320 []              # empty brackets for projected-out fields
```

`render_entity` projects the value then hands it to a template that reads
four fixed keys, so projection could only remove, never add. `compact_text`
now receives the projection: `task_summary`/`tasks_summary` render only the
present template keys and append projected-but-unrendered fields as
`label: value` (dependency edges as their `display_id` list):

```bash
$ carryctx task show CTX-0027 --fields display_id,status,title,depends_on,blocks   # 0.5.4
CTX-0027 [planned] 0.5.4 dogfood create with description — needs: CTX-0026
$ carryctx task show CTX-0027 --fields display_id                                  # 0.5.4
CTX-0027
```

Default (unprojected) `task show`/`task list` output is byte-identical to
0.5.3. In `vectojs` the real dependency edge `CTX-0321 → blocks: CTX-0320`
became visible for the first time in text mode.

## Regression tests

10 new integration tests in `tests/output_test.rs`:

- description: create persists + round-trip re-read; edit sets description
- depend kind: text-mode error message; `--json` envelope + no edge created;
  valid `--kind strong` still succeeds; `--status bogus` renders too
- projection: `depends_on` shown in compact text via `--fields` and via
  `[output.fields]` config; `--fields display_id` yields a clean token (no
  `[]`); `task list --fields display_id` has no padded empty columns

## Quality gates (0.5.4)

- `cargo fmt --check` clean
- `RUSTFLAGS="-D warnings" cargo clippy --workspace -- -D warnings` clean
- `cargo test` — 39 lib tests + all integration suites pass
- `cargo build --release` succeeds; CI (Quality/Test/CodeQL/Actionlint) green on PR #72

## Release status (v0.5.4)

Published: crates.io, npm (`carryctx` root + 5 platform packages), GitHub
Release (6 binaries + deb/rpm/apk/pkg.tar.zst), Homebrew, Scoop.
Publish AUR job failed twice because `aur.archlinux.org` was down for
maintenance ("The AUR is down due to maintenance. We will be back soon.");
retry once the outage clears.
