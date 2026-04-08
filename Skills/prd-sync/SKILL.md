---
name: prd-sync
description: Automatically consolidate completed Spec Phases into PRD documents for compliance and traceability. Use when 5+ conversation-sourced Phases have been completed, or when the user requests a PRD update.
---

# PRD Sync Skill

Automatically generate or update PRD documents from completed Spec Phases that originated from conversation confirmations (not from a formal PRD). This ensures compliance traceability without blocking development.

## When to Trigger

The Planning Agent should run this skill when ANY of the following is true:

1. **Auto-trigger**: 5 or more Spec Phases with `来源: 对话确认` have been marked `✅ 已完成` since the last PRD sync.
2. **Manual trigger**: The user asks to "update PRD" or "sync PRD".
3. **Milestone trigger**: A major release or review is approaching.

## Process

### 1. Scan the Spec

Read `specs/functional-spec.md`. Collect all Phases where:
- `状态` = `✅ 已完成`
- `来源` starts with `对话确认` (not already linked to a PRD)

If no qualifying Phases exist, report "No PRD sync needed" and stop.

### 2. Cluster by theme

Group the collected Phases by functional area or module. Use §3 架构决策 and the Phase descriptions to identify natural clusters. Each cluster becomes a section in the PRD.

### 3. Determine PRD target

Check `prd/` directory in the project:
- If no PRD files exist, create `prd/PRD-001-consolidated.md`.
- If PRD files exist, find the latest consolidated PRD and determine if the new Phases fit as an addendum or warrant a new PRD file.
- Use sequential numbering: `PRD-001`, `PRD-002`, etc.

### 4. Generate PRD content

For each cluster, write using the standard PRD template structure:

```markdown
## {{Cluster Title}}

### Problem Statement
（从 Phase 描述反向推导：这些 Phase 解决了什么问题）

### Solution
（从 Phase 描述和验收标准组合推导）

### User Stories
（从 Spec §2 和 Phase 覆盖用户故事提取，保留原编号）

### Implementation Decisions
（从 Spec §3 架构决策中提取与这些 Phase 相关的条目）

### Spec Phases Covered
- Phase N: {{标题}} (AC-N.1 ~ AC-N.M)
- Phase M: {{标题}} (AC-M.1 ~ AC-M.K)
```

### 5. Back-fill Spec 来源 field

For each synced Phase, update `来源` from:
```
对话确认 (2026-04-04)
```
to:
```
对话确认 → PRD-003 §2.1
```

This creates the bi-directional traceability link.

### 6. Update Spec §1 概述 if needed

If the accumulated changes have shifted the project direction, update §1.1 背景与动机 and §1.2 目标 to reflect the current state. Add a `<!-- 最后更新: YYYY-MM-DD by PRD Sync -->` comment.

### 7. Report

Output a summary:
```
PRD Sync 完成:
- 新增/更新 PRD: prd/PRD-003-consolidated.md
- 覆盖 Phase: 4, 5, 6, 8, 11
- Spec 来源已回填: 5 个 Phase
- §1 概述: 未更新 / 已更新
```

## PRD Directory Structure

```
prd/
  PRD-001-{{feature}}.md      ← Epic 正式访谈产出
  PRD-002-{{feature}}.md      ← Epic 正式访谈产出
  PRD-003-consolidated.md     ← Agent 自动汇总（对话确认 Phase）
  PRD-004-consolidated.md     ← Agent 自动汇总（下一批）
```

## Rules

1. **Never block development** — PRD sync is a post-hoc activity, never a prerequisite.
2. **Never fabricate** — Only describe what was actually implemented (from Spec + done Phases).
3. **Preserve original PRDs** — Never modify PRD files created from formal interviews. Consolidated PRDs are separate files.
4. **Keep it concise** — Each cluster section should be 20-40 lines. This is a compliance document, not a novel.
