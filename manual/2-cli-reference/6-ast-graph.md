# AST 语法树图谱

- `carryctx graph scan`: 扫描并更新本地代码图谱。
- `carryctx graph query`: 搜索特定文件或符号的调用者。
- `carryctx graph explain`: 生成某个组件的语义解释。
- `carryctx graph export`: 导出图谱结构。

JSON 信封约定（0.6.0 起）：

- `graph scan` 的 `data` 键为 snake_case：`dry_run`、
  `nodes_created`、`edges_created`、`error_count`。
- 命令信封的 `command` 字段统一使用点分名称：`graph.edges`、
  `graph.add-node`、`graph.link`、`graph.extract-deps`、
  `graph.scan`、`graph.export`。
- 变更类子命令（add-node/link/extract-deps/scan）受全局 `--dry-run`
  门控：dry-run 下返回 `operation.applied = false` 的预览信封且不写库。
