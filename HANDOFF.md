# HANDOVER

## Session goals (incomplete)

**Picked up #65 — test block `t[...]` boolean expression.** Started brainstorming but hit a spec/grammar conflict that needs a design decision before proceeding.

## Current state

- **drlx-parser project repo** `main` at `03f69ea`, clean, pushed.
- **javaparser-mvel** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **MVEL3** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Workspace** `main`, clean.

## Key decisions

- **Spec line 348 confirmed as typo** — `timestamp = new Date()` inside `[]` should be a boolean test (e.g. `timestamp != null`), not an assignment. Test blocks contain only boolean expressions.
- **`==` → `.equals()` mapping** — will be implemented as part of #65 (spec line 153: within `[]` blocks only).
- **Grammar conflict with list/map access** — both test blocks and list/map access use `expression '[' ... ']'`. Multi-expression (`t[a, b]`) is unambiguous (comma disambiguates), but single-expression (`t[x == y]`) clashes with array/map access syntax. This needs a design decision before implementation can proceed.

## Open issues

- **ListDataStore ordering** — *Unchanged — `git show HEAD~2:HANDOFF.md`*

## Immediate next action

Resume #65 brainstorming — resolve the `[]` grammar ambiguity (single-expression test block vs array access), then complete spec and plan.
