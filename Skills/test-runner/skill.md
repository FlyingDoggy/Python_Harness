# Test Runner Skill

Use this skill when the user wants to run tests, check code quality, or verify merge readiness for a project.

## Commands

### Run Tests
```bash
cd Workspace/<project-name>
uv run pytest --tb=short -v
```

### Run Tests with Coverage
```bash
cd Workspace/<project-name>
uv run pytest --cov=src --cov-report=term-missing --cov-report=html:reports/htmlcov
```

### Full Quality Check (Merge Readiness)

Run all checks in sequence and report results:

```bash
cd Workspace/<project-name>

# 1. Tests
uv run pytest --tb=short -v

# 2. Lint
uv run ruff check .

# 3. Format check
uv run ruff format --check .

# 4. Type check
uv run mypy src/
```

Report a summary table:

| Check | Status | Details |
|-------|--------|---------|
| Tests | PASS/FAIL | X passed, Y failed |
| Lint | PASS/FAIL | N issues |
| Format | PASS/FAIL | N files |
| Type check | PASS/FAIL | N errors |

If all pass, indicate the branch is ready to merge. If any fail, list what needs fixing.
