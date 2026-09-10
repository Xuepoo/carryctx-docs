# Mergeable, Git-Managed CarryCtx State (Merge Milestone Design)

**Status:** Accepted for implementation (design PR #12, 2026-09-10).
Implementation is merged to `carryctx-cli` main but not yet released
(CTX-0139–CTX-0144 merged; CTX-0145 merged snapshot commits, CTX-0146 e2e, and
the DEC-0052 local-only ref guard PR #154 still in flight; CTX-0147 owns the
docs sync in `carryctx-docs`).

**Task:** CTX-0138 (`carryctx-cli`; commander `cmd-001`). This is Phase 2 of
`design/2026-09-09-ctxpack-export-import.md`, which deferred merge, three-way,
DAG, and conflict UX, and it supersedes the "wait for observed conflicts before
designing merge" stance in `ROADMAP.md` under the user directive that all
workflows live under Git management.

**Goal:** make CarryCtx state mergeable across parallel worktrees, cloned
branches, and machines: row-level identity and delete semantics, a three-way
merge with the base taken from the ctxpack DAG, conflict staging and
resolution, and a local-only snapshot Git ref (`refs/carryctx/local`) that
records one commit per state snapshot with the same parent edges as the
manifest.

**Directive (2026-09-10):** all workflows under Git management; the CarryCtx
state should be git-like and mergeable — parallel worktrees and branches must
merge their state.

---

## 0. Context and binding constraints

Verified state of `carryctx-cli` on 2026-09-10 (0.9.1), before this design:

- `import --mode merge` refuses with `UNSUPPORTED_OPERATION`
  (`crates/carryctx-cli/src/application/import.rs::resolve_mode`).
- The v1 manifest already carries `parents: []` and `sequences: {}`;
  readers shape-check and otherwise ignore them
  (`crates/carryctx-pack/src/manifest.rs`).
- Replace import builds a candidate database and swaps it atomically through
  the restore journal; rows insert through a column whitelist that fails
  closed on unknown columns (`import.rs::insert_row`).
- `reconcile_sequences` already computes `max(display_id) + 1` per kind and
  never rewinds; import preserves ULIDs.
- `events` is append-only by trigger and historical payloads are preserved
  byte-for-byte.
- Hard deletes exist for `task_dependencies`, `scopes`, `worktrees`,
  `team_members`, and `project prune` (which archives, then deletes tasks and
  child rows). There are no tombstones, so a merge cannot distinguish "deleted
  on one side" from "never seen on this side".
- The `bitty-terminal` repositories publish redacted, publish-only ctxpack
  snapshots to `<repo>-workflow` mirrors through a custom script
  (`bitty-devtools/scripts/publish-ctxpack.sh`); those mirrors are explicitly
  not merge sources.

Binding constraints (inherited, not renegotiated):

1. **Zero network in the binary.** Local Git objects are fine; `git push` and
   `git fetch` stay user transport (`architecture/state-transport-boundary.md`
   §4, `architecture/zero-network-policy.md`).
2. **Fail closed.** A failed or conflicted operation leaves the live database
   and refs untouched. Whole-state replacement keeps requiring `--yes`;
   merge authors changes from validated rows and takes a verified pre-merge
   backup, so it does not.
3. **Append-only audit is never rewritten.** Merge appends `project.merged`
   and auto-resolution events in the same transaction as the state change.
4. **ULIDs are identity; display ids are allocator artifacts.**
5. **Public contracts move together.** Envelopes, exit codes, config keys, and
   format versions are documented in the same change as implementation.

---

## 1. Merge model

### 1.1 Row identity

Every mergeable row is keyed by a stable identity that survives transport:

| Kind                | Identity key                                               |
| ------------------- | ---------------------------------------------------------- |
| ULID tables         | `id` (ULID; never regenerated on import)                   |
| `team_members`      | `(team_id, agent_id)` — 2-part tombstone `row_id` form     |
| `graph_edges`       | `(source_id, target_id, relation_type)`                    |
| `task_dependencies` | `id`, plus semantic edge `(task_id, prerequisite_task_id)` |
| `scopes`            | `id`, plus semantic key `(task_id, pattern)`               |
| `projects`          | single row per project                                     |
| `sequences`         | `(project_id, kind)` — derived, reconciled separately      |
| `tombstones`        | `(project_id, table_name, row_id)`                         |

Composite identity keys MUST match the storage delete-path convention
byte-for-byte. The tombstone `row_id` is `canonical_composite_row_id(...)` — a
JSON array of the key components — and omits `project_id`, because the
tombstone's own `project_id` column scopes it. A merge always runs within one
project (`project_id` is constant), so `team_members` is keyed by
`(team_id, agent_id)`. An engine key that disagrees with the tombstone
`row_id` never matches a deletion and would resurrect a deleted row.

Display ids (`CTX-xxxx`, `DEC-xxxx`, `PX-xxxx`, `HO-xxxx`) are **not** identity.
When two independently allocated rows collide on a display id during merge,
the incoming row is renumbered from the target's sequence and an audit event
plus a warning records the change. References that matter (dependencies,
handoffs, events) point at ULIDs and survive renumbering; free-text mentions
in notes do not, and that is documented, not repaired.

### 1.2 Update semantics

The base/ours/theirs comparison decides _whether_ a row changed; election
decides _which value wins_ when both sides changed it. Defaults:

- **Row-level last-writer-wins by `updated_at`**, parsed as RFC 3339 instants.
  Rows are elected whole; fields are never mixed between sides.
- **NULL-union for monotonic facts** after election: if the winner has NULL
  and the loser a non-NULL value for `started_at`, `completed_at`,
  `accepted_at`, `declined_at`, `ended_at`, or `decisions.superseded_by`, the
  non-NULL value is kept. Setting a fact is never lost to a NULL.
- **Deterministic tie-break:** equal timestamps fall back to canonical row
  JSON (sorted keys, machine-local columns removed) and then `id`, so the
  result does not depend on which side is "ours".
- **Task status lattice** (checked before election): a terminal status
  (`completed`, `cancelled`) beats a non-terminal one; `completed` versus
  `cancelled` is a blocking conflict; otherwise election applies.
- **Auto-resolutions are recorded**, never silent: each LWW election where
  both sides changed is written to the merge report and appended as a
  `merge.auto_resolved` audit event. `--strict-edits` promotes every such
  election to an open conflict for review.
- **Clock skew caveat:** `updated_at` is machine-local; a badly skewed clock
  can win an election it should lose. This is accepted for row edits because
  the loser is recorded and recoverable; destructive semantics (deletes,
  terminal status) never rely on timestamps alone.

Tables without `updated_at` (`task_dependencies`, `scopes`, `graph_edges`) are
immutable-after-create: identical keys union, differing contents are a
blocking `immutable_edit` conflict.

### 1.3 Deletes and tombstones

Hard deletes lose the information merge needs. The milestone adds a
`tombstones` table (schema 0018):

```sql
CREATE TABLE tombstones (
  project_id  TEXT NOT NULL,
  table_name  TEXT NOT NULL,
  row_id      TEXT NOT NULL,   -- PK value, or canonical composite key
  deleted_at  TEXT NOT NULL,
  deleted_by  TEXT,
  reason      TEXT,
  PRIMARY KEY (project_id, table_name, row_id)
);
```

Rules:

1. Every hard-delete path writes tombstones for the deleted row **and every
   row deleted by cascade or unlink** in the same transaction (scopes,
   dependencies, worktree unbind, team-member removal, `project prune` and
   its children). A delete-then-export parity test enforces coverage.
2. Tombstones are exported (`tombstones.jsonl`), union by key, and keep the
   earliest `deleted_at`.
3. Merge delete rules:

   | base   | ours      | theirs    | result                        |
   | ------ | --------- | --------- | ----------------------------- |
   | row    | tombstone | unchanged | delete (tombstone wins)       |
   | row    | tombstone | edited    | **blocking** `delete_vs_edit` |
   | row    | tombstone | tombstone | delete, earliest `deleted_at` |
   | row    | unchanged | tombstone | delete                        |
   | row    | edited    | unchanged | keep edit                     |
   | absent | tombstone | present   | treat as `delete_vs_edit`     |
   | absent | tombstone | absent    | carry tombstone forward       |

4. **Absence without a tombstone is not a delete.** Rows missing from a
   bundle and absent from base are "unknown"; ours is preserved. This keeps
   pre-tombstone v1 bundles safe at the cost of not propagating legacy
   deletes.
5. Tombstone retention is explicit: nothing prunes them automatically;
   `doctor` reports counts and a future `doctor --prune-tombstones` may
   define a merge-history window.

`progress_items` already soft-delete through `status = 'removed'`, which is
authoritative for that table; session `abandoned`, task `cancelled`, and
handoff `closed` are statuses, not deletes.

### 1.4 Append-only tables

`events`, `checkpoints`, and `checkpoint_corrections` are immutable and merge
by union on `id`. Identical ids must carry identical content; a mismatch is a
tamper signal that fails the bundle with `VALIDATION_FAILED` before any write.
No conflict kind exists for these tables. Merging appends exactly one
`project.merged` event, plus `merge.auto_resolved` and
`merge.display_id_renumbered` events for the changes above.

### 1.5 Sequences and watermarks

`sequences.next_value` is a per-kind floor derived from allocated display ids.
Merge takes the maximum across base/ours/theirs and then re-runs
`reconcile_sequences`, so counters never rewind and renumbered rows can never
collide with future allocations. `sequences.kind` entries for both sides'
task prefixes are kept.

Manifest v2 optionally adds `watermarks` (per table: exported row count and
the maximum `updated_at`/`occurred_at` observed). Watermarks are an
optimization and sanity check only — they can short-circuit "unchanged since
base" comparisons and validate counts, but merge correctness never depends on
them.

### 1.6 Project identity

`project_id` must match between local state and bundle; a mismatch refuses
with `STATE_CONFLICT` (never silently fork identity, matching fresh/replace
behavior). On merge:

- `projects.repository_root` and `projects.git_common_dir` are re-anchored to
  the target; the incoming anchors are never compared.
- `name` and `task_prefix` drift (a bundle from a divergent config) produces
  a warning; identity columns are kept from ours, and both display-id
  prefixes are tracked in `sequences`.
- `.carryctx/config.toml` with a different `project.id` refuses, as today.

### 1.7 Machine-local columns

These columns are excluded from change detection and never overwritten from
the incoming side: `projects.repository_root`, `projects.git_common_dir`,
`worktrees.normalized_path`, `worktrees.git_common_dir`, and
`sessions.working_directory` (kept as history, not compared). Worktrees are
machine-local records: ours win, incoming rows are kept only when their path
exists at the target and the active-task slot is free; otherwise they are
pruned and audited exactly like replace import. Merge must reconcile dangling
references before insert (pruned `worktrees.id` and aliased agent ids can
appear in `sessions.worktree_id`, `sessions.agent_id`,
`tasks.owner_agent_id`, `handoffs.from_agent_id`/`to_agent_id`,
`events.actor_agent_id`, `teams.commander_agent_id`) by nulling or remapping
them with warnings — the same technique the loader already uses for dangling
`events.task_id`.

### 1.8 Schema and format compatibility

| Bundle format | Bundle schema | Local schema | Result                                                                                              |
| ------------- | ------------- | ------------ | --------------------------------------------------------------------------------------------------- |
| v1            | any ≤ local   | current      | accepted as implicit v2: `parents = []`, no tombstones                                              |
| v2            | equal         | current      | full merge                                                                                          |
| v2            | older         | current      | rows insert through the column whitelist; missing columns get defaults; unknown columns fail closed |
| v1 or v2      | newer         | current      | `UNSUPPORTED_OPERATION` (exit 10); no writes                                                        |

`schema_version` and `format_version` stay independent. Format v1 bundles keep
importing unchanged; only merge quality degrades (no DAG, no deletes). A future
format v3 must ship a v2→v3 migrator before any writer emits it.

---

## 2. Three-way merge

### 2.1 Base acquisition

`base = merge-base(ours, theirs)` over the export-id DAG:

1. **ours** is the live database; its position is `snapshot_state.last_export_id`
   (recorded by `export --snapshot`) or an explicit `--base`/ref override.
2. **theirs** is the bundle; its parents are `manifest.parents` (v2).
3. The DAG lives in the local snapshot ref (`refs/carryctx/local`): each
   commit's trailers carry its `export_id` and parents, so ancestor snapshots
   are readable with Git plumbing and no network.
4. The **newest common ancestor** wins. Criss-cross histories pick the
   greatest export id deterministically; recursive merge bases are out of
   scope.
5. Fallbacks, in order: explicit `--base <dir|export-id|git-ref>`; local
   snapshot cache (`<state-dir>/snapshots/<export_id>/`); otherwise a
   **base-less 2-way merge** that applies the same row policies but can only
   detect conflicts through tombstones and structural keys. The envelope and
   merge report state `base: null` and `degraded: true`; `--require-base`
   refuses instead (exit 8).

### 2.2 Algorithm

```text
1. validate bundle (existing fail-closed reader), identify project
2. resolve base; materialize base/ours/theirs row sets
3. canonicalize rows (sorted keys, machine-local columns removed)
4. per table, per identity key: classify as unchanged / ours-only /
   theirs-only / both-changed / deleted / delete-vs-edit
5. apply policy table -> MergePlan {writes, deletes, renumbers, aliases,
   auto_resolutions, conflicts}
6. build candidate database: fresh DB + merged rows in LOAD_ORDER, with
   reference reconciliation before insert
7. if conflicts: write merge session (candidate + conflicts), leave live DB
   and refs untouched, exit MERGE_CONFLICTS (3)
8. if clean: verified pre-merge backup, journal, atomic swap,
   project.merged + auto-resolution events, optional merged snapshot commit
```

Determinism requirements: step 4/5 are commutative and idempotent —
`merge(A,B) == merge(B,A)` and `merge(M,M) == M` — and are enforced by the
pure-engine fixture matrix (CTX-0141).

### 2.3 Conflict taxonomy and default policy

| Kind                    | Trigger                                           | Default                                  |
| ----------------------- | ------------------------------------------------- | ---------------------------------------- |
| `row_edit`              | both sides edited the same row                    | auto LWW + recorded resolution           |
| `status_gap`            | `completed` vs `cancelled` on a task              | blocking                                 |
| `delete_vs_edit`        | tombstone one side, content edit the other        | blocking                                 |
| `unique_key`            | colliding unique key that is not display-id/agent | blocking (e.g. two teams named `core`)   |
| `immutable_edit`        | same key, different content, immutable table      | blocking                                 |
| `display_id_collision`  | different ULIDs occupy one display id             | auto renumber incoming + audit           |
| `agent_name_collision`  | same `(project_id, name)`, different ULIDs        | auto alias incoming to ours + remap refs |
| `dependency_kind`       | same edge, `strong` vs `informational`            | auto: `strong` wins                      |
| `identity_mismatch`     | `project_id` differs                              | refuse `STATE_CONFLICT` (3)              |
| `redacted_bundle`       | manifest `redacted: true`                         | refuse `UNSUPPORTED_OPERATION` (10)      |
| `base_required_missing` | `--require-base` and no ancestor                  | refuse `VALIDATION_FAILED` (8)           |

`--strict-edits` turns `row_edit` auto-resolutions into blocking conflicts.

### 2.4 Merge sessions

Staging lives outside SQLite under the project state directory, so an
unresolved merge never leaks into normal queries:

```text
<git-common-dir>/carryctx/merges/<merge_id>/
├── merge.json        # ids, base, source ref/dir, counts, status, degraded flag
├── conflicts.json    # conflict records + resolutions (append-only list)
├── candidate.sqlite  # merged result, conflicts left at "ours"
└── theirs/           # materialized incoming bundle (dir or from-git)
```

One active merge per project; a second `--mode merge` refuses until
`conflict apply` or `conflict abort`. `doctor` reports stale sessions
(unapplied, older than a threshold) so crashes are visible. `--dry-run` runs
steps 1–5 and writes nothing.

### 2.5 Resolution UX

```bash
carryctx import ./bundle-b --mode merge            # stages conflicts, exit 3
carryctx import refs/remotes/origin/carryctx-local --mode merge --from-git
carryctx conflict list [--all] [--merge <id>]
carryctx conflict show <conflict-id> [--format json]
carryctx conflict resolve <conflict-id> --ours|--theirs [--set field=value]
carryctx conflict apply [--merge <id>] [--skip-open]   # atomic swap
carryctx conflict abort [--merge <id>]
```

`conflict list` reports open conflicts plus auto-resolutions (`--all`);
`show` prints base/ours/theirs rows and the policy reason; `resolve` appends
a resolution; `apply` refuses while open conflicts remain unless
`--skip-open`, then swaps the candidate, appends `project.merged`, cleans the
staging directory, and (with `--snapshot-ref`) writes the merge commit;
`abort` deletes the session with no database change. No `--yes` is required:
merge composes state and takes a verified pre-merge backup; `--yes` remains
reserved for `replace` and other whole-state discards.

### 2.6 Errors and exit codes (public API)

| Condition                                  | code                    | exit |
| ------------------------------------------ | ----------------------- | ---- |
| Bundle format/schema newer than reader     | `UNSUPPORTED_OPERATION` | 10   |
| Redacted bundle used as a merge source     | `UNSUPPORTED_OPERATION` | 10   |
| Bundle invalid / tamper / `--require-base` | `VALIDATION_FAILED`     | 8    |
| Project id mismatch / conflicting mode use | `STATE_CONFLICT`        | 3    |
| Conflicts staged; live DB untouched        | `MERGE_CONFLICTS`       | 3    |
| `conflict apply` with open conflicts       | `MERGE_CONFLICTS`       | 3    |
| No active merge session for `conflict *`   | `RESOURCE_NOT_FOUND`    | 7    |
| Git ref missing / not a repo               | `GIT_ERROR`             | 4    |
| Unexpected SQLite failure during staging   | `DATABASE_ERROR`        | 5    |

`--dry-run` writes nothing; JSON errors carry `details.mergeId` and
`details.conflicts` where applicable.

---

## 3. Git integration

### 3.1 Local snapshot ref layout

One local-only ref per clone, `refs/carryctx/local`, sharing the repository's
common Git directory across worktrees. **One commit per snapshot**, with the
pack directory at the commit root (so `git show <ref>:tasks.jsonl` works and
`git diff` between snapshots reads as a state diff):

```text
c3  merge snapshot (parents c2, c9)   chore(ctxpack): merge 01M... (01M... + 01M...)
c2  snapshot (parent c1)              chore(ctxpack): snapshot 01M... (main @ abc1234)
c1  initial snapshot (no parent)      chore(ctxpack): snapshot 01M...
```

The ref deliberately lives outside `refs/heads/*` (DEC-0052, issue #138): a
plain `git push` — even `push --all` — cannot move it, so publishing
unredacted state requires an explicit user refspec. The public redacted
publication ref `refs/heads/carryctx-snapshots` is a _different_ ref reserved
for the redaction/publication flow; unredacted `export --snapshot` refuses it.
CarryCtx never pushes any ref.

Each commit message carries trailers used to reconstruct the DAG without any
index file:

```text
CarryCtx-Export-Id: 01M...
CarryCtx-Parents: 01M...,01M...
CarryCtx-Source: <repo>@<short-sha> (branch)
```

Commits are created with Git plumbing (`hash-object`, `mktree`,
`commit-tree`, `update-ref` with compare-and-swap) so no index or worktree is
mutated and concurrent worktrees cannot corrupt each other. Creating the ref
is a local Git object write; pushing it is user transport, exactly as the
state-transport boundary allows.

### 3.2 Export

```bash
carryctx export --pack-format dir -o ./pack/ --snapshot \
    [--snapshot-ref refs/carryctx/local]
```

After the bundle is written and validated, the commit is created with parent
= current ref tip (if any); `manifest.parents` records the tip's export id so
the bundle is self-describing even outside Git. `snapshot_state.last_export_id`
and `last_snapshot_commit` are updated. A plain `export` without `--snapshot`
stays `parents = []`.

### 3.3 Clone and branch history

A clone gets `refs/remotes/origin/carryctx-local` when the local ref has been
pushed with an explicit refspec, and `git fetch` (user-run, like any
transport) brings updates. State
is per repository, not per code branch: worktrees of one clone share the
SQLite database, so there is nothing to merge between them; merges happen
between clones/machines (or a clone and a transported bundle). Because the
ref is orthogonal to code branches, `source.git_branch` in the manifest
records which code line an export came from.

### 3.4 Import from a Git ref

```bash
carryctx import --from-git <ref> --mode merge [--base <ref>] [--snapshot-ref <ref>]
```

The bundle is materialized from the ref's tree into the merge staging
directory using the fixed `PACK_TABLE_FILES` list, then runs the normal
validate/merge path. Ancestor snapshots for base resolution are read from the
same ref's history. This is fully offline.

### 3.5 Relation to native `git merge`

`git merge` of snapshot commits is **unsupported and documented as such**.
Line-level JSONL merges ignore ULID identity, unique display ids, sequences,
tombstones, and referential order, so a text-merged tree can be internally
inconsistent. CarryCtx owns the semantic merge and writes the merge commit
itself; users who want the Git-level operation followed by state merge run
`carryctx import --from-git ... --mode merge` and then `export --snapshot`,
which produces the two-parent commit. A text-merged snapshot that reaches
`import` is caught by validation only where it violates structural invariants;
that is a safety net, not a supported path.

### 3.6 Redaction and public repositories

Snapshots on the local ref are unredacted by definition; redaction destroys
merge fidelity (`***REDACTED***` would overwrite real values). Policy
(DEC-0052, issue #138):

- The local unredacted snapshot ref lives under `refs/carryctx/` (default
  `refs/carryctx/local`), deliberately outside `refs/heads/*`: a plain
  `git push` (even `--all`) cannot move it, and CarryCtx never pushes it.
  Publishing unredacted state requires an explicit user refspec.
- Public redacted publication uses a dedicated branch
  `refs/heads/carryctx-snapshots` (publication flow). The refs stay distinct:
  unredacted export refuses `refs/heads/carryctx-snapshots` and every other
  `refs/heads/*` target with `INVALID_ARGUMENTS`.
- **Never push an unredacted snapshot ref to a public repository.** Use a
  private state remote, an encrypted transport, or exchange pack directories
  directly.
- Redacted bundles (`manifest.redacted: true`) are publication artifacts and
  are refused as merge sources; fresh/replace import may still accept them.
- A guard test in CTX-0144 asserts the local-only ref never uses or moves the
  public ref and cannot be published by a default `git push`.

### 3.7 Status of CTX-0122

CTX-0122 ("publish project state snapshots to `carryctx-snapshots` branch")
was closed unmerged (PR #134). Its redaction and redacted-publication behavior
folds into a publication follow-up after CTX-0144/CTX-0145, using the distinct
public ref `refs/heads/carryctx-snapshots` (DEC-0051 #7, DEC-0052, issue #138).
The `git_snapshot.rs` domain type in `carryctx-core` is a different concept
(per-checkpoint worktree state).

---

## 4. Rollout for the bitty-terminal repositories

Current mechanism: seven repositories (`bitty`, `bitty-docs`,
`bitty-website`, `bitty-devtools`, `bitty-mcp`, `bitty-plugin-sdk`,
`bitty-plugin-template`) publish redacted ctxpack directories to their
`<repo>-workflow` mirrors on the mirror `main` branch, one
`<UTC-timestamp>-<sha>/` directory per merge, with `LATEST`, `source.json`,
an export-time redaction pass, and a round-trip self-test. Mirrors are
publish-only; merge-back is unsupported.

**Decision: coexist now, converge the format later — and never make a
public mirror a merge source.**

| Stage | Scope                                             | Action                                                                                                                                                                                                                                                |
| ----- | ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0     | now–v0.10                                         | No bitty change. Merge milestone is opt-in for CarryCtx itself; mirrors untouched.                                                                                                                                                                    |
| 1     | v0.10 release                                     | Pilot the snapshot ref on `carryctx-cli` (dogfood). For bitty repos, use the ref only to a private remote or not at all; keep exchanging pack dirs (scp/Syncthing) as today.                                                                          |
| 2     | after one release cycle, if the ref proved useful | Converge publishing: `publish-ctxpack.sh` derives the redacted workflow snapshot from the same export that was committed to the ref, so mirror and merge history share one `export_id`; mark the manifest `redacted: true`. Retire the second export. |

Rationale for coexist instead of an immediate switch: the mirrors are the
review and privacy surface (public visibility + redaction + self-test) and a
merge transport must be unredacted; collapsing them now would either publish
unredacted state or break merge fidelity. Converging at the format level
(v2 manifests, parents, tombstones) keeps one serialization while two
audiences are served.

Migration notes:

- Publisher must run carryctx ≥ 0.10.0 (v2 writer); v1 snapshot directories
  remain readable and are never rewritten.
- The redaction script must treat `tombstones.jsonl` like any other JSONL and
  stamp `redacted: true` into `manifest.json` without changing counts.
- `LATEST`, `source.json`, and the directory layout stay unchanged, so mirror
  consumers see no break.
- The snapshot ref is per repository and never pushed to public product repos
  (§3.6); unredacted transport is a private or encrypted channel.
- CarryCtx-repo follow-up work for the bitty rollout (publish-script update)
  belongs to the bitty repositories' own CarryCtx state, not
  `carryctx-cli`.

---

## 5. Acceptance criteria

- **AC1 — v1 compatibility.** Existing v1 bundles import (fresh and replace)
  unchanged; a v1 reader refuses a v2 bundle with `UNSUPPORTED_OPERATION`; v2
  export→import round-trips `parents`, `sequences`, and tombstones.
- **AC2 — three-way convergence.** Two clones with disjoint edits merge to
  the union; exit 0; `project.merged` is appended; task/progress/checkpoint
  counts equal the union.
- **AC3 — conflict staging.** Same-row concurrent edits under `--strict-edits`
  (or a delete-vs-edit) leave the live DB hash-identical, produce
  `MERGE_CONFLICTS` exit 3 with `mergeId`, and `conflict show` displays
  base/ours/theirs.
- **AC4 — resolution.** `conflict resolve --ours|--theirs` + `conflict apply`
  produce exactly the chosen row and clean the session; `abort` leaves the DB
  unchanged.
- **AC5 — deletes.** delete-vs-untouched applies the delete; delete-vs-edit
  conflicts; both-delete keeps the earliest `deleted_at`; delete-then-export
  shows tombstones for every removed row including cascades.
- **AC6 — identity.** Display-id collisions are renumbered with an audit
  event and never corrupt dependencies; agent name collisions alias and
  remap references; sequences never rewind.
- **AC7 — audit.** Events merge by union with byte-stable payloads; no event
  is updated or deleted by a merge.
- **AC8 — git.** `export --snapshot` produces one commit per snapshot with
  correct trailers and parents; `import --from-git` equals importing the same
  directory; a two-clone fetch/merge/snapshot/push cycle over a local remote
  works offline.
- **AC9 — fail closed.** Any failure (invalid bundle, newer format, kill
  during apply) leaves the live DB and refs untouched; the pre-merge backup
  and journal recover on next open.
- **AC10 — contract hygiene.** `cargo fmt --check`, Clippy `-D warnings`,
  `cargo test`, markdownlint, and the contract-version metadata check pass;
  `cli-specification.md`, `configuration.md`, `ROADMAP.md`, `TODO.md`,
  `state-transport-boundary.md` §5, the manual, and `CHANGELOG.md` are
  updated in the same release.

## 6. Test plan

- **Unit (pure engine, CTX-0141):** fixture matrix over
  base/ours/theirs — unchanged, one-sided edits, both-sided edits, status
  lattice, NULL-union, tombstone cases, immutable tables, composite keys,
  unique-key collisions, display-id renumbering, agent aliasing, sequence
  floors; commutativity and idempotence properties.
- **Format (CTX-0139):** manifest v1/v2 validate-reject matrix; v1→v2
  migration; counts with tombstones; tampered parents.
- **Storage (CTX-0140):** delete-then-export tombstone parity; migration
  backup gate; FK integrity.
- **Integration (CTX-0142/0143/0146):** v1 and v2 round-trip; redacted
  refusal; dry-run writes nothing; conflict staging hash check; kill-recovery
  via journal; one-active-merge enforcement; apply/abort/renumber envelopes.
- **Git e2e (CTX-0144/0145/0146):** disposable temp repos with a local bare
  remote (no network): clone A/B, divergent state, snapshot/push/fetch,
  merge, merge commit with two parents, next merge resolves the correct base;
  ref CAS under concurrent updates.
- **Transport e2e (optional, podman):** ssh fixture in workspace
  `recording/`, as in the ctxpack v1 plan; evidence recorded under
  `carryctx-docs/reports/` with commands, versions, and outcomes.

## 7. Alternatives considered

| Alternative                                                | Why rejected                                                                                                                                                    |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Keep SQLite as truth, no snapshots branch (v1 status quo)  | No base, no deletes, no history; parallel machines cannot merge safely.                                                                                         |
| Event-sourced JSONL as primary state, SQLite as projection | Largest possible change; rewrites persistence, queries, migrations, and every command; not needed to satisfy git-like mergeability.                             |
| Native `git merge` over JSONL snapshots                    | Line-level merges violate ULID identity, display ids, sequences, FK order, and tombstones; cannot detect semantic conflicts.                                    |
| CRDTs for every table                                      | High complexity and storage overhead; terminal statuses and deletes need human judgment anyway.                                                                 |
| Per-field LWW (`updated_at` per column)                    | Requires per-field timestamps that the schema does not have and would blend rows from different editors; row election plus NULL-union is simpler and auditable. |
| Conflict markers inside the live database                  | Conflicted state would be queryable by agents and exported accidentally; staging outside SQLite fails closed.                                                   |
| Push snapshot refs automatically from the binary           | Network code and credentials in Core; violates the zero-network boundary.                                                                                       |

## 8. Open questions and decisions needed

DEC-0051 (2026-09-10) ratified the defaults; resolutions are inlined below.

1. **Conflict default strictness.** Is auto-LWW for `row_edit` acceptable as
   the default (with `--strict-edits` opt-in), or should edits block by
   default for lower-surprise merges?
   _Resolved by DEC-0051 #1: auto-LWW by default, `--strict-edits` opt-in._
2. **Status semantics.** Confirm the terminal-wins lattice and that
   `completed` vs `cancelled` is the only blocking status pair.
   _Resolved by DEC-0051 #2: terminal-wins; `completed` vs `cancelled` is the
   only blocking pair._
3. **Tombstones vs `deleted_at` columns.** This design uses a side table;
   confirm that deleting rows should never alter ordinary read queries.
   _Resolved by DEC-0051 #3: tombstone side table, no read-path change._
4. **Snapshot-ref push policy for public repos.** Confirm that unredacted
   snapshots are never pushed to public product repos and that `-workflow`
   mirrors stay redacted, review-only.
   _Resolved by DEC-0051 #4 / DEC-0052: distinct refs — local-only unredacted
   `refs/carryctx/local`, public redacted `refs/heads/carryctx-snapshots`; the
   local ref is never pushed by CarryCtx and is refused the public name._
5. **Base-less merge default.** Degrade to tombstone-aware 2-way with a
   warning, or refuse unless `--base`/`--require-base`?
   _Resolved by DEC-0051 #5: degrade with a warning; `--require-base` refuses
   with `VALIDATION_FAILED` (exit 8)._
6. **New error code.** Is `MERGE_CONFLICTS` (exit 3) acceptable as public
   API, or should it collapse into `STATE_CONFLICT`?
   _Resolved by DEC-0051 #6: `MERGE_CONFLICTS` exit 3._
7. **CTX-0122 disposition.** Re-scope it under CTX-0144/CTX-0145 or close it
   as superseded.
   _Resolved by DEC-0051 #7: PR #134 closed unmerged; redacted publication
   folds into a publication follow-up after CTX-0144/CTX-0145 (issue #138)._
8. **Format v1 support window.** How long must v1 bundles stay mergeable
   (degraded) before readers may require v2?
   _Resolved by DEC-0051 #8: v1 imports supported for one release cycle after
   the v2 writers ship._
9. **Snapshot commit cadence.** One commit per export vs debounced/amended
   commits for high-frequency exports; and whether plain `export` should gain
   `--snapshot` by default in the future.
   _Resolved by DEC-0051 #9: one commit per export, `--snapshot` stays
   opt-in; debouncing revisited later._

## 9. Work breakdown (created in `carryctx-cli`, 2026-09-10)

All tasks were created with agent `cmd-001`, `Priority: P1`,
`Area`/`Labels` lines in their descriptions, and these dependencies:

| Task     | Deliverable                                                                  | Depends on                   |
| -------- | ---------------------------------------------------------------------------- | ---------------------------- |
| CTX-0139 | ctxpack format v2: parents DAG, tombstones, redacted flag                    | CTX-0138                     |
| CTX-0140 | Schema 0018: `tombstones` + `snapshot_state`; delete paths record tombstones | CTX-0138                     |
| CTX-0141 | Pure three-way merge engine: LWW rows, status lattice, base selection        | CTX-0138, CTX-0139, CTX-0140 |
| CTX-0142 | `import --mode merge`: staging, conflict report, atomic apply                | CTX-0141                     |
| CTX-0143 | `conflict list/show/resolve/apply/abort` UX                                  | CTX-0142                     |
| CTX-0144 | local snapshot ref: commit-per-snapshot export, import-from-git              | CTX-0139, CTX-0142           |
| CTX-0145 | Merged snapshot commits after `--mode merge`                                 | CTX-0142, CTX-0144           |
| CTX-0146 | Three-way fixtures + two-clone git e2e + conflict matrix                     | CTX-0142, CTX-0143, CTX-0145 |
| CTX-0147 | Docs sync: cli-spec, configuration, ADR/roadmap/TODO, manual, skill          | CTX-0143, CTX-0145           |

Suggested land order: 0139/0140 → 0141 → 0142 → 0143/0144 → 0145 → 0146 → 0147. Each task follows the repository lifecycle (worktree, branch, PR,
independent review, CI, merge) and keeps docs in `carryctx-docs` coupled to
the public-contract changes.

## 10. References

- `design/2026-09-09-ctxpack-export-import.md` — ctxpack v1 contract, deferred
  Phase 2 (this document completes the deferral).
- `design/002-workspace-crates.md` — `carryctx-pack` owns manifest/format/JSONL
  and is the intended home for pure merge logic.
- `architecture/state-transport-boundary.md` — local Git objects allowed,
  push/fetch are user transport; §5 "merge stays deferred" is superseded here.
- `architecture/zero-network-policy.md` §5 — whole-file `sync` LWW is not a
  merge; conflict-aware flows are this design.
- `ROADMAP.md`, `TODO.md` — merge is promoted from "explicitly later" to the
  current milestone; ctxpack hardening items remain prerequisites or parallel
  work.
- `bitty-devtools/scripts/publish-ctxpack.sh` — current workflow-mirror
  publisher referenced in §4.

## History

| Date       | Change                                                                                                                                                                                                                          |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-09-10 | Initial design: merge model, three-way algorithm, conflict UX, snapshots ref, bitty rollout, task breakdown CTX-0139–CTX-0147.                                                                                                  |
| 2026-09-11 | Implementation status recorded: CTX-0139–CTX-0144 merged to `carryctx-cli` main (unreleased); CTX-0145/CTX-0146 and the DEC-0052 local-only ref guard (PR #154) in flight; all §8 open questions resolved by DEC-0051/DEC-0052. |
