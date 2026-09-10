# State–Transport Boundary

**Document Path:** `carryctx-docs/architecture/state-transport-boundary.md`
**Status:** Accepted (2026-09-09). Complements `architecture/zero-network-policy.md`
and retires the remote-sync roadmap items named in §6.

## 1. Decision

> **CarryCtx owns state semantics. External tools own transport.**

CarryCtx serializes, validates, stores, and refuses to corrupt project state.
Moving bytes between machines — `git push`, `scp`, Syncthing, NAS replication,
`tar` pipes — is the user's job, done with tools that already do it well.
No network code enters the `carryctx` binary.

## 2. Context

By 0.8.2 the layering had already emerged in the implementation:

```text
SQLite (`<git-common-dir>/carryctx/state.sqlite`)  = persistence
ctxpack dir (`manifest.json` + JSONL)              = interchange
Git / SSH / NAS / Syncthing / rclone               = transport (user-chosen)
```

The design record (`design/2026-09-09-ctxpack-export-import.md`) made
persistence-vs-interchange explicit, and the zero-network policy already banned
network code from Core. What was missing was a standing test for the next
proposal: when someone asks "should this feature go into Core?", this document
is the answer. It also retires planning text that predates the invariant:
`ROADMAP.md` still staged `v0.1`/`v0.2` and listed remote-sync adapters and
GitHub Issue/PR synchronization, and `TODO.md` still carried pre-Core
graph/team phases as if they were greenfield work.

## 3. What belongs in Core

A capability belongs in the `carryctx` binary only if **all five** hold.
Otherwise it belongs in an external tool, a skill preset, the website docs,
or a separate utility.

1. **Local-only inputs and effects.** It reads and writes local state:
   SQLite, local files, local Git objects. A filesystem path is acceptable;
   a URL, hostname, or protocol is not.
2. **No network stack.** It must keep the CI dependency gate green
   (`architecture/zero-network-policy.md` §3): no `reqwest`/`hyper`/`rustls`
   class of dependencies, direct or indirect.
3. **State semantics.** It defines identity (ULIDs), validation, transactions
   with audit atomicity, migration compatibility, or conflict refusal.
   Pure movement of bytes is not state semantics.
4. **Scriptable boundary.** Every multi-machine flow must be expressible as
   local CarryCtx commands composed by outside tools:
   `export` → transport → `import`. If a flow cannot be staged this way,
   the Core side is missing — not the transport side.
5. **Fail-closed offline.** The airplane test: with no network, the command
   either completes fully or refuses with a typed error and writes nothing.
   Silent partial application is never acceptable.

## 4. Worked examples

| Belongs in Core                                                                                                                                              | Belongs outside Core                                                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| `export` / `import`: serialization, validation, replace-import, three-way merge with conflict staging/resolution, re-anchor, pre-import and pre-merge backup | `tar` / `zstd` / `age` pipes, `scp` / `ssh`, snapshot-ref `git push` / `fetch` |
| `sync push` / `pull` as local-path whole-file copy                                                                                                           | Anything with URL-, host-, or protocol-aware arguments                         |
| Hook dispatch plus local script execution (designed in CTX-0010)                                                                                             | The sync/backup/notify scripts themselves                                      |
| `preset install` from a local directory                                                                                                                      | Preset download (`git clone`, done by the user)                                |
| MCP server over stdio                                                                                                                                        | Network MCP transports, cloud dashboards                                       |
| `doctor`, integrity checks, migration guards                                                                                                                 | Remote health checks of other machines                                         |

Two readings deserve emphasis. First, `carryctx sync` is grandfathered **only**
under its current contract: a local `--remote` path, whole-file,
last-writer-wins, with `SYNC_PROJECT_MISMATCH` on project-id skew
(zero-network policy §5). Anything beyond a local path belongs in a separate
utility. Second, skill presets such as `git-backed-sync` are the intended
composition layer — Core commands plus user-run transport — and they must
never auto-import: silently replacing local state from a remote bundle is
refused by design.

## 5. Consequences

1. **Semantic merge lives in Core; transport stays external.** The merge
   milestone (design `2026-09-10-mergeable-git-managed-state.md`) has landed
   ctxpack v2 (`parents` DAG, tombstones, `redacted` flag), the three-way row
   merge, conflict staging and resolution (`import --mode merge`, `conflict
list/show/resolve/apply/abort`), and local snapshot Git refs
   (`export --snapshot`, `import --from-git`). These changes are merged to the
   `carryctx-cli` main branch but are **not yet released** (DEC-0051 /
   DEC-0052; merge-snapshot commits and the local-only ref guard are still in
   flight). The boundary is unchanged: snapshot refs are local Git objects,
   `git push` and `git fetch` remain user transport, and the binary neither
   pushes nor fetches. A public redacted publication flow and
   `snapshot log/diff` UX remain follow-ups, not Core network features.
2. **No native single-file compression in v1.** The interchange stays a
   `dir` layout (git-diff friendly, zero new dependencies); single-file
   movement is a documented external recipe
   (`tar -cf - <dir> | ssh …`, `age`, …).
