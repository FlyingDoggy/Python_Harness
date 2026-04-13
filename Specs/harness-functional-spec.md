# Python Harness 功能规格说明书

**版本**: 0.2.0
**日期**: 2026-04-13
**状态**: 活跃维护

**变更记录**:
- v0.2.0 (2026-04-13): 同步 CLAUDE.md 演进——新增 Tiered Workflow、Critic Review Loop、PRD 全链路（write→plan→sync）、12 Skills、web-app 模板、双语 README 规则；移除 MCPs/Scripts 幽灵目录
- v0.1.0 (2025-03-15): 初始草案

---

## 1. 概述

### 1.1 项目定位

Python Harness 是一个**元项目（Meta-Project）**——它本身不是一个应用程序，而是一个用于创建、管理和开发 Python 项目的**开发工作台**。它为基于 Claude Code 的 AI 辅助开发提供统一的基础设施，包括项目脚手架、开发规范、工具链集成和跨项目复用的资源。

### 1.2 核心理念

- **规范驱动开发（Spec-Driven Development）**：先写规范，再写代码，以规范为核心驱动整个开发流程
- **AI 原生工作流**：以 Claude Code 为主要开发工具，通过 Skills、MCP、Prompts 实现高效的人机协作
- **规划与执行分离**：人类与规划 Agent 进行高层讨论（需求、架构、决策），规划 Agent 将工作分解为任务并分发给多个编码 Agent 并行执行
- **项目隔离**：每个项目拥有独立的虚拟环境、Git 仓库和工作空间，互不干扰
- **约定优于配置**：通过统一的项目结构和开发约定，减少每次新建项目时的决策成本

### 1.3 适用场景

- 快速启动新的 Python 项目（CLI 工具、Web 服务、数据处理、自动化脚本等）
- 需要规范化管理多个相关或独立 Python 项目
- 使用 Claude Code 进行 AI 辅助开发的工作环境

---

## 2. 系统架构

### 2.1 目录结构

```
Python_Harness/                  # Harness 根目录（自身是一个 Git 仓库）
├── CLAUDE.md                    # Harness 级别的 Agent 配置（双角色 + Tiered Workflow）
├── .gitignore                   # 排除 Workspace 下的所有项目
│
├── Specs/                       # Harness 自身的规格说明
│   └── harness-functional-spec.md
│
├── Skills/                      # 跨项目复用的 Agent Skills（12 个）
│   ├── project-init/            # 项目初始化
│   ├── write-a-prd/             # PRD 深度访谈与编写
│   ├── prd-to-plan/             # PRD → 规格 Phase 转换
│   ├── prd-sync/                # PRD 自动回填（5+ 对话确认 Phase 后触发）
│   ├── spec-driven-dev/         # 规范编写与评审
│   ├── critic-review/           # Critic Agent 对抗性审查
│   ├── design-an-interface/     # 接口设计（"Design It Twice"）
│   ├── test-runner/             # 质量门禁（test/lint/type）
│   ├── code-review/             # 代码审查
│   ├── improve-codebase-architecture/ # 架构改进（模块深化）
│   ├── frontend-design/         # 前端 UI 设计质量
│   └── auto-improve/            # 自主改进循环（自动 PR）
│
├── Prompts/                     # 可复用的提示词模板
│   ├── spec-template.md         # 功能规格模板
│   ├── task-template.md         # 任务文件模板
│   ├── review-checklist.md      # 代码审查清单
│   └── test-plan-template.md    # 测试计划模板
│
├── Templates/                   # 项目模板（5 种）
│   ├── default/                 # 通用 Python 库/工具
│   ├── cli-app/                 # CLI 应用
│   ├── web-api/                 # Web API 服务
│   ├── data-pipeline/           # 数据处理管道
│   └── web-app/                 # 复合 Web 应用（Python 后端 + React 前端）
│
├── Docs/                        # Harness 文档
│   └── getting-started.md       # 快速开始指南
│
└── Workspace/                   # 所有项目的工作区（.gitignore 排除）
    ├── AI-PPT/                  # AI PPT 架构师
    ├── FC-CDT-AI-OPS/           # AI DevOps 平台
    ├── flowdeck/                # Flow Deck 项目
    ├── Product-Knowledge-Base/  # 产品知识库
    ├── TA-Doc-Parsing-Tool/     # 文档解析工具
    └── .../
```

