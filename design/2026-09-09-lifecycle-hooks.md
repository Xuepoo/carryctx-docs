# Lifecycle Hooks System Design

**Status:** Draft for review, 2026-09-09. Design only — implements nothing.

**Task:** CTX-0010 (owner: doc-2, commander cmd-001, program 001).

**Scope:** offline-first hook extension boundary. CarryCtx dispatches local
events and runs local commands; transport (`git push`, `scp`, Syncthing,
notifications) belongs to user scripts outside the binary
(see `architecture/state-transport-boundary.md`,
`architecture/zero-network-policy.md`).

**Goal:** separate the two hook layers that are conflated today, define the
trust and execution guardrails that must exist before any repository-provided
script auto-executes, and give wave-2 a minimal first step (thin shim +
composition) that adds no new auto-execution surface.

**Grounding (read 2026-09-09, implementation unchanged by this doc):**

- `carryctx-cli/src/commands/hooks.rs` — the only hook code in Core today:
  two fat shell templates (`POST_COMMIT_HOOK`, `PREPARE_COMMIT_MSG_HOOK`)
  that shell out to `carryctx context --format json | grep | cut`, atomic
  install via `write_atomic`, `.bak` backup under `--force`, jj-colocation
  refusal, `status` reporting `managed_by_carryctx` by `"CarryCtx"` marker.
- `carryctx-cli/src/main.rs` — `Commands::Hooks`, global flags (`--format`,
  `--quiet`, `--dry-run`, `--yes`, `--non-interactive`); no `--no-hooks`
  flag exists today.
- `carryctx-cli/src/application/runtime.rs` — `InvocationContext`; no hook
  fields exist today. There is no `src/application/hooks*` module.
- `carryctx-cli/src/adapter/git.rs` — `isolate_git_env` (hook runners such
  as lefthook set `GIT_DIR`/`GIT_INDEX_FILE`; raw spawns must isolate).
- Workspace research note 001 (lifecycle layer + thin-shim/composition
  sections) is the problem statement this design answers.

---

## 0. Binding constraints

1. **Zero network.** Hook dispatch plus local script execution may enter Core
   (state-transport-boundary §4, first row); the scripts themselves never do.
   No network dependency may enter the binary to support hooks
   (zero-network-policy §3 CI gate stays green).
2. **Local-only effects.** A hook command is a local executable path plus
   arguments. No URL-, host-, or protocol-aware hook configuration.
3. **No new auto-execution before trust exists.** The thin shim (phase 1)
   must not widen what auto-executes; the lifecycle engine (phase 2) ships
   only with the §2 trust model enforced.
4. **Layered architecture.** Dispatch logic lives in `src/application/`
   (new module), never in `src/commands/` handlers and never as SQL or Git
   subprocesses from handlers. Every state-changing dispatch appends its
   audit event in the same transaction (workspace rule).
5. **Public contract.** New flags, subcommands, envelopes, exit codes, and
   config keys are public API: `cli-specification.md`, `configuration.md`,
   and `CHANGELOG.md` move in the same change as any implementation.
6. **`--dry-run` never fires hooks.** Dry runs write nothing and execute
   nothing; hook dispatch is an execution.

---

## 1. Two layers

Today `carryctx hooks` means exactly one thing: **Git hooks integration**
(two files under `.git/hooks/`). The research note asks for a second thing:
**CarryCtx lifecycle hooks** (reactions to `export`, `import`,
`checkpoint`, task transitions, session end). These are different layers
with different owners, triggers, and risk profiles, and the design keeps
them separate:

```text
Git hooks (triggered BY git, owned by the repo's hook system)
  git.post-commit            -> today: fat shell, auto-checkpoint
  git.prepare-commit-msg     -> today: fat shell, prepend task id
  git.pre-push               -> future candidate, not phase 1 or 2

CarryCtx lifecycle hooks (triggered BY carryctx, owned by carryctx config)
  carryctx.pre-export        -> blocking: may veto the export
  carryctx.post-export       -> notify:  e.g. git add/commit/push snapshot branch
  carryctx.pre-import        -> blocking: may veto the import
  carryctx.post-import       -> notify:  e.g. desktop notification, CI kick
  carryctx.post-checkpoint   -> notify:  e.g. backup state.sqlite copy
  carryctx.post-task-transition -> notify: claim/complete/cancel events
  carryctx.post-session-end  -> notify:  e.g. worktree cleanup prompt
```

### 1.1 Event catalog (normative for phase 2)

| Event                           | Fires (application layer, not handler)                                        | Blocking?                                                                   | Payload on stdin (JSON)                                    |
| ------------------------------- | ----------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `carryctx.pre-export`           | after export validation, before any write or audit event                      | **yes** — non-zero exit aborts with `HOOK_VETOED` (exit 3), nothing written | `{event, project_id, export_args, target_path}`            |
| `carryctx.post-export`          | after `project.exported` committed                                            | no                                                                          | `{event, project_id, export_id, manifest_path, counts}`    |
| `carryctx.pre-import`           | after bundle validation, before backup/swap                                   | **yes** — non-zero exit aborts with `HOOK_VETOED` (exit 3), nothing written | `{event, project_id, bundle_manifest, mode}`               |
| `carryctx.post-import`          | after `project.imported` committed                                            | no                                                                          | `{event, project_id, import_id, mode, pruned_worktrees}`   |
| `carryctx.post-checkpoint`      | after checkpoint committed (including auto-checkpoint from `git.post-commit`) | no                                                                          | `{event, project_id, checkpoint_id, task_id, commit_sha?}` |
| `carryctx.post-task-transition` | after claim/complete/cancel committed                                         | no                                                                          | `{event, project_id, task_id, from, to}`                   |
| `carryctx.post-session-end`     | after session end/pause committed                                             | no                                                                          | `{event, project_id, session_id, reason}`                  |

Rules:

- `pre-*` hooks are the only veto points. A vetoed operation reports
  `HOOK_VETOED` with the hook's stderr tail attached; the operation's own
  error precedence is unchanged otherwise (validation errors still fire
  before any hook runs).
- `post-*` hook failure never fails the operation. It is reported per the
  §4 failure policy (warning + audit event), never as an operation error.
- Dispatch order for multiple hooks on one event is config-file order,
  sequential, deterministic. No parallel execution in v1 (keeps failure
  attribution and audit ordering trivial).
- Hook stdin is the only structured input. Relevant context is also
  exported as `CARRYCTX_*` env vars (see §4) for shell-script ergonomics,
  but stdin JSON is normative.
- The Git layer stays trigger-compatible: `git.post-commit` /
  `git.prepare-commit-msg` keep their current observable behavior through
  phase 1 (same checkpoint, same message prefix); only the mechanism
  changes (shim → dispatch → Rust, §5).

---

## 2. Trust model

Running `[[hooks.*]]` commands means auto-executing repository-provided
code — a supply-chain boundary equivalent to Git hooks or `npm scripts`.
Cloning a malicious repository and running CarryCtx must not execute its
scripts without an explicit user decision.

### 2.1 Trust store (XDG state, never the repo)

- Location: OS-appropriate XDG state dir
  (`~/.local/share/carryctx/trusted-hooks.json`; resolved via the existing
  `XdgPaths` adapter, not hardcoded).
- Keyed by **project id** (the ULID in `.carryctx/config.toml`), never by
  path: paths are re-anchorable across machines (ctxpack re-anchor policy),
  ids are stable.
- Minimal shape:

```json
{
  "trusted": {
    "<project-id>": { "trusted": true, "decided_at": "2026-09-09T00:00:00Z" }
  }
}
```

- A repository can never declare itself trusted: nothing under
  `.carryctx/` or version control is consulted for the trust decision.
- Optional hardening (wave-2 decides; not required for v1): also store a
  hash of the hook config section, and re-prompt when the hook commands
  change after trust was granted. At minimum, `hooks status` must show
  when trust was granted so a stale grant is visible.

