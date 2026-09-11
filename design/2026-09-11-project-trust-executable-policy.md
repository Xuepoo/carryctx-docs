# Project Trust for Executable Policy

**Status:** Implemented for review, 2026-09-11 (CTX-0100). The gate, the
user-local registry, the `[security]` global key, and the `carryctx trust`
command ship in `carryctx-cli`; wiring an executor through the gate remains a
separate task (§1, §10). Superseded decision records from the earlier
lifecycle-hooks draft are noted in §10.

**Task:** CTX-0100 (`carryctx-cli`; owner `ctx-0100-impl`). Supersedes and
generalizes the trust-store sketch in `design/2026-09-09-lifecycle-hooks.md`
§2 (that document's `trusted-hooks.json` / `carryctx hooks trust` naming is
superseded; its execution guardrails §4 remain the consumer contract).

**Goal:** make the boundary between safe declarative `.carryctx` data and
executable repository-provided policy explicit, and deny external policy by
default until a local user grants and the global security gate allows it.
Nothing under `.carryctx/` or version control may ever grant trust.

---

## 1. Problem and scope

`.carryctx/config.toml` is version-controlled and repo-controlled. It mixes:

- **Declarative data** — ids, names, task prefixes, branch templates, output
  preferences, cleanup policies. Reading these cannot execute anything.
- **Executable policy** — today `[verification].commands` (arguably a shell
  command list), and in the future lifecycle-hook commands
  (`design/2026-09-09-lifecycle-hooks.md` §3) and automation shell/webhook
  actions (`CTX-0095`–`CTX-0097`, deferred).

Cloning an untrusted repository and running `carryctx` must never execute its
command list because the repository said so. This document defines the trust
model that gates every external (repository-provided) execution surface.
Built-in actions shipped inside the binary are trusted by construction.

**In scope:** the trust decision, its storage, its CLI, the global gate, the
fail-closed rules, and the contract future executors must call.

**Out of scope:** the executor itself (verification runner, lifecycle engine,
automation engine). This task ships the gate and its observable status; it
does not add a new way to execute commands. A separate task must wire each
executor through the gate before that executor can run at all.

## 2. Declarative vs executable split

| Class                | Examples                                                                                              | Trust needed                          |
| -------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Declarative          | `project.*`, `git.*`, `session.*`, `task.*`, `context.*`, `output.*`, `worktree.cleanup.*`            | none                                  |
| Executable, external | `verification.commands`, future `[[hooks.*]]` commands, future automation `shell` / `webhook` actions | global gate **and** per-project trust |
| Executable, built-in | checkpoint capture, worktree cleanup, task transitions, event append                                  | none (shipped code)                   |

"External" means _the argv/URL originates from the repository's
`.carryctx/` files_. A value that originates from the user's environment or
global config is not external.

The set of external surfaces is a closed list in code
(`application::trust::collect_external_policy`). Adding a new executable
surface requires adding it to that function **and** to this document; a
surface that is not listed cannot be gated and therefore must not execute.

## 3. Threat model

- **Adversary:** a malicious or compromised repository, or any contributor
  able to edit `.carryctx/config.toml` in a PR.
- **Attack:** the user clones / pulls the repository and runs a `carryctx`
  command; the repository's command list executes.
- **Assets:** the user's shell, filesystem, credentials, and network.
- **Security objectives:**
  1. Repo-controlled data never grants trust (`no second trust root`).
  2. External policy is denied by default on a fresh machine.
  3. A global user-controlled gate can hard-disable external policy even
     after per-project trust exists.
  4. Trust is bound to the _policy content_, so a repository cannot copy a
     trusted project's id and then swap in different commands.
  5. Malformed, unreadable, or too-open trust state fails closed (untrusted).
  6. No secret (token, env value) is written to the registry or to the audit
     event payload.
- **Non-objectives:** defending against a malicious local user or root;
  OS-level sandboxing; network egress filtering (see
  `architecture/zero-network-policy.md`).

