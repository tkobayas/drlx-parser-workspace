# HANDOVER

## Session goals (mixed result)

1. ✅ Rule-annotation LSP tolerance fix (epilogue to prior session) — shipped
2. ⏸️ Tier 2 (#7 GroupElement infra + #8 `not /pattern`) — **5 of 12 tasks landed, then discovered a plan gap and paused**
3. ✅ Gap decomposed into new sub-issues #14 (type inference) and #15 (RuleUnit runtime), spec + plan written for #14

## Current state

### What landed in the project repo (`/home/tkobayas/usr/work/mvel3-development/drlx-parser`, branch `main`)

```
cd63875 test(not): add failing happy-path notSuppressesMatch (RED)
307bc45 fix(grammar): notElement requires trailing comma         ← plan gap #1 (Task 3 omission)
2f5d9af feat(grammar): add notElement production for single-element `not`
77724b1 feat(lexer): add NOT keyword token
dcaa791 refactor(ir): generalize RuleIR to tree-shape LHS         ← #7 infra, atomic refactor
83402f2 fix(parser): tolerate rule annotations in LSP visitor     ← prior session follow-up
```

- `NotTest#notSuppressesMatch` is **currently RED** (intentional — waiting for #8 Task 5 to land). All other tests pass.
- Baseline before resuming: run the suite and confirm 50 green + 1 RED.

### What landed in the workspace repo

- `specs/2026-04-20-rule-annotations-design.md` (prior session)
- `specs/2026-04-21-group-element-and-not-design.md` — #7/#8 design (`fd1b7c1`)
- `specs/2026-04-21-type-inference-from-unit-class-design.md` — #14 design (`ebe3fbf`)
- `plans/2026-04-21-group-element-and-not-implementation.md` — #8 plan, **Tasks 5-13 pending**
- `plans/2026-04-21-type-inference-from-unit-class-implementation.md` — #14 plan, **Tasks 1-10 + final**
- `blog/2026-04-21-tk01-where-does-person-come-from.md` — pivot entry

### GitHub issues (repo: `tkobayas/drlx-parser`)

- `#4` open — DrlxCompiler enhancement round 1 (parent epic, contains the 7 sub-issues as native GitHub sub-issues)
- `#7` open — GroupElement infra
- `#8` open, **blocked on #14** — `not /pattern`
- `#9..13` open — downstream features
- `#14` open — **NEXT UP**: unit-class type inference (compile-time only)
- `#15` open — RuleUnitInstance runtime integration (post-#14)

## The blocker that stopped execution

Mid-#8 at Task 5 (GREEN for `visitNotElement`): the DRLX spec form `not /persons[age < 18]` has no explicit type, but the runtime builder requires `typeName` to resolve a Java class. `var p : /persons[...]` has the same problem — `resolveType("var")` throws.

Solution path: read a required unit class's `DataSource<T>` / `DataStore<T>` fields to infer types. Decomposed into #14 (compile-time) + #15 (runtime). Brainstormed + planned #14; closing the session before executing it.

## Immediate next action

**Open a fresh session** and dispatch #14 Task 1:

1. Read `plans/2026-04-21-type-inference-from-unit-class-implementation.md` (Task 1).
2. Invoke `superpowers:subagent-driven-development` (matches this epic's cadence).
3. Dispatch #14 Task 1 implementer — adds `unitName` to `CompilationUnitIR`, proto, translator, visitor. No behaviour change; baseline tests stay green.

After #14 closes: resume #8 at Task 5 (re-open `plans/2026-04-21-group-element-and-not-implementation.md`). The RED `NotTest#notSuppressesMatch` will go GREEN once `visitNotElement` lands.

## Gotchas discovered this session (for future)

- **ANTLR trailing-comma convention.** `rulePattern` has `','` baked in; any new `ruleItem` alternative needs the same or the item-separator logic in `ruleBody` breaks. Fixed in `307bc45` after Task 4's RED test failed with `"mismatched input ','"` instead of the expected visitor error. Cost: one small commit, caught by the "RED must fail for the RIGHT reason" discipline.
- **`gh api` integer fields.** `sub_issue_id` is integer-typed; must use `-F` (capital) not `-f` (lowercase, sends string and 422s). Applies to any typed proto field in gh API.
- **Subagent test-count hallucinations.** A Task 3 subagent reported "100 tests passing" (actual: 50, 45+5 double-counted). "Do not trust the report" gate caught it — always grep actual `Tests run:` output.
- **The `var` sentinel.** DRLX grammar accepts `var p : /persons[...]` but no runtime test exercises it. `"var"` is decided to be treated as "no explicit type" in #14's `resolvePatternType`.

## References (locate, don't open)

| Topic | Path |
|-------|------|
| #7/#8 spec | `specs/2026-04-21-group-element-and-not-design.md` |
| #7/#8 plan (Tasks 5-13 pending) | `plans/2026-04-21-group-element-and-not-implementation.md` |
| #14 spec | `specs/2026-04-21-type-inference-from-unit-class-design.md` |
| #14 plan (NEXT UP) | `plans/2026-04-21-type-inference-from-unit-class-implementation.md` |
| Candidates tracker | `specs/IMPLEMENT_SYNTAX_CANDIDATES.md` |
| This session's blog entry | `blog/2026-04-21-tk01-where-does-person-come-from.md` |
| Drools source (read-only reference) | `/home/tkobayas/usr/work/mvel3-development/drools` |
| Key Drools classes | `drools-base/.../GroupElement.java`, `GroupElementFactory.java`, `RuleImpl.java`; `drools-ruleunits/drools-ruleunits-api/.../DataStore.java` |

## Key commands

```bash
# Build + test (project repo)
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install

# Regen proto (bundled protoc, NOT system)
/tmp/protoc-25.5/bin/protoc --java_out=/home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java \
    --proto_path=/home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/proto \
    /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/proto/drlx_rule_ast.proto

# Git in project repo (use -C, never cd)
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser <cmd>
```

## Session cadence learned

- Subagent-driven-development with two-stage review (spec + code quality) worked well for Tasks 1-4.
- Haiku was fine for trivial tasks (lexer token, grammar addition); standard sonnet for multi-file refactors.
- Native GitHub sub-issues (via `gh api repos/.../issues/N/sub_issues -F sub_issue_id=<id>`) are nicer than task-list-in-parent-body.
