# CarryCtx Master TODO & Roadmap

This document serves as the master plan for the CarryCtx ecosystem, moving from a simple state tracker to the universal "Git of Agent Context."

## P1: Improve Determinism & Ecosystem Foundations

These tasks focus on establishing the standard for Presets and eliminating ambiguity in rules and workflows.

- [x] **Design `preset.schema.json`**: Create a declarative schema for Capability Packs (Profiles, Rules, Workflows, Permissions).
- [x] **Define Instruction Precedence**: Document and enforce strict precedence (Platform Policy > User Instruction > Project Rules).
- [x] **Implement `carryctx preset` commands**:
  - `carryctx preset install <name>`
  - `carryctx preset activate <name>`
  - `carryctx preset list`
- [x] **Integrate Workflow State into Core**: (Skipped: Decided to keep workflow step execution delegated to the agent's prompt reading skill).
- [x] **Supply Chain Security**: Add permission manifests, integrity hashes (SHA-256), and `.carryctx/presets.lock`.

## P2: Platform Capability & Plugin Ecosystem

These tasks focus on delivering CarryCtx to various IDEs and Agent environments natively via `carryctx-plugins`.

- [x] **Architect `mcp-server-carryctx`**: Build the unified Model Context Protocol server exposing `carryctx-cli` commands as tools.
- [x] **Claude Code Adapter**: Create the specific `.claude-plugin/plugin.json` generator.
- [x] **Cursor Adapter**: Create the `.cursor-plugin/plugin.json` generator and compile our Markdown rules into Cursor `.mdc` format.
- [x] **OpenCode Adapter**: Build the TypeScript runtime adapter for `@opencode-ai/plugin`.

## Phase 2: Context Graph

Evolving from a linear state machine to a semantic graph of the project.

- [ ] **Design Context Graph Schema**: Define nodes (file, module, decision, bug, task, agent) and edges (depends, changed, fixed, related).
- [ ] **Graph Queries**: Allow agents to query "Why was this file changed?" or "What tasks depend on this module?".

## Phase 3: Agent Team Memory

Scaling from single-agent contexts to multi-agent swarms.

- [ ] **Subagent Shared State**: Enable Planner, Developer, Reviewer, and Tester agents to seamlessly pass `carryctx` context pointers without copying massive prompts.

## Phase 4: The Ultimate Vision

- [ ] **Native Integration**: Achieve out-of-the-box standard integration in major LLM tooling.
- [ ] **Preset Marketplace**: Launch a decentralized registry for CarryCtx Presets, heavily audited for security.

## Follow-ups from 0.5.0 triage (2026-08-10, see reports/)

- [ ] **Wire `touch_activity`**: `sessions.last_activity_at` equals `started_at` on every session in vectojs — nothing calls `touch_activity`, so session timing (and the `stale_after` staleness rule) have no data to work from. Decide where to call it (progress note, checkpoint, resume, periodic) and whether `stats` should show tracked time vs. span.
- [ ] **`mark_stale_sessions` is dead code**: defined in `application/session.rs` but never invoked by any command; stale sessions are never auto-marked. Call it on `resume`/`session start` (with the config's `stale_after`).
- [ ] **Release workflow `Build (${{ matrix.target }})` check** renders as `skipping` on PRs — the matrix template does not resolve for PR checks; fix the workflow so the build gate actually reports.