## 4. Trust decision

A project may run an **external** action iff all of:

1. `security.allow_project_commands == true` in the **global** config
   (default `false`), and
2. the project id is present in the trust registry with `trusted: true`, and
3. the stored `policy_fingerprint` equals the fingerprint of the currently
   declared external policy.

A **built-in** action is always allowed and never consults the registry.

Rationale for the two-key model: a leaked, copied, or accidentally
world-readable trust file alone cannot enable execution (key 1 still off);
and a globally-enabled user who has never trusted the project is still
protected (key 2 off). Both keys are user/global controlled; neither can be
set by the repository.

### 4.1 Policy fingerprint

`policy_fingerprint = sha256(serde_json::to_string(external_policy))`, where
`external_policy` is the ordered list of external command argv arrays declared
by the project. Order is significant. The fingerprint is stored at grant time
and re-computed at evaluation time. A mismatch (commands added, removed, or
reordered after trust was granted) is treated as **untrusted** and requires an
explicit re-grant. This is the anti-id-spoofing control required by
objective 4.

## 5. Trust store

- **Location:** `${XDG_STATE_HOME:-$HOME/.local/state}/carryctx/trusted-projects.json`.
  It lives in the _state_ directory (configuration.md §2.2), not the data
  directory, because it is user-local persistent state. Resolved through the
  existing `XdgPaths`, never hardcoded.
- **Format** (`schema_version` is mandatory; unknown versions fail closed):

```json
{
  "schema_version": 1,
  "trusted": {
    "01M0JJGJ9XV1RTWWK65JWEX6KN": {
      "trusted": true,
      "project_name": "carryctx-cli",
      "decided_at": "2026-09-11T03:46:33Z",
      "decided_by": "ctx-0100-impl",
      "policy_fingerprint": "sha256:..."
    }
  }
}
```

- **Key:** the immutable project id from `.carryctx/config.toml`
  (`project.id`). Never a path, never `project.name`.
- **Permissions:** the parent directory is created `0700`; the file is
  created and atomically replaced `0600`. On load, if the file mode has any
  group/other bit set (`mode & 0o077 != 0`), the registry is treated as
  **untrusted** and flagged `registry_insecure`; CarryCtx never silently
  repairs and then trusts a file it found too open. On non-Unix platforms the
  permission check is skipped (documented limitation).
- **Writes** are atomic: write a temp file with `0600` in the same directory,
  `fsync`, `rename`. A failed write never leaves a half-written registry.
- **Write safety:** `grant` refuses to write when an existing file is
  malformed, unreadable, insecure, or an unknown schema version
  (`CONFIGURATION_ERROR`); it never overwrites trust state it cannot safely
  own. `revoke` on such a file is a no-op with a warning — the fail-closed
  load already treats every project as untrusted.
- **Repo isolation:** nothing under `.carryctx/` is ever read to decide
  trust. `project.name` is stored for human display only and never compared.

## 6. CLI surface

```text
carryctx trust grant [--yes]
carryctx trust revoke [--yes]
carryctx trust list
carryctx trust status
```

- `grant` writes a registry entry for the resolved project id, with the
  current policy fingerprint. It requires `--yes` (mirroring
  `import --mode replace`); without it, and regardless of TTY, it refuses
  with `INVALID_ARGUMENTS` (exit 2). This is the non-interactive contract:
  a human or CI must pass an explicit confirmation flag. `grant` requires an
  identity (`--agent`/`CARRYCTX_AGENT`) and project state for the audit event.
- `revoke` removes the entry (idempotent). Revocation is fail-safe, so it
  does not require `--yes`.
- `list` prints the trusted project ids and decision metadata; it never
  consults the current project's policy and never executes anything.
