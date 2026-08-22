# Agent-Team Orchestration Design (Commander + Subagents)

**Status:** Implemented in carryctx-cli v0.6.0; revised 2026-08-21 for the Team Coordination proposal, the positioning correction in **DEC-0004**, and commander-design review; see Sections 1.1 and 1.3.

**Date:** 2026-08-21

**Task:** CTX-0037 (gates CTX-0038 through CTX-0041, CTX-0043, CTX-0044; CTX-0042 and CTX-0047 deferred)

**Release:** v0.6.0

**Binding constraint:** CarryCtx is a management tool, not an orchestration framework. Section 1.1 states the boundary and governs every other section. Read it first.

**Goal:** Add durable, queryable task-management primitives that an external commander or harness can use when parallel work is useful, without making parallel execution the default or breaking any existing single-agent workflow.

## Review disposition (2026-08-21)

This review narrows the ownership boundary. CarryCtx records tasks, dependencies, ownership, scopes, progress, checkpoints, decisions, handoffs, and audit events. The execution harness owns process spawning, prompt routing, worktree lifecycle, heartbeats, retries, and concurrency limits. A commander decides whether to work inline, batch tasks, or delegate them; CarryCtx does not encode that judgment as a scheduler or policy engine.

The design keeps assignment/reassignment, dependency-aware read projections, scope-overlap reporting, per-call agent identity, and atomic audited state changes. It now adds one durable, project-scoped Team coordination record and read-only Team projections. Role configuration is advisory metadata only. Context is rebuilt from existing structured records; it is not a token optimizer. Task leases, expiry, orphan classification, reclaim timing, mandatory recording cadence, capacity warnings, and session behavior gates are deferred until an actual harness integration demonstrates that they are required. The detailed lease material below is future design material, not a v0.6 implementation contract.

## 1. Scope

### 1.1 Positioning: a management tool, not an orchestration framework

Read this subsection before adding anything to this design. It is the constraint that decides whether a proposed feature belongs in CarryCtx at all, and it is binding per **DEC-0004**.

CarryCtx is a **management tool**. It records project state and surfaces the information needed to make good dispatch decisions. It is **not** an orchestration framework and does not compete with one. Plugin frameworks — `oh-my-openagent` for opencode, and equivalents for other harnesses — already occupy the orchestration-runtime niche: they spawn workers, route prompts, and own the execution loop. CarryCtx sits underneath them as the shared, durable state substrate that survives any single agent process.

The consequence that shapes Sections 5, 7, and 11: **orchestration is situational, and dispatch granularity is a commander judgment call.** Concretely:

- A very small task may not warrant a subagent at all. The commander doing it inline is often correct.
- Two near-identical tasks, or two tightly coupled ones, are frequently better handled by **one** subagent than fanned out to several. Batching is a legitimate strategy, not a policy violation.
- Two tasks touching the same file are a scope-overlap signal for the commander; CarryCtx does not decide whether they must be serialized or can be batched.

None of those calls is derivable from a static rule, and this backlog demonstrates it in both directions.

- **Should have been batched.** CTX-0035 (stale workspace `AGENTS.md` priorities) and CTX-0036 (README `sync` claims versus the zero-network policy) were dispatched to two separate subagents. Both are small documentation reconciliations, they overlap in subject matter, and one subagent should have taken both. Two were spun up because nothing surfaced their similarity.
- **Needs commander judgment.** CTX-0033 fixes text-format dispatch in `src/output.rs`, which has no match arm for `resume` or `context` and falls through at `src/output.rs:837`. CTX-0034 adds `carryctx task graph` with mermaid, dot, and JSON renderers, which lands in that same per-command match. They carry no dependency edge, so a dependency-only view would miss the overlap.

A tool enforcing "one task per agent" gets the first case wrong. A tool trusting the dependency graph alone gets the second wrong. Both need a commander looking at the graph, the file scopes, and the content — which is why CarryCtx's job is to put all three in front of it, cheaply, and then record the decision.

So the division of labour is:

| CarryCtx does                                                            | CarryCtx does not                                                  |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| Surface claimable work with dependency and scope information             | Decide the fan-out, grouping, or how many workers a wave deserves  |
| Expose the dependency graph and file-scope overlaps between tasks        | Refuse a dispatch because two tasks share a scope                  |
| Report task ownership and activity for inspection                        | Cap how many tasks one agent may hold, or define a capacity policy |
| Record who did what, when, and why, in an append-only audit log          | Spawn, supervise, schedule, or kill agent processes                |
| Surface recorded inactivity and preserve an explicit reassignment audit  | Infer process death, expire claims, or automatically reassign work |
| Protect data integrity: ownership, atomicity, dependency and cycle rules | Encode team-organisation policy as a rejected state transition     |

The asymmetry in the right-hand column is deliberate and load-bearing throughout this document. **Enforcement is reserved for correctness, never for policy.** Ownership checks (Section 4.4), transactional audit writes (I11), dependency gating, cycle prevention, and the single shared database (I17) are all strictly enforced, because violating them corrupts or loses state. How a team chooses to divide its work is advisory, because the tool has strictly less information than the commander does.

When in doubt about a new feature, ask whether it prevents data loss or merely prescribes a working style. The first belongs here; the second belongs in the harness.

### 1.2 In scope and out of scope

This design specifies the primitives required for a commander or external harness to inspect and record work against one project database. It does not specify how workers are spawned, grouped, routed, or run concurrently.

In scope for v0.6:

- A project-scoped persistent Team, minimal membership/commander relations, optional Task association, and read-only `team status` / `team context` projections (Section 1.3).
- A role and identity model layered on the existing `agents` table (Section 3).
- Reassignment of existing tasks, including an explicit authorization model (Section 4).
- A dependency-aware ready queue with stable ordering and topological depth (Section 5).
- Read-only Team and task-relevant context projections assembled from durable records (Sections 3.4, 3.5, and 6). No token budget or cache is promised.
- A minimal declarative configuration schema for subagent roles that degrades gracefully when absent (Section 7).
- Explicit, audited ownership recovery using task state (Section 8). Runtime orphan detection is future work.
- An advisory durability practice describing useful records for workers (Section 9).
- Concurrency invariants for N simultaneous writers (Section 10).
- A resolution for the undelivered `single_active_task_per_agent` policy: multiple active tasks remain permitted and the old documentation is corrected (Section 11).

Explicitly out of scope:

- Process and execution lifecycle. CarryCtx does not spawn, schedule, route, supervise, heartbeat, retry, or kill agent processes. The host harness owns those concerns.
- Runtime Team workers, Team Sessions, prompt caches, copied chat history, model routing, and context-window/token optimization.
- Network and remote synchronization. Consistent with the workspace policy, CarryCtx remains offline-first. Multi-machine agent teams are `future`.
- Automatic task decomposition. The commander decides what tasks exist; CarryCtx records and orders them.
- Dispatch policy. CarryCtx does not decide how many subagents to run, which tasks to batch together, or when a task is too small to delegate. Per Section 1.1 these are commander decisions informed by CarryCtx data.
- Any redesign of the verified primitives listed in Section 2.2.

### 1.3 Team Coordination proposal

The v0.6 coordination boundary is a persistent **Team** owned by the project. A Team is a durable management grouping, not a runtime worker pool. It gives a commander and members a stable coordination identity that survives session boundaries and linked Git worktrees while reusing the existing project records.

The smallest viable model is:

| Concept          | Durable representation                                                    | Meaning                                                                                                   |
| ---------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Team             | One project-scoped `teams` row                                            | Stable coordination group and human-readable name                                                         |
| Membership/role  | `team_members` relation from Team to existing Agent, with advisory `role` | Which registered agents participate and what responsibility label they carry                              |
| Commander        | `commander_agent_id` on Team, referencing an existing Agent               | The agent currently responsible for planning and coordination; it is a relationship, not a new agent kind |
| Task association | Nullable `team_id` on existing Task                                       | Optional link from a task to the Team coordinating it                                                     |

Conceptual schema:

```text
teams(
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(id),
  name TEXT NOT NULL,
  commander_agent_id TEXT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
)
team_members(
  project_id TEXT NOT NULL,
  team_id TEXT NOT NULL,
  agent_id TEXT NOT NULL,
  role TEXT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  PRIMARY KEY (project_id, team_id, agent_id)
)
tasks.team_id TEXT NULL
```

Every association is project-scoped, including associations whose project column is not visible in a CLI projection. The migration MUST add these required keys and foreign keys: `projects(id)` remains the project parent; `agents(project_id, id)`, `teams(project_id, id)`, and `team_members(project_id, team_id, agent_id)` are unique/primary keys; `team_members(project_id, team_id)` references `teams(project_id, id)` with `ON DELETE CASCADE`; `team_members(project_id, agent_id)` references `agents(project_id, id)` with `ON DELETE CASCADE`; and `tasks(project_id, team_id)` references `teams(project_id, id)` with `ON DELETE SET NULL`. A task from project A therefore cannot point at a Team from project B, and a Team from project A cannot contain an Agent from project B. The implementation MUST also retain the single-column primary keys used by existing records and add the composite unique keys needed by these foreign keys.

`teams.commander_agent_id` is nullable, but commander membership is enforceable: `teams(project_id, id, commander_agent_id)` is a foreign key to `team_members(project_id, team_id, agent_id)` when `commander_agent_id` is non-NULL. Team creation therefore inserts the Team with a NULL commander, inserts the member, and assigns the commander in one transaction. Updating or adding a commander must first ensure that the target Agent is a member of that same Team; the composite foreign key is the final database guard. The selected removal policy is **reject**: removing the current commander from `team_members` fails with a state-conflict error and leaves both rows unchanged. Callers must first assign another existing member or explicitly clear the commander through a separate audited Team update; clearing is allowed and leaves the Team temporarily without a commander. No cascade may silently orphan or replace the commander relation.

The schema does not persist session IDs, worktree paths, prompts, chat content, model/provider selection, capabilities, or liveness.

No TeamTask, TeamSession, copied chat history, prompt cache, runtime worker, scheduler, model router, or agent launcher is introduced. Existing Agent, Session, Task, Dependency, Scope, Worktree, Checkpoint, Decision, Handoff, and Event entities remain the source of detail. A Team is a durable index over those records, not a second task or context store.

The following invariants are normative:

- **I22.** A Team belongs to exactly one project and is stored in the authoritative Git-common state database.
- **I23.** Session is execution-scoped. Starting or ending a Session never creates, deletes, or resets Team membership or a Task's `team_id`.
- **I24.** Team membership references existing Agent records. Ending a session, switching sessions, or changing linked worktrees does not remove membership.
- **I25.** A Task may have no Team or one Team; association is optional and does not change Task lifecycle, ownership, dependency gating, or scope semantics.
- **I26.** A Team commander is an existing Agent relation and a member of that same Team/project. A Team has at most one commander at a time; changing or explicitly clearing it is an audited Team state change. Removing the current commander is rejected until the commander is replaced or cleared. Team membership is not implied by the current Session.
- **I27.** Team context is rebuilt from durable Team membership, Task, Dependency, Scope, Worktree, Checkpoint, Decision, Handoff, and Event records. Chat transcripts, prompts, and model context windows are not durable inputs.
- **I28.** Team read projections are read-only. They do not claim Tasks, create Sessions, reserve work, start processes, or mutate liveness.

The Team model does not make parallel execution the default. A commander may work inline, batch related Tasks, delegate, or serialize work. CarryCtx reports facts and records decisions; the harness owns execution.

## 2. Evidence and problem statement

### 2.1 Measured evidence

Measurements taken on real project databases:

| Observation                        | Measurement                                                     | Implication                                                                                            |
| ---------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Dependency graph density (vectojs) | 424 tasks, 396 completed, 26 dependency edges (~6%)             | Almost no graph was expressed, so nothing was schedulable in parallel even in principle                |
| Session count (vectojs)            | 37 sessions for 184 tasks, one agent                            | The sequential re-read pattern is the dominant workflow                                                |
| Handoff direction (vectojs)        | 16 of 18 handoffs are self-directed (agent to itself)           | The multi-agent primitive exists and is effectively unused                                             |
| Context cost (vectojs)             | `resume` ~3356 tokens vs `status` ~51 tokens                    | Every new session pays a full re-read; a dispatched subagent cannot afford this per task               |
| Write concurrency                  | 96 concurrent writes across 24 processes, zero failures, 326 ms | The storage layer already sustains a team; WAL plus `busy_timeout=10000` (`src/adapter/sqlite.rs:140`) |

The 6% edge density and the 89% self-directed handoff rate are the core finding: the product already has multi-agent primitives, but nothing makes expressing a graph or dispatching to a peer cheaper than doing the work sequentially in one long session. This design targets that cost asymmetry, not the absence of features.

