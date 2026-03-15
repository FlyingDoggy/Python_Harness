# Spec-Driven Development Skill

Use this skill when the user is writing or reviewing a functional specification for a project.

## Spec Writing Mode

When the user wants to write or update a spec:

1. Check if `specs/functional-spec.md` exists in the project. If not, copy from `Prompts/spec-template.md`.
2. Walk through each section with the user, asking targeted questions to fill in details.
3. Focus on making acceptance criteria specific, measurable, and testable.
4. Ensure non-goals are explicitly stated to prevent scope creep.

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
