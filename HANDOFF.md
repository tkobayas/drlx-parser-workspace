# HANDOVER

## Session goals (done)

1. ✅ #19 closed — `DrlxLambdaConstraint.getListenedPropertyMask` overrides via new `Evaluator.getReadProperties()`. Two-repo: MVEL3 (`mvel3-read-props`, merged upstream as [mvel/mvel#423](https://github.com/mvel/mvel/issues/423)), DRLX ([PR #21](https://github.com/tkobayas/drlx-parser/pull/21), merged).
2. ✅ Spec + plan committed; both have a "Deviations during implementation" addendum capturing what shipped vs. what was designed.
3. ✅ Blog entry `tk01` for 2026-04-27 (`constraint-mask-deviations`).
4. ✅ CLAUDE.md updated with MVEL3 add-dir note + SNAPSHOT install rule.

## Current state

### Test suite
- **111 tests passing, 0 failures.** Was 108 → +3 condition+watch-list cases in `PropertyReactiveWatchListTest`.
- MVEL3 full suite: 720 passing.

### GitHub issues
- `#13` CLOSED (last session). `#19` CLOSED (this session). `mvel/mvel#423` MERGED upstream.
- Next candidates (open): `#12` if/else branching, `#16` RuleIR proto experiment.

### Migration policy
*Unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## Immediate next action

Decide next ticket with user. Two-way choice:
- **#12 if/else branching** — grammar work, larger scope.
- **#16 RuleIR proto experiment** — infrastructure refactor.

## Gotchas (new this session)

- **`MVELCompiler.registerAndRename` uses `findFirst(MethodDeclaration.class)`** (`MVELCompiler.java:238`). If you add methods to a generated MVEL3 evaluator class, emit them *after* the eval method (after `MVELToJavaRewriter.rewriteChildren`). Otherwise `LambdaUtils.createLambdaKeyFromMethodDeclaration` operates on the wrong method and fails with `StringIndexOutOfBoundsException`.
- **MVEL3's symbol resolver rejects bare getter calls in user expressions.** `getBasePay()` doesn't compile even with declarations populated — drools-model uses `_this.getX()` form because that's produced by *its own rewriter*, not user-written. Don't assume drools-model expression shapes work as MVEL3 input.
- **DRLX puts every JavaBeans property in MVEL3's `available` set.** `DrlxLambdaCompiler.extractDeclarations:53-69` registers each property as a `Declaration`. Any analyser logic that branches on `available.contains(name)` will treat all property reads as known declarations — design accordingly.
- **JavaParser's `FieldAccessExpr.name` is a `SimpleName`, not a `NameExpr`.** Visitors that visit `NameExpr` correctly skip the field-name part of `p.name`. Cross-pattern join (`name == p.name`) handling falls out for free.

Older gotchas: *unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## References

| Topic | Path |
|-------|------|
| #19 spec | `specs/2026-04-27-constraint-mask-design.md` |
| #19 plan | `plans/2026-04-27-constraint-mask-implementation.md` |
| Latest blog entry | `blog/2026-04-27-tk01-constraint-mask-deviations.md` |
| MVEL3 source | `/home/tkobayas/usr/work/mvel3-development/mvel` (add-dir if MVEL3 changes needed) |
| DRLX PR for #19 | https://github.com/tkobayas/drlx-parser/pull/21 |
| MVEL3 PR | mvel/mvel#423 (merged) |

## Key commands

*Unchanged — retrieve with:* `git show HEAD~1:HANDOFF.md`