### 2.2 Verified working behavior — do not redesign

The following was verified against 0.5.8 and is load-bearing for this design:

- **Dependency-driven promotion.** Completing a prerequisite auto-promotes dependents from `planned` to `ready`.
- **Claim gating.** `task start` against unmet strong dependencies fails with `DEPENDENCY_INCOMPLETE`, exit code 3, error on stderr, stdout clean.
- **Atomic claim.** The conditional update requiring `owner_agent_id IS NULL AND status = 'ready'` maps zero changed rows to `TASK_ALREADY_CLAIMED`.
- **Create-time assignment.** `task create --assignee <agent>` sets `owner_agent_id`.
- **MCP per-call identity.** Passing `--agent` through MCP tool arguments attributes correctly. It is functional but undocumented; CTX-0041 documents it.

### 2.3 Confirmed gaps

1. **No reassignment.** `task edit` has no `--assignee` and no `assign` subcommand exists, though `carryctx task --help` advertises "Create, assign, review, and complete tasks". A commander cannot hand an existing task to an idle subagent.
2. **No dependency-aware ready queue.** `task list --status ready` is a raw status filter. It does not mean "unblocked AND unclaimed AND role-matched", and it exposes no ordering a planner can use to build waves.
3. **No cheap context.** There is no lease or pointer mechanism, so each subagent pays the full `resume` cost measured in Section 2.1.
4. **Session hazard.** `session start` silently auto-ends the agent's prior session ("Auto-ended by new session", `src/application/session.rs:30-36`). If two subagents share one identity, the second destroys the first session with no warning.
5. **Documentation asserts an unenforced policy.** `single_active_task_per_agent` defaults to `true` (`requirements.md:2956`, `configuration.md:302`) but is not enforced. One agent was observed holding 17 in-progress tasks. Section 11 resolves this by correcting the documentation, not by adding enforcement.

### 2.4 The grounding incident

This design is grounded in a real failure, not a hypothetical. The first dispatched attempt at CTX-0037 itself terminated abruptly while holding the task `in_progress`, having recorded zero progress items and zero checkpoints. The observable aftermath:

- All work was lost. Nothing distinguished "an agent is thinking hard about this design" from "the owner is a dead process".
- `doctor` reported only `{"check":"tasks.in_progress","status":"info","message":"17 task(s) currently in progress"}` — a neutral status count that cannot distinguish a live claim from an abandoned one.
- Recovery required a third party to run `task release` on a task it did not own. That succeeded, which exposed the missing ownership check now filed as **CTX-0044**.
- The `TASK_ALREADY_CLAIMED` error suggested "Ask the current owner to release the task" (`cli-specification.md:1066-1075`). That suggestion is unactionable when the owner is a dead process, and it is the only guidance the product offers.

Three requirements follow directly, and they are why Sections 8 and 9 carry the most weight in this document:

- Liveness must be observable, not inferred from status (Section 8).
- Reclaim must be authorized and audited rather than achieved by an unchecked `release` (Sections 4.4 and 8.4).
- Durability must be a contract a subagent is obligated to satisfy incrementally, not an optional courtesy performed at completion time (Section 9).

## 3. Role and identity model

### 3.1 One agent record per concurrent worker

Agents are already first-class records in the `agents` table with `carryctx agent register --name --provider --role`. The orchestration model adds no new identity entity. It adds one hard rule:

> **Invariant I1.** Every concurrently executing worker MUST have its own agent record. Two live processes MUST NOT share one agent identity.

This is not stylistic. Gap 4 in Section 2.3 makes identity sharing actively destructive: `session start` silently auto-ends the prior session for the same agent, so a second subagent booting under a shared identity destroys the first subagent's session with no warning. Enforcing I1 makes that hazard unreachable for correctly configured teams, and Section 8.6 adds a detection path for teams that violate it anyway.

The `agents.role` column already exists and is currently a free-form descriptive string. This design promotes it to a functional field: role participates in ready-queue filtering (Section 5) and in role-declared capability matching (Section 7). Role values remain open strings — consistent with the existing treatment of provider names as open and domain statuses as closed sets — so a project may define any role vocabulary.

### 3.2 Commander and subagent

Two behavioral kinds are distinguished by an agent-level attribute, not by a separate table:

| Kind        | Population                                                                 | Owns                                                                              | Typical commands                                                        |
| ----------- | -------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `commander` | Exactly one recommended per project; more than one is permitted but warned | The plan, the dependency graph, task creation, assignment, and dispatch decisions | `task create`, `task depend`, `task assign`, `task ready`               |
| `subagent`  | N                                                                          | Executing assigned work and recording durable state through existing commands     | `task start`, `progress *`, `checkpoint`, `decision add`, `task review` |

`commander` and `subagent` remain useful descriptive kinds for the existing Agent record and per-call identity. Team coordination does not use them as role admission or as a replacement for the Team commander relation. The Team's `commander_agent_id` is authoritative for that Team; an Agent may be a commander of one Team and a member of another only when explicitly recorded. No role configuration grants permission.

Where the existing authorization design needs an actor classification, it may continue to use the nullable `agents.kind` column:

```text
agents.kind TEXT NULL CHECK (kind IS NULL OR kind IN ('commander','subagent'))
```

`NULL` means unclassified and behaves exactly as 0.5.8 does today: full privileges, no team semantics. This is the mechanism that keeps existing single-agent projects working unchanged (Section 12). A project only acquires team semantics once it classifies at least one agent.

```bash
carryctx agent register --name planner --kind commander --role planning
carryctx agent register --name impl-1 --kind subagent --role implementation
carryctx agent register --name reviewer-1 --kind subagent --role review
```

`agent register` is documented as accepting an existing name to sync the record, so `--kind` and `--role` on an already-registered agent update it in place and append `agent.updated`.

### 3.3 Team view

The Team projections replace the proposed global roster command as the durable coordination surface. `team status` answers "what Team exists, who belongs to it, who is commander, and which associated Tasks are active". `team context` answers "what structured project state is relevant to this Team". They are read-only and work from persisted records; they do not claim to measure process liveness or optimize a context window.

```bash
carryctx team status --json
```

```json
{
  "schema_version": 1,
  "command": "team.status",
  "success": true,
  "data": {
    "team": {
      "id": "01M0HTEAM1P82HPWKJ475KD12W",
      "name": "release-0.6",
      "project_id": "01KY6ZK0TMQM5ANGZ97T68C71G",
      "commander_agent_id": "01M0HNQG1P82HPWKJ475KD12WM",
      "created_at": "2026-08-21T08:30:00Z",
      "updated_at": "2026-08-21T08:30:00Z"
    },
    "members": [
      {
        "agent_id": "01M0HNQG1P82HPWKJ475KD12WM",
        "name": "planner",
        "kind": "commander",
        "role": "planning",
        "active_session_id": "01M0HP7QF4TT0V2Y2W9C5A3XKQ",
        "active_task_count": 0
      },
      {
        "agent_id": "01M0HNR52KP9Q1B7X4D8E2GTYV",
        "name": "impl-1",
        "kind": "subagent",
        "role": "implementation",
        "active_session_id": null,
        "tasks": [
          {
            "display_id": "CTX-0038",
            "status": "in_progress",
            "team_id": "01M0HTEAM1P82HPWKJ475KD12W"
          }
        ],
        "active_task_count": 1
      }
    ],
    "counts": {
      "total": 2,
      "commanders": 1,
      "subagents": 1,
      "unassigned": 0
    }
  },
  "meta": { "timestamp": "2026-08-21T08:30:00Z" }
}
```

The `active_task_count` field is descriptive only. A commander may use it as one input when deciding whether to batch or delegate, but CarryCtx does not define a capacity hint or act on the count. No `liveness`, heartbeat, lease, or expiry field is emitted by Team projections in v0.6.

### 3.4 Minimal Team CLI surface

Team writes are administrative state changes, not execution commands:

```bash
carryctx team create --name <NAME> [--commander <AGENT_REF>] [--json]
carryctx team member add <TEAM_REF> --agent <AGENT_REF> [--role <ROLE>] [--json]
carryctx team member remove <TEAM_REF> --agent <AGENT_REF> [--json]
carryctx team commander set <TEAM_REF> --agent <AGENT_REF> [--json]
carryctx task team set <TASK_REF> --team <TEAM_REF|none> [--json]
carryctx team status [<TEAM_REF>] [--json]
carryctx team context [<TEAM_REF>] [--agent-for <AGENT_REF>] [--task <TASK_REF>] [--json]
```

`team create`, membership changes, commander changes, and Task association changes append `team.created`, `team.member_added`, `team.member_removed`, `team.commander_changed`, and `task.team_changed` respectively in the same transaction as their state change. `--commander` on create atomically creates the initial membership and commander relation. All new mutation responses use `team`, `member`, `commander`, `task`, `previous_team_id`, and `operation` fields as applicable. There is no Team delete in the minimal surface; retaining an empty Team preserves durable history and avoids ambiguous cascading deletion. A future administrative lifecycle may add archive semantics after usage evidence.

The commander target must be a member of the Team. Adding a member does not change Agent, Session, Task ownership, or role configuration. Removing a member does not clear Task ownership or delete durable records; the commander must use existing task assignment/release operations if work needs rerouting.

### 3.5 Team context projection

```bash
carryctx team context [<TEAM_REF>] [--agent-for <AGENT_REF>] [--task <TASK_REF>] [--json]
```

The projection returns structured references and summaries, not copied conversations:

```json
{
  "schema_version": 1,
  "command": "team.context",
  "success": true,
  "data": {
    "team": { "id": "01M0HTEAM1P82HPWKJ475KD12W", "name": "release-0.6" },
    "view": "commander",
    "members": [],
    "tasks": [],
    "dependencies": [],
    "scope_conflicts": [],
    "blockers": [],
    "conflicts": [],
    "latest_checkpoints": [],
    "decisions": [],
    "handoffs": [],
    "recent_events": [],
    "rebuild": { "source": "durable_records", "session_id": null }
  },
  "warnings": [],
  "meta": { "timestamp": "2026-08-21T08:30:00Z" }
}
```

The commander view may include the Team graph, associated Task progress, blockers, scope overlaps, conflicts, recent Checkpoints, Decisions, Handoffs, and relevant Events. A member view is task-relevant by default: the selected or owned Task, its dependencies and scopes, related progress, latest Checkpoint, Decisions, Handoffs, and directly relevant conflicts. Callers may request a specific Task. The exact projection ordering follows existing `context` relevance rules, but no token budget, cache, copied prompt, or claim that this is a context-window optimizer is part of the contract.

## 4. Assignment semantics

### 4.1 What already works

`task create --assignee <agent>` sets `owner_agent_id` at creation time and is verified. This design does not change it. The gap is exclusively about existing tasks.

### 4.2 Why `assign` is a subcommand, not an `edit` flag

`cli-specification.md` Section 16.6 is explicit: `task edit` modifies only `--title`, `--priority`, and `--description`, and "owner 与 status 不使用 edit 修改" — owner and status are not modified via edit; they go through the transition commands. Adding `--assignee` to `task edit` would contradict that contract and would bury an ownership transfer inside a metadata-edit audit event.

Therefore reassignment is a new transition subcommand:

```bash
carryctx task assign <TASK_REF> --to <AGENT_REF> [--force] [--reason <text>] [--dry-run]
carryctx task unassign <TASK_REF> [--reason <text>]
```

`task unassign` clears `owner_agent_id` while preserving status where the state machine permits, distinguishing an administrative de-assignment by a commander from a worker's own voluntary `task release`.

### 4.3 Status interaction

Assignment sets ownership. It does not start work. The status effect is deliberately minimal and total:

| Task status at assign time | Ownership result     | Status result           | Notes                                                                             |
| -------------------------- | -------------------- | ----------------------- | --------------------------------------------------------------------------------- |
| `planned`                  | `owner_agent_id` set | unchanged `planned`     | Pre-assigning future work is legitimate; the task remains dependency-blocked      |
| `ready`                    | `owner_agent_id` set | unchanged `ready`       | The subagent then runs `task start`, which requires an existing owner             |
| `blocked`                  | `owner_agent_id` set | unchanged `blocked`     | Reassigning a blocked task to whoever can unblock it is a normal commander action |
| `in_progress`              | replaces owner       | unchanged `in_progress` | **Requires `--force`.** This is a live-work steal; see Section 4.4                |
| `review`                   | replaces owner       | unchanged `review`      | Requires `--force`. Normal path for routing to a reviewer                         |
| `completed`, `cancelled`   | rejected             | unchanged               | `INVALID_TASK_TRANSITION`, exit code 3                                            |