### 2.2 Commands

```bash
carryctx hooks trust     # trust lifecycle hooks for this project id
carryctx hooks untrust   # revoke (default state: untrusted)
carryctx hooks status    # git-layer install state + lifecycle trust state
```

- `status` (extended, JSON-stable) reports both layers: installed Git
  shims with `managed_by_carryctx`, detected foreign managers (§6),
  lifecycle trust (`trusted: true|false`, `decided_at`), and configured
  lifecycle events. It never executes anything.
- `trust`/`untrust` write only XDG state, never the repo, and work outside
  any Git repository as long as the project id resolves.

### 2.3 First-run and `--no-hooks` escape

- When lifecycle hooks are configured but the project id is untrusted:
  - TTY: print what would execute (event → command list) and prompt
    `Trust hooks for this project? [y/N]`; `N`/empty aborts the operation
    with `HOOKS_UNTRUSTED` (exit 3), executing nothing.
  - Non-TTY / `--non-interactive` / CI: refuse closed with
    `HOOKS_UNTRUSTED` (exit 3). Never prompt where nobody can answer, and
    never silently skip: skipping would make `export → push` pipelines
    silently stop pushing.
- Global escape (new public flag, phase 2):

```bash
carryctx --no-hooks <subcommand> ...
# plus env equivalent, following the existing CARRYCTX_* convention:
CARRYCTX_NO_HOOKS=1 carryctx <subcommand> ...
```

- `--no-hooks` disables all lifecycle dispatch for the invocation and is
  recorded in the audit event (`hooks_skipped: true`) so skipped automation
  is visible, not silent.
- The Git layer keeps working under `--no-hooks` only insofar as Git
  invokes the shim: the shim dispatches through `carryctx`, which sees the
  flag via env propagation only if wave-2 implements env threading through
  the shim. Default: Git-triggered dispatch ignores `--no-hooks` from an
  unrelated interactive shell (it is a different process); suppressing
  auto-checkpoint remains `hooks uninstall`. Wave-2 must document whichever
  semantics it implements — this is an open question (§9, Q3).

---

## 3. Configuration surface (phase 2 sketch, not implemented here)

TOML under the existing config precedence (`config.toml` <
`config.local.toml` < env; exact section placement follows
`configuration.md` conventions at implementation time):

```toml
[hooks]
timeout_secs = 30          # default per-hook timeout; hard cap 300
max_output_bytes = 65536   # captured stdout+stderr per hook; past this, truncated with marker
max_depth = 1              # reentrancy budget, see §4

[[hooks.post_export]]
command = ["./scripts/sync.sh"]   # argv array: no shell interpolation, no PATH search surprises
on_failure = "warn"               # post-*: warn | ignore; pre-*: veto is implicit, see below
```

Validation at the config boundary (fail-closed): `command` must be a
non-empty argv array whose executable exists and is executable at dispatch
time, otherwise the operation refuses before running anything;
`on_failure` accepts only `warn`/`ignore` for `post-*` (a `pre-*` hook's
non-zero exit is always a veto regardless of `on_failure`); unknown keys
follow the existing `config_compat` error/warn behavior.

---

## 4. Execution guardrails

### 4.1 Timeout

- Default 30 s per hook, configurable per `[hooks]` section, hard-capped
  (proposed 300 s; wave-2/post-001 finalizes). Timeout kills the child
  (SIGKILL after SIGTERM grace) and is treated as failure: veto for
  `pre-*`, warning + audit for `post-*`.
- Timeout values come from local config only, never from the bundle or the
  event payload.

### 4.2 Output limits

- Capture at most `max_output_bytes` (default 64 KiB) across stdout+stderr
  per hook. Past the limit the child is allowed to finish but further
  output is discarded, and the record notes `output_truncated: true`.
