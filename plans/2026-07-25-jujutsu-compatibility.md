# Jujutsu (jj) Compatibility Roadmap

**Status:** Phases 1-4 shipped (this repo, `[Unreleased]` in `carryctx-cli/CHANGELOG.md`). Phase 3 landed as a detect-and-refuse guard rather than a `jj workspace add` integration — see the Phase 3 write-up below for why.

**Date:** 2026-07-25

**Goal:** Let teams using [Jujutsu](https://jj-vcs.dev/) (colocated with Git) use CarryCtx without data corruption or silently-broken commands, without CarryCtx taking on jj as a hard dependency or degrading the Git experience for everyone else.

## 1. Why this is a real, not hypothetical, need

Jujutsu is a Git-compatible VCS with a different mutation model (no staging area, working-copy auto-snapshot, an operation log instead of reflog). Its most common deployment mode is **colocated**: `jj git init --colocate` keeps a real `.git/` directory alongside `.jj/`, so any tool that talks to Git directly (like CarryCtx) can see the repository. This is the mode this plan targets; a jj repo with no `.git/` at all is out of scope (see §6).

More teams are adopting jj incrementally, colocated, inside otherwise-Git repositories and workflows. CarryCtx's job — persisting agent task state across Git worktrees and sessions — collides directly with the two things jj changes the most: multi-workspace layout and the commit lifecycle. This is worth planning for now rather than after users hit it in the wild.

## 2. What was actually verified (not assumed)

Findings below come from installing jj 0.43.0, creating real colocated repositories, and running the actual `carryctx` binary against them, not from reading jj's documentation alone.

### 2.1 Works today, no changes needed

`carryctx init`, `agent register`, `task create/claim/complete`, `progress *`, `decision add`, `session start/end` — all of this is pure local SQLite state with no Git dependency at all. Verified working identically to a plain Git repo.

`carryctx status`'s `branch`/`head` fields read correctly via `git symbolic-ref` / `git rev-parse HEAD`, because jj colocate keeps those Git refs in sync for the current workspace.

### 2.2 Bug, unrelated to jj (found while testing, worth fixing regardless)

`GitCli::get_snapshot` and `GitCli::discover` in `carryctx-cli/src/adapter/git.rs` call `git rev-parse HEAD` and silently coerce failure to `""` (`get_snapshot`) or `None` (`discover`) via `.ok()` / `.unwrap_or_default()`. On a brand-new repository with zero commits (true in *any* backend, Git or jj), `rev-parse HEAD` fails with "unknown revision", and the checkpoint's `head` field ends up as an empty string rather than `null`, which is what every other "absent" field in the same struct uses. Small, contained fix; not a compatibility issue, just an inconsistency this investigation surfaced.

### 2.3 Data-quality gap: checkpoint `staged_files`/`untracked_files` are unreliable under jj

`git status --porcelain`'s two-column XY code (index status, worktree status) is what `GitCli::get_snapshot` parses to fill `staged`/`modified`/`untracked`. Under plain Git, column X ("staged") only changes when the user runs `git add`. Under jj colocate, jj's automatic working-copy snapshotting (triggered by nearly any `jj` command, including read-only ones like `jj status`) writes directly to the Git index as a side effect, with observed behavior that is inconsistent by file state:

- A newly created file went from `??` (untracked) to `A` in the index column with the worktree column blank (staged) after running `jj status` — a read command the user did not intend as "git add".
- A modification to an already-tracked file stayed `M` in the worktree column only, with the index column blank (unstaged), across multiple jj commands (`jj status`, `jj describe`) in the same test session.

The net effect: CarryCtx's `staged_files` vs `untracked_files` split, which is meant to answer "did the agent deliberately stage this", stops meaning that under jj. `dirty` (any change present) and diff stats (insertions/deletions from `git diff --numstat`) remain accurate; only the staged/unstaged/untracked three-way split is compromised.

### 2.4 Broken: `carryctx worktree create` produces a directory jj does not recognize

`carryctx worktree create` shells out to `git worktree add`. This succeeds and Git sees a valid worktree. But jj has its own, separate multi-checkout primitive, `jj workspace add`, backed by its own bookkeeping (`jj workspace list`), and it does not discover directories created purely via `git worktree add`.

Verified: after `carryctx worktree create CTX-0001`, the resulting directory has no `.jj/`, and `jj workspace list` from the main repo does not list it. Running `jj status` from inside that directory does not fail outright, but resolves paths relative to the *parent* repository's working copy (observed output referenced files as `../../.carryctx/README.md` from inside what should have been an independent worktree) — i.e. it silently operates on the wrong logical workspace rather than erroring. This is a real correctness hazard, not just a missing feature: a user who runs `carryctx worktree create` then works inside it with `jj` commands is operating on a workspace jj doesn't think exists.

### 2.5 Broken: `carryctx hooks install` hooks never fire under jj

`carryctx hooks install` writes to `.git/hooks/post-commit` and `.git/hooks/prepare-commit-msg`, expecting `git commit` to trigger them (the `post-commit` hook auto-creates a checkpoint). jj never calls `git commit`; it writes commits directly to the Git object store and syncs refs via `jj git export`, a path that does not invoke Git's hook mechanism at all. Verified: running `jj commit` in a repo with CarryCtx hooks installed produced zero new checkpoints. jj also has no hook system of its own (checked `jj util --help` and the full command tree; nothing resembling Git's hooks exists), so there's no jj-side event to redirect to instead — this has to be solved differently (see §4.3).