Assigning a `ready` task and leaving it `ready` is intentional: it preserves the verified `task start` semantics, which require an existing owner and gate on strong dependencies. Assignment therefore never bypasses `DEPENDENCY_INCOMPLETE`.

### 4.4 Authorization model — the contract CTX-0044 enforces

CTX-0044 records that `task release` performs no ownership check, so any agent can release any other agent's task. That defect was found because recovering the incident in Section 2.4 required exploiting it. The following model is normative for `release`, `assign`, `unassign`, and `reclaim`, and CTX-0044 is the task that enforces it.

> **Invariant I2.** An operation that removes or replaces another agent's ownership MUST be either (a) performed by the owner, (b) performed by a `commander`, or (c) justified by an expired lease. Otherwise it is rejected.

| Operation                                     | Actor is owner | Actor is `commander`         | Actor is peer `subagent`  | Actor `kind` is `NULL`              |
| --------------------------------------------- | -------------- | ---------------------------- | ------------------------- | ----------------------------------- |
| `task release`                                | Allowed        | Allowed, audited as `forced` | `TASK_NOT_OWNED` (exit 9) | Allowed — legacy behavior preserved |
| `task assign --to` (unowned target)           | n/a            | Allowed                      | Allowed                   | Allowed                             |
| `task assign --to` (owned, not `in_progress`) | Allowed        | Allowed                      | `TASK_NOT_OWNED` (exit 9) | Allowed                             |
| `task assign --to --force` (`in_progress`)    | Allowed        | Allowed, audited as `forced` | `TASK_NOT_OWNED` (exit 9) | Allowed                             |
| `task unassign`                               | Allowed        | Allowed                      | `TASK_NOT_OWNED` (exit 9) | Allowed                             |
| `task reclaim` (future lease design)          | n/a            | Deferred                     | Deferred                  | Deferred                            |

The final column is the backward-compatibility hinge. An agent with `kind` `NULL` retains today's unchecked behavior, so no existing script breaks. Ownership enforcement activates for agents that have opted into a team role. This means CTX-0044's fix is a behavior change **only** for agents explicitly registered as `subagent`, which is what makes it shippable without a major version bump.

Attempting a privileged operation without authorization returns:

```json
{
  "schema_version": 1,
  "command": "task.release",
  "success": false,
  "error": {
    "code": "TASK_NOT_OWNED",
    "message": "Task CTX-0038 is owned by agent impl-1, not by impl-2.",
    "details": {
      "display_id": "CTX-0038",
      "owner_agent_id": "01M0HNR52KP9Q1B7X4D8E2GTYV",
      "owner_name": "impl-1",
      "actor_agent_id": "01M0HNR88XA2C4F1M7K3P5RQZW",
      "actor_kind": "subagent",
      "lease_state": "live",
      "lease_expires_at": "2026-08-21T09:15:00Z"
    },
    "suggestions": [
      "Run carryctx task show CTX-0038 to inspect current ownership.",
      "If the owner is unresponsive, run carryctx task reclaim CTX-0038 --reason <text> once the lease expires at 2026-08-21T09:15:00Z.",
      "A commander may reassign immediately: carryctx task assign CTX-0038 --to impl-2 --force."
    ]
  }
}
```

Every suggestion is actionable and names the exact command. This directly repairs the failure in Section 2.4, where the only suggestion offered was to ask a dead process for cooperation. Note that `lease_expires_at` is surfaced in the error itself, so a caller learns when reclaim becomes legal without a second call.

### 4.5 Assignment result and audit

`task assign` emits `task.assigned`; `task unassign` emits `task.unassigned`. Both are written in the same transaction as the ownership change, per the workspace rule that every state-changing use case appends an audit event transactionally.

```json
{
  "schema_version": 1,
  "command": "task.assign",
  "success": true,
  "data": {
    "task": {
      "display_id": "CTX-0038",
      "id": "01M0HNTZ3BFYHDFD8D3864VKSD",
      "status": "ready",
      "owner_agent_id": "01M0HNR52KP9Q1B7X4D8E2GTYV",
      "previous_owner_agent_id": null,
      "title": "Add task assignment primitive (assign/reassign to another agent)"
    },
    "assignment": {
      "assigned_by_agent_id": "01M0HNQG1P82HPWKJ475KD12WM",
      "forced": false,
      "reason": "wave 1 dispatch",
      "lease": null
    },
    "operation": { "applied": true }
  },
  "warnings": [],
  "meta": { "timestamp": "2026-08-21T08:30:00Z" }
}
```

Assignment does not inspect or report capacity. Batching related tasks remains a commander or harness decision, and no configuration makes `task assign` fail because of a task count.

## 5. Ready-queue contract

### 5.1 Command

`task list --status ready` is a raw status filter and stays exactly as it is. A separate read-only claimable-work projection may expose dependency completion, ownership, scope overlap, and stable ordering. It is a query for a commander or harness, not a scheduler-facing queue and not a reservation:

```bash
carryctx task ready [--role <role>] [--agent-for <AGENT_REF>] [--limit <n>]
                    [--include-claimed] [--max-depth <n>] [--json]
```

| Flag                      | Meaning                                                                        |
| ------------------------- | ------------------------------------------------------------------------------ |
| `--role <role>`           | Only tasks whose `required_role` is unset or equals this role                  |
| `--agent-for <AGENT_REF>` | Shorthand for the named agent's role, and additionally annotates capacity      |
| `--limit <n>`             | Truncate after ordering. Default 50                                            |
| `--include-claimed`       | Also return owned-but-unstarted tasks, for commander visibility. Default false |
| `--max-depth <n>`         | Only tasks at topological depth `<= n`                                         |

### 5.2 Membership definition

> **Invariant I3.** A task appears in the default ready queue if and only if all of the following hold: status is `ready`; every strong dependency is `completed`; `owner_agent_id IS NULL` unless `--include-claimed`; and `required_role` is unset or matches the requested role. Lease state is not part of the v0.6 queue contract.

This is strictly narrower than `--status ready`. The difference is the point of the command: `--status ready` answers "what is in the ready state", while `task ready` answers "what can be picked up right now by this worker". Informational dependencies never gate membership, consistent with the existing rule that only strong dependencies gate claim, start, and ready queries.

### 5.3 Ordering guarantee

> **Invariant I4.** The queue is totally ordered and deterministic. Two invocations against an unchanged database MUST return an identical sequence.

Ordering keys, applied in strict sequence:

1. `topological_depth` ascending — shallowest first, so unblocking work outranks leaf work.
2. `priority` descending in the closed order `urgent`, `high`, `normal`, `low`.
3. `unblocks_count` descending — a task that unblocks more dependents is scheduled earlier.
4. `created_at` ascending — older work first, which prevents starvation.
5. `id` ascending — the ULID tiebreaker that makes the total order absolute.

Key 5 exists so the ordering is never ambiguous even for two tasks created in the same transaction. Without it "deterministic" would be unenforceable in a test.

### 5.4 Topological depth and wave planning

`topological_depth` is the longest strong-dependency path from any root to the task. Roots have depth 0. It is computed over strong edges only and is well-defined because cycle insertion is already rejected.

Depth is a useful graph signal, not a wave plan. Tasks at the same depth can still overlap in file scope or be better batched, so the commander or harness must decide how to group them. CarryCtx only makes the dependency and scope facts cheap to inspect.

```json
{
  "schema_version": 1,
  "command": "task.ready",
  "success": true,
  "data": {
    "tasks": [
      {
        "display_id": "CTX-0038",
        "id": "01M0HNTZ3BFYHDFD8D3864VKSD",
        "title": "Add task assignment primitive (assign/reassign to another agent)",
        "status": "ready",
        "priority": "urgent",
        "owner_agent_id": null,
        "required_role": "implementation",
        "topological_depth": 1,
        "unblocks_count": 2,
        "blocked_by_count": 0,
        "lease_state": "none",
        "created_at": "2026-08-21T08:08:03Z"
      },
      {
        "display_id": "CTX-0040",
        "id": "01M0HNTZ4XP5D0X7M39FBTQ7ZV",
        "title": "Config schema for declaring subagent roles and responsibilities",
        "status": "ready",
        "priority": "high",
        "owner_agent_id": null,
        "required_role": null,
        "topological_depth": 1,
        "unblocks_count": 0,
        "blocked_by_count": 0,
        "lease_state": "none",
        "created_at": "2026-08-21T08:08:04Z"
      }
    ],
    "grouping": null,
    "counts": {
      "returned": 2,
      "claimable": 2,
      "excluded_owned": 3,
      "excluded_leased": 1,
      "excluded_role_mismatch": 0,
      "excluded_dependency_incomplete": 4
    },
    "filters": {
      "role": null,
      "agent_for": null,
      "limit": 50,
      "include_claimed": false,
      "max_depth": null
    }
  },
  "warnings": [],
  "meta": { "timestamp": "2026-08-21T08:30:00Z" }
}
```

The `counts` block is a deliberate diagnostic. An empty queue is otherwise indistinguishable from a misconfigured role filter, and a commander that cannot tell "nothing is ready" from "everything is role-mismatched" will either stall silently or dispatch nothing. `excluded_dependency_incomplete` in particular tells the commander the graph is the bottleneck rather than the workforce.

Grouping is deliberately absent from this response. A commander may batch or split returned tasks using descriptions, scopes, dependencies, and current context; no `waves` field or scheduler abstraction is part of the CarryCtx contract.

### 5.5 The queue is advisory, not a reservation

Reading the queue reserves nothing. Two commanders reading simultaneously see the same task and may both dispatch it. That is acceptable and intentional: the verified atomic claim (`owner_agent_id IS NULL AND status = 'ready'`) is the single point of truth, and the loser receives `TASK_ALREADY_CLAIMED`. Adding reservation to the read path would duplicate an already-correct concurrency control. Callers should treat a queue entry as a hint and handle `TASK_ALREADY_CLAIMED` as a normal, expected outcome rather than an error condition.

The projection never filters on capacity. Deciding whether an agent has enough work, or whether related tasks should be batched, is the commander's and harness's call.

## 6. Context projections; leases deferred

The v0.6 primitive is the read-only `team context` projection in Section 3.5, with existing task context as its member-level detail. It may return Tasks, Dependencies, Scopes, open progress, latest Checkpoints, Decisions, Handoffs, conflicts, and relevant Events without mutating ownership or creating a worker lifecycle record. The caller may use it before inline work, batching, or delegation. No worker is required to acquire a lease, pointer, or runtime token before reading or editing through normal CarryCtx commands.

Task leases, renewal, expiry, liveness, and reclaim are intentionally deferred. They combine a heartbeat protocol with a runtime failure policy that belongs to the execution harness. If a future harness proves that durable task-level ownership leases are needed, they require a separate design and compatibility review.

### 6.1 The cost problem

`resume` costs ~3356 tokens against ~51 for `status`. A worker handling one task may need a narrower read projection, but whether and when to fetch it is a caller decision. Team context makes durable records easier to retrieve; it is not specified as a token budget, cache, or context-window optimizer.

### 6.2 Deferred lease proposal (rejected for v0.6)

The former lease proposal is retained below only as rejected/future material. It must not be read as a v0.6 command contract, schema migration, JSON field set, or acceptance criterion. A future harness integration may propose it again in a separate design.

A lease is a **scoped context payload plus a time-bounded claim marker**. Combining them is the central design decision here: the moment a subagent receives context is exactly the moment it becomes responsible for the task, so binding the two into one call means a worker cannot begin work without simultaneously becoming observable to orphan detection. Two separate commands would permit exactly the failure in Section 2.4 — an agent holding work with no liveness record.

```bash
carryctx task lease <TASK_REF> [--ttl <duration>] [--scope <level>] [--renew] [--json]
```

| Flag               | Meaning                                                                 |
| ------------------ | ----------------------------------------------------------------------- |
| `--ttl <duration>` | Lease lifetime. Defaults to `[orchestration].lease_ttl`                 |
| `--scope <level>`  | `pointer`, `task`, or `full`. Default `task`                            |
| `--renew`          | Extend an existing lease held by this agent without re-emitting context |

### 6.3 Scope levels and measured budget

| Scope     | Contains                                                                                                                            | Target budget         | Use                                              |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------- | --------------------- | ------------------------------------------------ |
| `pointer` | IDs, status, title, lease metadata, and URIs for fetchable detail. No prose bodies                                                  | < 150 tokens          | Commander polling many tasks; liveness renewal   |
| `task`    | Pointer plus description, open progress items, blockers, strong dependency states, latest checkpoint summary, task-scoped decisions | < 800 tokens          | **Default.** A subagent executing one task       |
| `full`    | Everything `resume` returns, plus lease metadata                                                                                    | ~3356 tokens observed | Escalation when `task` scope proves insufficient |