- Hook stdout is **captured, never forwarded** into CarryCtx's own stdout:
  JSON envelopes stay parseable (the existing `grep`-over-`context --format
json` fragility in `hooks.rs` is exactly what this rule retires).
- On failure/veto, the stderr tail (bounded, e.g. last 2 KiB) is attached
  to the error envelope. Full captured output goes to the audit record,
  never secrets-scanned — hook output is user script output, logged as-is
  with the truncation marker.

### 4.3 Reentrancy guard

The research note's loop (`post-export → script → carryctx export →
post-export → …`) must be structurally impossible:

- Every dispatch sets `CARRYCTX_HOOK_EVENT=<event>` and increments
  `CARRYCTX_HOOK_DEPTH` in the child's environment.
- Any `carryctx` invocation with `CARRYCTX_HOOK_DEPTH >= max_depth`
  (default 1) runs with lifecycle dispatch disabled for its entire
  subtree. The inner command itself succeeds; only its hook emission is
  suppressed, and the suppression is noted in verbose output.
- Depth state is env-only (process-subtree scoped), never persisted:
  concurrent shells and agents each get their own budget, and a crashed
  hook cannot leave a stale lock behind.
- Git-triggered dispatch (`git.post-commit` → auto-checkpoint → would that
  fire `post-checkpoint` → script → commit → …?) terminates by the same
  rule: the auto-checkpoint runs at depth ≥ 1, so its `post-checkpoint`
  does not re-dispatch. The chain is length-bounded by construction.

### 4.4 Failure policy

| Hook kind                                 | Exit 0         | Non-zero exit                                                        | Timeout          |
| ----------------------------------------- | -------------- | -------------------------------------------------------------------- | ---------------- |
| `pre-*`                                   | proceed        | veto: `HOOK_VETOED`, exit 3, nothing written                         | veto (same)      |
| `post-*` (`on_failure = "warn"`, default) | record success | operation still succeeds; stderr warning + `hook.failed` audit event | same as non-zero |
| `post-*` (`on_failure = "ignore"`)        | record success | operation succeeds; audit event only                                 | audit event only |

- Error codes are new public API: `HOOK_VETOED` (operation refused by a
  `pre-*` hook), `HOOKS_UNTRUSTED` (configured but untrusted, §2.3),
  `HOOK_FAILED` (diagnostic code inside the `hook.failed` audit payload,
  never the operation's exit code for `post-*`).
- Every dispatch — fired, skipped (`--no-hooks`, `--dry-run`, depth cap),
  vetoed, failed — appends an audit event in the same transaction as the
  operation it annotates, so `event list` shows the automation history.

---

## 5. Thin-shim direction (phase 1, wave-2 implements)

Long-term, Git hook files carry no logic. They are three-line shims:

```sh
#!/bin/sh
exec carryctx hooks dispatch git.post-commit "$@"
```

and all behavior (resolve active task via library call, create checkpoint,
append event, prefix commit message) lives in Rust behind
`carryctx hooks dispatch <event>`:

```text
Git
 ↓
.git/hooks/post-commit
 ↓
carryctx hooks dispatch git.post-commit
 ↓
