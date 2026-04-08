---
name: todo-step
description: Work on a step from a todo*.md file. Use when the user says "let's work on step N", "implement step N", "implement next step from todo" or similar. Covers the full workflow: review step description for consistency, implement it, run tests, commit, then mark the step as done.
---

Work on a step from `todo*.md`. If there's a few markdown files named todo - ask which one to pick. Follow this workflow:

## 1. Review the step

Read `todo*.md` and locate the requested step. If user did not mention specific step - find first on not marked as done (no ✅ in title). Before implementing:
- Check the step description for internal consistency (e.g. unreachable cases, contradictions, dependencies on other steps).
- Point out any issues and discuss with the user before proceeding.
- Confirm the user wants to proceed as written (or with agreed adjustments).

## 2. Explore before coding

Read the relevant existing code before writing anything. Understand:
- Which files will need to change.
- How similar features were implemented (use them as the pattern to follow).
- Whether backend changes are needed or if this is frontend-only.

If you discover inconsistencies or potential issues discuss with the user before proceeding.

## 3. Implement

Make the changes. Follow the project's existing conventions exactly — same patterns, same file structure, same naming style.


## 4. Run tests

Always run tests after implementation. Work is not done until all tests pass:
- Backend: `go test ./...`
- Frontend: `cd web && npm test -- --run`
- Frontend build/typecheck: `cd web && npm run build`

Fix any failures before proceeding.

## 5. Commit

Commit the changes with a short, descriptive message (no "feat:"/"fix:" prefixes — just plain English). Always include the co-author trailer:
```
Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

## 6. Mark the step as done

Add ✅ to the step heading in todo file and commit that change separately with a message like "Mark Step N as completed in todo*.md".
