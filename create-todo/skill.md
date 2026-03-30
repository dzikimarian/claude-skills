---
name: create-todo
description: Split a spec .md file into a todo_*.md implementation plan. Use when the user says "split X into steps", "create todo from spec", "plan implementation for X.md" or similar. Takes a spec file path as argument.
---

Split a spec `.md` file into a `todo_*.md` step-by-step implementation plan. Argument: path to the spec file (e.g. `product-mgr.md`).

## 1. Read the spec thoroughly

Read the entire spec file. Understand the full scope: new domains, extensions to existing domains, UI, API, DB schema, cross-domain interactions, temporal behavior, and post-MVP items (exclude those from the plan).

## 2. Explore the existing codebase

Before planning steps, understand what already exists:
- Domain structure (`internal/*/`)
- Frontend structure (`web/src/`)
- How similar features were built (patterns to follow)
- What can be extended vs. what needs to be created from scratch

This informs step sizing — extending an existing domain is smaller than building one from scratch.

## 3. Design vertical slices

Each step should be a **vertical slice** — delivering testable end-to-end functionality where possible. Group related backend and frontend work together so each step ships something a user can see or interact with.

**Ordering principles:**
- Small domain extensions to existing code first (quick wins, unblock later steps)
- UI scaffolding (navigation, empty shells) early so subsequent steps have a home
- New domains + their UI together when feasible; if the domain is too large, split by capability (e.g. CRUD first, then event-sourcing, then the assembly endpoint)
- Interaction layers (drag-and-drop, command menu) after rendering is solid
- Event history / audit features last

**Sizing guidelines:**
- **Default to small.** Each step should deliver exactly one observable thing — something the user can see or interact with immediately after the commit. If you can't describe what's visible after the step in one sentence, it's too large.
- A step can span BE + FE if they're tightly coupled (e.g. "add field + display it") and together produce something visible in one sitting.
- Pure-backend steps are OK when the frontend genuinely can't show anything yet — but they still need a clear observable deliverable (tests pass, API responds correctly).
- When a step bundles multiple independent concerns (e.g. "navigation + new domain + panel + commands"), split it. Each concern that a user could notice separately should be its own step.
- If you're uncertain whether a step is too large, split it. Prefer 4 small steps over 1 large one. Sub-steps (3a, 3b, 3c…) are fine and expected for larger features.

**BE/FE boundary — the most common source of "too big":**

A step that has a complex or non-obvious backend/frontend boundary is almost always too big. The symptom is implementors being uncertain which layer owns what. When this applies, **split by layer** and label each sub-step explicitly as "Backend only" or "Frontend only".

When splitting by layer, the backend sub-step must describe the API contract precisely:
- What raw data the backend returns (field names, types, shape)
- What the backend does NOT compute (e.g. grouping, UI-level aggregations)

The frontend sub-step then describes what it derives client-side from that data.

**"Purely frontend computation" pattern:** Some features look like they need backend logic but are fully derivable client-side from data the backend already returns. Identify these explicitly in the step and state it plainly (e.g. "Dropzones are a frontend-only concept — never referenced in the backend. The backend returns `po_ids` per product and `po_person_ids` per team; all grouping and validity logic lives in the frontend."). This prevents over-engineering the backend and avoids confusion during implementation.

## 4. Write the todo file

Create `todo_<name>.md` (derive name from spec filename). Write for a **senior engineer** — convey intent and constraints, not implementation recipes.

**Format per step:**

```markdown
### Step N: Short descriptive title

One or two sentences of context — what this step delivers and why it matters.

Concise description of the work. Focus on:
- **What** to build (entities, endpoints, UI elements, behaviors)
- **Key constraints** from the spec (limits, validation rules, temporal behavior, graceful degradation)
- **Important design decisions** that aren't obvious from the code (e.g. "event-sourced, not CRUD", "no cross-domain FK", "auto-assigns to team if needed")

Don't list every file to touch, every function signature, or every test case. The implementer will read the code and figure that out. Do include spec-specific details they'd otherwise have to hunt for (default values, sort orders, color codes, validation rules, drop-target semantics).

---
```

**Document header:** Include a reference line pointing to the source spec and any mockups.

**What NOT to include:**
- File paths or function names (the implementer will find them)
- Test case lists (covered by project rules in CLAUDE.md)
- "Backend:" / "Frontend:" / "Tests:" section headers for every step — only split when it aids clarity
- Implementation instructions that any senior engineer would derive from reading existing code

**What TO include:**
- Spec-specific constraints that are easy to miss (max 3 POs, unordered PO set identity, ghost degradation)
- Non-obvious behaviors (auto-assign PO to team, cross-team move creates undo toast, dropzone merging)
- Default/seed data values
- Sort orders, visual specs (colors, border styles) that the implementer needs
- API endpoint paths and semantics from the spec
- DB schema decisions (event-sourced vs. CRUD, temporal tracking pattern)

## 5. Review with user

Present the plan as a summary table (step number, title, size estimate, what it delivers) and ask if the slicing feels right before finalizing.
