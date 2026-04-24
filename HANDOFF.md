# HANDOVER

## Session goals (done)

1. ✅ #13 closed — property-reactive watch list (second `[]` block on `oopathRoot`). Grammar, IR, proto, visitor, runtime validation (unknown / duplicate / duplicate-wildcard). Eight commits pushed (`b295dfd..f1b1bc7`).
2. ✅ Scope discovery mid-session: `DrlxLambdaConstraint` doesn't override `getListenedPropertyMask`, so alpha mask defaults to AllSetButLastBitMask and defeats the watch list when alpha constraints are present. Verified via Codex. Scoped down, filed **#19** for the constraint-mask override (with drools-model-compiler `LambdaConstraint.java:166-188` as reference).
3. ✅ Spec + plan committed to workspace; blog entry `tk03` for 2026-04-24.

## Current state

### Test suite
- **108 tests passing, 0 failures.** Was 96 → +3 parse-only (`DrlxParserTest`) + 7 behavioural (`PropertyReactiveWatchListTest`) + 2 proto round-trip (`DrlxRuleAstParseResultTest`).
- No-persist mode: 75 running + 33 skipped (baseline +3, inheriting `@DisabledIfSystemProperty` from `DrlxBuilderTestSupport`).

### GitHub issues
- `#13` CLOSED with summary comment (limitation noted; links to #19).
- `#19` OPEN — `DrlxLambdaConstraint.getListenedPropertyMask`. Full context + file:line refs + drools-model reference embedded in the issue body.
- Next candidates (open): `#12` if/else branching, `#19` constraint mask (finishes #13), `#16` RuleIR proto experiment.

### Migration policy
*Unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## Immediate next action

Decide next ticket with user. The three-way choice is:
- **#19 constraint mask** — finishes what #13 started. Reference impl in `drools-model-compiler/.../LambdaConstraint.java:166-188`. Tests already written (the behavioural cases currently restricted to empty-conditions form would extend once the mask lands).
- **#12 if/else branching** — grammar work, larger scope.
- **#16 RuleIR proto experiment** — infrastructure refactor.

## Gotchas (new this session)

- **`EntryPoint.update(fh, obj)` (2-arg) hard-codes modification mask to `AllSetBitMask.get()`** (`NamedEntryPoint.java:287-293`). For external tests that care about property reactivity, always use the 3-arg form: `ep.update(fh, obj, "propName1", "propName2")`. In rule consequences, `modify($x) { setX(...) }` compiles to the 3-arg form.
- **TypeDeclaration must be registered on the KieBase for `isPropertyReactive(ruleBase, objectType)` to return true.** Pattern for programmatic KieBase construction: register `TypeDeclaration.createTypeDeclarationForBean(cls, PropertySpecificOption.ALWAYS)` on a `KnowledgePackageImpl` keyed by the class's own package (not the rule's package). Mirror `drools-model-compiler/.../KiePackagesBuilder.java:1317-1335`. Without this the entire property-reactive machinery stays dormant.
- **`AllSetButLastBitMask` is "all bits except bit 0" (the traitable bit), not literally AllSet.** But for non-trait classes, it's effectively "watch every real property" — which is why the default `Constraint.getListenedPropertyMask` behaves as a catch-all.

Older gotchas: *unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## References

| Topic | Path |
|-------|------|
| #13 spec | `specs/2026-04-24-property-reactive-watch-list-design.md` |
| #13 plan | `plans/2026-04-24-property-reactive-watch-list-implementation.md` |
| Latest blog entry | `blog/` → `2026-04-24-tk03-watch-list-half-shipped.md` |
| #19 (follow-up) | GitHub issue — constraint-mask override reference implementation |
| ReactiveEmployee fixture | `drlx-parser-core/src/test/java/org/drools/drlx/domain/ReactiveEmployee.java` |
| PropertyReactiveWatchListTest | `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java` |

## Key commands

*Unchanged — retrieve with:* `git show HEAD~1:HANDOFF.md`
