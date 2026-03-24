# Python Harness

[English](README.md)

一个用于构建 Python 项目的元项目（Meta-Project），提供 AI 辅助的规范驱动开发。包含项目脚手架、开发规范、可复用技能，以及由 Claude Code 驱动的多 Agent 并行工作流。

## 工作原理

```
你 ←→ 规划 Agent                         编码 Agent 1        编码 Agent 2
      │                                      │                    │
      │ 讨论需求和设计                         │                    │
      │ 编写功能规格                           │                    │
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

1. **规格** — 与规划 Agent 讨论需求，它会编写 `specs/functional-spec.md`。
2. **分解** — 规划 Agent 将规格拆分为独立任务，放入 `.tasks/backlog/`。
3. **分发** — 规划 Agent 创建 git worktree 并启动编码 Agent。
4. **开发** — 编码 Agent 并行工作：TDD 开发，提交到功能分支。
5. **合并** — 规划 Agent 审查代码，运行质量检查，合并到 main。

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
│   ├── spec-driven-dev/   规格编写与评审
│   ├── test-runner/       质量检查
│   └── code-review/       代码审查
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
