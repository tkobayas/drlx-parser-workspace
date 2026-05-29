# HANDOVER

## Session goals (completed)

**Implemented #56 — passive query invocation (`?/queryName(...)`).** One-line fix in `DrlxRuleAstRuntimeBuilder.buildQueryElement()`: hardcoded `openQuery=true` → `!patternIr.passive()`. Added compile-time guard for passive invocation of queries with agenda `do` blocks. Two tests.

## Current state

- **drlx-parser project repo** `main` at `d0f1f81`, clean, pushed.
- **javaparser-mvel** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **MVEL3** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Workspace** `main`, clean.

## Key decisions

- **Priority axis:** *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Test pattern for passive queries:** use `withSession` + entry points (not `DrlxRuleUnitInstance`), insert reactive side first to create a complete match, then verify passive-side insertion doesn't wake the rule.

## Open issues

- **ListDataStore ordering** — *Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate next action

Pick another feature from epic #77 or #78 to implement next. Remaining in #77: #57 (result binding), #60 (named access), #61 (@Rule/@DataSource annotations).
