# Merge Milestone Verification (CTX-0146) — 2026-09-11

Tool version: `carryctx 0.9.1` (branch base `8857867`). Branch `carryctx/ctx-0146`
in `carryctx-cli`. All fixtures are disposable Git repositories under the system
temp directory; every test runs offline.

Toolchain: `rustc 1.98.0 (88d9e12ae 2026-08-18)`, `cargo 1.98.0 (797e8a9bc 2026-08-05)`,
`git 2.55.0`.

## Verdict

- **PASS-WITH-NOTES.** The merge milestone acceptance criteria AC1–AC9 are
  covered by executable tests; two real production defects in the agent-alias
  path were found and fixed during the task and its independent review.
- **Defect 1 (found and fixed during authoring):** the pure merge engine's
  agent-name aliasing did not remap `team_members.agent_id`. Because the
  composite team-member key ends in the agent id, an aliased incoming agent
  left a dangling reference and the candidate load failed with
  `team_members.agent_id FOREIGN KEY constraint failed` (`DATABASE_ERROR`),
  aborting an otherwise-clean merge. Fixed by adding
  `("team_members", "agent_id")` to `AGENT_REFERENCE_COLUMNS` in
  `crates/carryctx-pack/src/merge/mod.rs`, with an engine regression test and a
  CLI-boundary matrix test.
- **Defect 2 (found in independent review, fixed here):** the defect-1 remap
  could itself collide. When the alias survivor and the aliased loser are
  **both** members of the same team, rewriting the loser's `agent_id` makes two
  `team_members` rows share `(project_id, team_id, agent_id)`, and the candidate
  insert failed `UNIQUE constraint failed: team_members.project_id,
team_members.agent_id, team_members.team_id` (`DATABASE_ERROR`, exit 5). Fixed
  by adding `reconcile_team_members_after_remap` (design §2.3), which keeps one
  row per composite identity — LWW on `updated_at`, then canonical-frame and
  identity-key tie-breaks (mirroring `lww`, so commutative and idempotent) — and
  records a `team_member_alias` auto-resolution rather than a blocking conflict.
  Keeping the survivor-keyed row also keeps the
  `teams(project_id, id, commander_agent_id) -> team_members` foreign key valid
  when the collision is the commander membership. Regression tests cover both
  the plain dual-membership and the commander case at the engine and CLI
  boundaries.
- **New tests:** 17 (12 in `tests/merge_matrix_test.rs`, 2 in
  `tests/git_e2e_test.rs`, 3 engine unit tests in
  `crates/carryctx-pack/src/merge/tests.rs`).

## Commands and outcomes

```bash
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
CARGO_BUILD_JOBS=2 cargo test --workspace --no-fail-fast
just ci
```

<!-- RESULTS -->

All gates passed:

- `cargo fmt --check` — clean.
- `cargo clippy --workspace --all-targets -- -D warnings` — clean.
- `CARGO_BUILD_JOBS=2 cargo test --workspace --no-fail-fast` — **711 passed, 0 failed, 3 ignored**.
- `just ci` (fmt-check, lint, typecheck, test, markdownlint, actionlint, package-smoke) — **exit 0**.
- New tests: **17** (12 `merge_matrix_test`, 2 `git_e2e_test`, 3 engine unit tests).

> Note: an initial full-suite run was executed while a second `cargo test`
> process from another worktree was active; both share global `/tmp` fixture
> names, which produced four spurious failures in unrelated suites
> (`agent_identity_test`, `checkpoint_test`, `concurrency_test`). A clean serial
> re-run with no concurrent test process passed fully, as recorded above.

## AC-by-AC verdict

