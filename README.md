# Python Harness

[中文版](README.zh-CN.md)

A meta-project for building Python projects with AI-assisted, spec-driven development. It provides project scaffolding, development conventions, reusable skills, and a multi-agent parallel workflow powered by Claude Code.

## How It Works

```
You ←→ Planning Agent                  Coding Agent 1      Coding Agent 2
       │                                    │                    │
       │ interview → write PRD (Epic)        │                    │
       │  or confirm in chat (Patch/Feature) │                    │
       │ write/update spec (phases + AC)     │                    │
       │ decompose into tasks                │                    │
       │                                    │                    │
       ├── dispatch ──────────────────────→ TDD on branch       │
       ├── dispatch ─────────────────────────────────────────→ TDD on branch
       │                                    │                    │
       │ keep discussing next features      │ develop & test     │ develop & test
       │                                    │                    │
       │                           done ←───┘                    │
       │ review & merge                     │           done ←───┘
       │ review & merge                     │                    │
       │                                    │                    │
       │ (every 5 phases) auto PRD sync     │                    │
```

**Planning Agent** runs in this directory — talks to you, writes specs, breaks work into tasks, dispatches coding agents, reviews and merges.

**Coding Agents** run in git worktrees — pick up tasks, write tests first, implement, pass quality gates. They never merge to main.

You keep discussing next features while coding agents work in parallel.

## Quick Start

### Prerequisites

- Python 3.11+ (`uv python install 3.11`)
- [uv](https://docs.astral.sh/uv/getting-started/installation/)
- Git 2.30+
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

### Create a Project

Open Claude Code in this directory and say:

> "创建一个新的 CLI 项目 my-tool，用于批量处理 CSV 文件"

The planning agent will scaffold `Workspace/my-tool/` with everything ready to go.

### Available Templates

| Template | Stack | Use Case |
|----------|-------|----------|
| `default` | Pure Python | Libraries, utilities |
| `cli-app` | click + rich | Command-line tools |
| `web-api` | FastAPI + uvicorn | REST APIs |
| `web-app` | FastAPI + React/TypeScript/Vite/Tailwind | Fullstack web applications |
| `data-pipeline` | polars | Data processing |

## Development Workflow

The planning agent adapts the process based on scope:

| Tier | Scope | Flow | PRD? |
|------|-------|------|------|
| **Patch** | 1 task | Chat confirm → Task → Dispatch | No |
| **Feature** | 2-5 tasks | Chat confirm → Append Phase to Spec → Tasks | No |
| **Epic** | 5+ tasks | PRD Interview → Spec → Tasks | Yes |

1. **Discuss & PRD** — For Epics: deep interview to produce a PRD. For Patches/Features: confirm in conversation.
2. **Spec** — Create or update `specs/functional-spec.md` with phases, acceptance criteria, and traceability.
3. **Decompose** — Break each phase into independent tasks in `.tasks/backlog/`.
4. **Dispatch** — Create git worktrees and launch coding agents.
5. **Build** — Coding agents work in parallel: TDD, commit to feature branches.
6. **Merge** — Planning agent reviews, runs quality gate, merges to main.
7. **PRD Sync** — Every 5 completed conversation-sourced phases, auto-generate consolidated PRD for compliance.

All projects follow strict rules:
- No code without a spec
- No implementation without tests (TDD)
- No merge without passing: `pytest` + `ruff check` + `ruff format --check` + `mypy`
- No direct commits to `main`

## Repository Structure

```
├── CLAUDE.md          Agent instructions (planning agent reads this)
├── Specs/             Harness specifications
├── Skills/            Reusable Claude Code skills
│   ├── project-init/      Project scaffolding
│   ├── write-a-prd/       PRD interview & creation
│   ├── prd-to-plan/       PRD to spec phases
│   ├── prd-sync/          Auto-consolidate PRD from completed phases
│   ├── spec-driven-dev/   Spec writing & review
│   ├── design-an-interface/ Interface design ("Design It Twice")
│   ├── test-runner/       Quality gate checks
│   ├── code-review/       Code review process
│   ├── improve-codebase-architecture/ Module deepening
│   ├── frontend-design/   High-quality UI design
│   └── auto-improve/      Autonomous improvement loop
├── Templates/         Project scaffolds
│   ├── default/           Base template (shared by all)
│   ├── cli-app/           Click CLI boilerplate
│   ├── web-api/           FastAPI boilerplate
│   ├── web-app/           Fullstack: FastAPI backend + React frontend
│   └── data-pipeline/     Polars pipeline boilerplate
├── Prompts/           Reusable templates
│   ├── spec-template.md       Functional spec structure
│   ├── task-template.md       Task file format
│   ├── review-checklist.md    Code review criteria
│   └── test-plan-template.md  Test planning structure
├── Docs/              Documentation
└── Workspace/         All managed projects (gitignored)
```

## Project Structure (generated)

Each project under `Workspace/` gets:

```
my-project/
├── CLAUDE.md              Self-contained coding agent instructions
├── pyproject.toml         uv config (deps, tools, build)
├── .tasks/                Task queue for agent coordination
│   ├── backlog/           Unassigned tasks
│   ├── queue-agent-N/     Per-agent work queues
│   ├── in-progress/       Currently executing
│   ├── done/              Completed, awaiting review
│   └── merged/            Archived
├── specs/                 Functional spec (Chinese)
├── prd/                   PRD documents (auto-synced)
├── src/<package>/         Source code (src layout)
├── tests/                 unit / integration / e2e
└── reports/               Coverage, etc.
```

## Conventions

| What | Language |
|------|----------|
| Specs, docs | Chinese |
| Code, comments, commits | English |

Commit format: `<type>: <description>` — types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

Branch naming: `feat/`, `fix/`, `refactor/`, `docs/`, `test/` + description

## License

Private project.
