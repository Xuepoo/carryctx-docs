# CarryCtx 工程与开发规范

**文档路径：** `carryctx-docs/engineering-standards.md`
**文档版本：** v0.8.0
**适用版本：** CarryCtx v0.8.x

**当前架构：** CarryCtx v0.8.x 使用 Rust 2024、Cargo、`rusqlite` 和原生 CLI 二进制，主要通过 Cargo、GitHub Releases 及平台包分发；npm 仅为可选 wrapper 分发渠道。

> **历史范围：** 本文保留的 TypeScript/Bun、`bun:sqlite`、`package.json` 和 npm-first CLI 内容属于 v0.1 设计记录，仅用于解释历史决策，不适用于 CarryCtx v0.8.x.

---

## 1. 技术基线

CarryCtx 采用统一技术栈：

```text
Language:        Rust 2024
Build Tool:      Cargo
Runtime:         Native executable
Database:        SQLite through rusqlite
Test Runner:     cargo test
Distribution:    Cargo, GitHub Releases, platform packages
Primary OS:      Linux
Secondary OS:    macOS
Planned OS:      Windows
```

Rust edition and Cargo dependency versions are fixed per release.

依赖升级通过独立 PR 完成，不允许在功能 PR 中顺带大规模升级工具链。

---

# 2. Runtime 决策（历史 v0.1）

CarryCtx v0.1 是：

```text
Bun-first CLI
```

npm 是发布渠道，但 v0.1 不承诺 Node.js Runtime 兼容。

CLI 入口：

```typescript
#!/usr/bin/env bun
```

原因：

- 使用 Bun 统一 package manager、runtime、test 和 bundler
- 直接使用 `bun:sqlite`
- 减少原生 SQLite 第三方依赖
- 后续可以生成 standalone executable

Bun 当前没有实现 `node:sqlite`，因此 v0.1 不同时维护 `node:sqlite` 与 `bun:sqlite` 两套 Adapter。

P1 阶段可以使用：

```bash
bun build --compile
```

生成独立二进制；Bun 的 standalone executable 支持包含 `bun:sqlite`。

---

# 3. 核心依赖（历史 v0.1）

建议初始依赖：

```text
citty              CLI 命令解析
@clack/prompts     交互式终端输入
zod                Runtime Schema 验证
smol-toml          TOML 解析和序列化
picomatch          Path Scope Glob 匹配
ulidx               ULID 生成
```

数据库直接使用：

```typescript
import { Database } from "bun:sqlite";
```

v0.1 不使用 ORM。

原因：

- Schema 较稳定且关系明确
- 需要直接控制 Transaction
- 需要明确管理 Migration
- 需要直接使用 SQLite PRAGMA
- 避免 ORM 抽象增加 CLI 启动成本

---

# 4. TypeScript 规范（历史 v0.1）

`tsconfig.json`：

```json
{
  "compilerOptions": {
    "lib": ["ESNext"],
    "target": "ESNext",
    "module": "Preserve",
    "moduleResolution": "bundler",
    "moduleDetection": "force",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "types": ["bun"],
    "noEmit": true,

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "exactOptionalPropertyTypes": true,
    "useUnknownInCatchVariables": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts", "tests/**/*.ts", "*.config.ts"],
  "exclude": ["dist", "coverage", ".cache"]
}
```

Bun 官方推荐 TypeScript 项目使用 `module: "Preserve"`、`moduleResolution: "bundler"`、`allowImportingTsExtensions`、`verbatimModuleSyntax` 和严格类型检查。

Bun 负责执行和转译 TypeScript。

TypeScript Compiler 负责独立类型检查：

```bash
bunx tsc --noEmit
```

---

# 5. 源代码结构

## 5.1 历史 v0.1（TypeScript/Bun）

```text
carryctx/
├── src/
│   ├── cli.ts
│   ├── commands/
│   ├── application/
│   ├── domain/
│   ├── repositories/
│   ├── adapters/
│   │   ├── config/
│   │   ├── filesystem/
│   │   ├── git/
│   │   ├── sqlite/
│   │   ├── terminal/
│   │   └── xdg/
│   ├── schemas/
│   ├── output/
│   ├── errors/
│   └── utils/
├── migrations/
├── skills/
├── templates/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── fixtures/
│   └── helpers/
├── docs/
├── scripts/
├── .github/
└── dist/
```

## 5.2 当前 v0.8.2+ Workspace（4+1 Crates，002-P1 已落地 carryctx-core）

