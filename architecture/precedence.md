# Instruction Precedence Rules

As CarryCtx aggregates context from multiple layers (Agent Defaults, Project Rules, Active Presets, Workflows, User Prompts), agents will inevitably encounter conflicting instructions.

To prevent erratic behavior, all CarryCtx-compliant agents MUST resolve conflicts using the following strict **Hierarchy of Precedence** (1 being the highest priority, overriding all below it).

## Hierarchy of Precedence

### 1. Platform Security & Sandbox Policies (Highest)

**Scope**: Host machine, plugin adapter, and preset manifests.
**Definition**: Rules defined in `.carryctx/presets.lock` or `preset.schema.json` permissions block, as well as the Agent's root safety guidelines.
**Example**: "Agent is not permitted to execute `rm -rf /` or `git push` without confirmation."
**Conflict Resolution**: Cannot be overridden by ANY entity, including explicit User Prompts or Project Rules. If the user asks the agent to break out of its filesystem scope, the agent must refuse.

### 2. Explicit User Prompts

**Scope**: Current chat session.
**Definition**: Direct conversational instructions provided by the human user.
**Example**: "Ignore the Python style guide for this specific script and just write it quickly."
**Conflict Resolution**: Overrides Workflows, Rules, and Personas. The human user is the ultimate authority in their local session, as long as the request does not violate Level 1 Security constraints.

### 3. Active Task Workflow (SOPs)

**Scope**: Current active `carryctx task`.
**Definition**: The current active step in a `.carryctx/workflows/*.md` blueprint that the agent is executing.
**Example**: "Step 2: Run all unit tests before creating a PR."
**Conflict Resolution**: Overrides general Project Rules. If a workflow explicitly demands a behavior for a specific task lifecycle phase, that behavior wins over generic repository rules.

### 4. CarryCtx Project Rules (Constraints)

**Scope**: Entire repository.
**Definition**: Rules loaded from `.carryctx/rules/` per the `use-carryctx` skill's presets-rules-personas guidance (e.g., `.carryctx/rules/frontend.md` or rules inherited from active Presets).
**Example**: "Always use `snake_case` for variables. Never use Tailwind CSS."
**Conflict Resolution**: Overrides Personas and General Knowledge. These are the absolute engineering standards of the project.

### 5. Personas & Profiles

**Scope**: Agent communication and stylistic behavior.
**Definition**: Configurations defined in `.carryctx/personas/` or Preset profiles (e.g., `Reviewer`, `Architect`).
**Example**: "Respond in concise technical Chinese. Do not use emojis."
**Conflict Resolution**: Overrides General Knowledge. Shapes how the agent presents the output, but cannot override functional rules or user instructions.

### 6. Agent General Knowledge (Lowest)

**Scope**: Base Model capabilities.
**Definition**: The underlying LLM's pre-training data and default system prompts not managed by CarryCtx.
**Example**: The LLM's natural tendency to write code in a certain way.
**Conflict Resolution**: Yields to all of the above.

---

## Resolution Matrix Example

**Scenario**:

1. Base LLM (Level 6) defaults to formatting output nicely with emojis.
2. Preset Persona (Level 5) says: "No emojis, purely technical text."
3. Preset Rule (Level 4) says: "All database queries must use an ORM."
4. Active Workflow (Level 3) says: "Write a raw SQL migration script."
5. User Prompt (Level 2) says: "Actually, let's use the ORM for this migration instead."
6. User Prompt (Level 2) also says: "Add an emoji at the end of the response."

**Resulting Behavior**:

- The agent **will** add an emoji (User Prompt Level 2 overrides Persona Level 5).
- The agent **will** use the ORM (User Prompt Level 2 overrides Workflow Level 3).