The budgets are contractual and testable: the acceptance test in Section 16 asserts a `task`-scope payload for a representative task stays under 800 tokens. Without a numeric ceiling the scope levels would drift back toward `resume` as fields accrete, which is the exact failure mode this section exists to prevent.

`full` is retained so escalation never requires abandoning the lease. A subagent that finds `task` scope inadequate re-leases at `full` rather than falling back to `resume`, which would silently drop its liveness record.

Scope is a property of the response, not of the lease. It selects how much context this call returns and is echoed back in the payload, but it is not persisted (Section 6.4.2), so re-leasing at a wider scope does not rewrite lease state.

### 6.4 Lease record and states

Leases are the one piece of new state this design adds, so the shape is argued from requirements rather than from what might be useful later. Section 1.1 applies to schema as much as to behavior: a management tool should store what it must to detect an orphan and audit a reclaim, and nothing beyond it.

#### 6.4.1 Could liveness ride on existing tables?

Worth settling first, because if it can, the table should not exist.

| Candidate                | Why it cannot carry the lease                                                                                                                                                                                                                                                                               |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sessions`               | Wrong granularity and wrong lifecycle. A session is per-agent, not per-task, so it cannot express which of several held tasks went silent. Worse, `session start` silently auto-ends the prior session (gap 4), so the record a liveness check depends on can vanish without a write to the task            |
| `progress_items`         | No expiry and no exclusivity. A note records that something happened at a time; it cannot express "this claim is valid until T" or "at most one holder". Deriving expiry from the newest note would mean every read reinterprets history, and a task with no notes is indistinguishable from an expired one |
| `tasks.started_at` alone | Already tried, in effect. This is exactly the 0.5.8 state that produced the Section 2.4 incident: `in_progress` plus a timestamp cannot distinguish a working agent from a dead one                                                                                                                         |

The requirement none of them meets is the conjunction: **at most one holder per task, with a deadline, surviving session churn.** That is a distinct fact about a task, so it gets its own record.

#### 6.4.2 Minimum viable shape

Reduced from eleven columns to six. Each remaining column is forced by a stated requirement:

```text
task_leases(
  id            TEXT PRIMARY KEY,
  task_id       TEXT NOT NULL REFERENCES tasks(id),
  agent_id      TEXT NOT NULL REFERENCES agents(id),
  renewed_at    TEXT NOT NULL,
  expires_at    TEXT NOT NULL,
  released_at   TEXT NULL
)
```

| Column        | Forced by                                                                                                                                                         |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`          | Audit. `task.reclaimed` carries `lease_id` (Section 13.2), so the lease that was broken must be nameable after the fact                                           |
| `task_id`     | I7. The orphan predicate is per-task, and the partial unique index below needs it                                                                                 |
| `agent_id`    | R3 and I2. Reclaim authorization compares the actor against the holder, and `agent team` groups held tasks by agent                                               |
| `renewed_at`  | Diagnostics. Surfaces as `last_heartbeat_at` and drives `tasks.no_progress`. Not derivable from `expires_at` minus TTL, since the TTL may differ between renewals |
| `expires_at`  | I7 and lease-state derivation. This is the deadline the whole mechanism exists to express                                                                         |
| `released_at` | Terminal state, plus the partial unique index predicate that enforces one live lease per task                                                                     |

Five columns were removed:

| Removed         | Reason                                                                                                                                                                                                                                         |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `acquired_at`   | Redundant with the `lease.acquired` event, which already timestamps acquisition in the audit log. Storing it twice creates two sources that can disagree                                                                                       |
| `scope`         | Nothing reads it. Scope shapes one response payload at lease time and has no bearing on liveness, orphan detection, or reclaim. Persisting a formatting choice as durable state is exactly the accretion the Section 6.3 budgets guard against |
| `session_id`    | The `agents.identity_conflict` check is answerable from `sessions` directly, which already records agent and active state. A denormalized copy here would need to stay consistent with a table that auto-ends rows behind it                   |
| `worktree_path` | Already recorded on the session, and never consulted by the orphan predicate. A lease is on a task, and per I17 all worktrees share one database, so the path adds no information about who holds what                                         |
| `release_kind`  | The event log already distinguishes the four cases as distinct event types (`lease.released`, `lease.reclaimed`, and completion). A `CHECK`-constrained enum duplicating event types is a second vocabulary to keep in sync                    |

The removals share a shape: each was either a copy of something the audit log already records, or a value nothing queries. Neither justifies durable state in a tool whose value is that its records are trustworthy.

#### 6.4.3 The smaller alternative, and why the table still wins

Six columns is not the floor. Two nullable columns on `tasks` — `lease_agent_id` and `lease_expires_at` — would satisfy the I7 orphan predicate, make "one live lease per task" true by construction with no index, and add no table at all. It is worth stating why that was not chosen, since it is the more aggressive reading of this revision.

Two reasons, in order of weight:

1. **Write contention on the hot row.** A lease renews on every recording command (Section 6.5), so renewal is the most frequent write in a team workload. Putting it on `tasks` means every heartbeat updates the same row that `BEGIN IMMEDIATE` claim transactions serialize on (I12), turning a cheap liveness ping into a contender for the one lock the design depends on. A separate table keeps heartbeat traffic off the claim path.
2. **Reclaim history.** Overwriting two columns loses the sequence of holders. `task.reclaimed` events preserve the audit trail, but reconstructing "this task was picked up and abandoned three times" would require replaying the event log rather than reading rows. For a tool whose purpose is answering what happened, that is the wrong trade.

The first reason is the decisive one, and it is a correctness-adjacent argument rather than a preference: it protects the serialization point that the verified 96-writer concurrency result depends on.

A partial unique index enforces one live lease per task:

```sql
CREATE UNIQUE INDEX task_leases_one_live ON task_leases(task_id) WHERE released_at IS NULL;
```

`lease_state` is a closed derived set:

| State      | Condition                                   | Meaning                                    |
| ---------- | ------------------------------------------- | ------------------------------------------ |
| `none`     | No lease row                                | Never leased, or fully released            |
| `live`     | `released_at IS NULL AND expires_at > now`  | An agent is actively responsible           |
| `expired`  | `released_at IS NULL AND expires_at <= now` | **Orphan candidate.** Section 8 governs it |
| `released` | `released_at IS NOT NULL`                   | Terminal, retained for audit               |

Expiry is evaluated lazily at read time against the current clock. No background process exists, consistent with Section 1's exclusion of process supervision. A lease row is never mutated into `expired`; the state is always derived, so a clock adjustment cannot leave a stale persisted verdict.

Deriving rather than storing the state is also why `expires_at` and `released_at` are sufficient. A `lease_status` column would need writing at expiry, which would require the background process Section 1 excludes.

### 6.5 TTL default

`[orchestration].lease_ttl` defaults to **`30m`**.

The reasoning, in order of weight:

1. It must comfortably exceed a realistic single work step. A subagent implementing a task with tests can easily run 10 to 20 minutes between recordable milestones. A TTL near that duration would expire live workers, and a false orphan verdict is the most damaging failure this design can produce — it invites a commander to steal work from a healthy agent.
2. It must be short enough that recovery is not gated on human patience. The incident in Section 2.4 stranded a task indefinitely. Thirty minutes bounds that.
3. It is deliberately shorter than `[session].stale_after` (`2h`). Task-level liveness should be detected well before session-level staleness, because the task claim is the contended resource, not the session.
4. Renewal is cheap. A `pointer`-scope `--renew` costs under 150 tokens, so a 30-minute TTL does not pressure an agent into expensive periodic calls.

Renewal is not automatic. Any progress-recording command listed in Section 9.3 renews the lease as a side effect in the same transaction, which means an agent that honors the durability contract never has to think about renewal. This coupling is intentional: it makes the cheapest path to staying alive identical to the cheapest path to being durable.

### 6.6 Lease payload

```json
{
  "schema_version": 1,
  "command": "task.lease",
  "success": true,
  "data": {
    "lease": {
      "id": "01M0HQ12ZK8N4T6R9V3B7C5DEF",
      "task_id": "01M0HNTZ3BFYHDFD8D3864VKSD",
      "agent_id": "01M0HNR52KP9Q1B7X4D8E2GTYV",
      "scope": "task",
      "lease_state": "live",
      "renewed_at": "2026-08-21T08:30:00Z",
      "expires_at": "2026-08-21T09:00:00Z",
      "ttl": "30m"
    },
    "task": {
      "display_id": "CTX-0038",
      "id": "01M0HNTZ3BFYHDFD8D3864VKSD",
      "title": "Add task assignment primitive (assign/reassign to another agent)",
      "status": "ready",
      "priority": "urgent",
      "owner_agent_id": "01M0HNR52KP9Q1B7X4D8E2GTYV",
      "required_role": "implementation",
      "description": "..."
    },
    "context": {
      "scope": "task",
      "open_progress": [
        {
          "display_id": "PX-0009",
          "item_type": "note",
          "status": "open",
          "content": "..."
        }
      ],
      "blockers": [],
      "dependencies": [
        {
          "display_id": "CTX-0037",
          "kind": "strong",
          "status": "in_progress",
          "satisfied": false
        }
      ],
      "latest_checkpoint": {
        "display_id": "CP-0004",
        "summary": "...",
        "created_at": "2026-08-21T07:55:00Z",
        "git": { "branch": "main", "head": "a1b2c3d", "dirty": false }
      },
      "decisions": [],
      "fetch": {
        "full_context": "carryctx context --task CTX-0038 --format markdown",
        "resume": "carryctx resume --task CTX-0038"
      }
    },
    "durability_contract": {
      "must_record": [
        "progress",
        "checkpoint_on_milestone",
        "decision_on_tradeoff"
      ],
      "renew_by": "2026-08-21T09:00:00Z",
      "renew_command": "carryctx task lease CTX-0038 --renew",
      "on_expiry": "Task becomes an orphan candidate and may be reclaimed by any agent via carryctx task reclaim."
    }
  },
  "warnings": [],
  "meta": { "timestamp": "2026-08-21T08:30:00Z" }
}
```

The `context.fetch` block is the pointer mechanism: rather than embedding expensive content, the payload names the exact command that would retrieve it. The subagent escalates only if it actually needs to, so the common case never pays for the uncommon one.

`durability_contract` is embedded in the response deliberately. The expectations in Section 9 are delivered to the subagent at the moment it takes the work, so compliance does not depend on the agent having read this document or on the dispatching prompt remembering to restate them. Given that the Section 2.4 failure was precisely an agent not recording anything, shipping the contract inside the payload is the highest-leverage intervention available — and it works by informing the agent rather than by restricting it, which is the pattern Section 1.1 prefers throughout.

### 6.7 Lease and ownership are distinct

Ownership (`owner_agent_id`) is durable assignment. A lease is time-bounded active responsibility. They are separate because a task can legitimately be owned but not actively worked on — pre-assigned at `planned`, or parked between waves — and because expiry must be able to invalidate active responsibility without silently erasing the commander's assignment decision.

Consequences:

- Leasing a task the agent does not own does not transfer ownership. It records responsibility; `TASK_NOT_OWNED` still governs privileged operations.
- Lease expiry does not clear `owner_agent_id`. It only makes the task reclaimable.
- `task complete`, `task release`, and `task reclaim` all close any live lease in the same transaction, each emitting its own `lease.*` event to record how the lease ended.

## 7. Configuration schema for subagent roles

### 7.1 Requirement

Role configuration is optional project metadata. A project may name a few role labels and short responsibilities for human and agent orientation, but role configuration does not activate a team, select a preset, assign work, grant permission, define queue admission, or describe execution behavior. The harness may interpret these labels in its own configuration. This is CTX-0040.

### 7.2 Should a role just be a preset?

CTX-0040 asks this directly, and it deserves a real answer rather than a new parallel concept added by reflex.

The `preset` system already exists and already owns capability packs: profiles, rules, workflows, personas, permissions, integrity hashes, and `.carryctx/presets.lock`. A role definition must not duplicate that behavior or turn CarryCtx into a prompt-routing layer.

So the answer is: **a role is not a new behavioral concept, and behavior stays entirely in presets.** What remains after removing behavior is small — two things the preset system genuinely cannot answer, because they are properties of the _project's task queue_ rather than of an agent's instructions:

1. A human-readable responsibility label.
2. Optional advisory file globs, if a commander wants them surfaced alongside existing task scopes.

