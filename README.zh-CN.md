# Python Harness

[English](README.md)

一个用于构建 Python 项目的元项目（Meta-Project），提供 AI 辅助的规范驱动开发。包含项目脚手架、开发规范、可复用技能，以及由 Claude Code 驱动的多 Agent 并行工作流。

## 工作原理

```
你 ←→ 规划 Agent                         编码 Agent 1        编码 Agent 2
      │                                      │                    │
      │ 访谈 → 写 PRD（Epic 级）               │                    │
      │  或对话确认需求（Patch/Feature 级）      │                    │
      │ 编写/更新规格（分期 + 验收标准）          │                    │
      │ 分解为任务                             │                    │
      │                                      │                    │
      ├── 分发任务 ─────────────────────────→ 在分支上 TDD 开发     │
      ├── 分发任务 ──────────────────────────────────────────→ 在分支上 TDD 开发
      │                                      │                    │
      │ 继续讨论下一批功能                      │ 开发 & 测试         │ 开发 & 测试
      │                                      │                    │
      │                            完成 ←─────┘                    │
      │ 审查 & 合并                            │          完成 ←────┘
      │ 审查 & 合并                            │                    │
      │                                      │                    │
      │ （每完成5个阶段）自动 PRD 同步           │                    │
```

**规划 Agent** 运行在本目录——与你交互，编写规格，将工作拆分为任务，分发给编码 Agent，审查并合并代码。

**编码 Agent** 通过 git worktree 在独立目录中运行——领取任务，先写测试再写实现，通过质量检查。它们不会合并到 main。

你可以在编码 Agent 并行工作的同时，继续与规划 Agent 讨论下一批功能。

## 快速开始

### 前置条件

- Python 3.11+（`uv python install 3.11`）
- [uv](https://docs.astral.sh/uv/getting-started/installation/)
- Git 2.30+
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

### 创建项目

在本目录打开 Claude Code，说：

> "创建一个新的 CLI 项目 my-tool，用于批量处理 CSV 文件"

规划 Agent 会自动在 `Workspace/my-tool/` 下创建完整的项目结构。

### 可用模板

| 模板 | 技术栈 | 适用场景 |
|------|--------|---------|
| `default` | 纯 Python | 库、工具 |
| `cli-app` | click + rich | 命令行工具 |
| `web-api` | FastAPI + uvicorn | REST API 服务 |
| `web-app` | FastAPI + React/TypeScript/Vite/Tailwind | 全栈 Web 应用 |
| `data-pipeline` | polars | 数据处理管道 |

## 开发工作流

规划 Agent 根据需求规模自适应流程：

| 层级 | 规模 | 流程 | 需要 PRD？ |
|------|------|------|----------|
| **Patch** | 1 个任务 | 对话确认 → 任务 → 分发 | 否 |
| **Feature** | 2-5 个任务 | 对话确认 → 追加 Phase 到规格 → 任务 | 否 |
| **Epic** | 5+ 个任务 | PRD 访谈 → 规格 → 任务 | 是 |

1. **讨论 & PRD** — Epic 级：深度访谈产出 PRD；Patch/Feature 级：对话确认即可。
2. **规格** — 创建或更新 `specs/functional-spec.md`，包含分期、验收标准和来源追溯。
3. **分解** — 将每个 Phase 拆分为独立任务放入 `.tasks/backlog/`。
4. **分发** — 创建 git worktree 并启动编码 Agent。
5. **开发** — 编码 Agent 并行工作：TDD 开发，提交到功能分支。
6. **合并** — 规划 Agent 审查代码，运行质量检查，合并到 main。
7. **PRD 同步** — 每完成 5 个对话确认来源的 Phase，自动生成合并 PRD 用于合规追溯。

所有项目遵循严格规则：
- 没有规格不写代码
- 没有测试不写实现（TDD）
- 未通过检查不合并：`pytest` + `ruff check` + `ruff format --check` + `mypy`
- 禁止直接提交到 `main`

## 仓库结构

```
├── CLAUDE.md          Agent 指令（规划 Agent 读取）
├── Specs/             Harness 自身的规格说明
├── Skills/            可复用的 Claude Code 技能
│   ├── project-init/      项目脚手架
│   ├── write-a-prd/       PRD 访谈与创建
│   ├── prd-to-plan/       PRD 转规格分期
│   ├── prd-sync/          自动汇总 PRD（已完成 Phase 合规回填）
│   ├── spec-driven-dev/   规格编写与评审
│   ├── design-an-interface/ 接口设计（"双方案对比"）
│   ├── test-runner/       质量检查
│   ├── code-review/       代码审查
│   ├── improve-codebase-architecture/ 架构优化（模块深化）
│   ├── frontend-design/   高质量 UI 设计
│   └── auto-improve/      自动化改进循环
├── Templates/         项目模板
│   ├── default/           基础模板（所有模板共用）
│   ├── cli-app/           Click CLI 脚手架
│   ├── web-api/           FastAPI 脚手架
│   ├── web-app/           全栈应用：FastAPI 后端 + React 前端
│   └── data-pipeline/     Polars 数据管道脚手架
├── Prompts/           可复用模板
│   ├── spec-template.md       功能规格结构
│   ├── task-template.md       任务文件格式
│   ├── review-checklist.md    代码审查清单
│   └── test-plan-template.md  测试计划结构
├── Docs/              文档
└── Workspace/         所有项目的工作区（已 gitignore）
```

## 生成的项目结构

`Workspace/` 下的每个项目包含：

```
my-project/
├── CLAUDE.md              自包含的编码 Agent 指令
├── pyproject.toml         uv 配置（依赖、工具、构建）
├── .tasks/                任务队列（Agent 协调）
│   ├── backlog/           待分配
│   ├── queue-agent-N/     各 Agent 的工作队列
│   ├── in-progress/       执行中
│   ├── done/              已完成待审查
│   └── merged/            已归档
├── specs/                 功能规格（中文）
├── prd/                   PRD 文档（自动同步）
├── src/<package>/         源代码（src layout）
├── tests/                 unit / integration / e2e
└── reports/               覆盖率等报告
```

## 约定

| 内容 | 语言 |
|------|------|
| 规格说明、文档 | 中文 |
| 代码、注释、提交信息 | 英文 |

提交格式：`<type>: <description>` — type 取值：`feat`、`fix`、`refactor`、`test`、`docs`、`chore`

分支命名：`feat/`、`fix/`、`refactor/`、`docs/`、`test/` + 描述

## 许可证

私有项目。