## 3. Design principle for every fix below

**Detect, don't assume.** CarryCtx must never require jj to be installed, and must never change behavior for the (currently 100%) majority of users who are on plain Git. Every fix here is: detect that the repository is jj-colocated, and only then take a different code path. If jj is absent, behavior must be byte-for-byte identical to today.

Detection is cheap and already directly testable: `<git_common_dir>/../.jj` (i.e., a `.jj` directory as a sibling of `.git`) existing is a reliable, dependency-free signal, no need to shell out to `jj` or depend on it being on `PATH` just to detect it's not there.

## 4. Phased plan

### Phase 1 — Low-risk fixes, no jj-specific code paths (ship independently of the rest)

- Fix `head: ""` → `head: None` for the zero-commit case (§2.2). Pure bug fix, benefits every backend.
- Add a `carryctx doctor` check: "jj colocation detected" (informational, not a warning) whenever `.jj/` is found alongside `.git/`, so users get an explicit signal that CarryCtx knows about jj rather than silently doing something backend-specific. This is the seed of the detection logic Phase 2+ will reuse.

**Acceptance:** `cargo test` unaffected; new doctor check has a unit test using a temp dir with a fake `.jj/` marker (no real jj binary needed to test detection logic). **Done** — `GitCli::detect_jj_colocation` + 3 unit tests in `carryctx-cli/src/adapter/git.rs`; `doctor`'s `vcs.jj_colocation` check wired in `carryctx-cli/src/commands/doctor.rs`; additionally smoke-tested against a real jj 0.43.0 colocated repo (`jj git init --colocate`), confirming both the doctor check fires and `checkpoint`'s `head` field is `null` (not `""`) on a zero-commit repo.

### Phase 2 — Checkpoint data-quality: stop asserting staged/unstaged when it's not meaningful

Given §2.3's finding that jj makes the staged/unstaged split unreliable, the right fix is not to "parse it better" (there is no better parse; the underlying signal is jj's own inconsistent auto-snapshot behavior, not a CarryCtx bug), but to:

- When jj colocation is detected, still report `dirty` and diff stats (both remain accurate), but collapse `staged_files`/`modified_files`/`untracked_files` into a single accurate list of "changed files" rather than presenting a three-way split that doesn't hold. Add a field (e.g. `vcs_backend: "git" | "jj"`) to the checkpoint/snapshot payload so downstream consumers (the CLI reference docs, any Agent Skill reading this JSON) know why the shape differs.
- Document the field's meaning change in the CLI reference (`carryctx-website`) checkpoints page, gated on `vcs_backend`.

