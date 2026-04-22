# HANDOVER

## Session goals (done)

1. ✅ #9 closed — `exists /pattern` + `exists(/a, /b)` both shipped, mirroring the NOT landing.
2. ✅ Proto build migrated to `protobuf-maven-plugin` — future schema changes are `.proto`-only edits; no system `protoc` required.
3. ✅ #15 closed (deferred) — runtime RuleUnitInstance wiring handed off to Mark upstream.
4. ✅ Stale-`.tokens` gotcha auto-fixed: `maven-clean-plugin` now wipes `*.tokens`/`*.interp`/`gen/` from the source tree before ANTLR regenerates.
5. ✅ 8 commits pushed to `origin/main` (range `9d87fc9..bb6f3c4`).

## Current state

### Test suite
- **73 tests passing, 0 failures** (68 default + 5 no-persist).
- NotTest: 7 (unchanged).
- ExistsTest: 5 (new) — `existsAllowsMatch`, `existsWithOuterBinding`, `existsMultiElement_crossProduct`, `existsWithInnerBinding_failsParse`, `existsEmpty_failsParse`.

### GitHub issues
- `#8`, `#9`, `#14`, `#15` CLOSED.
- **Next candidate: `#11` `and(...)` / `or(...)` group CEs.** Helper `buildGroupElementFromOopaths` already takes a `Kind` arg — AND/OR become one-line wrappers. Drools AND/OR accept N children natively, so the AND-wrap condition in `buildLhs` likely simplifies rather than generalises further. Verify before writing the spec.

## Immediate next action

Start `#11`. Check `GroupElement.addChild` in Drools — does it still enforce single-child for AND/OR? Almost certainly not (the morning NOT gotcha source confirmed NOT/EXISTS are the only single-child CEs via `isNot() || isExists()`). If so, the `buildLhs` condition becomes `kind ∈ {NOT, EXISTS} && size > 1` as-is — no new case needed for AND/OR.

## Gotchas (new this session)

- **Stale `DrlxLexer.tokens` in `libDirectory` — now auto-mitigated.** Symptom (`mismatched input 'X' expecting {..., 'X', ...}` — expected-tokens list from parser tables disagrees with what the lexer emits) kept biting during #9. Root cause: ANTLR's `tokenVocab` resolution searches the grammar file's own directory; IDE-side ANTLR plugins drop stale `.tokens`/`.interp`/`gen/` there. Commit `bb6f3c4` adds a `maven-clean-plugin` execution at `generate-sources` that wipes those files before ANTLR regenerates. If an IDE-only build bypasses Maven, fall back to `mvn clean install`.
- **Proto build is now plugin-managed.** `drlx_rule_ast.proto` → regenerated on every `compile` by `protobuf-maven-plugin` via `os-maven-plugin` extension. No checked-in `DrlxRuleAstProto.java`. Schema changes = edit `.proto`, let Maven regen. No system `protoc` install needed.

Older gotchas: *unchanged — retrieve with* `git show HEAD~1:HANDOFF.md`

## References

| Topic | Path |
|-------|------|
| #9 spec | `specs/2026-04-22-exists-design.md` |
| #9 plan | `plans/2026-04-22-exists-implementation.md` |
| Latest blog entry | `blog/2026-04-22-tk02-exists-and-two-build-detours.md` |
| DRLXXXX.md spec reference | `docs/DRLXXXX.md` §"'not' / 'exists'" (line 595-600) |
| Drools source (read-only) | `/home/tkobayas/usr/work/mvel3-development/drools` |

## Key commands

*Unchanged — retrieve with:* `git show HEAD~1:HANDOFF.md`