- `status` reports the effective verdict for the current project using the
  frozen snake_case JSON convention (`cli-specification.md` §27.0):
  `project_id`, `project_name`, `trusted`, `decided_at`, `decided_by`,
  `global_allow_project_commands`, `external_policy_present`,
  `external_policy_fingerprint`, `command_count`, `effective` (`allowed` |
  `blocked` | `not_applicable`), `reason`, `registry_state`, and
  `registry_path`. It never executes anything.

`status.effective` is `blocked` for an untrusted project that declares
external policy, `not_applicable` when no external policy is declared, and
`allowed` only when all three §4 conditions hold. `status.reason` is one of
`allowed`, `no_external_policy`, `global_disabled`, `not_trusted`,
`policy_changed`; `status.registry_state` is one of `absent`, `ok`,
`malformed`, `unreadable`, `insecure`.

### 6.1 Non-interactive mode

`--non-interactive` and JSON/CI invocations never prompt. `grant` without
`--yes` fails closed. `revoke`, `list`, and `status` are safe and always
allowed. `--dry-run` prints what would change without writing the registry.

## 7. Fail-closed matrix

| Registry condition               | Loaded registry | `registry_state` | Effect on `effective` |
| -------------------------------- | --------------- | ---------------- | --------------------- |
| file absent                      | empty           | `absent`         | untrusted             |
| JSON malformed                   | empty           | `malformed`      | untrusted             |
| `schema_version` unknown/missing | empty           | `malformed`      | untrusted             |
| unreadable (permissions/IO)      | empty           | `unreadable`     | untrusted             |
| mode has group/other bits        | empty           | `insecure`       | untrusted             |
| valid registry                   | entries         | `ok`             | entry consulted       |

An untrusted/empty registry produces `effective = blocked` once the global
gate is on: `reason = not_trusted` normally, or `policy_changed` when a stale
entry exists for a different fingerprint. With the gate off the reason is
`global_disabled`. No declared external policy produces `not_applicable` /
`no_external_policy`. All three §4 conditions together produce `allowed`.

Every "blocked" case is non-fatal for unrelated commands: CarryCtx warns and
treats the project as untrusted rather than refusing to start. Execution
commands (future) must refuse.

## 8. Audit events

Grant and revoke append an event to the **project** event log in the same
invocation:

- `project.trust_granted` — payload
  `{ "policy_fingerprint": "sha256:...", "command_count": N }`.
- `project.trust_revoked` — payload `{}`.

The event carries `actor_agent_id` and `session_id` from the invocation. It
never contains command text, environment values, or paths (command text is
already repo-visible; omitting it removes any chance of leaking a secret
embedded in a command string).

Ordering: the registry write is authoritative and happens first; the event is
appended best-effort afterwards. If the event append fails, the trust change
still stands and a warning is emitted — trust is never rolled back to satisfy
an observability write, and an event is never written for a trust change that
did not persist. This is the one place where the "same transaction" rule is
relaxed because the two stores are different (XDG state vs project SQLite);
the relaxation is one-directional (event may be missing, never fabricated).

## 9. Configuration contract

Global config only (`~/.config/carryctx/config.toml`), plus environment:

```toml
[security]
allow_project_commands = false   # default
```

Environment override: `CARRYCTX_ALLOW_PROJECT_COMMANDS=true|false`.

A `[security]` table in the project `.carryctx/config.toml` is **ignored**
with a warning; the merged config always takes the security section from the
global file. This prevents a repository from loosening or tightening the
user's global security posture.

## 10. Relationship to lifecycle hooks (CTX-0010)

`design/2026-09-09-lifecycle-hooks.md` §2 specified a per-surface trust store
(`trusted-hooks.json`) and `carryctx hooks trust`. This document supersedes
that storage and command naming:

- one registry for all external policy (`trusted-projects.json`), and
- one top-level `carryctx trust` command.