```text
carryctx-cli/                  # Cargo workspace root
├── Cargo.toml                 # [workspace] members = crates/*
├── crates/
│   ├── carryctx-core/         # P1 已拆出：domain + repository traits + 纯 application + error
│   │   └── src/
│   │       ├── domain/        # Entity / Value Object / 状态机 / 不变量
│   │       ├── repository/    # 持久化契约（traits），无实现
│   │       ├── application/   # 纯用例（interchange/progress 等，不触 SQLite/Git/FS）
│   │       └── error.rs
│   ├── carryctx-sqlite/       # P2 目标：migrations + repository impl + state.sqlite 持久化
│   ├── carryctx-vcs/          # P3 目标：VcsBackend + Git Tier1 / jj optional
│   ├── carryctx-pack/         # P4 目标：ctxpack interchange（manifest/format_version/JSONL）
│   └── carryctx-cli/          # P5 目标：clap 解析 + commands + rendering + main 二进制
│       └── src/
│           ├── commands/
│           ├── adapter/       # 过渡期仍在根 crate，P2-P3 逐步迁入对应 crates
│           ├── application/   # 过渡期：含 SQLite/Git/FS 的用例仍在根，P2-P4 后收敛至 core
│           └── main.rs
├── migrations/project/
└── tests/
```

> **P1 交付边界（dfecd07）：**仅 `crates/carryctx-core` 已物理隔离并通过 `cargo check --workspace`；`carryctx-sqlite`/`vcs`/`pack`/`cli` 四个 crate 的完整抽离在 P2-P5 按序落地，期间根 `src/` 与 `crates/carryctx-core` 并存，CLI 契约零变化。

---

# 6. 架构分层（当前 v0.8；沿用 v0.1 分层原则，002 起以 workspace crates 物理隔离）

```text
CLI Layer               crates/carryctx-cli  (clap / commands / rendering / main)
    ↓
Application Layer       crates/carryctx-core (纯用例；过渡期部分用例仍在根 src/application)
    ↓
Domain Layer            crates/carryctx-core/domain
    ↓
Repository Interfaces   crates/carryctx-core/repository  (traits，无实现)
    ↓
Adapters                crates/carryctx-sqlite / carryctx-vcs / carryctx-pack / carryctx-cli
```

Workspace 依赖图（Cargo 强制）：

```text
core <- sqlite
core <- vcs
core <- pack
{ core, sqlite, vcs, pack } <- cli
```

`core` 禁止依赖 `rusqlite` / Git / `clap` / terminal / filesystem / network（`reqwest`/`hyper`/`rustls` 等）；仅允许 `serde`/`thiserror`/`ulid`/`chrono` 等纯数据依赖。P1 过渡期 `clap` 仍因 `TaskPriority` 的 `ValueEnum` 保留在 `core`，P5 移出。`sqlite`/`vcs`/`pack` 各自实现 `core` 定义的 traits，`cli` 聚合全部 crates 并提供二进制入口。

## CLI Layer

负责：

- 参数解析
- 命令路由
- 输出格式选择
- Exit Code
- 交互提示

不得包含业务状态转换。

## Application Layer

负责：

- Use Case
- Transaction 边界
- 权限检查
- Entity 协调
- Event 写入

## Domain Layer

负责：

- Entity
- Value Object
- 状态机
- Domain Error
- 业务不变量

Domain Layer（`crates/carryctx-core`）不得依赖：

- Bun API
- SQLite / `rusqlite`
- Git / `VcsBackend` 具体实现
- `clap` / Terminal / filesystem / network（P1 过渡期 `clap` 例外见 §6 依赖图注记）

## Adapter Layer

负责：

- Git CLI
- SQLite
- XDG Path
- TOML
- Terminal
- Clock
- ID Generator

v0.8 的实现使用 Rust 模块和 Cargo crate；SQLite adapter 通过
`rusqlite` 实现，命令入口不得绕过 application/domain/repository 分层直接执行 SQL。

---

# 7. 开发工具职责（历史 v0.1）

## 7.1 Bun

Bun 负责：

- 安装依赖
- Lockfile
- 执行 TypeScript
- 运行测试
- 打包
- 发布前构建

统一命令：

```bash
bun install
bun test
bun run
bun build
bun publish
```

必须提交：

```text
bun.lock
```

CI 使用：

```bash
bun install --frozen-lockfile
```

---

## 7.2 Oxfmt

Oxfmt 是项目唯一的主要源代码 Formatter。

负责：

- TypeScript
- JavaScript
- JSON
- JSONC
- Oxc 支持的配置文件

