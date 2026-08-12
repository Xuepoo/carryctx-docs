# 决策记录 (Decision)

记录架构或设计决策（ADR）。

- `carryctx decision add --title <TEXT> [--task <ID>]`: 记录新决策。
- `carryctx decision list [--task <ID>]`: 列出项目决策（可选按任务过滤）。
- `carryctx decision show <REF>`: 查看决策详情。
- `carryctx decision search <KEYWORD>`: 按关键字或内容搜索。
- `carryctx decision supersede <REF> --by <REF>`: 用新决策取代旧决策。

决策不允许直接删除；需要修正时应使用 `supersede` 保留历史关系。
