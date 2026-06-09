# HANDOVER

## Session goals (completed)

**Closed #88, #89 — removed @AgendaGroup, @AutoFocus, @RuleFlowGroup.** RuleUnit's `ActivationsManagerImpl` hardcodes `SimpleAgendaGroupsManager` which only supports the MAIN group. These annotations were silently set on `RuleImpl` but never enforced. This is by design — RuleUnit replaces the AgendaGroup/RuleFlowGroup concepts. 1 commit, 9 files changed (-301 lines), all 371 tests pass, 0 disabled.

## Current state

- **drlx-parser project repo** `main` at `750505a`, clean, not pushed.
- **javaparser-mvel** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **MVEL3** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Workspace** `main`, uncommitted (this handover).

## Key decisions

- **RuleUnit doesn't support agenda groups** — `ActivationsManagerImpl` hardcodes `SimpleAgendaGroupsManager`; `StackedAgendaGroupsManager` requires `InternalWorkingMemory` (not `ReteEvaluator`); `createRuleAgendaItem()` always puts rules in MAIN. Removed all three annotations rather than fight the architecture.
- **`@LockOnActive` works without `@AgendaGroup`** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Priority axis:** *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Open issues

- **ListDataStore ordering** — *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Immediate next action

Pick the next #78 sub-issue (#65 test blocks, #40 groupBy) or next priority from the backlog.
