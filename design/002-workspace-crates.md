# Workspace Crate Split Design (4+1)

**Status:** Shipped in 0.9.0 — Accepted (2026-09-09). P1 landed as `dfecd07`; P2 (`carryctx-sqlite`) landed on `refactor/002-P1-core`; P3-P5 tracked as follow-ons. All P1-P5 landed.

**Date:** 2026-09-09

**Task:** 002 (program 002, commander cmd-001). Research note: `recording/research/002.md`.

**Scope:** physical crate boundaries for the existing `carryctx-cli` monolith. No CLI contract change, no new user-visible flags.

**Goal:** isolate already-stable architecture boundaries into a Cargo workspace so `core` semantics, persistence, VCS, interchange, and CLI rendering can evolve independently without crate fragmentation.

## 1. Context

By 0.8.2 the implementation had already stabilized into

```text
domain / application / repository (traits) / adapter / commands
```

and into three runtime boundaries

```text
SQLite (state.sqlite + migrations + WAL/backup) = persistence
ctxpack dir (manifest + JSONL)                 = interchange
Git / jj (worktree/workspace, hooks, status)   = VCS
```

`recording/research/002.md` evaluated when to introduce `workspace + crates`. Before 0.3-0.4 the boundaries were still shifting; by 0.8.2 they are stable enough that a crate split reduces coupling rather than increasing it. The risk to avoid is crate fragmentation (`carryctx-task`, `carryctx-session`, `carryctx-agent`, ...) which raises maintenance cost without architectural benefit.

## 2. Decision

### 2.1 Workspace shape (4+1)

```text
carryctx-cli/                         # Cargo workspace root
├── Cargo.toml                        # [workspace] members = crates/*
├── crates/
│   ├── carryctx-core/                # domain + repository traits + pure application + error
│   ├── carryctx-sqlite/              # migrations + repository impl + state.sqlite persistence
│   ├── carryctx-vcs/                 # VcsBackend (Git Tier1, jj optional)
│   ├── carryctx-pack/                # ctxpack interchange (manifest, format_version, JSONL)
│   └── carryctx-cli/                 # clap + commands + rendering + main binary
├── migrations/project/
└── tests/
```

Only four library crates plus the binary crate. No per-entity crates.

### 2.2 Dependency graph (Cargo-enforced)

```text
core <- sqlite
core <- vcs
core <- pack
{ core, sqlite, vcs, pack } <- cli
```

`core` bans `rusqlite`, Git, `clap`, terminal, filesystem, and network (`reqwest`/`hyper`/`rustls` and equivalents). Allowed in `core`: `serde`, `thiserror`, `ulid`, `chrono` and other pure-data dependencies. `sqlite`/`vcs`/`pack` each implement traits defined in `core`; `cli` aggregates all crates.

P1 transition note: `clap` remains in `core` solely because `TaskPriority` derives `ValueEnum`; extraction is deferred to P5.

### 2.3 VcsBackend abstraction

Do not define `trait GitBackend`. Define

```rust
pub trait VcsBackend {
    fn kind(&self) -> VcsKind;
    fn repository_root(&self) -> Result<PathBuf>;
    fn head(&self) -> Result<Revision>;
    fn status(&self) -> Result<VcsStatus>;
    fn create_workspace(&self, request: WorkspaceRequest) -> Result<Workspace>;
    fn capabilities(&self) -> VcsCapabilities;
}
```

`capabilities()` is the load-bearing method:

```rust
pub struct VcsCapabilities {
    pub workspaces: bool,
    pub commit_hooks: bool,
    pub staging_area: bool,
    pub mutable_changes: bool,
}
```

| Backend    | workspaces | commit_hooks | staging_area | mutable_changes |
| ---------- | ---------- | ------------ | ------------ | --------------- |
| GitBackend | true       | true         | true         | false           |
| JjBackend  | true       | false        | false        | true            |

