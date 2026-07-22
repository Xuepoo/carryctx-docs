# CarryCtx Roadmap

## v0.1 — Local continuity loop

The first release establishes a complete offline workflow for coding agents and human developers:

1. Initialize a Git project and shared SQLite state.
2. Register an agent, create and claim a task, and start a session.
3. Track structured progress and Git-aware checkpoints.
4. End and resume work across terminals, agents, and linked worktrees.
5. Inspect deterministic project context, status, events, dependencies, and conflicts.
6. Diagnose stale or inconsistent state and migrate or back up the database safely.
7. Package the CLI and its generic Agent Skill for npm distribution.

The v0.1 completion gate is AC-001 through AC-012 in `requirements.md`, plus the engineering Definition of Done.

## v0.2 — Extensibility

- Stabilize a plugin contract after real v0.1 usage.
- Move or synchronize the generic skill with the standalone `carryctx-skills` repository.
- Add provider-specific skill guidance without relying on private Agent APIs.
- Improve export/import and shell completion.
- Expand macOS coverage and prepare Windows path behavior.

## Later releases

- MCP adapter
- Explicit remote synchronization adapters
- Multi-repository projects
- GitHub Issue and Pull Request synchronization
- Website and optional local dashboard
- Code impact and richer indexing adapters

These later items must remain optional and must not compromise the local-first, offline CLI.
