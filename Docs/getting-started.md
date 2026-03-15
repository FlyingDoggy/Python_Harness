# 快速开始指南

## 前置条件

| 工具 | 安装方式 |
|------|---------|
| Python 3.11+ | `uv python install 3.11` |
| uv | https://docs.astral.sh/uv/getting-started/installation/ |
| Git 2.30+ | https://git-scm.com/ |
| Claude Code | `npm install -g @anthropic-ai/claude-code` |

## 核心概念

本工作台采用**规划 Agent + 编码 Agent** 的并行协作模式：

- **规划 Agent**：运行在 Harness 根目录，与你直接交互。负责需求讨论、规格编写、任务分解、分发和合并。
- **编码 Agent**：通过 git worktree 在独立目录中运行，从任务队列中领取任务，独立完成 TDD 开发。可以有多个编码 Agent 并行工作。

你与规划 Agent 讨论的同时，编码 Agent 可以并行开发——无需等待。

## 工作流程

### 1. 创建新项目

在 Harness 根目录打开 Claude Code，说：

> "创建一个新的 CLI 项目 my-tool，用于批量处理 CSV 文件"

规划 Agent 使用 `project-init` skill 自动完成：
- 创建 `Workspace/my-tool/` 目录结构（含 `.tasks/` 任务队列）
- 生成 `pyproject.toml`、`.gitignore`、自包含的 `CLAUDE.md`
- 初始化 uv 虚拟环境和 Git 仓库
- 生成功能规格模板

可选模板：`default`、`cli-app`、`web-api`、`data-pipeline`

### 2. 讨论需求和编写规格

与规划 Agent 讨论：
- 软件整体构想和功能设计
- 关键技术决策（架构、组件选型），规划 Agent 会提供利弊分析
- 优先级排序

讨论结果会被写入 `specs/functional-spec.md`（中文）。

### 3. 任务分解与分发

规格确认后，规划 Agent 将功能拆分为独立的编码任务：
- 每个任务写成一个 Markdown 文件，放入 `.tasks/backlog/`
- 规划 Agent 决定需要几个编码 Agent
- 将任务分配到 `.tasks/queue-agent-1/`、`.tasks/queue-agent-2/` 等

### 4. 并行开发

规划 Agent 为每个编码 Agent 创建 git worktree 并启动：

```bash
git worktree add ../my-tool-agent-1 -b feat/csv-parser main
```

每个编码 Agent 独立执行：
1. 从队列中领取任务
2. 创建功能分支
3. TDD 开发（先写测试 → 再写实现）
4. 运行质量检查（pytest + ruff + mypy）
5. 标记任务完成

**同时**，你可以继续与规划 Agent 讨论下一批功能。

### 5. 审查与合并

编码 Agent 完成后，规划 Agent：
- 运行完整质量检查
- 对照规格进行代码审查
- 按正确顺序合并到 main
- 清理 worktree

## VS Code 中的并行开发

在一个 VS Code 窗口中即可完成：

```
VS Code 窗口（打开 Workspace/my-project/）
│
├── 侧边栏：Claude Code 扩展 → 规划 Agent（你在这里交互）
│
├── 终端 Tab 1：cd ../my-project-agent-1 && claude
│   └── 编码 Agent 1（在 feat/user-auth 分支工作）
│
├── 终端 Tab 2：cd ../my-project-agent-2 && claude
│   └── 编码 Agent 2（在 feat/csv-export 分支工作）
│
└── 终端 Tab 3：你自己的 shell（git、uv 等）
```

## 项目目录结构

```
Workspace/my-tool/
├── CLAUDE.md              自包含的编码 Agent 指令
├── pyproject.toml          uv 项目配置
├── .tasks/                 任务队列
│   ├── backlog/            待分配
│   ├── queue-agent-N/      各 Agent 的队列
│   ├── in-progress/        执行中
│   ├── done/               已完成待审查
│   └── merged/             已合并归档
├── specs/                  功能规格（中文）
├── src/<package>/          源代码（src layout）
├── tests/{unit,integration,e2e}/  测试套件
├── docs/                   文档
└── reports/                生成的报告
```

## 常用命令（在项目目录内执行）

```bash
uv sync                      # 安装依赖
uv run pytest                # 运行测试
uv run ruff check .          # lint 检查
uv run ruff format .         # 格式化
uv run mypy src/             # 类型检查
```

## 语言约定

- 规格说明、文档：**中文**
- 代码、注释、commit message：**英文**
