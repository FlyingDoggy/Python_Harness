---
name: critic-review
description: Launch an independent Critic Agent to adversarially review a design proposal. Use for Feature and Epic tiers before presenting designs to the human or before finalizing spec Phases. Skip for Patch tier and trivial implementation details.
---

# Critic Review

Launch an independent subagent to perform adversarial expert review of a design proposal. The Critic Agent thinks independently and challenges every aspect of the design.

## When to Invoke

- After the Planning Agent produces an initial design/architecture proposal
- After incorporating human feedback into a revised design
- Before finalizing spec Phases and acceptance criteria
- When the human explicitly requests expert review

## When to Skip

- Patch tier (trivially scoped, single change)
- Pure implementation details with no design decisions
- Urgent hotfixes where speed outweighs review depth

## Process

### 1. Prepare the proposal document

Before launching the Critic Agent, the Planning Agent must assemble:

- **The proposal**: complete design/architecture description with enough detail to evaluate
- **System context**: relevant codebase state, existing architecture, constraints, team capacity
- **Decision log**: what alternatives were considered and why they were rejected

### 2. Launch Critic Agent

Use `runSubagent` with the following prompt structure:

```
You are an independent expert reviewer. Your mandate is to be ruthlessly critical.
Find every flaw, every unstated assumption, every "sounds good but fails in practice" weakness.

## Proposal Under Review
{proposal — full text of the design}

## System Context
{codebase state, existing architecture, constraints, team size, timeline}

## Review Dimensions
- Logical soundness and unstated assumptions
- Missing considerations (operational, organizational, technical)
- Risk assessment (severity-rated: CRITICAL / HIGH / MEDIUM / LOW)
- Alternative approaches not considered
- Feasibility given team and timeline constraints

## Output Format
Structured critique organized by dimension. Each finding has:
- Title
- Severity: CRITICAL / HIGH / MEDIUM / LOW
- Description: what's wrong and why
- Evidence: specific references to the proposal
- Recommendation: what to do instead

End with:
- Severity summary table (count by level)
- Overall verdict: build as designed / simplify / abandon
- The single most important thing the proposal got wrong or missed
```

### 3. Process the critique

After receiving the Critic Agent's output:

1. **Triage findings by severity**: CRITICAL items must be addressed. HIGH items should be addressed. MEDIUM items are judgment calls. LOW items are noted.
2. **Accept or reject each finding**: Explicitly state your response to each CRITICAL and HIGH finding with reasoning.
3. **Revise the proposal**: Incorporate accepted findings into an improved design.
4. **Present to human**: Show the revised design along with a summary of what the Critic found and how you responded.

### 4. Iterate if needed

If the human provides additional feedback:
1. Revise the proposal based on human feedback
2. Re-launch Critic Agent with the updated proposal
3. Process new critique
4. Present revised version

Continue until the human approves.

## Quality Criteria for Critic Output

A good critic review should:
- [ ] Identify at least one non-obvious risk the Planning Agent missed
- [ ] Challenge at least one assumption with a concrete counter-scenario
- [ ] Propose at least one simpler alternative
- [ ] Provide severity ratings that are justified, not arbitrary
- [ ] Consider operational/organizational factors, not just technical ones

A poor critic review that should trigger re-invocation:
- Rubber-stamps the proposal with "looks good"
- Only finds trivial/cosmetic issues
- Doesn't address feasibility or alternatives
- Lacks concrete evidence or scenarios

## Flow Diagram

```
Human input
    │
    ▼
Planning Agent → initial proposal
    │
    ▼
Critic Agent → structured critique
    │
    ▼
Planning Agent → accept/reject findings → revised proposal
    │
    ▼
Human → feedback / approval
    │
    ├── feedback → revise → Critic reviews → ...
    └── approval → proceed to Spec / Tasks
```
