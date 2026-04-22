# HANDOVER

## Session goals (done)

1. ✅ #14 closed — unit-class type inference fully shipped.
2. ✅ #7 closed — tree-shape LHS refactor.
3. ✅ #8 closed — single-element + multi-element `not` both shipped, incl. `not(/a, /b)` with synthetic AND wrapper for multi-child Drools NOT.
4. ✅ 28 commits pushed to `origin/main` (range `83402f2..9d87fc9`).

## Current state

### Test suite
- **68 tests passing, 0 failures** (63 default + 5 no-persist).
- NotTest: 7 (notSuppressesMatch, notWithOuterBinding, protoRoundTrip_withNot, notWithInnerBinding_failsParse, notParens_singleElement, notMultiElement_crossProduct, notEmpty_failsParse).

### GitHub issues
- `#7` CLOSED. `#8` CLOSED. `#14` CLOSED.
- `#15` (runtime RuleUnitInstance wiring) still tracks MyUnit's dropped `RuleUnitData` interface.
- **Next candidate: `#9` `exists` (bare + paren)** — same shape as `not`, can likely ship both forms in one landing.

## Immediate next action

Start `#9` — `exists /pattern` and `exists(/a, /b)`. Check first: does Drools' `newExistsInstance()` also require a single child? If yes, reuse the AND-wrapping pattern added to `DrlxRuleAstRuntimeBuilder.buildLhs` for NOT. Spec + plan mirror `not`'s: same `GroupElementIR(EXISTS, [...])` shape, `Kind.EXISTS` slot already reserved in the IR enum.

## Gotchas (new this session)

- **Drools `newNotInstance()` requires exactly one child.** Multi-child NOT needs a synthetic AND wrapper: `NOT(AND(a, b))`. Fix added to `DrlxRuleAstRuntimeBuilder.buildLhs` (commit `be7346d`). Likely applies to EXISTS too — verify before writing #9 spec.
- **ANTLR `#label` on alternatives splits the Context class.** Labels generate per-alternative subclasses and remove the base class's shared accessors — any existing `ctx.someChild()` call on the parent type breaks. Use unlabelled alternatives when the base class has existing consumers.

Older gotchas: *unchanged — retrieve with `git show HEAD~1:HANDOFF.md`*

## References

| Topic | Path |
|-------|------|
| #8 plan (closed) | `plans/2026-04-22-multi-element-not-implementation.md` |
| #8 multi-element spec | `specs/2026-04-22-multi-element-not-design.md` |
| Latest blog entry | `blog/2026-04-22-tk01-not-a-b-two-gotchas.md` |
| DRLXXXX.md `not`/`exists` spec | `docs/DRLXXXX.md` §"'not' / 'exists'" (line 595-600) |
| Drools source (read-only) | `/home/tkobayas/usr/work/mvel3-development/drools` |

## Key commands

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*