### 2.2 项目内部结构（模板）

每个在 `Workspace/` 下创建的项目遵循统一的目录结构：

```
project-name/                    # 项目根目录（独立 Git 仓库）
├── CLAUDE.md                    # 项目级 Claude Code 配置（自包含）
├── pyproject.toml               # uv 项目配置（依赖、构建、工具）
├── uv.lock                      # uv 锁文件
├── .python-version              # Python 版本固定
├── .gitignore
│
├── .tasks/                      # 任务队列（规划 Agent 写入，编码 Agent 读取）
│   ├── backlog/                 # 待分配的任务
│   ├── queue-agent-<N>/         # 各编码 Agent 的工作队列
│   ├── in-progress/             # 正在执行的任务
│   ├── done/                    # 已完成待审查
│   └── merged/                  # 已合并归档
│
├── specs/                       # 项目规格说明
│   ├── functional-spec.md       # 功能规格
│   ├── api-spec.md              # API 规格（如适用）
│   └── changelog.md             # 变更记录
│
├── src/                         # 源代码
│   └── project_name/            # Python 包（使用 src layout）
│       ├── __init__.py
│       └── ...
│
├── tests/                       # 测试套件
│   ├── conftest.py              # pytest 公共 fixtures
│   ├── unit/                    # 单元测试
│   ├── integration/             # 集成测试
│   └── e2e/                     # 端到端测试（如适用）
│
├── docs/                        # 项目文档
│
└── reports/                     # 生成的报告（测试覆盖率等）
    └── .gitkeep
```

### 2.3 多 Agent 协作架构

#### 2.3.1 角色定义

系统中存在两类 Agent，各有明确的职责边界：

**规划 Agent（Planning Agent）**

规划 Agent 运行在 Harness 根目录，持有全局视角。它是唯一与人类直接交互的 Agent。

职责：
- 与人类讨论软件整体构想、功能设计、技术决策
- 根据 Tiered Workflow（见 §2.3.2）判断需求范围，决定是否需要 PRD
- 对 Feature/Epic 范围的设计发起 Critic Review（见 §2.3.3）
- 使用 `write-a-prd` 技能（Epic 范围）进行深度访谈并生成 PRD
- 使用 `prd-to-plan` 技能将 PRD 转化为规格 Phase
- 编写和维护功能规格（specs/functional-spec.md），每个 Phase 包含 `来源`、`状态` 字段和编号验收标准（AC-1.1, AC-1.2）
- 将规格分解为独立的、可并行的编码任务
- 决定需要多少个编码 Agent、如何分配任务
- 将任务写入任务队列（.tasks/ 目录）
- 监控编码 Agent 的进度和产出
- 协调合并顺序，处理潜在的分支冲突
- 进行最终的代码审查和合并决策
- 合并后标记 Spec Phase `状态` 为 `✅ 已完成`
- 在 5+ 对话确认 Phase 完成后，自动触发 `prd-sync` 技能回填 PRD

**编码 Agent（Coding Agent）**

编码 Agent 通过 git worktree 在独立的工作目录中运行，只能看到项目的 CLAUDE.md 和自己队列中的任务。

职责：
- 从任务队列中领取任务
- 创建功能分支，执行 TDD 开发（写测试 → 写代码）
- 运行质量检查（pytest + ruff + mypy）
- 在任务文件中更新状态和产出
- 完成后将分支标记为待合并

编码 Agent **不负责**：规格讨论、架构决策、跨任务协调、合并到 main。

#### 2.3.2 Tiered Workflow（分级工作流）

并非所有需求都需要完整的 PRD 仪式。规划 Agent 先评估需求范围，选择对应的流程：

| 级别 | 范围 | 流程 | PRD? |
|------|------|------|------|
| **Patch** | 1 个任务，单一变更 | 对话确认 → Task → Dispatch | 否 |
| **Feature** | 2-5 个任务，1 个 Phase | 对话确认 → Spec 追加 Phase → Tasks → Dispatch | 否 |
| **Epic** | >5 个任务，多 Phase | PRD 访谈 → Spec → Tasks → Dispatch | 是 |

- Patch 和 Feature 级别：Spec Phase 的 `来源` 字段记录 `对话确认 (日期)`
- 当 5+ 对话确认来源的 Phase 完成合并后，自动触发 `prd-sync` 技能回填 PRD

