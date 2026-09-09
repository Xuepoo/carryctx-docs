# CarryCtx CLI Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the complete Bun/TypeScript engineering baseline and a tested, idempotent `carryctx init` flow that discovers Git repositories, loads strict TOML configuration, creates shared SQLite state, and emits stable JSON.

**Architecture:** The CLI parses inputs and delegates to an application use case. Pure configuration/project values sit behind Git, TOML, filesystem, XDG, and SQLite adapters. Project bootstrap uses an external operation journal plus atomic file replacement; SQLite initialization and its audit Event commit together.

**Tech Stack:** Bun 1.3.14, TypeScript 7.0.2, Citty 0.2.2, Zod 4.4.3, smol-toml 1.7.0, bun:sqlite, bun:test, Oxfmt, Oxlint, Biome, Knip, Lefthook, Just

**Design:** `../design/2026-07-22-carryctx-cli-v0.1-design.md`

---

## File map

- `package.json`, `tsconfig.json`, `justfile` — runtime, compiler, scripts, and contributor entry points.
- `.oxfmtrc.jsonc`, `oxlint.config.ts`, `biome.jsonc`, `knip.jsonc` — source quality boundaries.
- `lefthook.yml`, `commitlint.config.ts`, `.markdownlint-cli2.jsonc` — local repository policy.
- `.github/workflows/ci.yml`, `.github/dependabot.yml`, `.github/actionlint.yaml` — local-verifiable CI and dependency automation definitions.
- `src/cli.ts`, `src/cli-app.ts`, `src/commands/init.ts` — executable entry, root command, and init parsing.
- `src/application/init-project.ts` — bootstrap orchestration and recovery protocol.
- `src/domain/project.ts`, `src/domain/config.ts` — I/O-free project and configuration values.
- `src/errors/carryctx-error.ts` — public errors and exit-code mapping.
- `src/output/envelope.ts`, `src/output/render.ts` — validated text/JSON result rendering.
- `src/adapters/git/git-cli.ts` — safe argument-array Git invocation and discovery.
- `src/adapters/config/config-schema.ts`, `src/adapters/config/config-loader.ts` — strict TOML schemas and precedence.
- `src/adapters/filesystem/atomic-file.ts`, `src/adapters/filesystem/operation-journal.ts`, `src/adapters/filesystem/admission-lock.ts` — recoverable multi-resource filesystem protocol.
- `src/adapters/sqlite/migrations.ts`, `src/adapters/sqlite/project-database.ts`, `src/adapters/sqlite/registry-database.ts` — schema, PRAGMAs, migration, project transaction, and registry.
- `src/adapters/xdg/xdg-paths.ts` — injected platform directory resolution.
- `migrations/project/0001_foundation.sql`, `migrations/registry/0001_registry.sql` — immutable initial schemas.
- `tests/unit/`, `tests/integration/`, `tests/helpers/` — domain, adapter, and temporary Git CLI coverage.

### Task 1: Pin the engineering baseline

**Files:**

- Create: `package.json`
- Create: `tsconfig.json`
- Create: `justfile`
- Create: `.oxfmtrc.jsonc`
- Create: `oxlint.config.ts`
- Create: `biome.jsonc`
- Create: `knip.jsonc`
- Create: `lefthook.yml`
- Create: `commitlint.config.ts`
- Create: `.markdownlint-cli2.jsonc`
- Create: `.github/actionlint.yaml`
- Create: `.github/workflows/ci.yml`
- Create: `.github/dependabot.yml`

- [ ] **Step 1: Create exact package metadata and scripts**

Create `package.json` with the approved scoped package and exact dependency versions:

```json
{
  "name": "@xuepoo/carryctx",
  "version": "0.1.0",
  "description": "Persistent project context for coding agents",
  "type": "module",
  "private": false,
  "packageManager": "bun@1.3.14",
  "engines": { "bun": ">=1.3.14" },
  "bin": { "carryctx": "./dist/cli.js" },
  "files": [
    "dist",
    "migrations",
    "skills",
    "templates",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "dev": "bun run src/cli.ts",
    "build": "bun build src/cli.ts --target=bun --outfile=dist/cli.js",
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "test:unit": "bun test tests/unit",
    "test:integration": "bun test tests/integration",
    "lint": "oxlint .",
    "format": "oxfmt --write . && biome check --write .",
    "format:check": "oxfmt --check . && biome check .",
    "markdownlint": "markdownlint-cli2 \"**/*.md\" \"#node_modules\" \"#dist\"",
    "knip": "knip"
  },
  "dependencies": {
    "@clack/prompts": "1.7.0",
    "citty": "0.2.2",
    "picomatch": "4.0.5",
    "smol-toml": "1.7.0",
    "ulidx": "2.4.1",
    "zod": "4.4.3"
  },
  "devDependencies": {
    "@biomejs/biome": "2.5.5",
    "@commitlint/cli": "21.2.1",
    "@commitlint/config-conventional": "21.2.0",
    "@types/bun": "1.3.14",
    "@types/picomatch": "4.0.3",
    "knip": "6.27.0",
    "lefthook": "2.1.10",
    "markdownlint-cli2": "0.23.1",
    "oxfmt": "0.60.0",
    "oxlint": "1.75.0",
    "typescript": "7.0.2"
  }
}
```

- [ ] **Step 2: Add the strict compiler configuration**

Create `tsconfig.json` exactly as the engineering standard specifies, with `src/**/*.ts`, `tests/**/*.ts`, and root config files included and generated directories excluded.

- [ ] **Step 3: Add focused quality-tool configurations**

Configure Oxfmt as the formatter, Oxlint as the linter, Biome only for import organization, Knip with `src/cli.ts` and migration/skill assets as entries, Commitlint with the conventional preset, Markdownlint with MD013 disabled for long technical tables, and Lefthook with the documented pre-commit/commit-msg/pre-push split.

- [ ] **Step 4: Add contributor and CI entry points**

Create `justfile` recipes `setup`, `dev`, `build`, `test`, `test-unit`, `test-integration`, `typecheck`, `lint`, `fmt`, `fmt-check`, `docs`, `knip`, `check-fast`, `check`, `actionlint`, `package`, and `ci`. Create Linux CI jobs for quality, unit, integration, Actionlint, and package smoke; create weekly Dependabot entries for npm and GitHub Actions. Do not add a publish workflow in this development phase.

- [ ] **Step 5: Install and lock dependencies**

Run: `bun install`

Expected: exit 0 and a new `bun.lock` containing only exact direct dependency constraints.

- [ ] **Step 6: Verify the empty-project toolchain**

Run: `bun run typecheck`

Expected: exit 0 with no TypeScript errors before source files exist.

Run: `bun run format:check`

Expected: exit 0.

- [ ] **Step 7: Commit the baseline**

```bash
git add package.json bun.lock tsconfig.json justfile .oxfmtrc.jsonc oxlint.config.ts biome.jsonc knip.jsonc lefthook.yml commitlint.config.ts .markdownlint-cli2.jsonc .github
git commit -m "chore: establish CLI engineering baseline"
```

### Task 2: Define public errors, envelopes, and CLI metadata

**Files:**

- Create: `tests/unit/output/envelope.test.ts`
- Create: `tests/integration/cli/version.test.ts`
- Create: `src/errors/carryctx-error.ts`
- Create: `src/output/envelope.ts`
- Create: `src/output/render.ts`
- Create: `src/cli-app.ts`
- Create: `src/cli.ts`

- [ ] **Step 1: Write failing envelope and exit-code tests**

Create tests that require `successEnvelope("project.list", { projects: [] }, clock)` to return schema version 1, command, success, data, empty warnings, and deterministic timestamp; require `CarryCtxError("RESOURCE_NOT_FOUND")` to map to exit 7; and require JSON error rendering to contain no ANSI bytes.

- [ ] **Step 2: Run the unit test and verify RED**

Run: `bun test tests/unit/output/envelope.test.ts`

Expected: FAIL because `src/output/envelope.ts` and `src/errors/carryctx-error.ts` do not exist.

- [ ] **Step 3: Implement the minimum public contract**

Implement `ExitCode` as the documented numeric union, `CarryCtxError` with stable `code`, safe `message`, `details`, `suggestions`, and `exitCode`, plus `successEnvelope`/`errorEnvelope` schemas in Zod. `renderResult` must write one JSON document to the correct stream and keep JSON success stderr empty.

- [ ] **Step 4: Verify GREEN**

