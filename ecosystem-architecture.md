# CarryCtx Ecosystem Architecture & Vision

## 1. The Ultimate Positioning: "The Git of Agent Context"

Currently, the Agent ecosystem suffers from massive fragmentation regarding project context (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `.windsurfrules`).

CarryCtx is **NOT** just another Markdown file manager, nor is it a user memory database (like `Mem0`) or an agent's internal working memory (like `Letta`).

**CarryCtx is the Project Knowledge Database and State Persistence Layer.**
It aims to be the **Agent Context Standard**.

Just as:

- Code → `Git`
- Dependencies → `npm`
- Containers → `Docker`
- **Agent Context → `CarryCtx`**

## 2. The 5-Layer Ecosystem Architecture

To prevent architectural drift and scale across platforms, the CarryCtx ecosystem is strictly divided into five layers:

### Layer 1: `carryctx-cli` (The Engine & Source of Truth)

- **Role**: Agent project state and memory engine. Written in Rust (moved from an earlier TypeScript prototype for long-term maintainability and stronger compile-time error checking).
- **Responsibilities**: Defines the exact state machine (Tasks, Progress, Checkpoints, Sessions). The CLI is the **only** source of truth for schema and state persistence.

### Layer 2: `carryctx-skills` (The Instructions)

- **Role**: Teaches the Agent *how* to use CarryCtx.
- **Responsibilities**: Lightweight instructions mapping to the core CLI. It should **not** recreate state machines in prompts. It relies on `carryctx-cli` for truth. Progressive disclosure is used via `references/` to keep context windows small.

### Layer 3: Presets (`carryctx-skills/presets/`) — The Capability Packs

- **Role**: Composable, versioned, and auditable Agent Capability Packs.
- **Responsibilities**: Structured, versioned, auditable capability packs, stored as JSON manifests (`presets/<category>/<slug>.json` + a companion `.md`) rather than a `preset.yaml`. A Preset combines:
  - **Profiles** (formerly Personas): Capabilities, review standards, and output structures.
  - **Rules**: Project constraints.
  - **Workflows**: SOPs and executable steps.
  - **Permissions**: Explicit boundaries (e.g., `gitPush: deny`).
- **Example**: `carryctx preset install rust-cli-maintainer`

### Layer 4: `carryctx-plugins` (Reserved, currently empty)

- **Role**: originally envisioned as platform-specific installation adapters for `Claude Code`, `Cursor`, `Codex`, `OpenCode`, and `Antigravity`.
- **Status (decided 2026-07-24)**: not built. `carryctx mcp` (see Layer 5 below) is a standard MCP stdio server; every MCP-compliant client reads the same `{"command": "carryctx", "args": ["mcp"]}` config directly, so a per-platform adapter package would only exist to generate that same JSON. An earlier attempt shipped three unverified TypeScript adapters with no build tooling and no confirmation they worked against the real target platforms; they were removed rather than fixed in place, since `carryctx-cli` moved to Rust specifically to avoid maintaining a second, weaker-tooled language stack. This repository stays reserved for a genuine future need (e.g. a platform requiring a compiled native extension, not a config file) rather than speculative adapters.

### Layer 5: `carryctx mcp` (The Runtime Protocol)

- **Role**: The unified cross-platform communication interface.
- **Responsibilities**: Exposing task/checkpoint/decision/progress/graph/project management via the Model Context Protocol (MCP) so any compliant agent can interact with project state natively.
- **Note**: this is a subcommand of `carryctx-cli` (`carryctx mcp`), not a separate repository. The 6 exposed tools and their actions are documented in `carryctx-cli/src/application/mcp.rs` and the CLI reference site.

## 3. Addressing the "Drift" Risk

A critical architectural constraint for CarryCtx is preventing **Command and State Model Drift** between the Skills documentation and the actual CLI implementation.

**Rule**: The CLI is the only state machine and schema source.

- `carryctx-skills` must align 1:1 with the CLI's `clap` commands.
- For example, `carryctx progress done` must not exist in a skill if the CLI expects `carryctx progress complete`.
- Skills should output machine-readable schema requests to the CLI, rather than attempting to manage state purely in the LLM's context window.

## 4. Supply Chain Security for Presets

As the Preset ecosystem grows, security becomes paramount. A Preset is treated as executable code.

- **Permissions Manifest**: Every preset manifest must explicitly declare needed permissions (`requires_filesystem`, `requires_network`, `requires_env`).
- **Auditability**: Presets must support lockfiles (`.carryctx/presets.lock`), SHA-256 integrity checks, and publisher signatures.
- **Rule Precedence**: Project Rules can never override Platform Security Policies or Explicit User Instructions. Repository content must be treated as untrusted input.
