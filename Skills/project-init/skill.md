# Project Init Skill

Use this skill when the user asks to create a new Python project. This skill initializes a new project under the Workspace/ directory with the standard structure, uv environment, git repo, task queue, and self-contained CLAUDE.md.

## Required Information

Before starting, confirm with the user:
1. **Project name** (kebab-case, e.g., `my-tool`)
2. **Brief description** (one sentence)
3. **Template** (default, cli-app, web-api, web-app, data-pipeline — default if not specified)

## Initialization Steps

Execute these steps in order:

### Step 1: Create directory structure

The `<package_name>` is the project name with hyphens replaced by underscores.

For **default, cli-app, web-api, data-pipeline** templates, create the project at `Workspace/<project-name>/` with:
```
.tasks/backlog/
.tasks/in-progress/
.tasks/done/
.tasks/merged/
specs/
src/<package_name>/
  __init__.py
tests/
  conftest.py
  unit/
  integration/
  e2e/
docs/
reports/
```

For the **web-app** template (Python backend + React frontend), create:
```
.tasks/backlog/
.tasks/in-progress/
.tasks/done/
.tasks/merged/
specs/
backend/src/<package_name>/
  __init__.py
backend/tests/
  conftest.py
  unit/
  integration/
  e2e/
frontend/src/
frontend/public/
docs/
reports/
```

### Step 2: Generate pyproject.toml

If using `default` template, use `Templates/default/pyproject.toml.template`. Otherwise use the template-specific version: `Templates/<template>/pyproject.toml.template`. For `web-app`, use `Templates/web-app/backend/pyproject.toml.template` → `backend/pyproject.toml`.

Replace in all templates:
- `{{PROJECT_NAME}}` → project name
- `{{PROJECT_DESCRIPTION}}` → description
- `{{PACKAGE_NAME}}` → package name (underscored)

### Step 2b: Copy template-specific source files

Each template includes boilerplate source and test files. Copy and replace placeholders:

- **cli-app**: `Templates/cli-app/src/cli.py.template` → `src/<package_name>/cli.py`, `Templates/cli-app/tests/test_cli.py.template` → `tests/unit/test_cli.py`
- **web-api**: `Templates/web-api/src/app.py.template` → `src/<package_name>/app.py`, `Templates/web-api/tests/test_app.py.template` → `tests/unit/test_app.py`
- **web-app**: Composite project. Copy backend files from `Templates/web-app/backend/` into `backend/`, and frontend files from `Templates/web-app/frontend/` into `frontend/`:
  - `backend/src/app.py.template` → `backend/src/<package_name>/app.py`
  - `backend/src/__init__.py.template` → `backend/src/<package_name>/__init__.py`
  - `backend/tests/conftest.py.template` → `backend/tests/conftest.py`
  - `backend/tests/test_app.py.template` → `backend/tests/unit/test_app.py`
  - `frontend/package.json.template` → `frontend/package.json`
  - `frontend/vite.config.ts.template` → `frontend/vite.config.ts`
  - `frontend/tsconfig.json.template` → `frontend/tsconfig.json`
  - `frontend/index.html.template` → `frontend/index.html`
  - `frontend/src/main.tsx.template` → `frontend/src/main.tsx`
  - `frontend/src/App.tsx.template` → `frontend/src/App.tsx`
  - `frontend/src/index.css.template` → `frontend/src/index.css`
- **data-pipeline**: `Templates/data-pipeline/src/pipeline.py.template` → `src/<package_name>/pipeline.py`, `Templates/data-pipeline/tests/test_pipeline.py.template` → `tests/unit/test_pipeline.py`
- **default**: no extra source files

All other files (.gitignore, .python-version) always come from `Templates/default/`. For CLAUDE.md, `web-app` uses `Templates/web-app/CLAUDE.md.template`; all others use `Templates/default/CLAUDE.md.template`.

### Step 3: Generate .gitignore

Copy from `Templates/default/gitignore.template`. For `web-app`, also add `node_modules/` and `dist/` entries.

### Step 4: Generate CLAUDE.md

For `web-app`, use `Templates/web-app/CLAUDE.md.template`. For all others, use `Templates/default/CLAUDE.md.template`. Replace placeholders.

### Step 5: Generate functional spec template

Copy `Prompts/spec-template.md` to `specs/functional-spec.md`, replacing `{{项目名称}}` with the project name and `{{日期}}` with today's date.

### Step 6: Create .python-version

Write `3.11` (or the user-specified version) to `.python-version`. For `web-app`, place it in `backend/.python-version`.

### Step 7: Initialize environments

For standard templates:
```bash
cd Workspace/<project-name>
uv sync
```

For `web-app`:
```bash
cd Workspace/<project-name>/backend
uv sync
cd ../frontend
npm install
```

### Step 8: Initialize Git repo

```bash
cd Workspace/<project-name>
git init
git add -A
git commit -m "chore: initialize project <project-name>"
```

### Step 9: Report to user

Confirm the project has been created. Remind the user of the next steps:
- Write the functional spec in `specs/functional-spec.md`
- After spec is complete, decompose into tasks in `.tasks/backlog/`
- Coding agents will be dispatched via git worktree to work on tasks in parallel