Run: `bun test tests/unit/output/envelope.test.ts`

Expected: PASS.

- [ ] **Step 5: Write a failing CLI version subprocess test**

The test runs `bun src/cli.ts --version`, expects exit 0, stdout `0.1.0`, and empty stderr.

- [ ] **Step 6: Verify the version test fails**

Run: `bun test tests/integration/cli/version.test.ts`

Expected: FAIL because the executable entry does not yet implement the version contract.

- [ ] **Step 7: Implement the Citty root command**

Create a `defineCommand` root with name `carryctx`, version `0.1.0`, the package description, and no business subcommands yet. `src/cli.ts` contains the Bun shebang and calls `runMain(createCli())` with a top-level safe error mapper.

- [ ] **Step 8: Run metadata, type, and build verification**

Run: `bun test tests/unit/output/envelope.test.ts tests/integration/cli/version.test.ts`

Expected: all tests PASS.

Run: `bun run typecheck`

Expected: exit 0.

Run: `bun run build`

Expected: exit 0 and executable `dist/cli.js`.

- [ ] **Step 9: Commit the public CLI shell**

```bash
git add src tests
git commit -m "feat(cli): add stable output and error contracts"
```

### Task 3: Discover Git projects safely

**Files:**

- Create: `tests/helpers/temp-git.ts`
- Create: `tests/integration/git/git-cli.test.ts`
- Create: `src/domain/project.ts`
- Create: `src/adapters/git/git-cli.ts`

- [ ] **Step 1: Write failing temporary-repository tests**

Create one normal Git repository and one linked worktree under Bun temporary directories. Assert discovery from a nested directory returns normalized absolute `repositoryRoot`, `gitCommonDir`, `worktreeRoot`, `head`, and branch; assert both worktrees return the same `gitCommonDir`; assert a non-repository throws `GIT_REPOSITORY_NOT_FOUND` with exit 4.

- [ ] **Step 2: Verify RED**

Run: `bun test tests/integration/git/git-cli.test.ts`

Expected: FAIL because `GitCli` is missing.

- [ ] **Step 3: Implement argument-array Git discovery**

Use `Bun.spawnSync(["git", "-C", startPath, ...args])`, never a shell string. Resolve root with `rev-parse --show-toplevel`, common dir with `rev-parse --path-format=absolute --git-common-dir`, branch with `symbolic-ref --quiet --short HEAD`, and HEAD with `rev-parse HEAD`. Represent detached HEAD with `branch: null` rather than an adapter failure.

- [ ] **Step 4: Verify GREEN and error mapping**

Run: `bun test tests/integration/git/git-cli.test.ts`

Expected: all discovery cases PASS.

- [ ] **Step 5: Commit Git discovery**

```bash
git add src/domain/project.ts src/adapters/git tests/helpers tests/integration/git
git commit -m "feat(project): discover repositories and linked worktrees"
```

### Task 4: Load and validate TOML configuration

**Files:**

- Create: `tests/unit/config/config-loader.test.ts`
- Create: `tests/integration/config/config-sources.test.ts`
- Create: `src/domain/config.ts`
- Create: `src/adapters/config/config-schema.ts`
- Create: `src/adapters/config/config-loader.ts`
- Create: `src/adapters/xdg/xdg-paths.ts`

- [ ] **Step 1: Write failing merge and validation tests**

Cover built-in defaults, recursive table merge, array replacement, CLI-over-environment precedence, `CARRYCTX_CONFIG`, nested `CARRYCTX_CONTEXT__MAX_EVENTS` coercion, strict unknown-key rejection, compatibility warning mode, and source attribution for the effective value.

- [ ] **Step 2: Verify RED**

Run: `bun test tests/unit/config/config-loader.test.ts tests/integration/config/config-sources.test.ts`

Expected: FAIL because the configuration loader is missing.

- [ ] **Step 3: Implement pure defaults and merge**

Define the full v0.1 built-in sections from `configuration.md`: project, git, session, task, context, checkpoint, output, and verification. Implement recursive object merge and whole-array replacement without mutating inputs.

- [ ] **Step 4: Implement strict layer loading**

Parse TOML with `smol-toml`, validate each final effective object with strict Zod schemas, collect source metadata by dotted key, and resolve layers in the approved order. Inject `XdgPaths` and environment maps so tests never depend on the developer home directory.

