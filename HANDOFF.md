# HANDOVER

## Session goals (completed)

**Fixed #87 — @NoLoop/@LockOnActive runtime enforcement.** `DataStore.update()` now propagates `InternalMatch` through `DataStoreSupport.update()`, giving `PropagationContext` the terminal node origin that `PhreakRuleTerminalNode` needs. Also sets `rule.setEager(true)` for both annotations. 2 commits, all 374 tests pass, 5 `@Disabled` (agenda group bugs #88/#89).

## Current state

- **drlx-parser project repo** `main` at `f4721e2`, clean, not pushed.
- **javaparser-mvel** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **MVEL3** — *Unchanged — `git show HEAD~1:HANDOFF.md`*
- **Workspace** `main`, uncommitted (this handover).

## Key decisions

- **`__match__` in MVEL context** — `InternalMatch` injected as `__match__` var in the MVEL evaluation map; `DataStoreUpdateRewriter` generates `DataStoreSupport.update(store, fact, __match__, "storeName")` calls.
- **`@LockOnActive` works without `@AgendaGroup`** — `blockedByLockOnActive()` recency check works against the MAIN group. Drools upstream doesn't test this combination but the behavior is correct.
- **Priority axis:** *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Open issues

- **#88** — `@AgendaGroup`/`@AutoFocus` not enforced. `SimpleAgendaGroupsManager` doesn't support named groups.
- **#89** — `@RuleFlowGroup` not enforced. Depends on #88.
- **ListDataStore ordering** — *Unchanged — `git show HEAD~5:HANDOFF.md`*

## Immediate next action

Fix #88 (agenda group support) or pick the next #78 sub-issue (#65 test blocks, #40 groupBy).
