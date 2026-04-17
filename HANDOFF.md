# HANDOVER

## Session Goals

1. Decide the next DRLX syntax feature to implement after inline cast (commit `0ce0b60`)
2. Plan positional syntax (`/locations("paris")`) for implementation
3. Trim dead/legacy visitor code from the parser package

## Current State

### Completed This Session

1. **Reviewed `docs/DRLXXXX.md` annotation coverage** — the spec does NOT list DRL7's `@salience` / `@no-loop` / `@agenda-group`; it specifies `@Description`, `@ExistanceDriven`, `@Tabled`, `@DataSource`, field-level `@Position`, etc. Given the gap, rule annotations/attributes are **ON HOLD** pending clarification from the spec designer. Noted in `IMPLEMENT_SYNTAX_CANDIDATES.md`.

2. **Reference source trees documented** (commit `0e6788b`) — `CLAUDE.md` now points to `/home/tkobayas/usr/work/mvel3-development/mvel` and `/home/tkobayas/usr/work/mvel3-development/drools` as read-only references for looking up MVEL / Drools internals (e.g. `@Position` handling in `ConstraintExpression.java`) instead of extracting jars from `~/.m2`.

3. **Created `Positional_PLAN.md`** — detailed implementation plan for positional syntax, including:
   - Grammar split: new `oopathRoot` rule carries positional `(...)`; navigation `oopathChunk` does not (prevents silent-drop on later chunks)
   - `@Position` resolution via a precomputed `Map<Integer, Field>` for the full class hierarchy; **fails on duplicates** (per-class or inherited collisions)
   - Proto schema: `repeated string positional_args = 6;` in `PatternParseResult`
   - Defensive synthesis: `fieldName == (argExpr)` (parens around RHS)
   - Test POJO: new `Location.java` with `@Position(0) city`, `@Position(1) district`
   - Tests: positive (`testPositionalSyntax`, `testPositionalAndSlotted`, `testPositionalSyntaxTwoArgs`) + negative (missing annotation, duplicate, inherited collision, complex RHS, non-root-chunk)
   - 7-phase implementation step order

4. **Codex second-opinion applied to the plan** — caught 5 substantive issues and reflected them in `Positional_PLAN.md`:
   - Silent-drop bug on later chunks → grammar split
   - `@Position` "first wins" unsafe → precompute map + fail on duplicates
   - Unparenthesized synthesis brittle → `field == (argExpr)`
   - Non-canonical visitors (`DrlxToJavaParserVisitor`, `DrlxToDescrVisitor`) also traverse `oopathChunk` — decide how to handle
   - Happy-path-only tests insufficient → add 5 negative tests

5. **Removed `DrlxToDescrVisitor`** (commit `0e6788b`) — legacy visitor that produced `drools-drl-ast` `PackageDescr`/`RuleDescr` for the old Drools DRL compiler path, which is no longer a target. Removed:
   - `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToDescrVisitor.java`
   - `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxToDescrVisitorTest.java`
   - `parseDrlxCompilationUnitAsPackageDescr()` + imports from `DrlxHelper.java`
   - Unused `drools-drl-ast` Maven dependency from `drlx-parser-core/pom.xml`
   - Two table entries in `docs/DESIGN.md`

6. **Marked `DrlxToJavaParserVisitor` as frozen** (commit `cac07f0`) — retained only as the base class for `TolerantDrlxToJavaParserVisitor`, which drlx-lsp uses for code completion via the token→AST-node map. Class-level Javadoc and `docs/DESIGN.md` now explicitly state that new DRLX syntax features are NOT propagated here.

### Architecture After This Session

```
drlx-parser/                        (parent POM)
├── drlx-parser-core/               (core library)
│   ├── src/main/java/              builder, parser, tools, util
│   ├── src/main/antlr4/            DRLX + MVEL3 grammars
│   ├── src/main/proto/             drlx_rule_ast.proto
│   └── src/test/java/              tests + domain POJOs
└── drlx-parser-benchmark/          (JMH benchmarks)
```

**Canonical pipeline** (single path — consolidation complete):
```
ANTLR → DrlxToRuleAstVisitor → DrlxRuleAstModel → DrlxRuleAstRuntimeBuilder → RuleImpl
                                (proto cache)         (uses DrlxLambdaCompiler)
```

