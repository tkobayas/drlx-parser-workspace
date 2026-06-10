# HANDOVER

## Session goals (completed)

**Closed #86 — implemented `@DateEffective` and `@DateExpires` rule annotations.** ISO-8601 date-only format (`yyyy-MM-dd`), parse-time validation via `LocalDate.parse()`, `Calendar` conversion for `RuleImpl` setters. Deterministic runtime tests using `PseudoClockScheduler.setStartupTime()`. 2 commits, 8 files changed (+501 lines), all 399 tests pass, 0 disabled.

## Current state

- **drlx-parser project repo** `main` at `c25c27e`, clean, pushed.
- **javaparser-mvel** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **MVEL3** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Workspace** `main`, uncommitted (this handover).

## Key decisions

- **ISO-8601 over legacy `DateUtils` format** — `yyyy-MM-dd` instead of `d-MMM-yyyy`. Locale-independent, validated by `LocalDate.parse()` natively.
- **Date-only, no time component** — activation at start of day (midnight) in system default time zone.
- **No mutual exclusion** — unlike `@Timer`/`@Duration`, both can coexist to form a date window.
- **Proto file needed for new Kind entries** — plan missed `DrlxRuleAstParseResult.java` switches and proto enum; 5 files changed, not 3. Compiler caught it.
- **Priority axis:** *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Open issues

- **ListDataStore ordering** — *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Immediate next action

Pick the next #78 sub-issue (#65 test blocks, #40 groupBy) or next priority from the backlog.
