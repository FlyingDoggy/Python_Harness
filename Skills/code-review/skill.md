# Code Review Skill

Use this skill when the user wants a code review before merging a feature branch.

## Process

### Step 1: Understand the context

1. Read the project's `specs/functional-spec.md` to understand what the feature should do
2. Run `git log main..HEAD --oneline` to see all commits on the feature branch
3. Run `git diff main...HEAD` to see all changes

### Step 2: Run automated checks

Execute the full quality check (see test-runner skill):
- Tests, lint, format, type check

### Step 3: Review code changes

Use the review checklist from `Prompts/review-checklist.md` to evaluate:

- **Functional correctness**: Does the code match the spec's acceptance criteria?
- **Code quality**: Readability, naming, complexity, duplication
- **Type safety**: Complete type annotations, mypy compliance
- **Test coverage**: Are all acceptance criteria tested?
- **Security**: Injection risks, sensitive data handling
- **Performance**: Obvious bottlenecks, unnecessary I/O

### Step 4: Report findings

Provide a structured report:

```
## Code Review Report

**Branch**: feat/xxx
**Commits**: N commits
**Files changed**: N files

### Automated Checks
(table from test-runner)

### Review Findings

#### Must Fix (Blocking)
- ...

#### Should Fix (Non-blocking)
- ...

#### Suggestions (Optional)
- ...

### Verdict: APPROVE / REQUEST CHANGES
```