Call sites branch on capabilities, not on `if jj { } else { }` scattered through the codebase. Domain concepts migrate from `worktree` (Git-specific) toward `Workspace` internally while `carryctx worktree` remains a CLI compatibility alias.

### 2.4 Git Tier 1, jj optional

- Git is the first-class default backend.
- `jj` colocated (`.jj/` present) is Tier 2 / optional runtime backend, selected via `[vcs] backend = "auto" | "git" | "jj"` with `auto` defaulting to `jj` when `.jj/` exists, otherwise `git`.
- No Cargo feature gate for `jj` in v1: dispatch via `Command::new("jj")` adds no heavy dependency, so a single binary ships both backends. A feature gate is reconsidered only if `jj` pulls in `libjj` or other large dependencies.

Stability:

| Backend      | Stability             |
| ------------ | --------------------- |
| Git          | Stable / Tier 1       |
| jj colocated | Experimental / Tier 2 |
| jj native    | Experimental          |

Fail-closed: where `jj` cannot safely emulate Git behavior, refuse with a typed error rather than silently diverging.

### 2.5 carryctx-pack is core-only

`carryctx-pack` owns `manifest`, `format_version`, JSONL encoding, validation, migration, and checksum. It depends on `carryctx-core` and on nothing else (`SQLite`, Git, CLI are out of scope). Future `ctxpack v2`, three-way merge, DAG, and external SDKs can depend on `carryctx-pack` directly.

### 2.6 Zero CLI contract change

Before and after the refactor, for the same `0.8.x` line:

```bash
carryctx task list
carryctx export --pack-format dir -o ./ctxpack-dir
carryctx import ./ctxpack-dir --mode replace --yes
```

behave identically. The refactor is internal only; external rendering, JSON envelopes, exit codes, and `stdout`/`stderr` separation are unchanged.

## 3. Consequences

1. **P1 lands narrowly.** Only `crates/carryctx-core` is physically extracted (domain + repository traits + pure `application/interchange` + `application/progress` + `error`). Root `src/` coexists during P2-P5; `cargo check --workspace` is the gate.
2. **P2-P5 follow the phase order.** `sqlite` adapters, then `VcsBackend` + Git/jj, then `pack` interchange, then `cli` commands/rendering. Each phase keeps `cargo test` green.
3. **Capability-based VCS.** New VCS call sites must go through `VcsBackend::capabilities()` rather than ad-hoc `detect_jj_colocation()` branches.
4. **Docs track the transition.** `engineering-standards.md` §5.2 and §6, `carryctx-cli/AGENTS.md`, `architecture/state-transport-boundary.md` §7, and `architecture/zero-network-policy.md` §5 now reference `crates/carryctx-*/src/...` with transitional fallbacks to `carryctx-cli/src/...` until each crate lands.
5. **No network in core.** The `zero-network-policy` CI gate (no `reqwest`/`hyper`/`rustls` family) applies to every new crate; `pack` in particular must stay network-free.

## 4. Alternatives considered

| Alternative                                                  | Why rejected                                                                                                                                                     |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Keep monolithic `src/`                                       | Coupling already visible; `core` purity relied on convention rather than `Cargo` enforcement.                                                                    |
| Per-entity crates (`carryctx-task`, `carryctx-session`, ...) | Fragmentation: many crates, no additional boundary, higher release and dependency overhead.                                                                      |
| `trait GitBackend` + `trait JjBackend`                       | Duplicates the abstraction; capability differences (hooks, staging, mutable changes) still need a shared vocabulary.                                             |
| `jj` as Cargo feature (`--features jj`)                      | Splits the binary matrix (`linux+git` vs `linux+git+jj` etc.) for negligible size saving while `jj` is a thin CLI wrapper. Revisit only with heavy `libjj` deps. |
| Single `carryctx-workspace` crate for all VCS                | Conflates Git and jj stability tiers and capability sets; `VcsCapabilities` already captures the variation more precisely.                                       |

## 5. Stage P1 deliverable

