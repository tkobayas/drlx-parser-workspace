# HANDOVER

## Session goals (done)

1. ✅ **#23 closed** — DRLXXXX `test` construct shipped end-to-end (proto + IR + lambda compiler + runtime + grammar + visitor + tests). 9 commits on `origin/main` (`a1e3101..511696c`).
2. ✅ Spec + plan for **#12 restructured** — depends on #23 (now landed); new slimmer 10-task plan replaces the original 14-task one.
3. ✅ Blog `tk02` written: `blog/2026-04-27-tk02-test-before-if-else.md`.
4. ✅ Property-reactivity finding: DRLX defaults to `PropertySpecificOption.ALWAYS`; `EvalCondition.isPatternScopeDelimiter()` is moot. Spec for #12 updated.

## Current state

### Test suite
- **DRLX: 125 passing, 0 failures.** Was 111 → +14 (#23: model 2, parse-result 2, eval-expression 3, eval-IR-builder 1, test-element-parse 2, test-element runtime 4).
- MVEL3 full suite: unchanged, 720 passing.

### GitHub issues
- `#23` CLOSED this session (test construct).
- `#22` OPEN (Form B if/else — multi-consequence; postponed).
- `#12` OPEN, **NEXT** (Form A if/else — plan ready, depends on shipped #23).
- `#16` still open (RuleIR proto experiment).

### Migration policy
*Unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## Immediate next action

Start **#12** (if/else, Form A). Plan: `plans/2026-04-27-if-else-branching-implementation.md`. 10 tasks, pure grammar + visitor desugar onto the `EvalIR` infrastructure shipped in #23. No risky bridging code remaining. Property-reactivity Task 9 verifies natural behaviour (no workaround needed).

## Gotchas (new this session)

- **DRLX uses `PropertySpecificOption.ALWAYS`** (`DrlxRuleAstRuntimeBuilder.java:105`). All pattern properties watched by default; `EvalCondition.isPatternScopeDelimiter() == true` does NOT inhibit re-evaluation in this configuration. If DRLX ever switches to `ALLOWED` mode, eval guards' property reads need explicit propagation (similar to #19).
- **`EvaluatorSink` already exists** at `org.drools.drlx.builder.EvaluatorSink`. Every lambda-carrying class (`DrlxLambdaConstraint`, `DrlxLambdaBetaConstraint`, `DrlxLambdaConsequence`, now `DrlxEvalExpression`) implements it. `compileBatch(...)` iterates `pendingLambdas` calling `target.bindEvaluator(resolved)`.
- **`EvalExpression.evaluate` signature**: `(BaseTuple, Declaration[], ValueResolver, Object) throws Exception` — 4 args, ValueResolver from `org.drools.base.base`. Tuple value extraction: `tuple.get(decl).getObject()` (canonical) or `tuple.getObject(decl)` (simpler, works for binding-decl).
- **`org.mvel3.Evaluator` is NOT a functional interface** — has multiple methods. Lambda-style instantiation fails at compile time.
- **Top-level parser entry**: `parser.drlxCompilationUnit()` returns `DrlxCompilationUnitContext` (DRLX-specific). `parser.compilationUnit()` returns the inherited Java/MVEL3 `CompilationUnitContext` which has no `ruleDeclaration()` accessor.

Older gotchas: *unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## References

| Topic | Path |
|-------|------|
| #12 spec (revised) | `specs/2026-04-27-if-else-branching-design.md` |
| #12 plan (slimmer, NEXT) | `plans/2026-04-27-if-else-branching-implementation.md` |
| #23 plan (DONE — for reference) | `plans/2026-04-27-test-construct-implementation.md` |
| Latest blog entry | `blog/2026-04-27-tk02-test-before-if-else.md` |
| #23 implementation commits | `git -C ~/usr/work/mvel3-development/drlx-parser log a1e3101..511696c` |

## Key commands

*Unchanged — retrieve with:* `git show HEAD~1:HANDOFF.md`
