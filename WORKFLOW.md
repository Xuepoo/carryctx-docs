# CarryCtx Development Workflow

## Repository boundaries

CarryCtx uses independent sibling repositories. Run Git and build commands from the repository they affect. The workspace root only routes work and holds disposable `recording/` artifacts.

The current delivery order is:

1. `carryctx-docs`
2. `carryctx-cli`
3. `carryctx-skills`
4. `carryctx-plugins`
5. `carryctx-website`

Documentation and CLI work may proceed together when a public contract needs correction. Product development in the final three repositories waits until the CLI v0.1 gate passes.

## Change lifecycle

1. Read the relevant requirement, CLI, configuration, and engineering sections.
2. Record or update the approved design in `design/`.
3. Write a task-level implementation plan in `plans/`.
4. Create a failing test that expresses one required behavior.
5. Run the test and confirm it fails for the expected missing behavior.
6. Implement the smallest coherent change that makes it pass.
7. Refactor only while the relevant and full tests remain green.
8. Update public documentation in the same workstream.
9. Run the repository quality and packaging gates.
10. Record release-level evidence under `reports/`.

## Commit policy

Use Conventional Commits with focused scopes, for example:

```text
feat(task): add atomic task claiming
fix(config): preserve project override precedence
test(checkpoint): cover dirty worktree capture
docs(cli): align handoff acceptance criteria
```

Do not mix unrelated repositories or user-owned changes in one commit.

## Verification levels

- Focused: the specific unit or integration test used during a red/green cycle.
- Fast: formatting check, typecheck, lint, and unit tests.
- Full: all quality checks, integration tests, CLI contracts, and package smoke tests.
- Release: full verification plus clean worktree, package contents, version consistency, and changelog checks.

The final CLI v0.1 report must map evidence to AC-001 through AC-012 rather than relying only on an aggregate test count.
