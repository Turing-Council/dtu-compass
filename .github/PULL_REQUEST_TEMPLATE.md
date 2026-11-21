## Summary
<!-- What does this PR change and why? -->

Closes #

## Requirement / Regression IDs
<!-- e.g., OPS-02, PLAN-12, §5 #9 (or N/A) -->

## How to test
<!-- Commands or steps a reviewer can run. Screenshots for UI changes. -->

## Risk / rollback
<!-- One line: what could break, and how to undo it (or "low, revert PR"). -->

## AI assistance
- [ ] None
- [ ] Assisted (autocomplete, chat suggestions)
- [ ] Agent-generated (most of the diff was written by an agent following `AGENTS.md`)

## Checklist
- [ ] I have read and understand every line of this diff
- [ ] PR title follows Conventional Commits (`feat(api): ...`, `fix(pipeline): ...`, `chore: ...`)
- [ ] Tests added/updated (including `domain/` coverage ≥ 90% if touched)
- [ ] Docs / ADR updated if behaviour or a decision changed
- [ ] If DB schema changed: Alembic migration is expand/contract (`OPS-05`)
- [ ] If API/tools changed: OpenAPI schema & typed web client regenerated