Neither is expressible in a preset without teaching presets about tasks, which would couple a distributable capability pack to one project's backlog. That is the justification for the `[orchestration.roles]` table existing at all, and it is also the reason it stays this small.

| Concern                                                          | Owner                                    | Rationale                                                                       |
| ---------------------------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------- |
| Behavioral instructions, rules, personas, workflows, permissions | `preset` and `carryctx-skills`           | Already implemented, with schema, lockfile, and SHA-256 integrity               |
| Which task scopes a role is associated with                      | `[orchestration.roles]` in `config.toml` | Properties of this project's task graph, not of a distributable capability pack |

> **Invariant I5.** `[orchestration.roles]` contains no behavioral instructions, preset selection, queue policy, capacity limit, or permission. It is advisory project metadata only.

Behavior remains in the harness and existing preset system. CarryCtx only stores and returns typed role metadata without parsing or materializing prompts.

The whole table is **advisory metadata**. Per Section 1.1, nothing in it grants or denies permission, and no key in it can cause a command to fail.

### 7.3 Schema

Every key below survived an explicit justification test: it must answer "who is on this team and what are they for" in a way that changes what a commander can see. Keys that only expressed policy were dropped, and 7.3.1 records them.

```toml
schema_version = 1

[orchestration.roles.implementation]
description = "Implements tasks and keeps tests green"
scopes = ["src/**", "tests/**"]

[orchestration.roles.review]
description = "Reviews completed implementation work"
```

| Key                                      | Type     | Default | Meaning                                             |
| ---------------------------------------- | -------- | ------- | --------------------------------------------------- |
| `orchestration.roles.<name>.description` | string   | none    | Human and agent-readable responsibility statement   |
| `orchestration.roles.<name>.scopes`      | string[] | `[]`    | Optional advisory globs surfaced with role metadata |

Two keys remain: `description` and `scopes`. They are annotations only; queue filtering uses task and agent fields supplied by the caller and never grants eligibility.

#### 7.3.1 Keys dropped, and why

| Dropped key                                                                      | Reason                                                                                                                                                                                                                                            |
| -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `max_parallel_subagents`                                                         | Pure fan-out policy, and the clearest case of framework machinery. How many workers to run is the harness's concern and the commander's judgment; CarryCtx has no basis for an opinion and no mechanism to act on one. Dropped per DEC-0004       |
| `require_role_match`                                                             | Admission control. It exists only to turn a role mismatch into a rejected claim, which CTX-0040 rules out. A role filter narrows a query; it does not grant eligibility. Its removal also deletes `ROLE_MISMATCH` and `ROLE_NOT_DECLARED`         |
| `lease_ttl`, `orphan_grace`                                                      | Runtime heartbeat and reclaim policy owned by the harness; no v0.6 lease lifecycle is specified                                                                                                                                                   |
| `enabled`                                                                        | Unnecessary feature gate; metadata and read projections must degrade naturally when absent                                                                                                                                                        |
| `default_lease_scope`                                                            | Runtime lease setting; deferred with leases                                                                                                                                                                                                       |
| `roles.<name>.kind`                                                              | Duplicate state. `kind` already lives on the agent record (Section 3.2), which is where authorization reads it. Declaring it per role invites a conflict between an agent registered `subagent` and a role declaring `commander`, with no arbiter |
| `roles.<name>.max_active_tasks`, `roles.<name>.active_task_hint`                 | Capacity policy belongs to the commander/harness; CarryCtx reports task ownership but does not prescribe a load signal                                                                                                                            |
| `roles.<name>.preset`, `roles.<name>.claims_status`, `roles.<name>.capabilities` | Behavior, admission, and permissions belong to the harness or preset system, not project state metadata                                                                                                                                           |

The pattern in this table is worth naming: four of the five genuine removals were mechanisms for _refusing_ something. That is the Section 1.1 boundary doing its work on the config surface.

### 7.4 Graceful degradation

> **Invariant I6.** The entire `[orchestration]` table is optional advisory metadata. Its absence is indistinguishable from 0.5.8 behavior, and no command becomes unavailable or more restrictive because it is missing.

Degradation rules, in precedence order:

1. No `[orchestration]` table: all existing commands and the new assignment/read primitives behave as before. Role data is absent and no task is blocked by configuration.
2. `[orchestration]` present but `roles` absent: task and agent role fields remain ordinary annotations. A requested role filter may return no matches, but it never rejects a claim or assignment.
3. A role referenced by an agent but undeclared in config: the agent still functions normally. Diagnostics may report the missing description as a warning, never as a fatal configuration error.
4. `required_role` set on a task: advisory in all configurations. It can annotate or narrow a read query but never blocks a claim or assignment.

The important rule is that missing metadata never gates the primitives. A commander can use assignment and read projections without first writing configuration.

Rule 2's warning-instead-of-empty-result is the same reasoning as the `counts` block in Section 5.4: a commander that cannot distinguish "nothing is ready" from "your filter matched nothing" will stall silently.

Unknown keys inside `[orchestration]` follow the existing `--config-compat` contract: `error` by default, downgradable to `warn`. This remains parse correctness, not execution policy.

### 7.5 Task-side field

One nullable column supports role routing:

```text
tasks.required_role TEXT NULL
```

Set via `task create --required-role <role>` or `task edit --required-role <role>`. Unlike owner and status, `required_role` is task metadata rather than lifecycle state, so it belongs on `task edit` and does not violate the Section 4.2 constraint.

## 8. Failure handling and explicit recovery (deferred)

The v0.6 contract records enough durable state for a commander or harness to inspect incomplete work and explicitly reassign it. It does not classify a process as dead, expire a claim, or reclaim work on a timer. Those are runtime inferences owned by the harness and are future work if a concrete integration requires them. Sections 8.2 through 8.7 are retained as rejected/future design notes only; their lease tables, liveness states, reclaim commands, doctor checks, and failure tests are not v0.6 implementation requirements.

Sections 8.2 through 8.6 below are retained as future design notes only. Their liveness states, expiry thresholds, reclaim command, doctor checks, and session gates are not v0.6 requirements or configuration keys.

This section is weighted most heavily because it addresses a failure that already happened (Section 2.4) rather than one that might. A subagent dying while holding a task is not an edge case in a team model; with N parallel workers it is the expected steady-state event.

### 8.1 What went wrong, restated as requirements

| Incident observation                                                 | Requirement                                                                        |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Task sat `in_progress` with a dead owner and nothing recorded        | R1: Liveness must be observable independently of status                            |
| `doctor` said only "17 task(s) currently in progress", status `info` | R2: `doctor` must classify held tasks by liveness and escalate severity            |
| Recovery needed a third party to `release` a task it did not own     | R3: Reclaim must be a first-class authorized operation, not an unchecked `release` |
| `TASK_ALREADY_CLAIMED` suggested asking a dead process to cooperate  | R4: Suggestions must be executable by the agent reading them                       |
| All work was lost                                                    | R5: Durability must be incremental and obligatory (Section 9)                      |

### 8.2 Liveness classification

Liveness is derived from lease state and recorded activity. It is an inference, never a process check, per Section 1.

| `liveness` | Condition                                  | Interpretation                                  |
| ---------- | ------------------------------------------ | ----------------------------------------------- |
| `live`     | Live lease, or activity within `lease_ttl` | Working. Do not disturb                         |
| `idle`     | No live lease and holds no task            | Available for dispatch                          |
| `expired`  | Holds a task whose lease is `expired`      | **Orphan candidate**                            |
| `unknown`  | Holds a task but never took a lease        | Legacy or non-compliant agent. Cannot be judged |

`unknown` is a required category, not a gap. Existing single-agent projects hold tasks without leases, and classifying them as `expired` would invite reclaiming healthy work. An `unknown` task is reported by `doctor` but is never auto-reclaimable; reclaiming it requires `--force` and an explicit reason.

### 8.3 Orphan definition

> **Invariant I7.** A task is an orphan candidate if and only if its status is `in_progress` or `review`, it has an owner, its lease state is `expired`, and `now > expires_at + orphan_grace`.

`orphan_grace` (default `5m`) is separate from the TTL so that an agent briefly slow to renew is not instantly declared dead. Effective reclaim threshold is therefore 35 minutes of silence by default.

Being an orphan candidate has no automatic consequence. Nothing is reclaimed, released, or reassigned without an explicit command. Automatic reclaim is deliberately not offered, because a false positive silently steals live work and the cost of that outweighs the convenience of automation. Detection is CarryCtx's job; deciding what to do about a silent agent is the commander's.

### 8.4 The reclaim path

```bash
carryctx task reclaim <TASK_REF> [--to <AGENT_REF>] [--reason <text>] [--force] [--dry-run] [--json]
```

| Behavior       | Rule                                                                                                                                                                                                        |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Authorization  | Any agent when lease is `expired` past grace; a `commander` at any time; `--force` required for `unknown` liveness                                                                                          |
| Default target | `--to` absent transfers to the invoking agent                                                                                                                                                               |
| Status effect  | Task returns to `ready` if strong dependencies are complete, otherwise `planned`, matching existing `release` semantics — unless `--to` is given, in which case ownership transfers and status is preserved |
| Lease effect   | Sets `released_at` on the stale lease and appends `lease.reclaimed`, which is what records the manner of release (Section 6.4.2)                                                                            |
| Audit          | Appends `task.reclaimed` in the same transaction                                                                                                                                                            |
| Preservation   | Never deletes progress items, checkpoints, decisions, or events belonging to the previous owner                                                                                                             |

The preservation rule is what makes reclaim safe to use. Recorded partial work survives the ownership change, so the incoming agent inherits it. This is precisely the property the Section 2.4 incident lacked — not because reclaim destroyed anything, but because nothing had been recorded to preserve.

Refusal before the grace period elapses:

```json
{
  "schema_version": 1,
  "command": "task.reclaim",
  "success": false,
  "error": {
    "code": "LEASE_STILL_LIVE",
    "message": "Task CTX-0038 has a live lease held by impl-1 until 2026-08-21T09:00:00Z.",
    "details": {
      "display_id": "CTX-0038",
      "owner_agent_id": "01M0HNR52KP9Q1B7X4D8E2GTYV",
      "owner_name": "impl-1",
      "lease_state": "live",
      "expires_at": "2026-08-21T09:00:00Z",
      "reclaimable_at": "2026-08-21T09:05:00Z",
      "orphan_grace": "5m"
    },
    "suggestions": [
      "Wait until 2026-08-21T09:05:00Z, then rerun carryctx task reclaim CTX-0038.",
      "A commander may reclaim immediately: carryctx task reclaim CTX-0038 --reason <text>.",
      "Inspect what the owner recorded: carryctx progress list --task CTX-0038."
    ]
  }
}
```

`reclaimable_at` is a precomputed timestamp rather than something the caller derives from TTL plus grace. R4 requires suggestions be executable; making a scheduler compute a deadline from two config values is how that requirement gets quietly broken.

Successful reclaim:

```json
{
  "schema_version": 1,
  "command": "task.reclaim",
  "success": true,
  "data": {
    "task": {
      "display_id": "CTX-0038",
      "id": "01M0HNTZ3BFYHDFD8D3864VKSD",
      "status": "ready",
      "owner_agent_id": null,
      "previous_owner_agent_id": "01M0HNR52KP9Q1B7X4D8E2GTYV"
    },
    "reclaim": {
      "reclaimed_by_agent_id": "01M0HNQG1P82HPWKJ475KD12WM",
      "previous_owner_name": "impl-1",
      "reason": "lease expired 41m ago, no progress recorded since 06:02",
      "forced": false,
      "lease_id": "01M0HQ12ZK8N4T6R9V3B7C5DEF",
      "lease_expired_at": "2026-08-21T09:00:00Z",
      "preserved": { "progress_items": 3, "checkpoints": 1, "decisions": 0 }
    },
    "operation": { "applied": true }
  },
  "warnings": [
    {
      "code": "ORPHAN_WORK_PRESERVED",
      "message": "3 progress item(s) and 1 checkpoint(s) from impl-1 were preserved. Review them before restarting: carryctx progress list --task CTX-0038"
    }
  ],
  "meta": { "timestamp": "2026-08-21T09:06:00Z" }
}
```

`--reason` is required unless `--force` is passed with `--yes`. A reclaim without a recorded rationale produces an audit trail that cannot answer why work changed hands, which is the same class of opacity the incident exposed.

### 8.5 `doctor` changes

The existing `tasks.in_progress` check is status-only and severity `info`. It is split so that liveness is visible, satisfying R2:

| Check                        | Severity                      | Condition                                                                            |
| ---------------------------- | ----------------------------- | ------------------------------------------------------------------------------------ |
| `tasks.in_progress`          | `ok` or `info`                | Count of `in_progress` tasks with `live` leases. Informational only                  |
| `tasks.orphaned_leases`      | `warn`, or `error` past grace | Tasks whose lease is `expired`. Names each task, owner, and expiry                   |
| `tasks.unleased_in_progress` | `info`                        | Tasks held without any lease (`unknown` liveness). Expected in single-agent projects |
| `agents.stale_holders`       | `warn`                        | Agents with `expired` liveness still holding tasks                                   |
| `agents.identity_conflict`   | `warn`                        | One agent with overlapping live sessions across worktrees, violating I1              |
| `tasks.no_progress`          | `warn`                        | `in_progress` more than `lease_ttl` with zero progress items and zero checkpoints    |

`tasks.no_progress` is the check that would have caught the Section 2.4 incident even in the absence of leases. It needs no new state — it is answerable today from existing tables — and it is the cheapest possible detector for "an agent took work and recorded nothing".

```json
{
  "check": "tasks.orphaned_leases",
  "status": "error",
  "message": "1 task(s) have expired leases past the 5m grace period and can be reclaimed.",
  "fix_command": "carryctx task reclaim CTX-0038 --reason \"expired lease\"",
  "tasks": [
    {
      "display_id": "CTX-0038",
      "owner_name": "impl-1",
      "lease_expired_at": "2026-08-21T09:00:00Z",
      "recorded_progress_items": 3,
      "reclaimable_at": "2026-08-21T09:05:00Z"
    }
  ]
}
```

`doctor --fix` MUST NOT reclaim tasks. Reclaim reassigns work between agents based on an inference about liveness; it is not a "safe repair" in the sense the existing contract uses, which is limited to explicitly classified safe repairs applied without confirmation. `doctor` therefore reports and emits `fix_command` strings, leaving execution to a deliberate operator or commander action.

### 8.6 Session hazard mitigation (future harness integration)

Gap 4 — `session start` silently auto-ending the prior session — may need mitigation in a harness integration. CarryCtx does not change this existing behavior in v0.6. A future integration could:

1. Have the harness use distinct agent identities for concurrent workers.
2. Add an explicit session mode or warning before replacing an active session.
3. `doctor` check `agents.identity_conflict` surfaces repeat offenders.

Any such behavior change requires its own CLI contract and compatibility review.

### 8.7 Failure matrix

| Failure                                    | Detection                                                        | Recovery                                               | Data loss            |
| ------------------------------------------ | ---------------------------------------------------------------- | ------------------------------------------------------ | -------------------- |
| Subagent dies holding a leased task        | Lease expires; `doctor` `tasks.orphaned_leases` `error`          | `task reclaim` after grace                             | Only unrecorded work |
| Subagent dies holding an unleased task     | `doctor` `tasks.unleased_in_progress` plus `tasks.no_progress`   | `task reclaim --force --reason`                        | Only unrecorded work |
| Subagent hangs but process is alive        | Lease expires if it stops recording                              | Same as death; indistinguishable by design             | Only unrecorded work |
| Commander dies mid-dispatch                | Assigned tasks keep owners; no lease taken means nothing expires | Another commander reads `task ready --include-claimed` | None                 |
| Two subagents share one identity           | `agents.identity_conflict`; `SESSION_AUTO_ENDED` warning         | Register distinct agents per I1                        | Session records only |
| Two agents race one claim                  | Atomic conditional update                                        | Loser handles `TASK_ALREADY_CLAIMED` normally          | None                 |
| Reclaim races the original owner returning | Partial unique index on live leases                              | Returning owner's next write fails; it must re-lease   | Only unrecorded work |
| Clock skew across worktrees                | Not detectable offline                                           | Grace period absorbs small skew                        | None                 |

"Only unrecorded work" appears in five rows. That repetition is the argument for Section 9: the blast radius of every liveness failure is bounded precisely by how recently the agent recorded something, so the durability contract is the real mitigation and leases are only the detector.

The last row is an accepted limitation. All timestamps come from the local clock of whichever process writes them, and CarryCtx has no network time source by policy. Severe skew between worktrees on one machine is not a realistic concern; a distributed team would need a different mechanism and is `future`.

## 9. Durability guidance for subagents (advisory)

### 9.1 Purpose

The product goal is that CarryCtx replaces scattered agent-local notes in `~/.claude`, `~/.codex`, `/tmp`, and stale `docs/` files. Project state must live with the project, be queryable, and not drift. That goal fails in exactly two ways, and both are addressed here:

- An agent keeps state in its own head or its own scratch files, so an abrupt exit loses it. This is what happened in Section 2.4.
- An agent writes state somewhere outside the database, so the database is no longer authoritative and drifts from reality.

The remedy is a contract that is explicit, checkable, and delivered to the subagent at lease time (Section 6.6). It is a contract in the sense of an agreed working standard that `doctor` can audit after the fact, not an admission-control gate — CarryCtx cannot observe what an agent chose not to write, so the leverage here is entirely in making recording cheap, obvious, and visibly absent when skipped.

### 9.2 Storage rules

> **Invariant I8.** Durable project state belongs in `<git-common-dir>/carryctx/state.sqlite`, written through CarryCtx. Agent-local scratch directories are for genuinely disposable artifacts and should not be the sole record of any decision, finding, or progress.

I8 is a contract with the agent, detected by `doctor`, not a gate the CLI can close. CarryCtx cannot see an agent's scratch files and will not try to; the enforcement available is making the recorded path cheap and the unrecorded path visible.

| State                                | Correct destination               | Never                      |
| ------------------------------------ | --------------------------------- | -------------------------- |
| What I am doing and how far I got    | `progress note` / `progress todo` | Agent-local notes          |
| A resumable milestone                | `checkpoint create`               | Commit messages alone      |
| A tradeoff and its rationale         | `decision add`                    | Inline code comments alone |
| A discovered blocker                 | `progress block`                  | Chat history               |
| Handing work to another agent        | `handoff create`                  | Prose in a prompt          |
| Disposable build logs, probe scripts | `tmp/`                            | The database               |

### 9.3 Mandatory recording points

> **Advisory.** A subagent should record durable state at each of the following points. No recording cadence, lease renewal, or completion gate is part of the v0.6 contract.

| Point                                        | Required command                                 | Rationale                                                                                                          |
| -------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| On taking work                               | `task lease` then `task start`                   | Establishes responsibility and observability                                                                       |
| On understanding the problem, before editing | `progress note` describing the intended approach | The single highest-value record. If the agent dies here, the successor inherits the analysis instead of redoing it |
| At each completed sub-step                   | `progress note` or `progress done`               | Bounds loss to one sub-step                                                                                        |
| On any non-obvious tradeoff                  | `decision add`                                   | Prevents silent re-litigation later                                                                                |
| On a blocker                                 | `progress block --reason`                        | Makes the blocker visible to the commander                                                                         |
| At a coherent, resumable milestone           | `checkpoint create`                              | The actual resume point                                                                                            |
| Before finishing                             | `task review` or `task complete`                 | Releases the lease and emits the matching `lease.*` event                                                          |

The "before editing" row is the one that matters most. An agent that records its plan before doing work converts a total loss into a warm start, and it costs one cheap call. The Section 2.4 failure was precisely the omission of this step.

> **Invariant I10.** Recording should not be deferred to completion time. Recording only at the end provides zero durability, because the failure mode being defended against is not reaching the end.

### 9.4 Bounded loss

The contract yields a stated guarantee:

> If a subagent exits abruptly at any moment, the recoverable state is everything recorded up to its last completed recording point. Maximum loss is one sub-step of work plus any unrecorded reasoning since that point.

This is testable: kill a subagent mid-task, then assert that `task reclaim` followed by `task lease` on a new agent returns the prior agent's progress items, decisions, and latest checkpoint in the payload.

### 9.5 Advisory posture

The posture is advisory and detective rather than blocking, because a hard block on `task complete` would push agents toward recording token-shaped noise to satisfy a counter — and because per Section 1.1 how thoroughly a team documents its work is a working-style question, not a data-integrity one:

| Level     | Mechanism                                                                                                     | Release                 |
| --------- | ------------------------------------------------------------------------------------------------------------- | ----------------------- |
| Advisory  | `durability_contract` block in the lease payload                                                              | v0.6                    |
| Detective | `doctor` check `tasks.no_progress`                                                                            | v0.6                    |
| Warning   | `task complete` warns `NO_DURABLE_RECORD` when a task completes with zero progress items and zero checkpoints | v0.6                    |
| Blocking  | Not defined. CarryCtx does not reject completion based on recording style.                                    | Future, separate review |

No strictness switch appears in the v0.6 configuration schema. A future harness may define its own recording policy, but CarryCtx should not turn documentation thoroughness into a rejected state transition without a separate product decision.

## 10. Concurrency invariants

The measured baseline — 96 concurrent writes across 24 processes, zero failures, 326 ms, WAL plus `busy_timeout=10000` (`src/adapter/sqlite.rs:140`) — shows the storage layer already sustains a team. This section states what must continue to hold, not what must be built.

> **Invariant I11.** Every state-changing use case writes its domain change and its audit event in one transaction. A team model must never expose a state change whose event is missing, because the event log is the only cross-agent record of who did what.
> **Invariant I12.** Task claim remains the single serialization point for ownership. It uses `BEGIN IMMEDIATE` and a conditional update requiring `owner_agent_id IS NULL AND status = 'ready'`; zero changed rows map to `TASK_ALREADY_CLAIMED` or `INVALID_TASK_TRANSITION`. No additional locking is introduced for assignment, leasing, or reclaim.
> **Future invariant.** If a separate lease design is accepted, at most one live lease may exist per task. This is not a v0.6 invariant.
> **Invariant I14.** Read commands take no write locks and never mutate. `task ready`, `team status`, and `team context` are read-only. Lease acquisition and renewal, if ever introduced, are writes.
> **Future invariant.** If leases are introduced, expiry must be evaluated without a background process. This is not a v0.6 requirement.
> **Future invariant.** If reclaim is introduced, it must be atomic and audited. No reclaim operation exists in the v0.6 CLI.
> **Invariant I17.** All N subagents share one database at `<git-common-dir>/carryctx/state.sqlite` regardless of worktree. Per-worktree databases are forbidden, since they would partition the team's state.
> **Invariant I18.** Concurrent contention surfaces as a domain error with exit code 3, never as a SQLite error with exit code 5. Exit code 5 is reserved for genuine lock exhaustion past `busy_timeout`.

I18 matters for schedulers: a commander must be able to distinguish "someone beat me to it, pick the next task" (exit 3, routine) from "the database is in trouble" (exit 5, stop and escalate). Conflating them would make a busy team look broken.

Verification target: 4 subagents plus 1 commander sustaining mixed lease, progress, checkpoint, and claim traffic with zero failures and no lost audit events.

## 11. `single_active_task_per_agent` decision

The v0.6 decision is simpler than the earlier capacity proposal: multiple active tasks remain allowed, and CarryCtx adds no capacity hint, warning, cap, or enforcement key. A commander may inspect task counts and decide whether batching is appropriate. The detailed `active_task_hint` material below is future work and is not part of the v0.6 contract.

### 11.1 Current state — and what the actual defect is

`single_active_task_per_agent` is documented as default `true` in both `requirements.md:2956` and `configuration.md:302`, and `cli-specification.md` Section 16.3 lists "单 Agent Task 限制" among the checks `task claim` must perform. It is not enforced. One agent was observed holding 17 in-progress tasks simultaneously.

It is worth being precise about what is broken here, because the obvious reading sends the fix in the wrong direction. **The defect is not that agents hold several tasks. The defect is that the documentation asserts a rule the code ignores.** Those are different bugs with opposite remedies: the first is fixed by adding enforcement, the second by making the documentation true. Section 1.1 decides which one applies — capping how many tasks an agent may hold is team-organisation policy, not correctness, so the code is right and the documentation is wrong.

The supporting evidence is that nothing broke. This key has been documented default-`true` and unenforced since it was introduced, across the entire 0.5.8 lifetime, and multi-holding agents produced no incident. Mechanical enforcement was never the working model, and the failure this design actually had to recover from (Section 2.4) was caused by a dead agent recording nothing, not by an agent holding too much.

### 11.2 Options

| Option                        | Effect                                                               | Assessment                                                                                                                                       |
| ----------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| A. Enforce globally at `true` | `task claim` and `task start` reject a second active task            | Faithful to the docs, but breaks any workflow relying on multi-holding, and forbids a reviewer batching several reviews                          |
| B. Drop the key entirely      | Delete it, document unlimited concurrency                            | Honest, but discards the one signal that distinguishes a focused worker from a runaway one, and leaves the commander with nothing to reason from |
| C. Per-role enforced cap      | Enforce `max_active_tasks`, resolved per role, defaulting to `1`     | **Rejected.** Turns a policy preference into a rejected state transition; see 11.3                                                               |
| D. Advisory capacity signal   | Report the load, and any declared hint, as data. Warn but never fail | **Recommended.** Keeps the signal, refuses the policy, makes the docs true                                                                       |