- [ ] **Step 5: Verify GREEN**

Run: `bun test tests/unit/config/config-loader.test.ts tests/integration/config/config-sources.test.ts`

Expected: all precedence, validation, and source tests PASS.

- [ ] **Step 6: Commit configuration loading**

```bash
git add src/domain/config.ts src/adapters/config src/adapters/xdg tests/unit/config tests/integration/config
git commit -m "feat(config): load strict layered TOML configuration"
```

### Task 5: Create secure project and registry databases

**Files:**

- Create: `migrations/project/0001_foundation.sql`
- Create: `migrations/registry/0001_registry.sql`
- Create: `tests/integration/sqlite/project-database.test.ts`
- Create: `tests/integration/sqlite/registry-database.test.ts`
- Create: `src/adapters/sqlite/migrations.ts`
- Create: `src/adapters/sqlite/project-database.ts`
- Create: `src/adapters/sqlite/registry-database.ts`

- [ ] **Step 1: Write failing database tests**

Open file-backed temporary databases and assert foreign keys are enabled, journal mode is WAL, busy timeout is 5000, migrations are recorded with checksums, repeated open is idempotent, a changed checksum is rejected, and adversarial project names are stored without affecting tables.

- [ ] **Step 2: Verify RED**

Run: `bun test tests/integration/sqlite`

Expected: FAIL because database adapters and migrations do not exist.

- [ ] **Step 3: Write the immutable foundation schemas**

Project migration creates `schema_migrations`, `projects`, `operations`, and append-only `events` with foreign keys, non-empty checks, JSON-text payload validation through application schemas, and indexes on Event time/type/project. Registry migration creates `schema_migrations`, `projects`, and `path_mappings` with unique normalized paths and a longest-prefix lookup index.

- [ ] **Step 4: Implement connection and migration runners**

Open with `create: true`, execute `foreign_keys = ON`, `journal_mode = WAL`, `busy_timeout = 5000`, and `synchronous = NORMAL`, then apply ordered SQL files in an exclusive transaction. Bind all inserted values with prepared statements. Compare the SHA-256 checksum of already applied migrations and map mismatches to exit 11.

- [ ] **Step 5: Implement atomic project/Event and registry writes**

Expose narrow methods `initializeProjectWithEvent`, `recordOperation`, `completeOperation`, `failOperation`, `registerProject`, and `resolveByLongestPath`. The project plus `project.initialized` Event uses one transaction; registry failure remains a warning at the application boundary.

- [ ] **Step 6: Verify GREEN and foreign-key integrity**

Run: `bun test tests/integration/sqlite`

Expected: all database, migration, checksum, parameter-safety, and idempotency tests PASS.

Run an integration assertion for `PRAGMA foreign_key_check`.

Expected: zero rows.

- [ ] **Step 7: Commit state storage**

```bash
git add migrations src/adapters/sqlite tests/integration/sqlite
git commit -m "feat(storage): add migrated project and registry databases"
```

### Task 6: Implement recoverable `carryctx init`

**Files:**

- Create: `tests/unit/filesystem/operation-journal.test.ts`
- Create: `tests/integration/cli/init.test.ts`
- Create: `src/adapters/filesystem/atomic-file.ts`
- Create: `src/adapters/filesystem/operation-journal.ts`
- Create: `src/adapters/filesystem/admission-lock.ts`
- Create: `src/application/init-project.ts`
- Create: `src/commands/init.ts`
- Modify: `src/cli-app.ts`

- [ ] **Step 1: Write failing filesystem protocol tests**

Require atomic writes to leave either the old or new complete file, require journals to survive database replacement, require a live admission lock to reject a second process, and require an expired lock with no local PID to be reported as repairable rather than stolen silently.

- [ ] **Step 2: Verify RED**

Run: `bun test tests/unit/filesystem/operation-journal.test.ts`

Expected: FAIL because filesystem protocol adapters are missing.

- [ ] **Step 3: Implement the filesystem protocol**

Use unique temporary sibling paths followed by rename for files. Store journals as mode-0600 JSON validated by Zod. Acquire `<git-common-dir>/carryctx/locks/command.lock/` with atomic directory creation and write PID, host, operation ID, and timestamp. Release only a lock owned by the current operation.

