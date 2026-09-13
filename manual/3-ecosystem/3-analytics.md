# 分析与统计 (Stats)

洞察项目效能。

- `carryctx stats`: 计算 Agent 总工时、任务完成率和图谱复杂度。支持输出 markdown, json 或 csv 格式。
- 空状态恢复提示（自 0.11.2 起，#183）：本地无 CarryCtx 状态、但仓库带有
  in-repo 发布 ref（`refs/heads/carryctx-snapshots`，clone 后通常可见为
  `origin/carryctx-snapshots`）时，文本模式 `stats` 会打印恢复指引
  （`git fetch` + `carryctx init --non-interactive` +
  `carryctx import --from-git <ref> --mode replace --yes`）。仅提示，
  不自动拉取或导入。