### 11.3 Why Option C is rejected

An earlier revision of this document recommended Option C. That recommendation is withdrawn per **DEC-0004**, and the reasoning is worth recording so it is not re-proposed.

Option C makes exceeding a task count a **rejected transition**. Under Section 1.1 that is backwards. Batching several related or near-identical tasks onto one subagent is a legitimate and often superior commander decision — CTX-0035 and CTX-0036 in this very backlog are the worked example, two small documentation edits in adjacent files that one subagent should have taken together. With a default cap of 1, CarryCtx would have **refused the correct dispatch** and forced the fan-out that actually wasted a subagent. A tool with strictly less context than the commander must not overrule it.

Option C's stated benefits survive without the enforcement. The capacity signal a commander needs is `active_task_count` and a declared hint; neither requires the power to reject. The "converts misinformation into a real invariant" argument was the weakest of the four, because it treats _any_ resolution of the contradiction as progress and ignores that one of the two resolutions is wrong on the merits.

### 11.4 Recommendation: no capacity policy

> **Invariant I19.** Holding multiple active tasks is permitted. No configuration causes `task claim`, `task start`, or `task assign` to fail because of a task count.

The only useful v0.6 signal is the existing task ownership and status data. A commander can count active tasks when it needs that information; CarryCtx does not turn the count into a role hint, warning, or policy.

### 11.5 Existing key and documentation correction

`[task].single_active_task_per_agent` is an existing public key. It remains accepted for compatibility, but it must no longer be documented as an enforced default. Its value has no effect on claim, start, or assignment in this design. The docs workstream should describe multiple active tasks as supported and may later deprecate the key through the normal compatibility process.

### 11.6 Rejected future enforcement

No `[orchestration].enforce_active_task_hint` key is defined. A capacity limit would be an execution policy owned by the commander or harness, and introducing an opt-in CarryCtx rejection is deferred until a concrete integration justifies it.

### 11.7 Documentation corrections

Resolved in the same workstream, per the docs authoring rule that contradictions are fixed rather than documented as two valid variants:

- `requirements.md:2956` and `configuration.md:302` stop describing `single_active_task_per_agent` as an enforced default-`true` limit and state that multiple active tasks are supported.
- `cli-specification.md` Section 16.3 removes "单 Agent Task 限制" from the list of checks `task claim` performs, because it performs no such check and will not.
- Both documents state plainly that multiple in-progress tasks per agent are supported; no new capacity configuration is added.

This is the deliverable of CTX-0043. For v0.6, the required behavior is only that an agent may hold multiple in-progress tasks and the shipped documentation matches that behavior. Capacity hints and warning semantics are future work and are intentionally absent from the v0.6 configuration surface.

## 12. Backward compatibility

> **Invariant I20.** The existing single-agent workflow MUST keep working unchanged. A project that never creates a Team and never writes `[orchestration]` MUST observe byte-identical behavior to 0.5.8 apart from additive fields.

### 12.1 Purely additive

Nothing here alters an existing command's success behavior for an unclassified agent:

| Addition                                                                                      | Kind                                                    |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `task assign`, `task unassign`, claimable-work read projection, `team status`, `team context` | New commands/read projections                           |
| `agent register --kind`                                                                       | New optional flag                                       |
| `task create --required-role`, `task edit --required-role`                                    | New optional flags                                      |
| `task list --status ready`                                                                    | Unchanged. `task ready` is separate                     |
| `agents.kind`, `tasks.required_role`, `tasks.team_id`                                         | New nullable columns                                    |
| `teams`, `team_members`                                                                       | New project-scoped Team tables                          |
| `[orchestration.roles]`                                                                       | New optional advisory metadata, absent by default       |
| `task.assigned`, `task.unassigned`                                                            | New event types                                         |
| `topological_depth`, `unblocks_count` in task JSON                                            | New fields; additive per the existing envelope contract |

### 12.2 Behavior changes and their gates

Four changes are not purely additive. Each is gated, and only the first is a v0.6 behavior change at all:

| Change                                                           | Gate                         | Effect when gate is off                             |
| ---------------------------------------------------------------- | ---------------------------- | --------------------------------------------------- |
| Ownership checks on `release` / `assign` / `unassign` (CTX-0044) | Actor's `kind` is `subagent` | Unclassified agents keep today's unchecked behavior |
| Lease/reclaim lifecycle and session behavior changes             | Future harness integration   | Existing behavior remains unchanged                 |

The `kind` gate on CTX-0044 is what makes an ownership check shippable in a minor release. The defect is real and worth fixing, but tightening authorization for every existing caller would be a breaking change; scoping enforcement to agents that explicitly opted into a team role fixes it exactly where the team model needs it.

### 12.3 Migration

The implementation MUST ship one forward-only project migration after the current highest bundled migration. It MUST be an immutable, checksummed migration source and MUST apply in the existing `BEGIN IMMEDIATE` transaction; the migration record is inserted before that transaction commits. It must not backfill Teams, memberships, or task associations: existing rows receive `agents.kind = NULL`, `tasks.required_role = NULL`, and `tasks.team_id = NULL`, and no Team is synthesized from Sessions, Agents, or worktrees.

The authoritative schema contract is:

| Table/field                                          | Required definition                                                         | Default | Required indexes/keys                                                                           | Foreign-key action                                                                            |
| ---------------------------------------------------- | --------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `agents.kind`                                        | Nullable `TEXT`; when non-NULL, `CHECK (kind IN ('commander', 'subagent'))` | `NULL`  | Preserve existing `agents(id)` and `agents(project_id, id)` unique key                          | None                                                                                          |
| `tasks.required_role`                                | Nullable `TEXT`; NULL or trimmed non-empty string                           | `NULL`  | `tasks(project_id, required_role, status, owner_agent_id)`                                      | None                                                                                          |
| `tasks.team_id`                                      | Nullable `TEXT`                                                             | `NULL`  | `tasks(project_id, team_id, status)` plus composite FK support key `tasks(project_id, team_id)` | `teams(project_id, id)` `ON DELETE SET NULL`                                                  |
| `teams.id`                                           | Existing stable non-empty `TEXT` primary key; `project_id` is non-NULL      | None    | `PRIMARY KEY (id)` and `UNIQUE (project_id, id)`                                                | `project_id -> projects(id) ON DELETE CASCADE`                                                |
| `teams.name`                                         | Non-NULL trimmed non-empty `TEXT`                                           | None    | `UNIQUE (project_id, name)`                                                                     | None                                                                                          |
| `teams.commander_agent_id`                           | Nullable `TEXT`; same-project member relation                               | `NULL`  | Composite FK target is `team_members` primary key `(project_id, team_id, agent_id)`             | Membership removal is rejected by the commander FK; Team deletion cascades through membership |
| `teams.created_at`, `teams.updated_at`               | Non-NULL trimmed timestamp `TEXT`                                           | None    | `teams(project_id, updated_at DESC)`                                                            | None                                                                                          |
| `team_members.project_id`                            | Non-NULL `TEXT` copied from the Team project                                | None    | Part of primary key and both composite FKs                                                      | `projects(id) ON DELETE CASCADE`                                                              |
| `team_members.team_id`                               | Non-NULL `TEXT`                                                             | None    | `PRIMARY KEY (project_id, team_id, agent_id)` and `team_members(project_id, team_id, agent_id)` | `teams(project_id, id) ON DELETE CASCADE`                                                     |
| `team_members.agent_id`                              | Non-NULL `TEXT`                                                             | None    | `team_members(project_id, agent_id, team_id)`                                                   | `agents(project_id, id) ON DELETE CASCADE`                                                    |
| `team_members.role`                                  | Nullable `TEXT`; NULL or trimmed non-empty string                           | `NULL`  | None beyond membership key                                                                      | None                                                                                          |
| `team_members.created_at`, `team_members.updated_at` | Non-NULL trimmed timestamp `TEXT`                                           | None    | `team_members(project_id, team_id, updated_at DESC)`                                            | None                                                                                          |

The exact composite constraints are mandatory, not merely query-time conventions: `team_members(project_id, team_id)` references `teams(project_id, id)`; `team_members(project_id, agent_id)` references `agents(project_id, id)`; `tasks(project_id, team_id)` references `teams(project_id, id)`; and `(teams.project_id, teams.id, teams.commander_agent_id)` references `(team_members.project_id, team_members.team_id, team_members.agent_id)` whenever the nullable commander value is present. SQLite's nullable composite-FK behavior permits a NULL commander, but no non-NULL commander can cross projects or bypass membership.

Every Team or membership write, every task-Team association write, every `agents.kind` or `tasks.required_role` write that is exposed as a state-changing command, and its corresponding audit event MUST execute in one application transaction. The event records the project ID and affected aggregate ID; a failed constraint rolls back the domain write and event together. Read projections remain read-only. No lease table, lease index, heartbeat column, runtime worker table, copied context, or prompt cache is introduced by this design.

The current CLI migration runner applies all bundled migrations greater than the database's maximum recorded version automatically when opening a writable project database, records checksums in `schema_migrations`, and commits the batch transactionally. It does **not** currently reject a database whose recorded version is newer than the binary's highest bundled version: `pending_migrations()` only filters `source.version > applied_version`, and checksum verification only checks applied versions that have a bundled source. Therefore this design makes no downgrade-safety claim. A future implementation MUST add an explicit guard before any read or write path: if `MAX(schema_migrations.version)` is greater than the binary's maximum bundled version, return `MIGRATION_REQUIRED` (or the existing unsupported-schema error) without applying migrations or mutating state; if a known migration checksum differs, return the existing checksum/migration error without mutation. The implementation MUST add tests for old-binary/new-schema opening and for checksum mismatch. Until that guard ships, a database migrated by a newer binary is unsupported by an older binary and must not be described as safely downgradable.

## 13. JSON naming convention and known inconsistency

### 13.1 The shipped convention and inconsistency

The codebase is not internally consistent, and this design does not silently pick a side. Observed in live 0.5.8 output:

| Location               | Convention                  | Example                                                     |
| ---------------------- | --------------------------- | ----------------------------------------------------------- |
| Envelope keys          | snake_case                  | `schema_version`, `command`, `success`, `data`, `meta`      |
| Entity record fields   | snake_case                  | `display_id`, `owner_agent_id`, `project_id`, `occurred_at` |
| **Event payload keys** | **mixed, historical**       | `session_id`, `handoffId`, `afterStatus`, `ownerAgentId`    |
| `doctor` data          | **mixed within one object** | `allOk` alongside `fix_command`                             |

The `doctor` case is the clearest evidence this is drift rather than an intentional two-convention design: a single response object mixes both styles.

The shipped `SuccessEnvelope`, `ErrorEnvelope`, and Rust entity records serialize
their field names as snake_case. Some command projections are hand-built and retain
historical camelCase keys, so the convention applies to the envelope and entity
records, not to an invented global rewrite of every `data` object. The old examples
in `cli-specification.md` and this design that used `schemaVersion` or `projectId`
were wrong against the implementation and are corrected to the shipped envelope.

### 13.2 Decision for this design

> **Invariant I21.** All new JSON keys introduced by this design use snake_case, in every position including event payloads.

snake_case is retained for all new envelope, entity, and event-payload keys because
it is the shipped convention for the envelope and entity records and the one external
consumers already parse.

New event payloads therefore use snake_case:

```json
{
  "event_type": "task.reclaimed",
  "payload": {
    "task_id": "01M0HNTZ3BFYHDFD8D3864VKSD",
    "display_id": "CTX-0038",
    "previous_owner_agent_id": "01M0HNR52KP9Q1B7X4D8E2GTYV",
    "new_owner_agent_id": null,
    "before_status": "in_progress",
    "after_status": "ready",
    "lease_id": "01M0HQ12ZK8N4T6R9V3B7C5DEF",
    "reason": "lease expired 41m ago",
    "forced": false
  }
}
```

This deliberately leaves existing camelCase payloads such as `task.claimed`
unchanged. Event payloads are opaque append-only JSON: historical keys are not
rewritten, aliased, or backfilled, and reads do not normalize them. New payloads use
snake_case. A future payload migration requires a separate versioned compatibility
decision; this task does not introduce dual emission or a broad schema migration.

### 13.3 Compatibility policy