#### 2.3.3 Critic Review Loop（对抗性审查）

对 **Feature** 和 **Epic** 级别的设计，规划 Agent 必须在提交给人类审核前，通过 Critic Agent 进行对抗性审查。

**触发条件**：
- 规划 Agent 产出初始设计/架构方案后
- 人类反馈后修改设计后
- 最终确定 Spec Phase 和验收标准前

**协议流程**：
```
人类输入
    │
    ▼
规划 Agent 构思 → 初始方案
    │
    ▼
Critic Agent 审查 → 结构化批评（CRITICAL / HIGH / MEDIUM / LOW）
    │
    ▼
规划 Agent 修订 → 改进方案
    │
    ▼
人类审核 → 反馈 / 批准
    │
    ├── 反馈 → 规划 Agent 修订 → Critic 再审 → ...
    └── 批准 → 进入 Spec / Tasks
```

**跳过条件**（以下情况可不执行 Critic Review）：
- Patch 级别（范围极小，单一变更）
- 纯实现细节，无设计决策
- 紧急修复（速度优先于审查深度）

#### 2.3.4 协作流程

```
人类 ←→ 规划 Agent                    编码 Agent 1    编码 Agent 2    编码 Agent N
    │                                    │               │               │
    │  讨论需求/架构/决策                   │               │               │
    │         │                           │               │               │
    │         ▼                           │               │               │
    │  更新 specs/functional-spec.md      │               │               │
    │         │                           │               │               │
    │         ▼                           │               │               │
    │  分解任务 → 写入 .tasks/             │               │               │
    │  决定 Agent 数量和分配               │               │               │
    │         │                           │               │               │
    │         ├──── task-001.md ─────────→ 领取            │               │
    │         ├──── task-002.md ──────────────────────→ 领取              │
    │         ├──── task-003.md ───────────────────────────────────→ 领取  │
    │         │                           │               │               │
    │  继续讨论下一批需求                   │  开发中         │  开发中         │  开发中
    │         │                           │               │               │
    │         │                    标记完成 ←┘               │               │
    │         │                           │               │               │
    │         ▼                           │        标记完成 ←┘               │
    │  审查 + 合并 Agent 1 的分支           │               │               │
    │  审查 + 合并 Agent 2 的分支           │               │        标记完成 ←┘
    │  审查 + 合并 Agent 3 的分支           │               │               │
    │         │                           │               │               │
    │         ▼                           │               │               │
    │  分配下一批任务...                    │               │               │
```

**关键设计点**：人类与规划 Agent 的讨论和编码 Agent 的开发**同时进行**。人类不需要等待编码完成就可以继续讨论下一批功能。

#### 2.3.5 任务队列机制

任务通过项目目录下的 `.tasks/` 目录管理，每个任务是一个 Markdown 文件：

```
.tasks/
├── backlog/              # 待分配的任务
│   ├── task-004.md
│   └── task-005.md
├── queue-agent-1/        # Agent 1 的工作队列
│   └── task-001.md
├── queue-agent-2/        # Agent 2 的工作队列
│   └── task-002.md
├── in-progress/          # 正在执行的任务（Agent 领取后移入）
│   └── task-003.md
├── done/                 # 已完成待审查的任务
│   └── task-000.md
└── merged/               # 已合并的任务（归档）
```

**任务文件格式**：

```markdown
# Task: <task-id>

## Summary
<一句话描述这个任务要做什么>

## Branch
<branch-name, e.g., feat/add-csv-export>

## Spec Reference
<对应 specs/functional-spec.md 中的章节编号和验收标准>

## Scope
- Files to create/modify: <列表>
- Dependencies on other tasks: <无 / task-xxx 必须先完成>
- Estimated complexity: small / medium / large

## Acceptance Criteria
- [ ] <从规格中提取的具体验收条件>
- [ ] <...>

## Status
- [ ] Branch created
- [ ] Tests written
- [ ] Implementation complete
- [ ] All quality checks pass (pytest, ruff, mypy)
- [ ] Ready for review

## Notes
<规划 Agent 给编码 Agent 的额外说明、技术提示、注意事项>
```

#### 2.3.6 任务分解原则

规划 Agent 在分解任务时应遵循以下原则：