- **Commit:** `dfecd07 refactor(core): hoist domain + ports + pure application into carryctx-core workspace crate` (ahead of `origin/main`, not yet in a PR).
- **Diff vs `origin/main`:** 40 files, `+4482` lines (workspace `Cargo.toml` + `crates/carryctx-core` with `domain/*`, `repository/*`, `application/{interchange,progress}`, `error.rs`, `lib.rs`).
- **Carried residuals in working tree:** `src/commands/hooks.rs` (916 LOC lifecycle-hooks follow-on), `src/commands/version.rs` (contract-version command per `state-transport-boundary` §7), `src/adapter/sqlite.rs` (`bundled_schema_version()`), `src/commands/mod.rs`, `src/main.rs`, `Cargo.lock` — 5 files, `+864` lines. Included in the PR scope; not reverted.
- **Verification:** `cargo check --workspace` green at `dfecd07` plus residuals. Full `cargo test` / `cargo clippy --workspace -- -D warnings` / `cargo fmt --check` are PR gates, not P1-only gates.

## 6. Remaining gaps for P2-P5

| Phase | Crate             | Scope                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Entry criteria                                                                                                                                                                                                                                                                                       |
| ----- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| P2    | `carryctx-sqlite` | `migrations/*`, `src/adapter/sqlite*`, `src/adapter/unit_of_work.rs`, repository impls, WAL/backup/restore, integrity check — **landed**                                                                                                                                                                                                                                                                                                                                                                                                                     | `core` traits stable; no new domain entities expected — P2 commit thins `src/` to re-exports                                                                                                                                                                                                         |
| P3    | `carryctx-vcs`    | `src/adapter/git.rs`, `src/adapter/xdg.rs` (VCS-adjacent), `VcsBackend` trait, `GitBackend` Tier 1, `JjBackend` optional + `capabilities()`                                                                                                                                                                                                                                                                                                                                                                                                                  | P2 green; `worktree` vs `workspace` naming decision recorded — **landed** on `refactor/002-P1-core` (`carryctx-vcs` 4 members, thin bridges in `src/adapter/git.rs`, `src/adapter/xdg.rs`+`src/lib.rs`re-exports,`cargo check --workspace`/`cargo test --workspace` green, CLI contract zero change) |
| P4    | `carryctx-pack`   | `src/domain/pack.rs`, `src/application/interchange.rs`, `src/application/export.rs` / `import.rs`, manifest/JSONL validation — **landed** on `refactor/002-P1-core` (`carryctx-pack` 5 members: `manifest`+`io`+`migration`+`checksum` reader/writer, core-only `carryctx-core` deps, thin bridges in `src/domain/mod.rs`+`src/application/interchange.rs`+`src/lib.rs`, `cargo check --workspace` / `cargo test --workspace` / `carryctx export --pack-format dir --help` green, zero CLI contract change)                                                  | `core` pack types frozen at `format_version` 1 — satisfied, v1 no migrator                                                                                                                                                                                                                           |
| P5    | `carryctx-cli`    | `src/commands/*`, `src/application/*` (remaining), `src/output.rs`, `src/error.rs`, `src/main.rs`, `clap` extraction from `core` — **landed** on `refactor/002-P1-core` (`carryctx-cli` workspace shell + `ValueEnum` extraction, `crates/carryctx-cli` owns `adapter`/`application` wiring/`output`, root `carryctx` thin facade `pub use carryctx_cli::*`, `cargo check --workspace` 5 members + root / `cargo test --workspace` / `cargo tree -p carryctx-core` no `rusqlite`/`clap` / `cargo tree -p carryctx-pack` core-only, zero CLI contract change) | P2-P4 green; CLI contract snapshot tests green                                                                                                                                                                                                                                                       |

Each phase updates `engineering-standards.md` §5.2 to mark the crate as landed and removes the corresponding transitional fallback from the architecture docs.