| AC   | Verdict        | Primary evidence                                                                                                                                                                                                                                                             |
| ---- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AC1  | PASS (note)    | `merge_matrix_test::ac1_v2_snapshot_round_trips_parents_sequences_and_tombstones`; existing `import_test` v1 fresh/replace + `import_merge_test::merge_v1_bundle_degrades_with_warning`. A "v1 reader refuses v2" is not representable: the binary ships one forward reader. |
| AC2  | PASS           | existing `import_merge_test::merge_disjoint_edits_applies_union_and_records_provenance`; `merge_snapshot_test` two-clone tests                                                                                                                                               |
| AC3  | PASS           | existing `merge_strict_edits_stages_conflict_and_leaves_live_db_untouched`; `conflict_test` show/list                                                                                                                                                                        |
| AC4  | PASS           | existing `conflict_test::conflict_resolve_*`, `conflict_apply_*`, `conflict_abort_*`                                                                                                                                                                                         |
| AC5  | PASS           | existing delete policies; new `ac5_delete_then_export_tombstones_cascaded_children`                                                                                                                                                                                          |
| AC6  | PASS + 2 fixes | existing renumber/alias; new `ac6_renumber_keeps_dependency_edge_and_sequence_floor`, `ac6_agent_alias_remaps_references_across_all_tables`, `ac6_agent_alias_dedups_dual_team_membership`, `ac6_agent_alias_dedups_commander_membership`; engine alias + dedup regressions  |
| AC7  | PASS           | new `ac7_events_union_is_byte_stable_and_never_mutated`; engine append-only tests                                                                                                                                                                                            |
| AC8  | PASS           | new offline `git_e2e_test::offline_two_clone_clone_fetch_merge_snapshot_cycle`; existing `snapshot_ref_test` + `merge_snapshot_test`                                                                                                                                         |
| AC9  | PASS           | new `ac9_invalid_and_newer_bundles_leave_db_and_ref_untouched`; existing journal/backup tests                                                                                                                                                                                |
| AC10 | PARTIAL        | workspace fmt/clippy/test gates pass here; markdownlint/contract-version/doc coupling belong to CTX-0147                                                                                                                                                                     |

## Conflict matrix (§2.3) — CLI boundary

`row_edit` (auto + `--strict-edits`), `delete_vs_edit`, `unique_key`,
`display_id_collision`, `agent_name_collision`, `identity_mismatch`,
`redacted_bundle`, `base_required_missing` were already covered. Added CLI
coverage for the three that were engine-only: `status_gap`
(`matrix_status_gap_blocks_at_cli`), `immutable_edit`
(`matrix_immutable_edit_blocks_at_cli`), and `dependency_kind`
(`matrix_dependency_kind_auto_resolves_at_cli`).

### `agent_name_collision` residual (resolved)

The alias path is an auto-resolution, not a blocking conflict; that semantics is
unchanged. Two defects in it are fixed: the missing `team_members.agent_id`
remap (defect 1) and the duplicate-membership UNIQUE collision the remap could
create (defect 2). Defect 2 is registered as a `team_member_alias`
auto-resolution. After the fix there is **no known residual gap** in the alias
path: the survivor and loser can both be members of the same team, and the
collision may involve the commander membership, with exactly one deterministic
row kept and all foreign keys satisfied.

## Known gaps / notes

- The optional ssh/podman transport e2e in design §6 was **not run**; it is
  explicitly optional and no podman dependency is asserted. The offline
  two-clone cycle is covered by `git_e2e_test`.
- `AC1`'s "v1 reader refuses a v2 bundle" cannot be exercised against the
  shipped binary because the reader is forward-compatible (v1 is treated as
  implicit v2). The newer-format refusal is covered at the format layer and via
  `import_test::future_format_version_refuses_as_unsupported` and the new
  merge-mode `ac9_...`.
- `AC10` documentation coupling is owned by CTX-0147; this task verifies only
  the Rust quality gates.
- Kill-during-apply recovery is covered by the existing restore-journal tests
  (`merge_swap_journal_*`, `conflict_apply_journal_recovery_installs_staged_candidate`,
  `apply_retry_after_mid_apply_failure_appends_exactly_one_merged`) and the new
  AC9 test's ref/DB-untouched assertions.

## Files

- `carryctx-cli/tests/merge_matrix_test.rs` (new)
- `carryctx-cli/tests/git_e2e_test.rs` (new)
- `carryctx-cli/crates/carryctx-pack/src/merge/mod.rs` (two alias-path fixes)
- `carryctx-cli/crates/carryctx-pack/src/merge/tests.rs` (regression tests)