1. **独立性**：每个任务应尽量独立，减少跨任务依赖。如有依赖，在任务文件中明确标注。
2. **原子性**：每个任务应是一个完整的功能单元，可以独立测试和交付。
3. **无文件冲突**：尽量避免两个并行任务修改同一个文件。如果不可避免，明确哪个任务先合并。
4. **规模适当**：每个任务的规模应在一个编码 Agent 的单次会话内可完成。过大的任务应进一步拆分。

#### 2.3.7 合并协调

规划 Agent 负责合并顺序的决策：

1. 没有依赖关系的任务：先完成的先合并。
2. 有依赖关系的任务：按依赖顺序合并。
3. 有文件冲突风险的任务：规划 Agent 指定合并顺序，后合并的 Agent 需要 rebase。
4. 合并前：规划 Agent 运行完整的质量检查，确认无回归。

#### 2.3.8 编码 Agent 数量决策

规划 Agent 根据以下因素决定编码 Agent 的数量：

- **任务数量**：可并行的独立任务越多，Agent 越多
- **任务依赖关系**：依赖链越长，并行度越低
- **文件冲突风险**：修改同一文件的任务越多，并行度越低
- **项目规模**：小项目 1-2 个 Agent，中等项目 2-3 个，大项目可更多
- **实际约束**：受限于可用的 Claude Code 实例数量

---

## 3. 核心功能

### 3.1 项目初始化

**功能描述**：通过标准化流程创建新的 Python 项目，自动完成环境搭建。

**初始化流程**：

1. 在 `Workspace/` 下创建项目目录结构（基于所选模板）
2. 使用 `uv init` 初始化 Python 项目
3. 生成 `pyproject.toml`，配置：
   - 项目元数据（名称、版本、描述）
   - Python 版本要求（默认 ≥ 3.11）
   - 开发依赖（pytest、ruff、mypy 等）
   - 工具配置（ruff 规则、pytest 选项等）
4. 使用 `uv sync` 创建虚拟环境并安装依赖
5. 初始化 Git 仓库（`git init`）
6. 生成项目级 `CLAUDE.md`
7. 从模板生成初始的功能规格说明文档

**支持的项目类型/模板**：

| 模板 | 说明 | 额外依赖示例 |
|------|------|-------------|
| `default` | 通用 Python 库/工具 | — |
| `cli-app` | 命令行应用 | click/typer, rich |
| `web-api` | Web API 服务 | fastapi, uvicorn, httpx |
| `data-pipeline` | 数据处理管道 | pandas, polars |
| `web-app` | 复合 Web 应用（Python 后端 + React 前端） | fastapi, React 18, TypeScript, Vite |

### 3.2 规范驱动开发（SDD）工作流

**功能描述**：以功能规格为中心，驱动整个开发生命周期。

**工作流阶段**：

```
规范编写 → 规范评审 → 设计 → 实现 → 测试 → 验证 → 迭代
```

**各阶段详述**：

#### 3.2.1 规范编写

- 提供结构化的规范模板（功能规格、API 规格等）
- 规范应包含：背景与目标、功能需求、非功能需求、接口定义、验收标准
- 使用 Claude Code 辅助撰写和完善规范

#### 3.2.2 规范评审

- 通过 Claude Code 的 Skill 执行规范自检：
  - 完整性检查：是否覆盖了所有必要内容
  - 一致性检查：术语、接口是否前后一致
  - 可测试性检查：验收标准是否可量化/可验证
  - 歧义检查：是否存在模糊不清的描述

#### 3.2.3 设计与实现

- 基于规范，使用 Claude Code 进行架构设计
- 采用 TDD 或 BDD 方式实现功能：
  1. 先根据验收标准编写测试用例
  2. 实现代码使测试通过
  3. 重构优化

#### 3.2.4 测试与验证

- 运行完整测试套件（unit → integration → e2e）
- 生成测试覆盖率报告
- 对照规范中的验收标准进行逐项验证
- 记录验证结果

### 3.3 跨项目工具链

以下工具和集成可供所有 Workspace 下的项目使用：

#### 3.3.1 包与环境管理 — uv

- 每个项目独立的 `pyproject.toml` 和 `.venv`
- 使用 `uv` 进行依赖解析和安装（替代 pip/poetry）
- 支持 Python 版本管理（`uv python`）

#### 3.3.2 代码质量 — ruff + mypy