Rust: resolve task → create checkpoint → append event
```

Why (from the current code):

- The `grep -o '"display_id":…'` pipeline in `hooks.rs:62,80` already broke
  once (camelCase vs snake_case; regression test at `hooks.rs:391` guards
  the exact string). Any envelope change breaks every installed hook
  silently. Dispatch moves parsing into Rust, where schema changes are
  compile-time errors.
- `--quiet` suppressing the JSON the grep needs (same regression test) is
  a whole class of shell-composition bug that disappears when the hook is
  a library call.
- Atomic install (`write_atomic`, `hooks.rs:239`) and loud chmod failure
  (`hooks.rs:255`) are retained; only the file _content_ shrinks to the
  shim.

Naming note: the research sketch uses singular `carryctx hook dispatch`.
The existing top-level command is plural (`Commands::Hooks`,
`carryctx hooks`). This design specifies plural
`carryctx hooks dispatch <event>` to avoid a second top-level name; a
singular hidden alias is acceptable if wave-2 finds it cheap, but the
plural form is the documented contract.

Migration: `hooks status` distinguishes legacy fat hooks (contain
`carryctx context` pipeline) from shims (contain `hooks dispatch`) and
reports `shim: true|false`; `hooks install` on a legacy hook reinstalls it
as a shim (same `--force`/`.bak` rules as today, plus §6 composition).
Behavioral parity (same checkpoint note format, same `[TASK]` prefix
rules, same merge/squash exemption) is a wave-2 acceptance criterion.

---

## 6. Composition policy (lefthook / husky / user hooks)

`--force` renaming an existing hook to `.bak` and overwriting it is
acceptable for two CarryCtx-owned files and unacceptable as an ecosystem
rule: lefthook, husky, and hand-written hooks all compete for the same
`.git/hooks/*` paths, and silent overwrite destroys user automation.

Policy (phase 1, wave-2 implements):

1. **Detect before touching.** On `install`, classify each target path:
   absent | CarryCtx-managed (contains `CarryCtx` marker) | foreign
   (exists, no marker). For foreign hooks, additionally detect known
   managers: `lefthook.yml`/`lefthook.yaml` at repo root, `.husky/`
   directory, or a hook body that delegates to one.
2. **Never silently overwrite.** Foreign hook + no `--force` → refuse with
   `HOOK_EXISTS` (today's behavior, kept). Foreign hook + `--force` →
   today's `.bak` rename stays, but the success output must warn that a
   foreign hook was displaced and name the backup path.
3. **Compose, don't displace (new `--compose` mode).** When a foreign hook
   exists, `install --compose` chains instead of replacing: the installed
   file runs the previous content first (inlined with a delimiter comment,
   original preserved verbatim below the marker), then `exec`s/​calls the
   CarryCtx shim. Order (foreign-first) keeps existing CI/status checks
   gating before CarryCtx side effects. If the foreign hook is a delegating
   stub to lefthook/husky (recognized patterns documented at implementation
   time), prefer registering CarryCtx as a lefthook job / husky script when
   that manager's config supports it, and fall back to file chaining with a
   clear message. `uninstall` in compose mode removes only the CarryCtx
   segment and restores the foreign body byte-for-byte; `--restore` from
   `.bak` remains for the `--force` path.
4. **Report.** `hooks status` gains `foreign_manager: none|lefthook|husky|
custom` per hook path so `doctor` can surface fragile compositions.
5. **Git subprocess hygiene.** Anything dispatch spawns inherits the
   `isolate_git_env` contract (`adapter/git.rs`): hook-runner-provided
   `GIT_DIR`/`GIT_INDEX_FILE` must not redirect CarryCtx's internal Git
   calls at a different repository.

---

## 7. Observability and envelopes

- `hooks dispatch` is an internal plumbing command: human-readable output
  suppressed by default (hook context forbids noise on stdout); `--json`
  emits the standard envelope (`hooks.dispatch`, exit codes per §4.4).
- Hook audit events (`hook.fired`, `hook.failed`, `hook.vetoed`,
  `hook.skipped` with `reason`) carry: event name, hook index, command
  argv[0] basename (never full user paths in TTY-facing output),
  exit/timeout, `output_truncated`, depth. Exact field names are fixed at
  implementation time in `cli-specification.md`.
- `doctor` gains checks: orphaned shims (binary renamed/missing),
  legacy fat hooks (suggest reinstall), untrusted-but-configured projects,
  foreign-manager compositions.

---

## 8. Non-goals (explicit)

- **No network in Core**, including from hooks: no webhook notifier, no
  `POST` on export, no polling registry. Notifications are user scripts
  invoking their own tools.
- **No full lifecycle engine in 001.** Phase 2 (§1–§4, trust store,
  `pre/post-*` dispatch) is post-001 work. Phase 1 is shim + composition
  only.
- **No `git.pre-push` management** in either phase (listed only as a future
  candidate; push-time behavior interacts with transport ownership).
- **No parallel hook execution**, no hook dependency DAG, no conditional
  (`only_if`) expressions in v1. Config-file order, sequential.
- **No Windows PowerShell shim** in phase 1 (`/bin/sh` only, as today);
  platform shims beyond that are a separate task with its own testing.
- **No `watch`/polling mode.** The jj-compatibility plan evaluated and
  deferred it; this design does not revive it. Under jj colocation the
  install refusal stays.
- **No secret injection** (no env-secret store, no redaction engine):
  hook output is logged as-is with truncation markers.
- **No second trust root.** No per-repo `trust: true` flag, no
  `config.local.toml` trust override — trust lives only in XDG state.

---

## 9. Phased rollout

**Phase 1 — thin shim + composition (wave-2, inside program 001).**
`hooks dispatch git.post-commit | git.prepare-commit-msg`, shim content
swap, legacy/shim detection in `status`, `--compose` + foreign-manager
detection, `isolate_git_env` through dispatch. No new auto-execution: the
shim fires exactly where the fat hook fired. Acceptance: byte-identical
observable behavior (checkpoint note, message prefix, merge/squash
exemption), regression tests kept green, temp-repo integration tests for
install/status/uninstall × {absent, managed, foreign, lefthook-stub},
plus the existing jj-colocation refusal untouched.

**Phase 2 — lifecycle engine (post-001).** Event catalog (§1.1), trust
store + `trust`/`untrust`/`status` (§2), `--no-hooks` + env (§2.3), config
surface (§3), timeout/output/reentrancy/failure policy (§4), `doctor`
checks (§7). Ships only as a whole: dispatch without trust does not land.
Acceptance: trust matrix (TTY prompt / non-TTY refuse / trusted runs /
revoked refuses), veto atomicity (vetoed export/import writes nothing,
provable via pre/post DB hash), recursion test (self-invoking hook
terminates by depth cap), timeout test (sleep-hook vetoes/fails on time),
output-limit test (yes-generator truncated with marker), composition
round-trip (compose → status → uninstall restores foreign byte-for-byte).

**Explicitly deferred:** merge-adjacent hooks (anything gating a future
`--mode merge`), `pre-push`, watch mode, parallel execution, PowerShell
shims.

---

## 10. Test plan (for implementers, not executed here)

- Unit: event catalog deserialization, veto-vs-warn matrix, env depth
  parsing (`CARRYCTX_HOOK_DEPTH` garbage → treat as maxed-out, fail
  closed), output truncation markers, trust-store CRUD against temp XDG.
- Integration (disposable Git repos under tempdir, existing harness
  style): shim parity, legacy→shim migration, compose/uninstall
  round-trip with lefthook-stub and husky-stub fixtures, veto atomicity
  via DB hash, recursion termination, timeout and output-limit fixtures.
- Contract: new envelopes/exit codes added to the CLI stability table;
  `cargo fmt --check`, Clippy `-D warnings`, `cargo test`,
  markdownlint on touched docs.

---

## Open questions for wave-2 implementers

1. `hooks dispatch` visibility: public subcommand (documented, stabilized)
   or hidden plumbing (`#[arg(hide = true)]`)? This design assumes public
   for debuggability; hiding is a one-line change if reviewers prefer.
2. Trust-store config-hash re-prompt (§2.1): worth the UX complexity in v1,
   or is `decided_at` visibility sufficient until real misuse is observed?
3. `--no-hooks` through the Git shim (§2.3): thread env from the committing
   shell, or document that Git-triggered dispatch ignores it? Either is
   defensible; it must be documented, not accidental.
4. `max_depth` default of 1 vs 2: depth 1 forbids even intentional
   two-stage chains (`post-export` → helper `carryctx checkpoint` that
   itself wants `post-checkpoint`); is there a real chain that needs 2?
5. Compose ordering (§6.3): foreign-first is specified — does any existing
   CarryCtx user depend on CarryCtx-first (prefix in message before
   lint)? Call it out in review if so.
