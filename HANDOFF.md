# HANDOVER

## Session goals (completed)

**Implemented #6 — 7 rule-level annotations.** `@NoLoop`, `@LockOnActive`, `@AutoFocus`, `@Disabled`, `@AgendaGroup`, `@ActivationGroup`, `@RuleFlowGroup`. ArgShape-based resolver refactor, proto enum, runtime application. 10 commits, 22 tests pass, 6 `@Disabled` (runtime bugs).

**Created 5 issues:** #85 (`@Duration`/`@Timer`), #86 (`@DateEffective`/`@DateExpires`) — deferred from #6. #87 (NoLoop not enforced), #88 (AgendaGroup not enforced), #89 (RuleFlowGroup not enforced) — runtime bugs discovered by DrlxRuleUnitInstance tests.

## Current state

- **drlx-parser project repo** `main` at `9b2deef`, clean, not pushed.
- **javaparser-mvel** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **MVEL3** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Workspace** `main`, uncommitted (this handover + blog).

## Key decisions

- **Marker-style booleans** — `@NoLoop` with no args, not `@NoLoop(true)`. Passing an argument is an error.
- **`@Disabled` over `@Enabled`** — marker `@Enabled` is redundant (rules enabled by default).
- **Empty-string rejection** — universal for all STRING-shape annotations, not just `@DataSource`.
- **Priority axis:** *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Open issues

- **#87** — `@NoLoop`/`@LockOnActive` not enforced. DataStore `update(DataHandle, T)` doesn't propagate `InternalMatch` to `PropagationContext`. Garden entry: `GE-20260609-dab2f5`.
- **#88** — `@AgendaGroup`/`@AutoFocus` not enforced. `SimpleAgendaGroupsManager` doesn't support groups.
- **#89** — `@RuleFlowGroup` not enforced. Depends on #88.
- **ListDataStore ordering** — *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Immediate next action

Fix runtime enforcement bugs (#87, #88) or pick the next #78 sub-issue (#65 test blocks, #40 groupBy).