**Acceptance:** a test fixture using a real jj colocated temp repo (CI needs jj installed for this one test binary, or it's marked `#[ignore]` and run manually / in a dedicated job — decide during implementation) confirms `dirty` and diff stats stay correct and `vcs_backend` is set. **Done** — `GitSnapshot`/`Checkpoint` gained `vcs_backend: "git" | "jj"` and `changed_files` (accurate merge of staged+modified+untracked); under jj colocation the three-way split is cleared, `changed_files` and `dirty` stay accurate. Migration `0008_jj_compat.sql` adds the two new checkpoint columns. `#[ignore]`-gated integration test `carryctx-cli/tests/checkpoint_test.rs::test_checkpoint_jj_colocation_reports_backend_and_changed_files` (run manually with `cargo test --test checkpoint_test -- --ignored`); verified against real jj 0.43.0.

### Phase 3 — Worktree creation: revised after further verification

The original plan for this phase (branch `carryctx worktree create` on backend, calling `jj workspace add <path>` instead of `git worktree add`) turned out to have a premise that doesn't hold. Further verification with real jj 0.43.0:

- `jj workspace add <path>` creates a workspace with **only a `.jj/` directory — no `.git/` at all**, even though the *primary* workspace is colocated. jj's colocation is a property of the primary checkout; secondary workspaces created via `jj workspace add` are jj-native only. Confirmed: `git rev-parse --show-toplevel` inside a directory created this way fails with "not a git repository", while `jj status` works fine there.
- This means switching `carryctx worktree create` to call `jj workspace add` would close the discovery gap (§2.4 — jj not recognizing the directory) but immediately reopen a different, worse one: every other `carryctx` command (`status`, `checkpoint`, `context`, etc.) that reads state via `GitCli::discover`/`get_snapshot` would fail entirely inside that same directory, since there is no `.git/` to find. That is not a partial fix, it is trading one broken command for a fleet of newly-broken ones.
- Building a genuinely working fix would require a second, jj-CLI-based state reader (parsing `jj status`/`jj log` instead of shelling out to `git`) for any directory with `.jj/` but no local `.git/`. That is exactly the "jj-native, no `.git/`" support this plan's own §5 (non-goals) puts out of scope — the boundary doesn't move just because the no-`.git/` directory happens to be a *secondary* workspace of an otherwise-colocated repo.

Given that, this phase shipped the boundary the original plan intended (§2.4's exact hazard: a `carryctx`-created directory that `jj` and jj-users can't safely operate in) using the "detect, don't assume" principle from §3, but as a **refusal** rather than a **translation**:

- `carryctx worktree create` now detects jj colocation (via the same `detect_jj_colocation` helper from Phase 1) and returns `VALIDATION_FAILED` with a message pointing at `jj workspace add <path>` directly, plus `carryctx worktree bind` from inside the *primary* colocated checkout if the new workspace needs to be tracked by CarryCtx. It never creates a broken directory.
- `worktree list`/`bind`/`show`/`status`/`unbind` are unchanged — they already operate on the primary colocated checkout, which has a real `.git/` and works today (§2.1).

**Acceptance:** an integration test creates a real jj colocated repo, runs `carryctx worktree create`, and verifies it refuses with a clear jj-specific error instead of creating a directory neither `jj` nor `carryctx` can safely use (closing the exact gap found in §2.4, by refusal rather than by a working `jj workspace add` translation). **Done** — guard added in `carryctx-cli/src/application/worktree.rs::create_worktree`; `#[ignore]`-gated integration test `carryctx-cli/tests/worktree_test.rs::test_worktree_create_refuses_under_jj_colocation` plus a plain-git regression test in the same file; verified against real jj 0.43.0 (confirmed `jj workspace add` produces a `.git/`-less directory, confirmed the refusal fires and no directory is created, confirmed plain Git worktree create is unaffected).

### Phase 4 — Hooks: replace the git-hook assumption, don't try to patch it

git-hook-based auto-checkpointing cannot be made to work under jj (§2.5 — there is no equivalent trigger point to hook into). Options to evaluate, not yet decided:

1. **Polling/watch mode**: a lightweight `carryctx watch` background process (opt-in, not default) that periodically snapshots via the same `GitCli`/jj-equivalent status check and creates checkpoints on detected changes, backend-agnostic by construction since it doesn't rely on any commit-time hook.
2. **Manual is the answer for jj, for now**: don't build a replacement; update `carryctx hooks install` to detect jj colocation and print a clear message ("auto-checkpoint-on-commit isn't available under jj; run `carryctx checkpoint` manually or use `carryctx hooks install --watch`" once/if option 1 ships) instead of silently installing hooks that will never fire.

Recommendation: ship the honest error/warning from option 2 in this phase regardless of whether option 1 is pursued, since it's a one-line detection + message and immediately stops the current silent-failure behavior. Option 1 is a larger, separate feature and should get its own plan if pursued.

**Acceptance:** `carryctx hooks install` under a detected jj-colocated repo either refuses with a clear message, or (if option 1 lands first) installs the watch-based alternative; either way, it never silently installs hooks that cannot fire. **Done** — shipped option 2: `carryctx hooks install` detects jj colocation (reusing `detect_jj_colocation`) and refuses with `VALIDATION_FAILED` and a message explaining `jj git export` bypasses Git's hook mechanism, pointing at manual `carryctx checkpoint` as the interim workflow. Option 1 (watch mode) not pursued — remains a candidate for its own future plan if manual checkpointing proves insufficient. `#[ignore]`-gated integration test `carryctx-cli/tests/hooks_test.rs::test_hooks_install_refuses_under_jj_colocation` plus a plain-git regression test; verified against real jj 0.43.0 (confirmed refusal fires, confirmed no hook files are written, confirmed plain Git hooks install is unaffected).

## 5. Explicit non-goals

- No jj-native (non-colocated, no `.git/`) support. That would require a second, entirely separate storage-reading path (jj's own backend, not Git's), a much larger undertaking with a currently-unclear user base size. Revisit only if colocated support proves insufficient for real users.
- No jj as a `Cargo.toml` dependency. Every interaction stays at the CLI-shelling-out level, matching how CarryCtx already treats Git, for the same reasons (stay aligned with the user's own jj version and config, don't vendor a VCS library).
- No changes to CarryCtx's core task/session/progress/checkpoint data model. This plan only touches the Git-snapshot-capture and worktree/hooks adapters.

## 6. Sequencing relative to other work

Phase 1 can land any time, independent of everything else (it's a plain bug fix plus an informational doctor check). Phases 2-4 should be sequenced as their own tracked tasks once this plan is approved, each phase behind its own PR and CHANGELOG entry, following the same "verify against the real tool before claiming it's fixed" discipline used for the MCP/task-dependency fixes in `carryctx-cli` v0.3.2. Do not batch all four phases into one PR; each is independently shippable and independently risky enough to review on its own.
