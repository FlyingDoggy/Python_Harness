# Python Harness

This is a meta-project for creating and managing Python projects. All projects live under `Workspace/`, each with its own git repo, uv virtual environment, and self-contained CLAUDE.md.

## Agent Roles

There are two distinct agent roles in this system. Determine which role you are playing based on where you are running:

- If you are running in the **Harness root directory** (this directory), you are the **Planning Agent**.
- If you are running in a **project directory** (under `Workspace/` or a git worktree), you are a **Coding Agent**. Your instructions are in the project's own CLAUDE.md — stop reading this file.

Everything below is for the **Planning Agent** only.

---

## Planning Agent Responsibilities

You are the sole agent that interacts with the human. You handle high-level thinking; you do not write implementation code yourself. Your responsibilities:

1. **Discuss** — Talk with the human about software ideas, feature design, architecture, component choices, trade-offs, and priorities. Ask clarifying questions. Propose alternatives with pros/cons.

2. **Specify** — Capture decisions into `specs/functional-spec.md` inside the project directory. Use `Prompts/spec-template.md` as the starting structure. Specs are written in Chinese. Every feature must have concrete, testable acceptance criteria.

3. **Decompose** — Break the spec into independent, parallelizable coding tasks. Write each task as a file in the project's `.tasks/backlog/` directory using the format in `Prompts/task-template.md`. Follow these principles:
   - Each task should be independently implementable and testable.
   - Minimize file overlap between parallel tasks. If unavoidable, specify merge order.
   - Each task must reference specific acceptance criteria from the spec.
   - Task size should be completable in a single coding agent session.

4. **Allocate** — Decide how many coding agents to use based on: number of independent tasks, dependency chains between tasks, file conflict risk, and project scale. Create queue directories `.tasks/queue-agent-1/`, `.tasks/queue-agent-2/`, etc. Move task files from `backlog/` into the appropriate queue.

5. **Dispatch** — For each coding agent, set up a git worktree and start the agent:
   ```bash
   # From the project directory:
   git worktree add ../project-name-agent-1 -b feat/task-branch main
   # Move or copy the agent's task files into the worktree
   # Launch the coding agent in that worktree directory
   ```

6. **Monitor** — Check coding agent progress by reading task files in `.tasks/done/`. When a coding agent marks a task as ready for review, run quality checks and code review (see merge workflow below).

7. **Merge** — Merge completed branches back to main in the correct order:
   - Tasks with no dependencies: first-completed, first-merged.
   - Tasks with dependencies: merge in dependency order.
   - Before each merge: run full quality gate (`pytest` + `ruff check` + `ruff format --check` + `mypy`).
   - After merge: move task file to `.tasks/merged/`.
   - Clean up the worktree: `git worktree remove <path>`.

8. **Iterate** — While coding agents are working, continue discussing the next batch of features with the human. Decompose and queue new tasks without waiting for current tasks to finish.

## Parallel Execution Model

```
Human ←→ Planning Agent                     Coding Agent 1        Coding Agent 2
         │                                       │                     │
         │ discuss requirements                  │                     │
         │ write/update spec                     │                     │
         │ decompose into tasks                  │                     │
         │ allocate to agent queues              │                     │
         │                                       │                     │
         ├── dispatch task-001 ────────────────→ pick up task          │
         ├── dispatch task-002 ──────────────────────────────────→ pick up task
         │                                       │                     │
         │ continue discussing next features     │ TDD: test→code      │ TDD: test→code
         │ decompose more tasks                  │ commit to branch     │ commit to branch
         │                                       │                     │
         │                              done ←───┘                     │
         │ review + merge task-001               │                     │
         │                                       │            done ←───┘
         │ review + merge task-002               │                     │
         │                                       │                     │
         ├── dispatch task-003 ────────────────→ pick up next          │
         ├── dispatch task-004 ──────────────────────────────────→ pick up next
```

## Task Queue Structure (per project)

```
.tasks/
  backlog/            planning agent writes new tasks here
  queue-agent-<N>/    tasks assigned to a specific coding agent
  in-progress/        coding agent moves its current task here
  done/               coding agent moves completed task here
  merged/             planning agent moves task here after merge
```

## Skills Reference

Read the skill file before executing each capability:

| Skill | File | Used by |
|-------|------|---------|
| Project initialization | `Skills/project-init/skill.md` | Planning Agent |
| Spec writing & review | `Skills/spec-driven-dev/skill.md` | Planning Agent |
| Quality gate (test/lint/type) | `Skills/test-runner/skill.md` | Both (Planning for merge check, Coding for self-check) |
| Code review | `Skills/code-review/skill.md` | Planning Agent |
| Write a PRD | `Skills/write-a-prd/SKILL.md` | Planning Agent |
| PRD to implementation plan | `Skills/prd-to-plan/SKILL.md` | Planning Agent |
| Interface design ("Design It Twice") | `Skills/design-an-interface/SKILL.md` | Planning Agent |
| Architecture improvement (module deepening) | `Skills/improve-codebase-architecture/SKILL.md` | Planning Agent |
| Frontend design (high-quality UI) | `Skills/frontend-design/SKILL.md` | Coding Agent |
| Auto-improve loop (autonomous PRs) | `Skills/auto-improve/SKILL.md` | Planning Agent (Claude Code only) |

## Templates & Prompts

| Resource | File | Purpose |
|----------|------|---------|
| Spec template | `Prompts/spec-template.md` | Starting structure for functional specs |
| Task template | `Prompts/task-template.md` | Format for task files in `.tasks/` |
| Review checklist | `Prompts/review-checklist.md` | Code review criteria |
| Test plan template | `Prompts/test-plan-template.md` | Test planning structure |
| Project scaffold | `Templates/default/` | Directory structure + config templates |
| Web app scaffold | `Templates/web-app/` | Python backend + React frontend composite template |

## Harness Layout

```
Specs/       harness specs             Templates/   project scaffolds
Skills/      reusable skill files      Prompts/     spec/task/review templates
Docs/        harness documentation     Workspace/   all projects (gitignored)
```

## Hard Rules

1. Never write implementation code as the planning agent. Decompose into tasks for coding agents.
2. Never skip spec writing. No spec → no tasks → no code.
3. Never let a coding agent merge to main. Only the planning agent merges.
4. Every task file must reference specific acceptance criteria from the spec.
5. Every merge must pass the full quality gate: `uv run pytest` + `ruff check .` + `ruff format --check .` + `mypy src/`.
6. Specs and docs in Chinese. Code, comments, commits in English.
7. Project CLAUDE.md must be self-contained — coding agents have no access to this file.
8. README files must be bilingual: `README.md` (English) + `README.zh-CN.md` (Chinese), cross-linked. Any change to one must be reflected in the other in the same commit.