- **ruff**：统一的 linting 和格式化工具
  - Harness 提供推荐的 ruff 规则集（写入项目模板的 `pyproject.toml`）
  - 项目可根据需要自定义
- **mypy / pyright**：静态类型检查
  - 推荐使用 strict 模式
  - 集成到 CI 和代码审查流程中

#### 3.3.3 测试框架 — pytest

- pytest 作为统一的测试框架
- 推荐插件集：
  - `pytest-cov`：覆盖率统计
  - `pytest-asyncio`：异步测试支持
  - `pytest-xdist`：并行测试执行
- 测试组织约定：
  - `tests/unit/`：纯逻辑测试，无外部依赖
  - `tests/integration/`：涉及外部服务/数据库的测试
  - `tests/e2e/`：端到端测试

#### 3.3.4 MCP 服务器集成

MCP 服务器通过 Claude Code / VS Code 环境直接配置使用，不再在 Harness 内单独维护配置目录。

常用 MCP 服务器：

| MCP 服务器 | 用途 | 场景 |
|-----------|------|------|
| Pylance | Python 语言智能（补全、类型、重构） | 所有 Python 项目 |
| Context7 | 第三方库文档查询 | 依赖文档查阅 |
| Playwright | 浏览器自动化 | Web 应用 E2E 测试、爬虫 |

#### 3.3.5 Agent Skills

Harness 提供以下可复用的 Skills（12 个）：

**PRD 与规划链路：**

| Skill 名称 | 功能 | 触发场景 |
|------------|------|---------|
| `write-a-prd` | PRD 深度访谈与编写 | Epic 范围需求分析时 |
| `prd-to-plan` | PRD → 规格 Phase 转换 | PRD 完成后转化为可执行规格 |
| `prd-sync` | PRD 自动回填合并 | 5+ 对话确认 Phase 完成后 |

**规格与设计：**

| Skill 名称 | 功能 | 触发场景 |
|------------|------|---------|
| `spec-driven-dev` | 规范编写与评审辅助 | 编写或审查规格时 |
| `critic-review` | Critic Agent 对抗性审查 | Feature/Epic 设计审查时 |
| `design-an-interface` | 接口设计（"Design It Twice"） | 关键接口设计时 |

**质量与审查：**

| Skill 名称 | 功能 | 触发场景 |
|------------|------|---------|
| `test-runner` | 质量门禁（pytest + ruff + mypy） | 实现完成后、合并前 |
| `code-review` | 代码审查 | 分支合并前 |

**架构与前端：**

| Skill 名称 | 功能 | 触发场景 |
|------------|------|---------|
| `improve-codebase-architecture` | 架构改进（模块深化） | 代码库结构需要优化时 |
| `frontend-design` | 前端 UI 设计质量 | Web 应用前端开发时 |

**项目与自动化：**

| Skill 名称 | 功能 | 触发场景 |
|------------|------|---------|
| `project-init` | 初始化新项目 | 用户要求创建新项目时 |
| `auto-improve` | 自主改进循环（自动 PR） | 规划 Agent 发起持续改进时 |

### 3.4 项目级 CLAUDE.md 管理

每个项目的 `CLAUDE.md` 是 Claude Code 与项目交互的核心配置文件。

**关键设计原则：自包含（Self-Contained）**

项目可能通过 `git worktree` 被检出到独立目录，交给其他 coding agent 开发。在这种场景下，agent 只能看到项目自身的 `CLAUDE.md`，无法访问 Harness 的 `CLAUDE.md`。因此，项目级 `CLAUDE.md` 必须包含该 agent 正确工作所需的**全部信息**，不能依赖 Harness 层的任何上下文。

**必须包含的内容**：

| 章节 | 内容 | 原因 |
|------|------|------|
| 项目概述 | 项目描述、目标 | agent 需要理解上下文 |
| 技术栈 | Python 版本、uv、主要依赖 | agent 需要知道用什么工具 |
| 项目结构 | 完整的目录说明（src layout、tests 分层） | agent 需要知道代码放哪 |
| 开发命令 | uv sync/pytest/ruff/mypy 的完整命令 | agent 需要知道怎么跑 |
| 任务工作流 | 如何从 .tasks/ 领取和完成任务 | agent 需要遵循任务流程 |
| 开发流程 | SDD 工作流（规范→测试→实现→验证） | agent 需要遵循正确流程 |
| Git 分支管理 | 分支命名、工作流、提交格式 | agent 需要遵循 git 规范 |
| 合并标准 | 测试/lint/format/mypy 全部通过的清单 | agent 需要知道交付标准 |
| 代码规范 | 语言约定、命名风格、类型注解要求 | agent 需要写符合规范的代码 |
| 双语 README | README.md (English) + README.zh-CN.md (Chinese) 互链 | 每次变更需同步更新 |
| 当前状态 | 开发阶段和进展 | agent 需要了解项目进度 |

