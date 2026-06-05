# HANDOVER

## Session goals (completed)

**Implemented #61 — @DataSource annotation.** New `DataSource.java` annotation, `DATASOURCE` IR kind + proto, visitor resolution/validation, runtime builder entry point override with dual-registration for self-reference resolution. 8 tests. All green, module installed, pushed.

**Closed epic #77** (Query enhancements) — all 6 sub-issues complete.

**Created #84** — `@Rule` annotation on Unit DataSource fields, split from #61, placed in epic #80 (de-prioritized, requires drools-side changes).

## Current state

- **drlx-parser project repo** `main` at `d7569af`, clean, pushed.
- **javaparser-mvel** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **MVEL3** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Workspace** `main`, uncommitted (this handover + blog).

## Key decisions

- **`@DataSource` queries only** — annotation rejected on non-query rules with a clear error.
- **Dual-registration in queryRegistry** — query registered under both override name and default lowercased name. The default name is needed for self-reference resolution in the query body (`buildSelfReferencePattern` path).
- **Priority axis:** *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Open issues

- **ListDataStore ordering** — *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Immediate next action

Pick the next epic to work on. Candidates: #78 (Rule metadata & syntax sugar), #79 (Conditional branching & named windows). #80 and #81 are de-prioritized.