**Parallel non-canonical paths** (do not target `RuleImpl`):
- `DrlxToJavaParserVisitor` → JavaParser + MVEL3 `OOPathChunk` AST — **frozen**; exists only as the base for:
- `TolerantDrlxToJavaParserVisitor` → used by drlx-lsp for code completion

## Decisions and Rationale

- **Rule annotations on hold, not abandoned** — the spec designer needs to confirm which annotation set DRLX supports before we pick between DRL7 parity (`@salience`, `@no-loop`, `@agenda-group`) vs. the DRLXXXX.md set (`@Description`, `@ExistanceDriven`, `@Tabled`, `@Position`, `@DataSource`). Pick positional syntax first because the spec is unambiguous there.

- **Positional uses `@Position` annotation**, not declared-field order. Matches Drools 7.x `ConstraintExpression.getFieldAtPosition` semantics and DRLXXXX.md §978. Reliance on `getDeclaredFields()` order is JVM-implementation-defined — explicit annotation is the only robust contract.

- **Duplicate/inherited `@Position` → error, not silent "first wins"** — the DRLXXXX spec doesn't sanction override semantics, and hidden precedence rules are bug factories. Fail at build time with a clear message naming both fields.

- **`fieldName == (argExpr)` with defensive parens** — ternaries, casts, or future MVEL quirks could reinterpret `fieldName == argExpr` if `argExpr` has its own operator. Cost is zero; safety is clear.

- **Grammar split `oopathRoot` vs `oopathChunk`** — positional only makes sense on the pattern type, not on path navigation segments. Enforcing this at the grammar level makes `/a/b("x")` a parse error rather than a silent-drop footgun.

- **Positional args `expression`, not `drlxExpression`** — keeps the grammar simple for the first cut. Consequence: DRLXXXX's `var postCode` out-binding inside positional (§504) is **actively unsupported**, not merely deferred. Separate feature if ever needed.

- **`DrlxToDescrVisitor` removed** rather than frozen — no consumers anywhere (verified in both `drlx-parser` and `drlx-lsp`). Carrying it forward would mean teaching it every new DRLX syntax feature for no benefit.

- **`DrlxToJavaParserVisitor` frozen, not removed** — its `TolerantDrlxToJavaParserVisitor` subclass is load-bearing for drlx-lsp completion. The LSP uses the token→AST-node map for cursor-scope resolution, which does NOT require semantic completeness of the AST. So we can freeze semantic parity without breaking the LSP.

## Problems Encountered and Resolutions

1. **Initial Plan agent missed non-canonical visitor blast radius** — Codex review caught that `DrlxToDescrVisitor` and `DrlxToJavaParserVisitor` also traverse `oopathChunk` and would silently drop positional args. Added §8b to `Positional_PLAN.md` to document the decision, then user chose to delete `DrlxToDescrVisitor` and freeze `DrlxToJavaParserVisitor`.

2. **First `/java-code-review` run had nothing staged** — workflow requires staged changes; files had been edited but not `git add`-ed. Staged and re-ran successfully.

3. **`rm` without `-f` prompted for confirmation** — tool required `-f` to delete non-interactively.

## Remaining Risks and Uncertainties

- **`DrlxToJavaParserVisitor` freeze policy enforcement** — the Javadoc warns future contributors, but nothing mechanically prevents a drive-by edit from adding positional handling. If it becomes a recurring temptation, consider splitting the file so `TolerantDrlxToJavaParserVisitor` doesn't have to subclass it.

- **MVEL3 `OOPathChunk` AST has no positional field** — if a future need arises to thread positional through the JavaParser AST path (e.g. for formatting or IDE-based rule rendering), MVEL3's `OOPathChunk` class would need extension. Not blocking anything now.

- **Rule annotations blocked on external decision** — can't proceed without spec designer input on which annotations DRLX formally supports.

- **Grammar split for `oopathRoot`** is a plan-level decision, not yet implemented — phase 1 of positional will need to handle the refactor carefully to avoid breaking existing `oopathChunk` references in the other visitors (even if they ignore positional, they still traverse the chunks).