详细模板见 `Templates/default/CLAUDE.md.template`。

---

## 4. Harness 级 CLAUDE.md 设计

Harness 根目录的 `CLAUDE.md` 定义了双角色 Agent 架构和完整的开发治理流程：

- 根据运行位置自动判断角色（Harness 根目录 = 规划 Agent，项目目录 = 编码 Agent）
- **Tiered Workflow**：Patch / Feature / Epic 三级范围判定，决定流程重量级
- **Critic Review Loop**：Feature 和 Epic 设计的对抗性审查协议
- 规划 Agent 9 项职责：Discuss & PRD → Specify → Decompose → Allocate → Dispatch → Monitor → Merge → PRD Sync → Iterate
- 编码 Agent 指令：在项目自己的 CLAUDE.md 中，与 Harness 无关
- **Hard Rules**：8 条不可违反的约束（含双语 README 规则）

详见根目录 `CLAUDE.md`。

---

## 5. 工作流详解

### 5.1 创建新项目的完整流程

```
用户请求 "创建一个新的 CLI 项目 my-tool"
         │
         ▼
    ┌─────────────┐
    │ 选择项目模板  │  ← cli-app
    └──────┬──────┘
           │
           ▼
    ┌─────────────────┐
    │ 创建目录结构     │  ← Workspace/my-tool/...
    └──────┬──────────┘
           │
           ▼
    ┌─────────────────┐
    │ uv init + 配置   │  ← pyproject.toml, .python-version
    └──────┬──────────┘
           │
           ▼
    ┌─────────────────┐
    │ 安装基础依赖     │  ← uv sync (dev deps)
    └──────┬──────────┘
           │
           ▼
    ┌─────────────────┐
    │ git init         │  ← 初始化独立仓库
    └──────┬──────────┘
           │
           ▼
    ┌─────────────────┐
    │ 生成 CLAUDE.md   │  ← 项目级配置
    └──────┬──────────┘
           │
           ▼
    ┌──────────────────┐
    │ 生成规格说明模板  │  ← specs/functional-spec.md
    └──────┬───────────┘
           │
           ▼
    ┌──────────────────┐
    │ 引导用户编写规范  │  ← 进入 SDD 工作流
    └─────────────────┘
```

### 5.2 Git 分支管理流程

所有项目严格遵循**分支开发模型**，禁止直接在 `main` 分支上进行开发。

#### 5.2.1 分支策略

```
main（主干）
  │
  ├── feat/add-user-auth        ← 功能分支
  ├── fix/login-timeout         ← 修复分支
  ├── refactor/db-layer         ← 重构分支
  └── docs/api-reference        ← 文档分支
```

**分支命名规范**：

| 前缀 | 用途 | 示例 |
|------|------|------|
| `feat/` | 新功能开发 | `feat/add-export-csv` |
| `fix/` | Bug 修复 | `fix/null-pointer-on-empty-list` |
| `refactor/` | 代码重构 | `refactor/extract-parser-module` |
| `docs/` | 文档更新 | `docs/update-api-spec` |
| `test/` | 测试补充 | `test/add-edge-case-coverage` |

#### 5.2.2 开发流程（分支生命周期）

```
1. 从 main 创建功能分支
   git checkout -b feat/my-feature main
       │
       ▼
2. 在功能分支上进行开发迭代
   （编写规格 → 编写测试 → 实现代码 → 提交）
       │
       ▼
3. 持续提交，保持提交粒度合理
   每个提交应是一个逻辑完整的变更单元
       │
       ▼
4. 开发完成后，执行合并前检查
   ├── 全量测试通过（unit + integration + e2e）
   ├── 代码质量检查通过（ruff lint + format）
   ├── 类型检查通过（mypy strict）
   ├── 测试覆盖率达标
   └── 代码审查通过
       │
       ▼
5. 合并到 main
   git checkout main
   git merge feat/my-feature
       │
       ▼
6. 删除已合并的功能分支
   git branch -d feat/my-feature
```

