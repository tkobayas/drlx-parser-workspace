# HANDOVER

## Session goals (completed)

**#74 Window test flakiness — root-caused and fixed.** `ListDataStore` uses `IdentityHashMap` which doesn't preserve insertion order. When events were added before `subscribe()`, the `length[3]` window replay was non-deterministic — producing 1, 2, or 3 instead of consistent 2. Fix: moved all event insertions in `WindowTest` to after instance creation. Also closed #34.

## Current state

- **drlx-parser project repo** `main` at `ec444bd`, clean, pushed.
- **javaparser-mvel** `compact-with` merged (per user).
- **MVEL3** `compact-with` merged (per user).
- **Workspace** `main`, clean.

## Key decisions

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Open issues

- **ListDataStore ordering** — `IdentityHashMap` in drools `ListDataStore.subscribe()` replays events in arbitrary order. User will file upstream drools issue. Workaround: always insert events after instance creation.
- `DataStoreUpdateRewriter` uses plain `JavaParser.parseBlock()` which can't parse `p{...}` syntax. Compact-with + `DataStore.update()` in the same consequence block needs either the rewriter to use MVEL3 parser, or statement reordering at DRLX level.

## Immediate next action

Pick next from epic #26: **#43** (pluggable operators) or another open issue.

## References

| Topic | Path |
|---|---|
| Epic #26 | https://github.com/tkobayas/drlx-parser/issues/26 |
| Bug #74 | https://github.com/tkobayas/drlx-parser/issues/74 |
