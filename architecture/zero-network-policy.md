# Zero Network Policy & Local-First Philosophy

**Document Path:** `carryctx-docs/architecture/zero-network-policy.md`
**Document Version:** v0.1

## 1. Core Commitment

CarryCtx is fundamentally designed as a **local-first** and **offline** tool. 
To ensure maximum security, privacy, and speed, we explicitly commit to the following principle:

> **CarryCtx Core never initiates network connections.**

### Specifically:
- **No telemetry upload**: Your project data remains on your machine.
- **No automatic update checks**: You manage your tools via your system's package manager.
- **No login or authentication required**: It is a pure local CLI.
- **No API keys needed**.
- **No cloud sync**: Syncing is outside the scope of the core tool.
- **No background registry polling**.
- **No GitHub API calls** from the core CLI.

---

## 2. Network Boundaries

While the ecosystem may involve network interactions, these must strictly occur *outside* the `carryctx` core binary.

### Safe Operations (Zero Network in Core):
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

### External Responsibilities (Network permitted but delegated):
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
If multi-device synchronization is required in the future, it must be developed as a completely separate, optional utility (e.g., `carryctx-sync`), or handled via explicit plugin adapters. The core CLI will only provide local `import`/`export` functionalities via JSONL streams, utilizing an append-only event model with ULIDs, timestamps, and conflict metadata.
