# AST 语法树图谱

- `carryctx graph scan`: 扫描并更新本地代码图谱。
- `carryctx graph extract-deps <path>`: 按需从单个文件提取 `depends_on` 边。
- `carryctx graph edges <target>`: 列出与某节点相连的所有边（ULID、精确
  节点名或无歧义名称后缀均可，0.11.4 起）。
- `carryctx graph add-node` / `graph link`: 手动建节点、连边。
- `carryctx graph export`: 导出图谱结构。

JSON 信封约定（0.6.0 起）：

- `graph scan` 的 `data` 键为 snake_case：`dry_run`、
  `nodes_created`、`edges_created`、`error_count`。
- 命令信封的 `command` 字段统一使用点分名称：`graph.edges`、
  `graph.add-node`、`graph.link`、`graph.extract-deps`、
  `graph.scan`、`graph.export`。
- 变更类子命令（add-node/link/extract-deps/scan）受全局 `--dry-run`
  门控：dry-run 下返回 `operation.applied = false` 的预览信封且不写库。

节点解析（自 0.11.4 起，CTX-0168）：`graph edges <target>` 按 ULID 精确
匹配、节点名精确匹配、无歧义名称后缀（`ends_with`，与 `export --focus`
一致）解析；唯一命中返回该节点的边，未命中返回 `RESOURCE_NOT_FOUND`，
多命中返回 `VALIDATION_FAILED` 并列出候选。

Rust 依赖提取（自 0.11.4 起，CTX-0169）：`extract-deps` / `scan` 展开
brace 分组 `use`（`use crate::path::{a, b};` 含嵌套分组、`self`/`Self`、
glob、`as` 别名）为逐个模块依赖；含字面量 `{`、`}` 或 `*` 的片段永不
作为依赖输出。
