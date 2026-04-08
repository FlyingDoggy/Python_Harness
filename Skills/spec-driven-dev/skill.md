# Spec-Driven Development Skill

Use this skill when the user is writing or reviewing a functional specification for a project.

## Spec Writing Mode

When the user wants to write or update a spec:

1. Check if `specs/functional-spec.md` exists in the project. If not, copy from `Prompts/spec-template.md`.
2. Determine the tier (Patch / Feature / Epic) based on scope.
3. **Epic**: A PRD should already exist. Use the PRD's Problem Statement, User Stories, Implementation Decisions, Testing Decisions, and Out of Scope to populate §1, §2, §5. Use the `prd-to-plan` skill to derive §3 架构决策 and §4 Phases.
4. **Feature**: Confirm requirements in conversation. Append a new Phase to §4 with `来源: 对话确认 (日期)`. Update §3 架构决策 if new decisions are needed.
5. **Patch**: Append a minimal Phase with 1-2 ACs, or go straight to Task if trivially scoped. Set `来源: 对话确认 (日期)`.
6. Walk through each new Phase with the user, ensuring acceptance criteria are specific, measurable, and testable. Use numbered AC format: AC-N.1, AC-N.2, etc.
7. Ensure non-goals are explicitly stated to prevent scope creep.
8. Keep §1 概述 accurate — update it if accumulated changes have shifted the project direction.

## Phase Lifecycle

Each Phase has a `状态` field that tracks progress:

```
🔲 待开发 → 🚧 开发中 → ✅ 已完成
```

When all tasks for a Phase are merged, the Planning Agent marks it `✅ 已完成`.

## PRD Traceability

Each Phase has a `来源` field:
- Epic Phases: `PRD-001` (reference to formal PRD)
- Feature/Patch Phases: `对话确认 (YYYY-MM-DD)` → later back-filled to `对话确认 → PRD-003 §2.1` by the `prd-sync` skill

## Three-Layer Workflow

```
PRD (what & why)  →  Spec (how, phases, AC)  →  Tasks (file-level work items)
```

The spec is the single source of truth that merges PRD content with architectural planning. There is no separate `plans/` document. PRD is auto-synced for compliance after every 5 conversation-sourced Phases are completed.

## Spec Review Mode

When a spec is ready for review, check:

### Completeness
- [ ] Background and motivation are clear
- [ ] Goals and non-goals are explicit
- [ ] All core use cases are described
- [ ] Each feature has acceptance criteria
- [ ] Non-functional requirements are addressed
- [ ] Implementation plan has phases

### Consistency
- [ ] Terminology is used consistently throughout
- [ ] Interface definitions match feature descriptions
- [ ] Acceptance criteria are achievable within stated constraints

### Testability
- [ ] Every acceptance criterion can be verified with a test
- [ ] Success/failure conditions are unambiguous
- [ ] Edge cases are addressed

### Ambiguity
- [ ] No vague terms like "should handle gracefully" without definition
- [ ] Data formats and types are specified
- [ ] Error behavior is explicitly described

Report findings as a structured checklist with pass/fail and recommendations.