> **P5 完成态（002 Complete）：**`crates/carryctx-cli` 已物理隔离并通过 `cargo check --workspace`（5 members + 根 `carryctx`）/ `cargo test --workspace` / `cargo clippy --workspace -- -D warnings` / `cargo fmt --check`；`clap::ValueEnum` 已从 `carryctx_core::domain::task::TaskPriority` 抽离至 `carryctx_cli::commands::task::TaskPriority` 侧 CLI enum translation（域保持 `serde`/`Default` 纯度，`core` 无 `clap`），`crates/carryctx-cli` 拥有 `adapter`（`config`/`filesystem`/`terminal` + `sqlite`/`git`/`xdg` re-exports）、剩余 `application` wiring、`output.rs`、`error` bridge；根 `carryctx`（`src/lib.rs`/`src/main.rs` + `Cargo.toml` workspace 5) 保留为 thin facade `pub use carryctx_cli::*` + `carryctx_core/vcs/pack` 透传，`cargo tree -p carryctx-core` 无 `rusqlite`/`clap`/`git2`/network，`cargo tree -p carryctx-pack` core-only；最终 `members = ["crates/carryctx-core","crates/carryctx-sqlite","crates/carryctx-vcs","crates/carryctx-pack","crates/carryctx-cli"]` with 根 aggregator，002 全量完成。

## 7. References

- `recording/research/002.md` — workspace + 4+1 crate research (phased migration, capability model).
- `carryctx-docs/engineering-standards.md` §5.2, §6 — workspace structure and dependency graph.
- `carryctx-docs/architecture/state-transport-boundary.md` §7 — contract-version sources (now `crates/carryctx-core/...`).
- `carryctx-docs/architecture/zero-network-policy.md` §3, §5 — network ban and local-only `sync` exception.
- `carryctx-docs/design/2026-09-09-ctxpack-export-import.md` — interchange v1 contract consumed by `carryctx-pack`.
- `carryctx-docs/design/2026-09-09-lifecycle-hooks.md` — hook dispatch boundary (stays in `cli`/`application`, not `core`).

## 8. History

| Date       | Change                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-09-09 | Initial version. Records 4+1 decision, `VcsBackend::capabilities()` choice, and P1 deliverable `dfecd07`.                                                                                                                                                                                                                                                                                                                                            |
| 2026-09-09 | P2 lands `carryctx-sqlite` (migrations + repos + WAL/backup/journal). `src/` thinned to bridges; `cargo check --workspace` (3 members) + `cargo test --workspace` green; CLI contract zero change.                                                                                                                                                                                                                                                   |
| 2026-09-09 | P3 lands `carryctx-vcs` (`VcsBackend` + `VcsCapabilities`, Git Tier1 `GitBackend`, jj optional runtime `JjBackend` via `Command::new("jj")`, no feature matrix, `auto_backend_kind` `.jj` rule, `XdgPaths` alongside VCS). `src/adapter/git.rs`, `src/adapter/xdg.rs`thinned to re-exports,`src/lib.rs`adds`vcs`surface,`Cargo.toml`4 members;`cargo check --workspace`/`cargo test --workspace`/`carryctx --help`/`carryctx worktree --help` green. |
| 2026-09-09 | P4 lands `carryctx-pack` (`manifest`/`format_version`/JSONL/validation/migration/checksum/reader-writer, core-only `carryctx-core` deps, `src/domain/pack.rs`+`src/application/interchange.rs` thin bridges, `cargo check --workspace` 5 members / `cargo test --workspace` green, `carryctx export --pack-format dir --help` / `carryctx import --help` zero CLI contract change).                                                                  |
| 2026-09-09 | P5 lands `carryctx-cli` (CLI shell + workspace wiring, `clap::ValueEnum` extraction from `core`, `crates/carryctx-cli` owns `adapter`/`application` wiring/`output`, root `carryctx` thin facade `pub use carryctx_cli::*`, 5 members + root, zero CLI contract change) — **002 Complete** (all 5 crates, internal refactor done).                                                                                                                   |