3. **Hooks are the extension boundary.** Backup, sync, notifications, and CI
   integration compose out of lifecycle hooks running local commands, not out
   of network features in Core. Trust, timeout, output limits, reentrancy
   guards, and failure policy are part of that design (CTX-0010), because a
   hook that auto-executes repository-provided scripts is a supply-chain
   boundary like Git hooks or `npm scripts`.
4. **Skills and website consume contracts; they do not define them.**
   When Core renames a flag (e.g. the `--pack-format` vs `--format` split),
   stale skill examples and website pages are bugs to be caught by CI, not
   dialects to be supported.

## 6. Non-goals (binding)

The following are out of Core scope, not deferred design:

- Remote synchronization adapters and cloud sync services.
- GitHub Issue and Pull Request synchronization (GitHub API calls from Core).
- Preset registries or marketplaces in Core (external clone plus local
  install only).
- A second user-manual source of truth (`manual/` is normative; the website
  manual generates or syncs from it in the website repository).

## 7. Contract-version metadata (machine-readable)

Four contracts must never drift silently again. Each has exactly one source
of truth; the table is the expected snapshot. A wave-2 CLI-repo task
implements the check this section specifies — docs work stops at the spec.

| Contract         | Source of truth                                                                                                                                                                                     | Current value                                                                                                        |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `cli`            | `carryctx-cli/Cargo.toml`, `package.version` (runtime cross-check: `carryctx --version`, MCP `initialize` server info — both `env!("CARGO_PKG_VERSION")`)                                           | `0.9.1`                                                                                                              |
| `ctxpack-format` | `crates/carryctx-core/src/domain/pack.rs` (`PACK_FORMAT` + `PACK_FORMAT_VERSION`; pre-P1 at `carryctx-cli/src/domain/pack.rs`)                                                                      | `carryctx-pack-dir`, `format_version` `2` (v2 unreleased-on-main; v1 readable for one release cycle per DEC-0051 #8) |
| `db-schema`      | `crates/carryctx-*/src/adapter/sqlite.rs` migration list (`crates/carryctx-sqlite` post-P2; pre-P2 at `carryctx-cli/src/adapter/sqlite.rs`), matching `migrations/project/NNNN_*.sql` (latest wins) | `18` (`0018_tombstones_snapshot_state`; unreleased-on-main)                                                          |
| `skill-surface`  | `carryctx-skills/skills/use-carryctx/SKILL.md` frontmatter `version`, plus the minimum CLI surface the skill claims to cover                                                                        | skill `1.1.0`; `min_carryctx 0.9.1` (0.8.x → 0.9.1 aligned; merge-milestone skill sync still pending)                |

The check compares this exact JSON shape with strict equality; any mismatch
fails the gate:

```json
{
  "contract_versions": {
    "cli": "0.9.1",
    "ctxpack_format": { "format": "carryctx-pack-dir", "format_version": 2 },
    "db_schema": 18,
    "skill_surface": {
      "skill": "use-carryctx",
      "version": "1.1.0",
      "min_carryctx": "0.9.1"
    }
  }
}
```

Check procedure (normative for the wave-2 implementation):

1. **Extract** actuals from pinned checkouts of `carryctx-cli` and
   `carryctx-skills`: parse `package.version` from `Cargo.toml`; read
   `PACK_FORMAT` / `PACK_FORMAT_VERSION` from `crates/carryctx-core/src/domain/pack.rs`
   (pre-P1 fallback: `src/domain/pack.rs`); take the maximum migration `version`
   in `src/adapter/sqlite.rs` (`crates/carryctx-sqlite` post-P2) and require a
   matching `migrations/project/NNNN_*.sql` file; parse the skill frontmatter
   `version` and its declared minimum CLI version.
2. **Compare** field-by-field against the shape above. Unknown fields,
   missing fields, and type changes (e.g. `db_schema` as string) fail —
   the shape is closed.
3. **Cross-check** the skill surface: every `carryctx` flag and subcommand in
   skill examples must exist in the CLI help of `min_carryctx`; a renamed
   flag with stale examples fails the same gate.
4. **Bump rule.** A version bump is one reviewable change: source-of-truth
   edit, plus this table, plus the `cli-specification.md`
   applicability note, plus `CHANGELOG.md`. Tagging follows the 0.9.0/current gate
   (ctxpack design §9, history anchor for 0.8.2 preserved): review closed, `cargo fmt --check`, Clippy with
   `-D warnings`, `cargo test`, markdownlint, package-smoke, and the
   acceptance matrix green.

## 8. References

- `architecture/zero-network-policy.md` (network ban, CI dependency gate,
  local-only `sync` exception).
- `design/2026-09-09-ctxpack-export-import.md` (interchange v1 contract,
  fail-closed import, deferred merge — completed by
  `design/2026-09-10-mergeable-git-managed-state.md`).
- Workspace research note 001 (2026-09-09): version drift (P0), hooks
  re-design, ROADMAP/TODO cleanup, ctxpack hardening before merge.