- [ ] **Step 4: Verify filesystem GREEN**

Run: `bun test tests/unit/filesystem/operation-journal.test.ts`

Expected: all atomicity and lock tests PASS.

- [ ] **Step 5: Write failing CLI initialization tests**

In a temporary Git repository run `carryctx init --json --non-interactive`. Assert one success envelope; `.carryctx/config.toml` and `.carryctx/README.md`; `.gitignore` contains only the local override rule addition; `<git-common-dir>/carryctx/state.sqlite`; project and `project.initialized` Event rows; registry entry under injected XDG state; mode-0600 state and journal files on Unix; and no state database under `.carryctx/`.

Add cases for repeated idempotent init, nested working directory, linked worktree sharing, invalid existing config, interrupted journal finalization, and `--force --yes --non-interactive` repairing configuration without replacing a readable database.

- [ ] **Step 6: Verify CLI RED**

Run: `bun test tests/integration/cli/init.test.ts`

Expected: FAIL because the init command is not registered.

- [ ] **Step 7: Implement the init use case**

Discover Git, derive the configured name/prefix/main branch, preflight existing files, acquire the admission lock, write the external prepared journal, build and verify a temporary migrated project database, atomically place database then project files, insert Project plus Event, complete the journal, and update the registry last. Return `created[]`, `reconciled[]`, paths, Project projection, warnings, and operation metadata without adapter-specific values.

- [ ] **Step 8: Register the Citty command and options**

Expose `--name`, `--task-prefix`, `--main-branch`, `--force`, `--minimal`, and `--install-skill`. Enforce `--force --yes` in non-interactive mode. Use the shared renderer so JSON success writes exactly one stdout envelope and errors write exactly one stderr envelope.

- [ ] **Step 9: Verify init GREEN**

Run: `bun test tests/integration/cli/init.test.ts`

Expected: all initialization, idempotency, linked-worktree, recovery, permission, and JSON contract tests PASS.

Run: `bun test`

Expected: all tests PASS.

- [ ] **Step 10: Commit project initialization**

```bash
git add src tests
git commit -m "feat(init): initialize recoverable shared project state"
```

### Task 7: Verify and document the foundation milestone

**Files:**

- Modify: `README.md`
- Create: `tests/integration/package/package-smoke.test.ts`
- Create locally without committing: `../recording/reports/2026-07-22-foundation-test-report.md`

- [ ] **Step 1: Write a failing package smoke test**

Build and dry-pack the package, assert the tarball includes only approved runtime assets, install it into a temporary directory, run `carryctx --version`, initialize a temporary Git repository, and run `carryctx status --json` only after the Status unit is available. For this foundation milestone, require version and init from the packed artifact.

- [ ] **Step 2: Verify RED before packaging fixes**

Run: `bun test tests/integration/package/package-smoke.test.ts`

Expected: FAIL on missing packaged migration resolution or executable permissions before final package adjustments.

- [ ] **Step 3: Fix package-relative asset resolution and documentation**

Resolve migrations relative to the built CLI location, preserve the Bun shebang, and keep the user-facing README limited to installation and CLI usage. Do not add internal development documentation to the CLI repository.

- [ ] **Step 4: Run the complete foundation gate**

Run: `just ci`

Expected: format, typecheck, lint, docs, Knip, unit, integration, Actionlint, build, and package smoke all exit 0.

- [ ] **Step 5: Record evidence**

Create the local-only report with the date, Bun/TypeScript/Git versions, commit, commands, test counts, AC-001 evidence, linked-worktree evidence, known platform limits, and the explicit list of later v0.1 units. Keep it under workspace `recording/` and do not add it to any Git repository.

- [ ] **Step 6: Commit milestone documentation**

In `carryctx-cli`:

```bash
git add README.md tests/integration/package
git commit -m "docs: document and verify CLI foundation"
```

## Plan self-review

- Spec coverage: Units A through E and AC-001 map to Tasks 1 through 7; later domain units remain in subsequent focused plans as required by the approved decomposition.
- Placeholder scan: this plan contains no deferred implementation marker inside its foundation scope; remote publishing and later domain units are explicitly outside this plan.
- Type consistency: `GitProject`, `EffectiveConfig`, `CarryCtxError`, operation journal records, Project/Event storage, and envelope fields retain the same names across tests and adapters.
