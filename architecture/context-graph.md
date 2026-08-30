# Context Graph Architecture

**Document Path**: `carryctx-docs/architecture/context-graph.md`  
**Status**: Draft (Phase 2)

## 1. Overview

As CarryCtx evolves from a linear state tracker into the "Git of Agent Context", it needs to support complex semantic relationships. The Context Graph allows agents to reason about the codebase holistically, answering questions like:

- "Why was this file changed?"
- "What tasks depend on this module?"
- "Which bug is blocking this feature?"

## 2. Universal Nodes (Entities)

CarryCtx already uses ULIDs (Universally Unique Lexicographically Sortable Identifiers) for all core entities (`tasks`, `agents`, `sessions`, `progress_items`, `checkpoints`). Because ULIDs are globally unique, we do not need to explicitly partition the graph by table type.

To represent codebase artifacts (files, modules) and conceptual artifacts (bugs, decisions) that don't fit into the existing core tables, we introduce a generic **`graph_nodes`** table:

```sql
CREATE TABLE graph_nodes (
    id TEXT PRIMARY KEY,          -- ULID
    node_type TEXT NOT NULL,      -- 'file', 'module', 'decision', 'bug', 'architecture'
    name TEXT NOT NULL,           -- e.g., 'src/main.rs', 'AuthModule', 'ADR-001'
    description TEXT,             -- Human/Agent readable summary
    metadata TEXT NOT NULL,       -- JSON format, extra details
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_name ON graph_nodes(name);
```

_Note: Existing core entities (Tasks, Agents) act as implicit graph nodes._

## 3. Universal Edges

To map the relationships between any two nodes (whether they live in `graph_nodes` or core tables like `tasks`), we introduce a **`graph_edges`** table:

```sql
CREATE TABLE graph_edges (
    source_id TEXT NOT NULL,      -- ULID of the source node
    target_id TEXT NOT NULL,      -- ULID of the target node
    relation_type TEXT NOT NULL,  -- 'depends_on', 'changed', 'fixed', 'related_to', 'blocked_by', 'implements'
    created_at TEXT NOT NULL,
    created_by TEXT,              -- ULID of the Agent/Session that created this edge
    metadata TEXT NOT NULL,       -- JSON format (e.g., {"line_numbers": [10, 20], "commit_hash": "..."})
    PRIMARY KEY (source_id, target_id, relation_type)
);

CREATE INDEX idx_graph_edges_target ON graph_edges(target_id);
CREATE INDEX idx_graph_edges_relation ON graph_edges(relation_type);
```

## 4. Common Semantic Relations

- **`depends_on`**: Task A depends on Task B; Module A depends on Module B.
- **`changed`**: Task/Session modified File A.
- **`fixed`**: Task A fixed Bug B.
- **`related_to`**: Loose coupling between two entities for context discovery.
- **`blocked_by`**: Progress/Task is blocked by Bug/Decision.
- **`implements`**: Task implements Decision/Architecture.

## 5. Query Patterns

Agents interacting via MCP will have access to graph queries:

- **Impact Analysis**: "Find all `module` nodes where `relation_type = 'depends_on'` and `target_id = <Module_ULID>`."
- **Context Gathering**: "Find all `file` nodes where `relation_type = 'changed'` and `source_id = <Task_ULID>`."
- **Root Cause**: "Find all `bug` nodes where `relation_type = 'related_to'` and `target_id = <File_ULID>`."