配置：

```text
.oxfmtrc.jsonc
```

命令：

```bash
bunx oxfmt --write .
bunx oxfmt --check .
```

禁止同时使用：

```text
Prettier
Biome Formatter
ESLint Formatting Rules
```

避免同一文件存在多个格式化来源。

---

## 7.3 Oxlint

Oxlint 是 TypeScript/JavaScript 的主要 Linter。

配置：

```text
oxlint.config.ts
```

命令：

```bash
bunx oxlint .
```

Oxlint 负责：

- Correctness
- Suspicious Pattern
- Import
- Promise
- Node/Bun Code Quality
- TypeScript Lint
- 项目自定义架构规则

Oxc 提供 TypeScript/JavaScript parser、linter 和 formatter，并支持独立配置文件。

---

## 7.4 Biome

Biome 不作为主要 Formatter，也不作为主要 JS/TS Linter。

Biome 仅负责：

- Assist
- Import Organization
- 补充结构检查
- 编辑器辅助
- Oxc 暂未覆盖的有限规则

`biome.jsonc`：

```json
{
  "$schema": "https://biomejs.dev/schemas/latest/schema.json",
  "formatter": {
    "enabled": false
  },
  "linter": {
    "enabled": false
  },
  "assist": {
    "enabled": true,
    "actions": {
      "source": {
        "organizeImports": "on"
      }
    }
  },
  "files": {
    "includes": ["src/**/*.ts", "tests/**/*.ts", "*.config.ts"]
  }
}
```

命令：

```bash
bunx biome check .
bunx biome check --write .
```

规则：

> 不允许 Oxfmt 与 Biome Formatter 同时启用。

---

## 7.5 Lefthook

Lefthook 管理 Git Hooks。

配置：

```text
lefthook.yml
```

Lefthook 是 Git hook manager，并通过配置文件定义 hook command 和 script。

安装：

```bash
just setup
```

不得依靠 npm `postinstall` 自动修改用户 Git Hook。

建议：

```yaml
pre-commit:
  parallel: true
  commands:
    format:
      run: bunx oxfmt --check {staged_files}
    lint:
      run: bunx oxlint {staged_files}
    markdown:
      glob: "*.md"
      run: bunx markdownlint-cli2 {staged_files}

commit-msg:
  commands:
    commitlint:
      run: bunx commitlint --edit {1}

pre-push:
  commands:
    verify:
      run: just check-fast
```

---

## 7.6 Commitlint

Commit message 遵循 Conventional Commits。

配置：

```text
commitlint.config.ts
```

基础配置：

```typescript
export default {
  extends: ["@commitlint/config-conventional"],
};
```

Commit 类型：

```text
feat
fix
docs
refactor
perf
test
build
ci
chore
revert
```

示例：

```text
feat(task): add atomic task claiming
fix(config): resolve project override precedence
docs(cli): document resume JSON schema
```

`@commitlint/config-conventional` 使用 Conventional Commits preset。

---

## 7.7 Markdownlint CLI2

Markdown 由：

```text
markdownlint-cli2
```

检查。

配置：

```text
.markdownlint-cli2.jsonc
```

命令：

```bash
bunx markdownlint-cli2 "**/*.md" "#node_modules" "#dist"
```

Markdownlint 负责：

- 标题层级
- 空行
- 列表风格
- Code Fence
- 行尾空格
- 链接格式
- 文档结构一致性

Markdown 不由 Oxfmt 强制格式化。

---

## 7.8 Knip

Knip 用于检测：

- 未使用依赖
- 未使用 devDependencies
- 未使用文件
- 未使用 Export
- 未使用 Type

命令：

```bash
bunx knip
```

Knip 官方定位包括检测 unused dependencies、exports 和 files。

配置：

```text
knip.jsonc
```

Knip 在 CI 中必须执行。

允许通过配置显式声明：

- CLI Entry
- Migration Entry
- Test Fixture
- Dynamic Import
- Skill Resource
- Package Export

不得通过大量全局 ignore 使 Knip 失去作用。

---

## 7.9 Just

`just` 是项目统一的人类命令入口。

`package.json` Scripts 是底层命令。

`justfile` 是开发者和 Contributor 的标准入口。

`just` 是 command runner，而不是 build system，并支持从子目录调用 recipe。

核心 Recipe：

```text
just setup
just dev
just build
just test
just test-unit
just test-integration
just typecheck
just lint
just fmt
just fmt-check
just docs
just knip
just check-fast
just check
just ci
just act
just package
just release-check
```

