# Zero Network Policy & Local-First Philosophy

**Document Path:** `carryctx-docs/architecture/zero-network-policy.md`
**Document Version:** v0.1

## 1. Core Commitment

CarryCtx is fundamentally designed as a **local-first** and **offline** tool.
To ensure maximum security, privacy, and speed, we explicitly commit to the following principle:

> **CarryCtx Core never initiates network connections.**

### Specifically

- **No telemetry upload**: Your project data remains on your machine.
- **No automatic update checks**: You manage your tools via your system's package manager.
- **No login or authentication required**: It is a pure local CLI.
- **No API keys needed**.
- **No cloud sync**: Remote and cloud syncing are outside the scope of the core tool. The shipped `carryctx sync` is a local filesystem copy only — see §5.
- **No background registry polling**.
- **No GitHub API calls** from the core CLI.

---

## 2. Network Boundaries

While the ecosystem may involve network interactions, these must strictly occur _outside_ the `carryctx` core binary.

### Safe Operations (Zero Network in Core)

- Initializing `.carryctx`
- Registering and switching Agents
- Session and Task management
- Checkpoint creation and restoration
- Git Worktree operations
- Rules and Personas loading
- Workflow execution
- Preset offline parsing and activation
- Local SQLite queries
- Local Git operations
- `carryctx doctor`
- `carryctx sync push` / `carryctx sync pull` against a local `--remote` path

### External Responsibilities (Network permitted but delegated)

Any operation requiring the internet must be handled by external systems (e.g., package managers, Git, or native Agent Plugins).

For example, downloading a preset:

```bash
# Correct: An external tool downloads the preset
git clone https://github.com/.../carryctx-presets
# Correct: CarryCtx installs it from the local disk
carryctx preset install ./carryctx-presets/rust-cli-maintainer
```

---

## 3. CI/CD Enforcement

To mathematically guarantee these principles, the CI pipeline enforces strict constraints on the build:

1. **Dependency Verification**: The `cargo tree` is automatically scanned on every commit. Any indirect or direct network dependencies (such as `reqwest`, `hyper`, `rustls`, `native-tls`, `tokio` networking features, etc.) will cause an immediate CI failure.
2. **Binary Size Budget**: To ensure the tool remains extremely lightweight and free of bloated network stacks, the binary must remain under strict size budgets (e.g., 4MB for Linux, 5MB for macOS, 6MB for Windows).

---

## 4. MCP Transports

When running as an MCP server, `carryctx` will strictly default to the `stdio` transport.
Agents (like Claude Code, Cursor, Codex) will launch `carryctx` as a local subprocess and communicate purely via `stdin/stdout` using JSON-RPC, keeping all interactions contained within the host machine with zero network overhead.

---

## 5. Syncing and Multi-device

Cloud synchronization is deliberately excluded from the core tool.
If multi-device synchronization is required in the future, it must be developed as a completely separate, optional utility (e.g., `carryctx-sync`), or handled via explicit plugin adapters. Remote synchronization adapters are tracked under "Later releases" in `ROADMAP.md` and are not implemented today.

### Scoped exception: local-only `carryctx sync`

The core CLI ships a `carryctx sync` command. It is **not** a network feature and does not violate §1:

- `sync push` copies `<git-common-dir>/carryctx/state.sqlite` to `<remote>/<git-common-dir-name>.sqlite` with `std::fs::copy` (`carryctx-cli/src/application/sync.rs:32`).
- `sync pull` copies that file back in the same way (`carryctx-cli/src/application/sync.rs:74`).
- `--remote` is a filesystem path, defaulting to `/tmp/carryctx-remote` (`carryctx-cli/src/commands/sync.rs:11`, `:17`).

The binary contains no network stack, so `--remote` can only ever resolve to a path the operating system already exposes. Reaching another machine is possible only if the user has independently mounted remote storage (NFS, SMB, or similar) — the transport is then owned by the OS, not by `carryctx`, which matches the "External Responsibilities" rule in §2.

Two constraints follow, and both are binding on future work:

1. `carryctx sync` must never gain a URL-, host-, or protocol-aware argument. Anything beyond a local path belongs in a separate utility or adapter.
2. The copy is whole-file and last-writer-wins. It is not a merge and carries no conflict resolution, so it is unsafe for concurrent multi-machine writes. Conflict-aware multi-device flows remain the job of the append-only JSONL `import`/`export` model, using ULIDs, timestamps, and conflict metadata.
