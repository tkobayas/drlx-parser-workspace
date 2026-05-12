# HANDOVER

## Session goals (done)

1. ✅ **#45 — `DataStore.update(T)` coercion.** JavaParser AST rewriter (`DataStoreUpdateRewriter`) wired into `DrlxRuleAstRuntimeBuilder` between consequence-text capture and MVEL3 compile. Cheap String guards make cost proportional to rules-with-DataStore-update, not total rule count. Project commits `b712f85..16bcacf`, pushed to `origin/main`.
2. ✅ **Off-plan addition: `DataStoreSupport`.** Static facade exposing `lookup(DataStore<?>, Object)`. Needed because MVEL3's symbol solver type-checks consequences at compile time via JavaParser symbol-solver-core; `lookup(Object)` lives on impl-package `InternalStoreCallback`, not on the public `DataStore<T>`. Spec had this flagged as runtime risk; it bites at compile.
3. ✅ **#45 closed** with summary referencing the new tests and the off-plan facade.

## Current state

### Test suite
- **DRLX: 170 passing, 0 failures.** Up from 162 baseline.
- `DataStoreCrudTest`: 5 (was 3) — `updateByObjectViaDataStore`, `updateOfMissingFactThrows` added.
- `DataStoreUpdateRewriterTest` (new): 11 unit tests, no MVEL3, no Drools.

### GitHub issues
- **Epic #26**: 13 open sub-issues, 5 closed (#27–#29 + #37 + now #45)
- High priority: #39 (accumulate), #40 (groupBy), #41 (queries)
- **#34** (compact `with`-block update) is now technically unblocked by #45, but it's a grammar change — much bigger than #45 was.

### Migration policy
*Unchanged — retrieve with* `git show 98b59ab:HANDOFF.md`

## Immediate next action

User-pick. From the open Epic #26: **#39 (accumulate)** or **#40 (groupBy)** or **#41 (queries)** — each is a meaningful chunk. **#34** (compact `with`-block update) is now the natural follow-up to #45 but it's a DRLX grammar + MVEL3 collaboration, larger scope than today.

## Gotchas (this session)

- **MVEL3 symbol solver rejects impl-only methods at compile time, not runtime.** Symptom: `Method 'lookup' cannot be resolved in context persons.lookup(p)` with stack frames in `JavaParserFacade.solveMethodAsUsage` and `MVELToJavaRewriter.maybeCoerceArguments`. Cause: MVEL3 transpiles via JavaParser symbol-solver-core, which sees only the static type. Fix: route impl-only methods through a static facade with a typed signature. Captured in garden as `GE-20260512-0cda17`.
- **DRLX `update(p)` infinite loop when consequence doesn't break the match.** Test hangs silently. Classic Drools 101: change a property the pattern depends on so the activation is removed post-update. The test fixture for `updateByObjectViaDataStore` uses `p.setAge(0)` against a pattern `age > 30`.
- **Don't trust spec runtime/compile-time framing without checking.** `update(T)` spec said the impl-only `lookup` was a runtime risk; it was actually a compile-time hard fail. Symbol-solver gates aren't intuitive.

For historical gotchas: *unchanged — retrieve with* `git show 98b59ab:HANDOFF.md`

## References

| Topic | Path |
|-------|------|
| Today's blog entry | `blog/2026-05-12-tk01-45-symbol-solvers-static-types.md` |
| #45 spec / plan | `specs/2026-05-12-drlx-datastore-update-coercion-design.md`, `plans/2026-05-12-drlx-datastore-update-coercion-implementation.md` |
| Garden gotcha | `~/.hortora/garden/jvm/GE-20260512-0cda17.md` |
| Rewriter | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreUpdateRewriter.java` |
| Static facade | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreSupport.java` |
| Previous handover | `git show 98b59ab:HANDOFF.md` |

## Key commands

*Unchanged — retrieve with:* `git show 98b59ab:HANDOFF.md`