The lifecycle-hooks execution guardrails (timeout, output cap, reentrancy,
failure policy) are unchanged and remain the consumer contract. When the
lifecycle executor lands, `collect_external_policy` must include its
`[[hooks.*]]` commands, and each dispatch must call the §4 gate before
spawning anything. The §2.3 `HOOKS_UNTRUSTED` exit code (3) is superseded by
the unified `TRUST_DENIED` code (exit 9).

## 11. Non-goals

- No network access, no registry polling, no remote trust root.
- No per-repository `trust = true` flag and no `config.local.toml` override:
  trust lives only in XDG state.
- No interactive grant prompt in v1 (explicit `--yes` only).
- No signature/attestation of the policy, no provenance beyond the
  fingerprint.
- No automatic re-trust when a policy changes.

## 12. Decisions and open questions

Resolved by this revision:

- **Registry location:** STATE is the single owner.
  `$XDG_STATE_HOME/carryctx/trusted-projects.json` is documented in
  `configuration.md` §2.2; the lifecycle-hooks draft's DATA location is
  superseded (§10).
- **Environment override:** `CARRYCTX_ALLOW_PROJECT_COMMANDS` (special
  environment variable, `cli-specification.md` §9). Only explicit truthy
  values enable the gate; anything else keeps the deny-by-default posture.
- **JSON field naming:** `trust` projections are snake_case per
  `cli-specification.md` §27.0.

Remaining open:

1. **Config-hash UX:** command changes require a manual re-grant. Is a
   `trust diff`/re-grant prompt worth adding once an executor exists?
2. **Grant scope:** `grant` currently requires being inside the project. A
   path/remote-URL keyed grant (for out-of-tree approval) is deferred.
3. **Windows permissions:** `0600` enforcement is Unix-only; document the
   residual risk or add an ACL check in a follow-up.
4. **Fingerprint input:** `[verification].commands` is the only surface today.
   Confirm the canonical serialization before a second surface is added.

## 13. Test evidence

Implemented and passing (see the CTX-0100 PR):

- `carryctx-core::domain::trust` unit tests (8): fingerprint stability and
  order sensitivity; the §7 verdict matrix; `authorize` returns `TRUST_DENIED`
  (exit 9) for blocked external actions; built-in surface always authorized;
  registry closed-shape/schema-version parsing.
- `carryctx-cli::adapter::trust_store` unit tests (5): absent is fail-closed;
  save round-trips and creates `0600` file / `0700` directory on Unix;
  malformed and unknown-`schema_version` are fail-closed; group-readable file
  is `insecure`.
- `carryctx-cli::adapter::config` unit tests (3): global `[security]` wins
  over a project table; missing global defaults to deny; env override only
  enables on explicit truthy values.
- `carryctx-cli::application::trust` unit tests (5): status/list projections.
- Integration `tests/trust_test.rs` (12, disposable git repo + isolated XDG):
  - untrusted project with `verification.commands` → `effective = blocked`,
    no registry created, unrelated built-in commands still succeed;
  - `grant` without `--yes` → `INVALID_ARGUMENTS` (exit 2), registry absent;
  - `grant --yes` → registry exists, file `0600` / dir `0700`, and
    `effective = allowed` once the global gate is on;
  - global gate off → blocked (`global_disabled`) even when trusted;
  - changing `verification.commands` after grant → blocked
    (`policy_changed`);
  - `revoke` → blocked again and idempotent;
  - malformed registry → blocked (`malformed`), `grant` refuses with
    `CONFIGURATION_ERROR` and never rewrites the file; group-readable registry
    → blocked (`insecure`);
  - `--dry-run` writes nothing; `list` works outside a repository; a
    project-level `[security] = true` is ignored.
  - audit: `event list` shows `project.trust_granted` / `_revoked` and no
    payload contains command text.

Contract: commands, error code, exit-code alias, and snake_case JSON are
documented in `cli-specification.md` §26.1/§28; `configuration.md` §2.2/§3.6/§9
documents the registry path and global key. Executor integration (a command
that actually spawns repository-provided argv through `authorize`) remains a
separate follow-up task by design (§1).