Renaming existing camelCase payload keys is a breaking change to a public contract and
is **out of scope**. No option below is selected by this design:

| Option | Approach                                                              | Cost                                          |
| ------ | --------------------------------------------------------------------- | --------------------------------------------- |
| A      | Leave existing payloads camelCase forever; new ones snake_case        | Permanent split; cheapest                     |
| B      | Emit both keys in payloads for one minor release, then drop camelCase | One release of payload bloat; clean end state |
| C      | Rename at the next major version with a documented break              | Cleanest; requires waiting                    |

Any future change must be filed separately and include a versioned migration and
consumer compatibility plan. Until then, the policy above is the complete,
backward-compatible behavior.

## 14. Error codes and exit codes

Only Team lookup, assignment, ownership, dependency, and persistence errors needed by the v0.6 management primitives belong in this contract. Lease-specific errors below are future placeholders and are not part of the release surface.

New error codes, mapped onto the existing exit-code scheme where 3 is a state conflict, 7 is a missing resource, 8 is input validation, and 9 is a permission or scope failure:

| Code                 | Exit | Raised by                                       | Meaning                                                |
| -------------------- | ---- | ----------------------------------------------- | ------------------------------------------------------ |
| `TASK_NOT_OWNED`     | 9    | `release`, `assign`, `unassign`                 | Actor is neither owner nor commander (Section 4.4)     |
| `LEASE_STILL_LIVE`   | 3    | `reclaim`                                       | Lease has not expired past grace (Section 8.4)         |
| `LEASE_NOT_HELD`     | 3    | `lease --renew`                                 | No live lease for this agent to renew                  |
| `LEASE_EXPIRED`      | 3    | Recording commands                              | Agent's lease expired; it must re-lease before writing |
| `LEASE_CONFLICT`     | 3    | `lease`                                         | Another agent holds a live lease                       |
| `COMMANDER_REQUIRED` | 9    | `assign --force`                                | Operation requires `kind = commander`                  |
| `AGENT_NOT_FOUND`    | 7    | `assign --to`, `reclaim --to`                   | Existing code, reused for a nonexistent target         |
| `TEAM_NOT_FOUND`     | 7    | `team status`, `team context`                   | Referenced Team does not exist                         |
| `TEAM_MEMBER_EXISTS` | 3    | Team membership mutation (future write surface) | Agent is already a member of the Team                  |

Three error codes from an earlier revision are deliberately absent. `AGENT_TASK_LIMIT_EXCEEDED` is gone because capacity no longer fails a claim (Section 11.4); the condition is now the warning `AGENT_OVER_TASK_HINT`. `ROLE_MISMATCH` and `ROLE_NOT_DECLARED` are gone with `require_role_match` (Section 7.3): a role filter narrows a query, and an undeclared role is a `doctor` warning, so neither needs to fail a command.

New warning codes: `AGENT_OVER_TASK_HINT`, `ORPHAN_WORK_PRESERVED`, `SESSION_AUTO_ENDED`, `NO_DURABLE_RECORD`, `ROLE_UNDECLARED`, `ORCHESTRATION_ROLES_MISSING`.

The ratio is intentional. Of the conditions this design newly detects, the ones tied to data integrity — ownership, lease exclusivity, atomic reclaim — are errors, and every condition describing how a team organises itself is a warning.

`LEASE_EXPIRED` on a recording command is a deliberate choice with a caveat. It prevents an agent that lost its lease from continuing to write as though it still holds the task, which would let two agents record into one task unknowingly. The caveat: it must not cause data loss. The error therefore instructs re-leasing and retrying, and the payload the agent tried to write is echoed back in `details.rejected_content` so a compliant agent can resubmit it verbatim.

Every new error MUST carry actionable suggestions per R4. Suggestions naming a wait must include a precomputed timestamp; suggestions naming a command must be copy-pasteable.

## 15. Delivery decomposition

Ordered so that each unit ships independently with its own exit evidence, and so the failure-handling work is not last.

| Unit | Task     | Owns                                                                | Depends on    | Exit evidence                                                                         |
| ---- | -------- | ------------------------------------------------------------------- | ------------- | ------------------------------------------------------------------------------------- |
| 1    | CTX-0044 | Ownership authorization model (I2), `TASK_NOT_OWNED`, `kind` column | None          | A `subagent` cannot release a peer's task; unclassified agent still can               |
| 2    | CTX-0038 | `task assign`, `task unassign`, `task.assigned`, `task.unassigned`  | Unit 1        | Reassignment across all statuses in Section 4.3; `--force` required for `in_progress` |
| 3    | CTX-0039 | Claimable-work read projection and stable dependency ordering       | Unit 1        | Deterministic ordering and exact dependency membership                                |
| 4    | CTX-0040 | Advisory role metadata and `tasks.required_role`                    | Unit 3        | Absent metadata is indistinguishable from 0.5.8 (I6)                                  |
| 5    | CTX-0043 | Multiple active tasks remain allowed; documentation correction      | Unit 4        | Multi-holding works by default and docs match behavior                                |
| 6    | —        | Existing task-relevant context and Team read projections            | Units 2, 3    | Read projections remain advisory and no runtime lifecycle is introduced               |
| 6a   | CTX-0037 | Persistent Team schema and read-only `team status` / `team context` | Units 2, 3, 4 | Team state survives Session and linked-worktree changes; projections are read-only    |
| 7    | CTX-0041 | Document and harden MCP per-call agent identity                     | Units 1, 3    | Concurrent MCP calls under distinct identities attribute correctly                    |
| 8    | —        | Cross-document contract reconciliation                              | All prior     | Public docs match the v0.6 management primitives                                      |

CTX-0042 and CTX-0047, including task context pointers, leases, expiry, orphan detection, and reclaim, remain deferred and are not v0.6 implementation units. The `single_active_task_per_agent` documentation correction belongs to CTX-0043. Envelope naming reconciliation is a separate follow-up under Unit 8. No new implementation task is created for Team in this document: CTX-0037 owns the design gate, and the implementation slice is added to its delivery decomposition without changing unrelated task statuses.

## 16. Testing strategy

### 16.1 Unit tests

Ready-queue ordering determinism across all five keys including the ULID tiebreaker; topological depth over strong edges only; authorization matrix in Section 4.4 for all four actor kinds; role degradation rules 1 through 4.

### 16.2 Integration tests

Concurrent `task claim` preserving `TASK_ALREADY_CLAIMED` with exit 3 and never exit 5; migration from a 0.5.8 database with no backfill; 4 subagents plus 1 commander sustaining mixed assignment, claim, progress, and checkpoint traffic with zero lost audit events.

### 16.3 Contract tests

Every new command in text, JSON, and non-interactive modes; stdout/stderr separation on all new error codes; exit codes for the v0.6 management primitives; `--dry-run` immutability for assignment operations; every new error's `suggestions` array non-empty with copy-pasteable commands.

### 16.4 Team projection tests

A fixture with two linked worktrees and multiple Sessions asserts that Team membership and `tasks.team_id` remain unchanged after Session end/start and worktree switching. `team status` and `team context` must be read-only, use snake_case fields, and rebuild from durable records. No token budget or comparative context-window assertion is required.

### 16.5 Failure-injection tests (future lease design)

The following future acceptance test applies only if a separate harness integration adopts the lease/reclaim design; it is not a v0.6 gate:

1. Register `commander` and `subagent`.
2. Subagent leases and starts a task, records one `progress note`.
3. Kill the subagent process abruptly.
4. Assert `doctor` reports `tasks.orphaned_leases` as `warn` before grace and `error` after.
5. Assert a peer `subagent` reclaim before grace fails with `LEASE_STILL_LIVE` and a precomputed `reclaimable_at`.
6. Assert the commander can reclaim immediately with `--reason`.
7. Assert the recorded progress note survives and appears in the new agent's `task lease` payload.
8. Assert `task.reclaimed` exists in the event log with the previous owner recorded.

Step 7 is the whole point of the design. Step 4's severity escalation is what the incident's neutral `info` message failed to provide.

### 16.6 Backward-compatibility tests

A 0.5.8-shaped project with one unclassified agent and no Team or `[orchestration]` metadata must produce identical output for `status`, `resume`, `context`, `task list`, `task claim`, `task release`, and `doctor` apart from documented additive fields. This suite is the executable form of I20 and should gate the release.

One case is called out separately because it is the regression this revision most needs protected: an agent claiming and starting several tasks concurrently under default configuration must succeed every time, at exit code 0, with no error and no warning. This is the executable form of I19, and it is what distinguishes the recommended Option D from the rejected Option C.

## 17. Open questions for the reviewer

These are deliberately left for the reviewer to decide.

1. **Event payload naming (Section 13.3).** CTX-0048 records the minimal
   backward-compatible policy: existing payload keys remain as stored, and new
   payload keys use snake_case. Any future migration requires its own versioned
   compatibility decision; this orchestration design does not dual-emit or rewrite
   historical payloads.
2. **Lease/reclaim scope.** CTX-0042 and CTX-0047 remain useful backlog candidates, but they are not gated v0.6 implementation units. They require a concrete harness integration and a new design review.
3. **`sync` command scope contradiction.** `carryctx sync` exists in the shipped 0.5.8 CLI, while the workspace `AGENTS.md` states network/remote sync is explicitly excluded to keep CarryCtx offline-first. I did not touch it, but it contradicts the stated scope and needs a ruling: remove it, or amend the scope statement.
4. **Durability policy.** The design recommends incremental progress and checkpoints, but does not make recording cadence or completion evidence a CarryCtx admission rule. Any stricter policy belongs in the harness or a separate product decision.
5. **Whether a future lease model is warranted.** Do not add it until a harness integration demonstrates a concrete need for task-level liveness and recovery beyond existing ownership, progress, checkpoint, and handoff records.

## 18. Commander-design review note (2026-08-21)

### Decisions

- CarryCtx is a lightweight durable management tool. The execution harness owns spawning, routing, retries, worktrees, heartbeats, and concurrency limits.
- Team is a project-scoped durable coordination record. Membership and Task association survive Session boundaries and linked worktrees.
- The minimum Team model is Team, membership/role relation, commander relation, and optional Task association; existing entities provide all other detail.
- `team status` and `team context` are read-only projections rebuilt from durable structured records. They are not prompt caches or token optimizers.
- Commander grouping is situational. Inline work, batching, serial execution, and delegation are all valid; CarryCtx exposes facts and does not emit waves or enforce fan-out.
- Preserve task assignment/reassignment, dependency and scope read projections, per-call identity, atomic ownership rules, append-only audit events, and read-only Team/task context projections.
- Roles are optional advisory metadata: responsibility text and optional scope labels only. Presets, capabilities, status admission, permissions, and capacity hints are outside the CarryCtx role schema.
- Multiple active tasks per agent remain allowed. The old `single_active_task_per_agent` documentation must be corrected rather than replaced with a cap or warning policy.
- Leases, expiry, orphan detection, reclaim timing, mandatory recording cadence, and session behavior gates are future work pending a concrete harness need.
- CTX-0048 policy is adopted: all new JSON and event payload fields use `snake_case`; historical payload keys are not rewritten.

### Open risks

- The existing CLI and docs are ahead of the shipped 0.5.8 command surface. New commands and fields require separate CLI contract updates before implementation.
- A harness that needs process-level liveness or automatic recovery will require a separate lease design, including clock, ownership, and migration review.
- Existing `requirements.md`, `configuration.md`, and `cli-specification.md` still contain the legacy single-task policy and must be reconciled in the CTX-0043/contract workstream.
- Existing JSON envelope and event-payload naming drift remains outside this review.

### Changed sections

- Sections 1, 1.2, and 1.3: clarified the management-tool boundary, persistent Team model, cross-session invariants, and v0.6 scope.
- Section 3: replaced the global roster emphasis with Team status/context projections and the minimal Team CLI/schema surface.
- Sections 5 and 6: removed wave grouping from the read projection and separated durable Team context from deferred leases.
- Section 7: reduced role configuration to advisory metadata and removed runtime/policy keys.
- Section 8 and Section 9: made liveness, reclaim, and recording cadence future/advisory rather than v0.6 runtime requirements.
- Section 11 and Section 12: removed capacity policy from v0.6 and updated compatibility claims.
- This review note records the final decisions and risks.

### Verification commands

```text
carryctx --project ../carryctx-cli agent show architect --json
carryctx --project ../carryctx-cli session start --agent architect --task CTX-0037 --provider codex --json
carryctx --project ../carryctx-cli progress note --agent architect --task CTX-0037 "..."
carryctx --project ../carryctx-cli checkpoint --agent architect --task CTX-0037 --no-git --json
git diff --check
git status --short
```