示例：

```just
setup:
    bun install
    bunx lefthook install

dev *args:
    bun run src/cli.ts {{args}}

typecheck:
    bunx tsc --noEmit

lint:
    bunx oxlint .

fmt:
    bunx oxfmt --write .
    bunx biome check --write .

fmt-check:
    bunx oxfmt --check .
    bunx biome check .

test:
    bun test

docs:
    bunx markdownlint-cli2 "**/*.md" "#node_modules" "#dist"

knip:
    bunx knip

check-fast:
    just typecheck
    just lint
    just test

check:
    just fmt-check
    just typecheck
    just lint
    just docs
    just knip
    just test

ci:
    just check
    just actionlint
    just package
```

---

## 7.10 Actionlint

Actionlint 静态检查：

```text
.github/workflows/*.yml
.github/workflows/*.yaml
```

命令：

```bash
actionlint
```

它用于检查 GitHub Actions 的：

- YAML Syntax
- Expression
- Matrix
- Action Input
- Shell Script
- Workflow Schema

Actionlint 可以自动发现仓库中的 Workflow 并检查错误。

配置：

```text
.github/actionlint.yaml
```

---

## 7.11 act

`act` 用于在本地运行 GitHub Actions Workflow。

命令：

```bash
just act
```

或：

```bash
act pull_request
```

`act` 的职责是本地预检，不是 CI 的 Source of Truth。

最终验收仍以 GitHub Actions Runner 结果为准。

`act` 官方定位是本地运行和测试 GitHub Actions Workflow。

---

# 8. `package.json` 规范（历史 v0.1）

```json
{
  "name": "@xuepoo/carryctx",
  "version": "0.1.0",
  "description": "Persistent project context for coding agents",
  "type": "module",
  "packageManager": "bun@1.3.14",
  "bin": {
    "carryctx": "./dist/cli.js"
  },
  "files": [
    "dist",
    "migrations",
    "skills",
    "templates",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "dev": "bun run src/cli.ts",
    "build": "bun build src/cli.ts --target=bun --outfile=dist/cli.js",
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "lint": "oxlint .",
    "format": "oxfmt --write .",
    "format:check": "oxfmt --check .",
    "biome": "biome check .",
    "markdownlint": "markdownlint-cli2",
    "knip": "knip"
  }
}
```

`package.json` 中的工具版本应固定为精确版本。

禁止：

```json
"typescript": "^7.0.2"
```

使用：

```json
"typescript": "7.0.2"
```

`bun.lock` 提供完整依赖锁定。

---

# 9. 工具配置文件（历史 v0.1）

仓库根目录必须包含：

```text
package.json
bun.lock
tsconfig.json
justfile
lefthook.yml
oxlint.config.ts
.oxfmtrc.jsonc
biome.jsonc
commitlint.config.ts
.markdownlint-cli2.jsonc
knip.jsonc
.actrc
.github/actionlint.yaml
```

不引入：

```text
.prettierrc
eslint.config.js
.husky/
Makefile
```

除非后续 ADR 明确修改工具链。

---

# 10. 测试规范（历史 v0.1）

使用：

```text
bun:test
```

以上 `bun:test` 和目录约定仅适用于 v0.1 TypeScript 实现。v0.8 使用
Rust 的 `cargo test`；当前测试布局和命令以 `carryctx-cli` 仓库的 Cargo 配置为准。

测试目录：

```text
tests/unit/
tests/integration/
tests/fixtures/
tests/helpers/
```

## Unit Test

覆盖：

- Domain State Machine
- Config Merge
- Task Dependency
- Context Ranking
- Error Mapping
- Path Resolution

## Integration Test

使用临时 Git repository 测试：

- `carryctx init`
- Git common directory
- 多 worktree
- SQLite transaction
- Checkpoint
- Resume
- Task claim race
- Database migration

## CLI Snapshot Test

测试：

- Human-readable output
- JSON output
- Error output
- Exit Code

Snapshot 中不得包含：

- 绝对用户路径
- 随机时间
- 不稳定 ULID
- 平台特定分隔符

需要通过 Fixture Normalizer 归一化。

---

# 11. GitHub Actions（历史 v0.1）

建议 Workflow：

```text
.github/workflows/
├── ci.yml
├── release.yml
└── codeql.yml
```

CI Job：

```text
quality
unit-test
integration-test
actionlint
package-smoke
```

`quality`：

```text
bun install --frozen-lockfile
just fmt-check
just typecheck
just lint
just docs
just knip
```

