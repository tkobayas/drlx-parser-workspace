# HANDOVER

## Session goals (done)

1. ✅ #11 closed — `and(...)` / `or(...)` group CEs + NOT/EXISTS-with-nested-CEs shipped in one landing.
2. ✅ Grammar refactor γ — CE `,` lifted to `ruleItem`, unified `groupChild` non-terminal used by all four CE paren forms. Single source of truth for future CE additions.
3. ✅ 14 commits pushed to `origin/main` (range `bb6f3c4..184daaa`).

## Current state

### Test suite
- **84 tests passing, 0 failures** (79 default + 5 no-persist).
- AndTest: 4 new (match, single-child, empty-parse-fail, bare-parse-fail).
- OrTest: 4 new — `orBothBranchesFire` confirms `LogicTransformer` top-level-OR expansion works through the new pipeline.
- NestedGroupTest: 3 new — `or(and, and)`, `and(/, not(/))`, `not(or(/, /))`.
- NotTest (7) + ExistsTest (5) unchanged — grammar refactor was behaviour-preserving for their inputs.

### GitHub issues
- `#11` CLOSED.
- **Next candidate: bound patterns as group children** — `and(var l : /locations, /persons[locationId = l.id])`. Not yet filed as an issue. It's the explicit "next ticket" flagged in the #11 spec §10 non-goals and the blog entry. Widens `groupChild` to accept `rulePattern`-like bindings; biggest grammar change. Gate to DRLXXXX §16 feeling "done."
- Other open tickets (informal order of closeness): `#10` passive `?/persons`, `#13` property-reactive watch list, `#12` if/else branching, `#16` RuleIR proto experiment.

## Immediate next action

Decide next ticket with user. Default pitch: file the bindings-in-group-CEs issue and spec it — it's the natural continuation of this session's work. Alternative: pivot to `#10` passive syntax (which also affects `groupChild` shape — worth sequencing with bindings to avoid double-touching the grammar).

## Gotchas (new this session)

- **Lexer token name collision with transitively imported grammars.** Adding `AND : 'and';` to `DrlxLexer` silently overrode `JavaLexer`'s pre-existing `AND : '&&';` (inherited via `Mvel3Lexer`). Error surfaced as "cannot create implicit token for string literal: `'&&'`" at line numbers that don't exist in the source file (they're post-import line numbers in the combined grammar — `DrlxParser.g4:723` when the source file is 134 lines). Fix: use namespaced token names. We renamed to `DRLX_AND` / `DRLX_OR` to match Drools' `DRL10Lexer` convention (`DRL_AND`, `DRL_OR`, …). **Rule for future grammar work:** before adding a DRLX lexer keyword token, grep the entire import chain for the proposed token *name*, not just the string literal.
- **ANTLR error line numbers can exceed source file length** when grammar imports are involved. Don't chase line numbers in the source; grep for the literal content in the error across the whole import chain.

Older gotchas: *unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## References

| Topic | Path |
|-------|------|
| #11 spec | `specs/2026-04-23-and-or-design.md` |
| #11 plan | `plans/2026-04-23-and-or-implementation.md` |
| Latest blog entry | `blog/2026-04-23-tk01-and-or-and-the-and-that-ate-double-ampersand.md` |
| DRLXXXX.md spec reference | `docs/DRLXXXX.md` §"'and' / 'or' structures" (line 573-593) |
| Drools DRL10 CE grammar | `~/usr/work/mvel3-development/drools/drools-drl/drools-drl-parser/src/main/antlr4/org/drools/drl/parser/antlr4/DRL10Parser.g4` — `lhsNot` / `lhsExists` (line 395, 402) |
| Drools `GroupElement.pack()` | `~/usr/work/mvel3-development/drools/drools-base/src/main/java/org/drools/base/rule/GroupElement.java` (line 143-183) |

## Key commands

*Unchanged — retrieve with:* `git show HEAD~1:HANDOFF.md`