#### 5.2.3 合并标准（Merge Criteria）

功能分支必须满足以下全部条件才可合并回 `main`：

- **测试全部通过**：`uv run pytest` 零失败
- **无 lint 错误**：`uv run ruff check .` 零警告
- **格式规范**：`uv run ruff format --check .` 通过
- **类型检查通过**：`uv run mypy src/` 零错误
- **规格一致性**：实现与功能规格中的验收标准一一对应
- **提交历史清晰**：每个提交信息清晰描述变更内容

#### 5.2.4 提交信息规范

```
<type>: <简要描述>

<详细说明（可选）>
```

**type 取值**：`feat`、`fix`、`refactor`、`test`、`docs`、`chore`

示例：
```
feat: add CSV export for user reports

Implement export functionality supporting UTF-8 encoding
and configurable delimiters. Closes #12.
```

### 5.3 典型开发迭代流程

```
1. 从 main 创建功能分支
       │
       ▼
2. 编写/更新功能规格
       │
       ▼
3. 评审规格（自检 + 人工确认）
       │
       ▼
4. 编写测试用例（基于验收标准）
       │
       ▼
5. 实现功能代码，逐步提交到功能分支
       │
       ▼
6. 运行测试 + lint + 类型检查
       │
       ├── 通过 → 7. 代码审查
       │               │
       │               ▼
       │          8. 满足合并标准 → 合并到 main，删除分支
       │
       └── 失败 → 修复后回到 6
```

---

## 6. 配置管理

### 6.1 pyproject.toml 推荐配置

以下是项目模板中 `pyproject.toml` 的推荐内容：

```toml
[project]
name = "project-name"
version = "0.1.0"
description = ""
readme = "README.md"
requires-python = ">=3.11"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.backends"

[tool.hatch.build.targets.wheel]
packages = ["src/project_name"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
addopts = [
    "-v",
    "--tb=short",
    "--strict-markers",
]

[tool.ruff]
target-version = "py311"
line-length = 100
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # flake8-simplify
    "TCH",  # flake8-type-checking
]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=4.0",
    "ruff>=0.4",
    "mypy>=1.9",
]
```

### 6.2 .gitignore 模板

#### Harness 级 .gitignore

```gitignore
# 排除所有项目工作区
Workspace/

# Python 通用
__pycache__/
*.py[cod]
.venv/

# IDE
.vscode/
.idea/
```

#### 项目级 .gitignore