`package-smoke`：

1. `bun run build`
2. `npm pack`
3. 在临时目录安装 tarball
4. 执行 `carryctx --version`
5. 初始化临时 Git 仓库
6. 执行 `carryctx init`
7. 执行 `carryctx status --json`

以上 Workflow、Bun 命令和 npm smoke test 是 v0.1 历史记录，不是 v0.8 的
CI 要求。v0.8 的发布 workflow 使用 `cargo build --release --locked` 构建各平台
原生二进制，并校验 tag、Cargo 版本和发布资产后再生成平台包。

---

# 12. Git Hook 与 CI 分工（历史 v0.1）

## Pre-commit

只运行快速检查：

- Oxfmt Check
- Oxlint staged files
- Markdownlint staged Markdown
- Biome Assist Check

目标时间：

```text
< 5 seconds
```

## Commit-msg

运行 Commitlint。

## Pre-push

运行：

- Typecheck
- Unit Test
- Oxlint

## CI

运行完整：

- Format
- Typecheck
- Lint
- Biome
- Markdownlint
- Knip
- Unit Test
- Integration Test
- Actionlint
- Package Smoke Test

Git Hook 不能替代 CI。

以上 Hook/CI 命令属于 v0.1 工具链记录。v0.8 的最低验证基线是
`cargo fmt --check`、`cargo check`、`cargo clippy --workspace -- -D warnings`
和 `cargo test`；发布前还必须验证 `cargo build --release --locked` 及目标平台资产。

---

# 13. 版本管理（历史 v0.1）

## Runtime 与 Compiler

精确固定：

```text
Bun 1.3.14
TypeScript 7.0.2
```

## npm Dev Dependencies

在 `package.json` 中使用精确版本。

## 外部 CLI

以下工具不一定作为 npm Dependency：

```text
just
act
actionlint
```

其最低版本记录在：

```text
docs/development.md
```

CI 中使用固定版本或固定 Action Commit。

---

# 14. Release 规范（历史 v0.1）

发布前执行：

```bash
just release-check
```

必须通过：

- Clean Git Worktree
- Format
- Typecheck
- Lint
- Markdownlint
- Knip
- Tests
- Actionlint
- Package Smoke Test
- Version Consistency
- Changelog Check

npm 包必须包含：

```text
dist/
migrations/
skills/
templates/
README.md
LICENSE
```

不得包含：

```text
src/
tests/
coverage/
.carryctx/
state.sqlite
```

以上 npm 包内容和 `just release-check` 流程仅为 v0.1 历史记录。v0.8 发布以
Cargo crate、GitHub Releases 原生二进制和平台包为准；npm 仅作为可选的带平台
原生二进制 wrapper 渠道，不是 CLI 的唯一发布物。

---

# 15. 代码质量原则

1. 不使用 `any`，除非有明确注释和边界封装。
2. 外部输入必须经过 Runtime Schema 验证。
3. Error 必须转换为 CarryCtx Domain Error。
4. 所有 SQL 使用 Parameter Binding。
5. Task 状态转换必须集中管理。
6. 所有关键写操作必须产生 Event。
7. 时间必须以 UTC ISO 8601 持久化。
8. 文件路径在数据库中使用规范化绝对路径或明确的 repository-relative path。
9. JSON 输出视为公共 API。
10. 不允许 CLI Command 直接执行 SQL。

---

# 16. Definition of Done

一个功能完成必须满足：

- 需求已实现
- Domain Test 已添加
- Integration Test 已添加
- JSON Output 已定义
- Error Code 已定义
- 文档已更新
- `just check` 通过
- `just package` 通过
- 没有新增 Knip 问题
- 没有未解释的 Lint Ignore
- Commit 符合 Conventional Commits

---

# 17. 最终工具职责矩阵（历史 v0.1）

| 工具              | 唯一职责                      |
| ----------------- | ----------------------------- |
| TypeScript        | 类型检查                      |
| Bun               | Runtime、Package、Test、Build |
| Oxfmt             | 主要 Formatter                |
| Oxlint            | 主要 JS/TS Linter             |
| Biome             | Assist 与补充检查             |
| Lefthook          | Git Hook 管理                 |
| Commitlint        | Commit Message                |
| Markdownlint CLI2 | Markdown                      |
| Knip              | 未使用代码和依赖              |
| Actionlint        | GitHub Actions 静态检查       |
| act               | 本地 Workflow 预检            |
| just              | 统一开发命令入口              |

任何新工具加入前，必须说明它是否与现有职责重叠。