## Concrete Next Actions (Priority Order)

1. **Implement positional syntax per `Positional_PLAN.md`** — 7 phases:
   1. Grammar split (`oopathRoot` + `oopathChunk`), regenerate ANTLR parser
   2. Extend `PatternIR` record with `List<String> positionalArgs`
   3. `DrlxToRuleAstVisitor.buildPattern` populates positional args via `extractPositionalArgs`
   4. Proto schema `repeated string positional_args = 6;`, regenerate `DrlxRuleAstProto.java`, update `DrlxRuleAstParseResult`
   5. `DrlxLambdaCompiler.resolvePositionalField` (precomputed hierarchy map, duplicate detection); `DrlxRuleAstRuntimeBuilder.buildPattern` synthesizes `field == (argExpr)` constraints
   6. New `Location.java` test POJO (`@Position(0) city`, `@Position(1) district`)
   7. Tests: `testPositionalSyntax`, `testPositionalAndSlotted`, `testPositionalSyntaxTwoArgs`, + 5 negative tests (missing / duplicate / inherited collision / complex RHS / non-root chunk)

2. **After positional is merged** — follow up with Tier 2 items from `IMPLEMENT_SYNTAX_CANDIDATES.md`: GroupElement infrastructure → `not`/`exists`/passive patterns.

3. **Revisit rule annotations** once the spec designer clarifies which annotations DRLX formally supports.

4. **Clean up leftover artifacts** (low priority, unchanged from prior session): `RuleAST_Improve.md`, `profile-cpu/`, `profile-wall/`, `profile1/`, `results.csv`, `tmp/`, `quick-check.diff`, `tmp-cc-bak.txt`.

## Files Modified This Session

| File | Change | Commit |
|------|--------|--------|
| `CLAUDE.md` | Reference source tree paths added | `0e6788b` |
| `docs/DESIGN.md` | Removed `DrlxToDescrVisitor` rows; updated `DrlxToJavaParserVisitor` row with freeze marker | `0e6788b`, `cac07f0` |
| `drlx-parser-core/pom.xml` | Removed `drools-drl-ast` dependency | `0e6788b` |
| `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToDescrVisitor.java` | **Deleted** | `0e6788b` |
| `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxToDescrVisitorTest.java` | **Deleted** | `0e6788b` |
| `drlx-parser-core/src/main/java/org/drools/drlx/util/DrlxHelper.java` | Removed `parseDrlxCompilationUnitAsPackageDescr` + imports | `0e6788b` |
| `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java` | Javadoc freeze notice | `cac07f0` |
| `IMPLEMENT_SYNTAX_CANDIDATES.md` | Marked rule annotations on-hold with DRLXXXX.md discrepancy note | (uncommitted) |

## Files Created This Session

| File | Purpose |
|------|---------|
| `Positional_PLAN.md` | Detailed implementation plan for positional syntax (Codex-reviewed) |

## Commits This Session

```
cac07f0 docs(parser): mark DrlxToJavaParserVisitor frozen for DRLX syntax
0e6788b refactor(parser): remove unused DrlxToDescrVisitor
```

## Key Commands

```bash
# Build all modules
mvn install -DskipTests

# Build + test core only
mvn -pl drlx-parser-core -am install -DskipTests && mvn -pl drlx-parser-core test

# Regenerate proto Java class (needed for phase 4 of positional impl)
/tmp/protoc/bin/protoc --java_out=drlx-parser-core/src/main/java \
    --proto_path=drlx-parser-core/src/main/proto \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto

# Regenerate ANTLR parser (needed for phase 1 of positional impl)
mvn -pl drlx-parser-core generate-sources
```

## Reference Source Trees (read-only)

- MVEL: `/home/tkobayas/usr/work/mvel3-development/mvel`
- Drools: `/home/tkobayas/usr/work/mvel3-development/drools`
  - `@Position` annotation: `kie-api/src/main/java/org/kie/api/definition/type/Position.java`
  - Positional resolution precedent: `drools-model/drools-model-codegen/src/main/java/org/drools/model/codegen/execmodel/generator/drlxparse/ConstraintExpression.java:81-94`