```gitignore
# Virtual environment
.venv/

# Python
__pycache__/
*.py[cod]
*.egg-info/
dist/
build/

# Testing
.coverage
htmlcov/
reports/*.html
reports/*.xml

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

---

## 7. 非功能需求

### 7.1 环境要求

| 依赖 | 最低版本 | 说明 |
|------|---------|------|
| Python | 3.11+ | 通过 uv 管理 |
| uv | 最新稳定版 | 包和项目管理 |
| Git | 2.30+ | 版本控制 |
| Claude Code | 最新版 | AI 辅助开发 |

### 7.2 设计原则

- **可扩展性**：新的项目模板、Skills、MCP 集成应易于添加
- **低耦合**：各项目之间完全独立，Harness 与项目之间通过约定而非强依赖关联
- **渐进式采纳**：用户可以只使用部分功能（例如只用项目初始化，不用 SDD 流程）
- **透明性**：所有自动化操作都应清晰可见，不做"魔法"操作

### 7.3 语言约定

- 规格说明、文档：使用**中文**
- 代码、注释、commit message：使用**英文**
- 变量命名：遵循 Python PEP 8（snake_case）
- **README 双语规则**：每个项目必须同时维护 `README.md`（English）和 `README.zh-CN.md`（中文），互相交叉链接。任何修改必须在同一个 commit 中同步更新两个文件

---

## 8. 实施计划

### Phase 1：基础设施 ✅

- [x] 定义 Harness 功能规格
- [x] 创建 Harness 目录结构
- [x] 编写 Harness 级 CLAUDE.md
- [x] 创建 .gitignore
- [x] 初始化 Git 仓库

### Phase 2：项目模板与初始化 ✅

- [x] 创建 default 项目模板
- [x] 编写项目初始化 Skill（Skills/project-init/）
- [x] 创建规格说明模板（Prompts/spec-template.md）
- [x] 创建代码审查清单（Prompts/review-checklist.md）
- [x] 创建测试计划模板（Prompts/test-plan-template.md）

### Phase 3：工具链集成 ✅

- [x] 编写测试运行 Skill（Skills/test-runner/）
- [x] 编写代码审查 Skill（Skills/code-review/）
- [ ] 配置 MCP 服务器（Python LSP、Context7、Playwright）— 已通过 Claude Code 环境直接使用

### Phase 4：SDD 工作流 ✅

- [x] 编写 spec-driven-dev Skill（编写 + 评审合一）
- [x] 完善 TDD 工作流集成（融入 test-runner Skill）

### Phase 5：多 Agent 协作架构 ✅

- [x] 设计规划 Agent / 编码 Agent 角色分工
- [x] 设计任务队列机制（.tasks/ 目录结构和任务文件格式）
- [x] 定义任务分解原则和合并协调策略
- [x] 更新 CLAUDE.md 支持双角色 Agent 指令
- [x] 更新项目 CLAUDE.md 模板，编码 Agent 可独立工作
- [x] 创建任务模板文件（Prompts/task-template.md）

### Phase 6：模板扩展 ✅

- [x] 创建 cli-app 项目模板
- [x] 创建 web-api 项目模板
- [x] 创建 data-pipeline 项目模板
- [x] 创建 web-app 复合模板（Python 后端 + React 前端）
- [x] 编写快速开始指南（Docs/getting-started.md）

### Phase 7：PRD 全链路与高级 Skills ✅

- [x] 编写 write-a-prd Skill（PRD 深度访谈）
- [x] 编写 prd-to-plan Skill（PRD → 规格 Phase 转换）
- [x] 编写 prd-sync Skill（PRD 自动回填）
- [x] 编写 critic-review Skill（对抗性审查）
- [x] 编写 design-an-interface Skill（接口设计）
- [x] 编写 improve-codebase-architecture Skill（架构改进）
- [x] 编写 frontend-design Skill（前端 UI 设计）
- [x] 编写 auto-improve Skill（自主改进循环）

### Phase 8：治理流程升级 ✅

- [x] 在 CLAUDE.md 中实现 Tiered Workflow（Patch/Feature/Epic 三级）
- [x] 在 CLAUDE.md 中实现 Critic Review Loop
- [x] 在 CLAUDE.md 中增加 PRD Sync 自动触发机制
- [x] 在 CLAUDE.md 中增加 Hard Rule #8（双语 README）
- [x] 规格 Phase 增加 `来源`、`状态` 字段和编号 AC

### Phase 9：持续优化（进行中）

- [ ] 根据实际使用反馈优化工作流
- [ ] 补充 Harness 文档（development-workflow.md 等）
- [ ] 完善跨项目知识积累机制

---

## 附录 A：术语表

| 术语 | 定义 |
|------|------|
| Harness | 本项目——Python 项目开发工作台 |
| Workspace | 所有项目的容器目录 |
| SDD | 规范驱动开发（Spec-Driven Development） |
| Skill | Agent 的可复用技能插件 |
| MCP | 模型上下文协议（Model Context Protocol） |
| uv | Astral 出品的 Python 包管理工具 |
| Planning Agent | 规划 Agent——与人类交互、编写规格、分解任务、协调合并 |
| Coding Agent | 编码 Agent——从任务队列领取任务、独立开发、TDD |
| Critic Agent | 评审 Agent——对设计方案进行对抗性审查（由规划 Agent 调用的子 Agent） |
| Task Queue | 任务队列——.tasks/ 目录下的任务文件集合 |
| Worktree | Git worktree——允许同一仓库在多个目录中同时检出不同分支 |
| Tiered Workflow | 分级工作流——根据需求范围（Patch/Feature/Epic）选择不同的流程重量级 |
| PRD | 产品需求文档（Product Requirements Document） |
| Critic Review | 对抗性审查——独立 Agent 对设计方案的严格批评与挑战 |
| Phase | 规格中的开发阶段，包含来源、状态和编号验收标准 |
| AC | 验收标准（Acceptance Criteria），编号格式如 AC-1.1 |
