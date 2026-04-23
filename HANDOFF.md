# HANDOVER

## Session goals (done)

1. ✅ #17 closed — bound patterns as children of `and/or/not/exists` paren CEs. `boundOopath` factored out of `rulePattern`, reused by `groupChild`.
2. ✅ 7 commits pushed to `origin/main` (range `6f5c4cf..3cffa03`).
3. ✅ Spec + plan committed to workspace; blog entry `tk02` written.

## Current state

### Test suite
- **94 tests passing, 0 failures** (89 default + 5 no-persist).
- New this session: `BindingInGroupTest` (4), `OrBindingScopeTest` (3), `NotExistsBindingTest` (2), `NestedGroupTest#andWithNestedNotCarryingBinding` (1 extension).
- All existing tests unchanged — `boundOopath` factor-out is behaviour-preserving at the top level.

### GitHub issues
- `#17` CLOSED. Summary comment left on issue.
- Next candidates (open): `#10` passive `?/persons`, `#13` property-reactive watch list, `#12` if/else branching, `#16` RuleIR proto experiment.
- Claim from blog entry: "DRLXXXX §16 feels done." The "and/or structures" section of the DRLX spec now has canonical bound-pattern examples and a branch-local scope paragraph.

### `docs/DRLXXXX.md` — newly tracked
`docs/DRLXXXX.md` was previously untracked in the project repo; commit `3cffa03` added it to git for the first time (1291 lines) alongside the scope-paragraph edit. User explicitly approved keeping it as-is rather than reverting. Future edits are normal in-place diffs.

## Immediate next action

Decide next ticket with user. Reasonable defaults: `#10` passive `?/persons` (natural since it also touches `groupChild` shape and the grammar now has a clean single-source-of-truth non-terminal to extend) or `#13` property-reactive watch list (different subsystem, no grammar interaction — good if user wants variety).

## Gotchas (new this session)

- **Factoring out an ANTLR non-terminal fans out to every Java reader of the old context's accessors.** When `rulePattern : boundOopath ','` moved the `identifier` / `oopathExpression` accessors onto `BoundOopathContext`, three places broke mid-plan: `DrlxToRuleAstVisitor` (expected — plan covered it), `DrlxToJavaParserVisitor` frozen-parent LSP visitor (not in plan), and `DrlxParserTest` parse-tree assertions (not in plan). Symptom: compile errors "cannot find symbol method identifier(int) on RulePatternContext". Lesson: before committing a grammar factor-out, grep for every call site of the old accessors (`ctx.identifier(0).getText()`, `ctx.oopathExpression()` on a `RulePatternContext`) and include all of them in the plan.
- **Drools throws `RuntimeException` at `KieBase.newKieSession()` when a sibling references an OR-internal binding outside the group.** Makes the branch-local OR scope assertion trivial: wrap `withSession` in `assertThatThrownBy`. No DRLX-layer enforcement needed.

Older gotchas: *unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## References

| Topic | Path |
|-------|------|
| #17 spec | `specs/2026-04-23-bindings-in-groups-design.md` |
| #17 plan | `plans/2026-04-23-bindings-in-groups-implementation.md` |
| Latest blog entries | `blog/2026-04-23-tk01-and-or-and-the-and-that-ate-double-ampersand.md`, `blog/2026-04-23-tk02-bindings-in-groups-and-the-readers-the-plan-missed.md` |
| DRLXXXX.md scope paragraph | `docs/DRLXXXX.md` §"'and' / 'or' structures" (after the nested-or example) |
| Drools `GroupElement.pack()` | `~/usr/work/mvel3-development/drools/drools-base/src/main/java/org/drools/base/rule/GroupElement.java` (line 143-183) — confirmed to preserve outer-AND bindings inside nested NOT |

## Key commands

*Unchanged — retrieve with:* `git show HEAD~1:HANDOFF.md`
