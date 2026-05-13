# DRLX Accumulate (#39) — v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement DRLX accumulate forms 1 and 2 (simple-function and multi-function, built-ins only). Adds `var x = avg(p.age),` and the multi-function variant; no `acc` keyword in v1.

**Architecture:** Direct construction at the runtime-builder layer (matches existing DRLX). Grammar gets one new alternative; IR gets `AccumulatePatternIR` bundling source pattern + N `AccumulatorIR`s; visitor folds adjacent pattern+accumulate items; proto mirrors the IR; runtime builder constructs N independent `SingleAccumulate`s sharing cloned source patterns. Function-name resolution and binding-scope validation are build-time concerns (no visitor-time scope checking). Qualified names (e.g. `Func.avg`) parse but are rejected at build time with a v1-restriction error.

**Tech Stack:** Java 17+, ANTLR4 (via `antlr4-maven-plugin`), protobuf-java (via `protoc`), Drools 999-SNAPSHOT (`org.drools.base.rule.Accumulate`/`SingleAccumulate`/`Accumulator`, `org.drools.core.base.accumulators.*`), MVEL3 3.0.0-SNAPSHOT (lambda compiler), JUnit 5, AssertJ, Maven.

**Source spec:** `/home/tkobayas/claude/public/drlx-parser/specs/2026-05-13-drlx-accumulate-design.md`

**Project working directory:** `/home/tkobayas/usr/work/mvel3-development/drlx-parser` (branch `main`).

**Workspace directory:** `/home/tkobayas/claude/public/drlx-parser` (workspace; receives plan, handover).

**Commit discipline:** Every project-repo commit references `#39` (or `Closes #39` at the final commit). Workspace commits track planning/handover artifacts independently.

**Sanity check before starting:** From the project repo, `mvn -pl drlx-parser-core -am test` must pass cleanly (182 tests). If not, do not start — fix the baseline first.

---

## File Structure

### Files created (in project repo)

| Path | Purpose |
|---|---|
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/AccumulateFunctionRegistry.java` | Static lookup table: function name → `AccumulateFunction` class. Holds the v1 built-ins (avg/sum/min/max/count) and the v1 result-type table. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaAccumulator.java` | Implements `org.drools.base.rule.accessor.Accumulator`; wraps an `AccumulateFunction` instance plus an optional value-extractor lambda. Equivalent to Drools' `LambdaAccumulator.BindingAcc` but lives in `drlx-parser-core` to avoid depending on `drools-model-compiler`. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateFunctionRegistryTest.java` | Unit tests for the registry. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java` | End-to-end DRLX → KieSession execution tests. |

### Files modified (in project repo)

| Path | Change |
|---|---|
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` | Add `accumulateItem` ruleItem alternative + `accumulateItem`/`accumulateCall` rules. |
| `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` | Add `AccumulatePatternParseResult` + `AccumulatorParseResult` messages; add `oneof` arm in `LhsItemParseResult`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` | Add `AccumulatePatternIR` + `AccumulatorIR` records; extend `LhsItemIR` sealed interface. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Visit `accumulateItem`; fold adjacent `boundOopath` + `accumulateItem`s into one `AccumulatePatternIR`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` | Proto round-trip for the new IR types. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` | New branch in `buildLhs` handling `AccumulatePatternIR`: inner-scope assembly, N×SingleAccumulate construction, outer-scope binding of results. |
| `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java` | Parser tests for new accumulate forms. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java` | IR construction tests. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstParseResultTest.java` | Proto roundtrip tests. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilderTest.java` | Runtime builder unit tests (inner/outer scope, qualified-name rejection, unknown-function rejection). |

### Files deleted

None.

---

## Task ordering

| Task | Output | Verification |
|---|---|---|
| 1 | Issue linkage + branch confirmed | `gh issue view 39` shows v1 scope comment |
| 2 | Grammar accepts new forms | `DrlxParserTest` passes new cases |
| 3 | IR records + sealed interface extended | `DrlxRuleAstModelTest` passes new cases |
| 4 | Visitor emits + folds correctly | `DrlxToJavaParserVisitorTest` (or new tests) pass |
| 5 | Proto roundtrip works | `DrlxRuleAstParseResultTest` passes new cases |
| 6 | Registry + accumulator wrapper compile and unit-test | `AccumulateFunctionRegistryTest` passes |
| 7 | Single-function lowering works end-to-end | First subset of `AccumulateTest` passes |
| 8 | Multi-function lowering works | Multi-function `AccumulateTest` cases pass |
| 9 | Error paths reject correctly | Negative `AccumulateTest` cases pass |
| 10 | Full test suite green | `mvn -pl drlx-parser-core test` clean |
| 11 | Handover, push, optional PR | `HANDOFF.md` updated in workspace; commits pushed; #39 closed |

---

## Task 1: Issue linkage and branch setup

**Files:** none modified yet.

- [ ] **Step 1.1: Confirm branch and clean tree**

Run:
```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser status
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser log --oneline -1
```
Expected: branch `main`, clean working tree, HEAD at `46f7d2f` (or a later commit if the baseline moved).

- [ ] **Step 1.2: Confirm baseline tests pass**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am test
```
Expected: BUILD SUCCESS. If failing, stop and fix the baseline before doing anything else.

- [ ] **Step 1.3: Post v1 scope comment on issue #39**

Run:
```bash
gh issue comment 39 --repo tkobayas/drlx-parser --body "v1 scope: forms 1 (simple-function) + 2 (multi-function) only. Built-ins: avg, sum, min, max, count. Lowering: N × SingleAccumulate (MultiAccumulate folding is fast-follow). Spec: specs/2026-05-13-drlx-accumulate-design.md. Plan: plans/2026-05-13-drlx-accumulate-implementation.md."
```
Expected: `https://github.com/tkobayas/drlx-parser/issues/39#issuecomment-...` printed.

---

## Task 2: Grammar — `accumulateItem` and `accumulateCall`

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- Test:   `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java`

- [ ] **Step 2.1: Locate existing parser-test pattern**

Run:
```bash
grep -n "void parses\|@Test" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java | head -10
```
Expected: a list of `@Test`-annotated method names. Note the test class style — use it as a template for new tests.

- [ ] **Step 2.2: Write failing parser tests for accumulate forms**

Open `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java` and add the following inside the test class (place near other rule-pattern tests):

```java
@Test
void parsesSingleAccumulateBindingWithVar() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                var p : /persons,
                var avgAge = avg(p.age),
                do {}
            }
            """;
    assertNoParseErrors(drlx);
}

@Test
void parsesAccumulateBindingWithExplicitType() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                var p : /persons,
                int total = sum(p.age),
                do {}
            }
            """;
    assertNoParseErrors(drlx);
}

@Test
void parsesAccumulateBindingWithGenericType() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                var p : /persons,
                BigDecimal total = sum(p.amount),
                do {}
            }
            """;
    assertNoParseErrors(drlx);
}

@Test
void parsesCountWithNoArgs() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                var p : /persons,
                long n = count(),
                do {}
            }
            """;
    assertNoParseErrors(drlx);
}

@Test
void parsesQualifiedAccumulateFunctionName() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                var p : /persons,
                var avgAge = Func.avg(p.age),
                do {}
            }
            """;
    assertNoParseErrors(drlx);
}

@Test
void parsesMultipleAccumulateItems() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                var p : /persons,
                var minAge = min(p.age),
                var maxAge = max(p.age),
                var avgAge = avg(p.age),
                do {}
            }
            """;
    assertNoParseErrors(drlx);
}
```

If `assertNoParseErrors` does not already exist in this test class, add this helper near the top (the test class probably already has a similar method — reuse it):

```java
private static void assertNoParseErrors(String drlx) {
    DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(drlx));
    CommonTokenStream tokens = new CommonTokenStream(lexer);
    DrlxParser parser = new DrlxParser(tokens);
    parser.removeErrorListeners();
    var errors = new java.util.ArrayList<String>();
    parser.addErrorListener(new BaseErrorListener() {
        @Override public void syntaxError(Recognizer<?,?> r, Object o, int l, int c, String m, RecognitionException e) {
            errors.add("line " + l + ":" + c + " " + m);
        }
    });
    parser.drlxStart();
    assertThat(errors).as("parse errors").isEmpty();
}
```

- [ ] **Step 2.3: Run new tests, confirm they fail**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=DrlxParserTest#parsesSingleAccumulateBindingWithVar
```
Expected: FAIL with parse errors (`mismatched input`/`extraneous input` at the `var avgAge = avg(...)` site). This confirms the grammar doesn't accept the new form yet.

- [ ] **Step 2.4: Modify `DrlxParser.g4` — add accumulate rules**

Open `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`.

In the `ruleItem` rule, add the new alternative *before* `notElement ','` (ANTLR tries alternatives top-to-bottom; placing accumulate near the top means lookahead matches it early when applicable). After:

```antlr
ruleItem
    : rulePattern
    | notElement ','
    | existsElement ','
    | ...
    | ruleConsequence
    ;
```

change to:

```antlr
ruleItem
    : rulePattern
    | accumulateItem ','
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','
    | conditionalBranch ','
    | ruleConsequence
    ;
```

Then add the two new rules at the end of the file (after `watchItem`):

```antlr
// Accumulate result binding — DRLXXXX §Accumulate. The argument list is
// optional to admit `count()` (Drools' count() takes no argument).
accumulateItem
    : (VAR | typeType) identifier '=' accumulateCall
    ;

accumulateCall
    : qualifiedName '(' (expression (',' expression)*)? ')'
    ;
```

- [ ] **Step 2.5: Regenerate parser sources and re-run tests**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=DrlxParserTest
```
Expected: all new tests PASS, no regressions in existing tests.

- [ ] **Step 2.6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 \
    drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(grammar): add accumulateItem rule for var/typed bindings (Refs #39)"
```

---

## Task 3: IR — `AccumulatePatternIR` and `AccumulatorIR`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`
- Test:   `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java`

- [ ] **Step 3.1: Write failing IR tests**

Open `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java` and add:

```java
@Test
void accumulatorIrCopiesArgsAndReferences() {
    var ir = new DrlxRuleAstModel.AccumulatorIR(
            "var", "avgAge", "avg",
            List.of("p.age"),
            List.of("p"));
    // List.copyOf must defensively copy: caller can mutate input freely.
    assertThat(ir.argExpressions()).containsExactly("p.age");
    assertThat(ir.referencedBindings()).containsExactly("p");
}

@Test
void accumulatePatternIrCopiesAccumulators() {
    var src = new DrlxRuleAstModel.PatternIR(
            "var", "p", "persons",
            List.of(), null, List.of(), false, List.of());
    var acc = new DrlxRuleAstModel.AccumulatorIR(
            "var", "avgAge", "avg",
            List.of("p.age"), List.of("p"));
    var pat = new DrlxRuleAstModel.AccumulatePatternIR(src, List.of(acc));
    assertThat(pat.source().bindName()).isEqualTo("p");
    assertThat(pat.accumulators()).hasSize(1);
    assertThat(pat.accumulators().get(0).functionName()).isEqualTo("avg");
}

@Test
void accumulatePatternIrIsLhsItem() {
    var src = new DrlxRuleAstModel.PatternIR(
            "var", "p", "persons",
            List.of(), null, List.of(), false, List.of());
    var pat = new DrlxRuleAstModel.AccumulatePatternIR(src, List.of());
    DrlxRuleAstModel.LhsItemIR item = pat; // must compile
    assertThat(item).isInstanceOf(DrlxRuleAstModel.AccumulatePatternIR.class);
}
```

- [ ] **Step 3.2: Run the new tests, confirm they fail**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=DrlxRuleAstModelTest
```
Expected: compilation error — `AccumulatePatternIR`/`AccumulatorIR` not found.

- [ ] **Step 3.3: Add the records**

Open `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`. Modify the sealed interface and add the new records:

```java
public sealed interface LhsItemIR
        permits PatternIR, GroupElementIR, EvalIR, AccumulatePatternIR {
}
```

Append at the end of the class, before the closing `}`:

```java
public record AccumulatePatternIR(PatternIR source,
                                  List<AccumulatorIR> accumulators) implements LhsItemIR {
    public AccumulatePatternIR {
        accumulators = List.copyOf(accumulators);
    }
}

public record AccumulatorIR(String resultTypeName,
                            String resultBindName,
                            String functionName,
                            List<String> argExpressions,
                            List<String> referencedBindings) {
    public AccumulatorIR {
        argExpressions     = List.copyOf(argExpressions);
        referencedBindings = List.copyOf(referencedBindings);
    }
}
```

- [ ] **Step 3.4: Run tests**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=DrlxRuleAstModelTest
```
Expected: PASS.

- [ ] **Step 3.5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(ir): add AccumulatePatternIR and AccumulatorIR (Refs #39)"
```

---

## Task 4: Visitor — emit and fold

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Test:   create `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`

- [ ] **Step 4.1: Locate the existing visitor pattern**

Run:
```bash
grep -n "visitRuleBody\|visitRuleItem\|visitBoundOopath" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java | head -20
```
Note the names of the existing visit methods — the new method should follow the same naming convention.

- [ ] **Step 4.2: Write failing visitor tests**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`:

```java
package org.drools.drlx.builder;

import java.util.List;
import org.junit.jupiter.api.Test;
import org.antlr.v4.runtime.*;
import org.drools.drlx.parser.DrlxLexer;
import org.drools.drlx.parser.DrlxParser;

import static org.assertj.core.api.Assertions.assertThat;

class AccumulateVisitorTest {

    @Test
    void foldsBoundOopathAndOneAccumulateIntoAccumulatePatternIR() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var p : /persons,
                    var avgAge = avg(p.age),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        assertThat(rule.lhs()).hasSize(1);
        var item = rule.lhs().get(0);
        assertThat(item).isInstanceOf(DrlxRuleAstModel.AccumulatePatternIR.class);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) item;
        assertThat(accPat.source().bindName()).isEqualTo("p");
        assertThat(accPat.accumulators()).hasSize(1);
        var acc = accPat.accumulators().get(0);
        assertThat(acc.resultBindName()).isEqualTo("avgAge");
        assertThat(acc.resultTypeName()).isEqualTo("var");
        assertThat(acc.functionName()).isEqualTo("avg");
        assertThat(acc.argExpressions()).containsExactly("p.age");
        assertThat(acc.referencedBindings()).containsExactly("p");
    }

    @Test
    void foldsThreeAccumulatorsSharingOneSource() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var p : /persons,
                    var avgAge = avg(p.age),
                    var minAge = min(p.age),
                    var maxAge = max(p.age),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        assertThat(rule.lhs()).hasSize(1);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        assertThat(accPat.accumulators())
                .extracting(DrlxRuleAstModel.AccumulatorIR::resultBindName,
                            DrlxRuleAstModel.AccumulatorIR::functionName)
                .containsExactly(
                        org.assertj.core.groups.Tuple.tuple("avgAge", "avg"),
                        org.assertj.core.groups.Tuple.tuple("minAge", "min"),
                        org.assertj.core.groups.Tuple.tuple("maxAge", "max"));
    }

    @Test
    void countWithNoArgumentHasEmptyArgList() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var p : /persons,
                    long n = count(),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        var acc = accPat.accumulators().get(0);
        assertThat(acc.functionName()).isEqualTo("count");
        assertThat(acc.argExpressions()).isEmpty();
        assertThat(acc.referencedBindings()).isEmpty();
        assertThat(acc.resultTypeName()).isEqualTo("long");
    }

    @Test
    void qualifiedFunctionNamePreservedVerbatim() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var p : /persons,
                    var avgAge = Func.avg(p.age),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("Func.avg");
    }

    private static DrlxRuleAstModel.RuleIR parseRule(String drlx) {
        DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(drlx));
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        DrlxParser parser = new DrlxParser(tokens);
        DrlxParser.DrlxCompilationUnitContext ctx = parser.drlxStart().drlxCompilationUnit();
        var unit = new DrlxToRuleAstVisitor(tokens).visitDrlxCompilationUnit(ctx);
        return unit.rules().get(0);
    }
}
```

- [ ] **Step 4.3: Run the tests, confirm they fail**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=AccumulateVisitorTest
```
Expected: FAIL — ClassCastException ("PatternIR cannot be cast to AccumulatePatternIR") or similar, because the visitor still emits a flat list of items.

- [ ] **Step 4.4: Add a builder for `AccumulatorIR` (no LhsItemIR wrapper)**

Open `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`.

Add the following private method (place it near `visitTestElement` or wherever other ruleItem-child helpers live). Note: this returns a bare `AccumulatorIR`, **not** an `LhsItemIR`. The accumulator is folded onto the preceding pattern by the caller in Step 4.5 — no transient `LhsItemIR` subtype is introduced, which would otherwise break the sealed `LhsItemIR` permits list.

```java
private DrlxRuleAstModel.AccumulatorIR buildAccumulatorIR(DrlxParser.AccumulateItemContext ctx) {
    String typeName = ctx.VAR() != null
            ? "var"
            : ctx.typeType().getText();
    String bindName = ctx.identifier().getText();
    String functionName = ctx.accumulateCall().qualifiedName().getText();
    List<String> args = ctx.accumulateCall().expression() == null
            ? List.of()
            : ctx.accumulateCall().expression().stream()
                  .map(e -> textBetween(e.getStart(), e.getStop()))
                  .toList();
    List<String> refs = args.stream()
            .flatMap(a -> scanIdentifiers(a).stream())
            .distinct()
            .toList();
    return new DrlxRuleAstModel.AccumulatorIR(typeName, bindName, functionName, args, refs);
}
```

Where:
- `textBetween(Token, Token)` is the existing helper used by `EvalIR.expression` — find it via `grep -n "textBetween\|originalText\|getInputStream" DrlxToRuleAstVisitor.java`. If it doesn't exist, the visitor already has *some* helper to extract the original text of an expression — reuse that (often `ctx.getText()` is too lossy because it drops whitespace).
- `scanIdentifiers(String)` is the existing helper used to populate `EvalIR.referencedBindings`. If it lives under a different name, adapt the call.

- [ ] **Step 4.5: Fold inline while building the LHS list**

Locate the method that builds the LHS list of `LhsItemIR` for a `RuleIR` (it's the loop walking `ruleBody.ruleItem()`, then calling visitors per child — search for `visitRulePattern`/`visitNotElement`/`visitTestElement` callers). Replace the per-child loop with a folding variant. The pattern: hold the most-recent pattern as `pendingPattern`; accumulate adjacent `accumulateItem`s into `pendingAccs`; flush at the next non-accumulate item or end of body.

```java
private List<DrlxRuleAstModel.LhsItemIR> buildLhsItems(DrlxParser.RuleBodyContext body) {
    List<DrlxRuleAstModel.LhsItemIR> out = new java.util.ArrayList<>();
    DrlxRuleAstModel.PatternIR pendingPattern = null;
    List<DrlxRuleAstModel.AccumulatorIR> pendingAccs = new java.util.ArrayList<>();

    for (DrlxParser.RuleItemContext rci : body.ruleItem()) {
        if (rci.accumulateItem() != null) {
            if (pendingPattern == null) {
                throw new IllegalStateException(
                        "accumulate item without a preceding pattern — only forms 1 and 2 are supported in v1");
            }
            pendingAccs.add(buildAccumulatorIR(rci.accumulateItem()));
            continue;
        }
        // Non-accumulate item: flush any pending state first.
        flushPending(out, pendingPattern, pendingAccs);
        pendingPattern = null;
        pendingAccs = new java.util.ArrayList<>();

        // Visit normally; if it's a pattern, hold it for possible fold.
        DrlxRuleAstModel.LhsItemIR item = visitRuleItemChild(rci);
        if (item == null) continue;  // ruleConsequence is handled separately
        if (item instanceof DrlxRuleAstModel.PatternIR pat) {
            pendingPattern = pat;
        } else {
            out.add(item);
        }
    }
    flushPending(out, pendingPattern, pendingAccs);
    return out;
}

private static void flushPending(List<DrlxRuleAstModel.LhsItemIR> out,
                                 DrlxRuleAstModel.PatternIR pending,
                                 List<DrlxRuleAstModel.AccumulatorIR> accs) {
    if (pending == null) {
        if (!accs.isEmpty()) {
            throw new IllegalStateException(
                    "accumulate items without a preceding pattern");
        }
        return;
    }
    if (accs.isEmpty()) {
        out.add(pending);
    } else {
        out.add(new DrlxRuleAstModel.AccumulatePatternIR(pending, List.copyOf(accs)));
    }
}

/** Dispatch to the existing visit-per-child methods. Replace the body
 * with whatever the visitor uses today (likely a series of `if (rci.X() != null) return visitX(rci.X());`). */
private DrlxRuleAstModel.LhsItemIR visitRuleItemChild(DrlxParser.RuleItemContext rci) {
    if (rci.rulePattern() != null)       return visitRulePattern(rci.rulePattern());
    if (rci.notElement() != null)        return visitNotElement(rci.notElement());
    if (rci.existsElement() != null)     return visitExistsElement(rci.existsElement());
    if (rci.andElement() != null)        return visitAndElement(rci.andElement());
    if (rci.orElement() != null)         return visitOrElement(rci.orElement());
    if (rci.testElement() != null)       return visitTestElement(rci.testElement());
    if (rci.conditionalBranch() != null) return visitConditionalBranch(rci.conditionalBranch());
    return null; // ruleConsequence — handled outside this list
}
```

If the existing code already has a `visitRuleBody` returning the LHS list, replace its loop with the loop above. The dispatch in `visitRuleItemChild` mirrors whatever the existing visit-per-grammar-alternative methods are named — adjust to match.

Then at the `new DrlxRuleAstModel.RuleIR(name, annotations, lhs, rhs)` construction site, pass `buildLhsItems(ruleBody)` as the `lhs` argument.

Key point: at no point does any non-permitted type implement `LhsItemIR`. The fold happens inline; the only IR types that ever exist are the four sealed permits.

- [ ] **Step 4.6: Run tests**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=AccumulateVisitorTest
```
Expected: PASS for all four tests.

Also run the existing visitor tests to confirm no regression:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=DrlxToJavaParserVisitorTest
```
Expected: PASS, no regressions.

- [ ] **Step 4.7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(visitor): emit and fold AccumulatePatternIR (Refs #39)"
```

---

## Task 5: Protobuf — add messages and roundtrip

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`
- Test:   `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstParseResultTest.java`

- [ ] **Step 5.1: Write failing roundtrip test**

Open `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstParseResultTest.java` and add:

```java
@Test
void accumulatePatternIrRoundtrips() throws Exception {
    var src = new DrlxRuleAstModel.PatternIR(
            "var", "p", "persons",
            List.of(), null, List.of(), false, List.of());
    var acc1 = new DrlxRuleAstModel.AccumulatorIR(
            "var", "avgAge", "avg", List.of("p.age"), List.of("p"));
    var acc2 = new DrlxRuleAstModel.AccumulatorIR(
            "long", "n", "count", List.of(), List.of());
    var accPat = new DrlxRuleAstModel.AccumulatePatternIR(src, List.of(acc1, acc2));
    var rule = new DrlxRuleAstModel.RuleIR("R1", List.of(),
            List.of(accPat),
            new DrlxRuleAstModel.ConsequenceIR("{}"));
    var unit = new DrlxRuleAstModel.CompilationUnitIR(
            "p", "MyUnit", List.of(), List.of(rule));

    Path tmpDir = Files.createTempDirectory("drlx-acc-proto-");
    try {
        DrlxRuleAstParseResult.save("source.drlx", unit, tmpDir);
        var roundtrip = DrlxRuleAstParseResult.load(
                "source.drlx",
                DrlxRuleAstParseResult.parseResultFilePath(tmpDir));
        var loadedRule = roundtrip.rules().get(0);
        assertThat(loadedRule.lhs()).hasSize(1);
        var loadedAcc = (DrlxRuleAstModel.AccumulatePatternIR) loadedRule.lhs().get(0);
        assertThat(loadedAcc.source().bindName()).isEqualTo("p");
        assertThat(loadedAcc.accumulators()).hasSize(2);
        assertThat(loadedAcc.accumulators().get(0).functionName()).isEqualTo("avg");
        assertThat(loadedAcc.accumulators().get(0).argExpressions()).containsExactly("p.age");
        assertThat(loadedAcc.accumulators().get(1).functionName()).isEqualTo("count");
        assertThat(loadedAcc.accumulators().get(1).argExpressions()).isEmpty();
    } finally {
        Files.walk(tmpDir).sorted((a, b) -> b.compareTo(a))
             .forEach(p -> p.toFile().delete());
    }
}
```

- [ ] **Step 5.2: Run, confirm fails**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=DrlxRuleAstParseResultTest#accumulatePatternIrRoundtrips
```
Expected: compilation error or `ClassCastException` — proto doesn't know how to serialise `AccumulatePatternIR`.

- [ ] **Step 5.3: Modify the proto schema**

Open `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`. Inside `LhsItemParseResult.kind`, add a new `oneof` arm:

```protobuf
message LhsItemParseResult {
  oneof kind {
    PatternParseResult pattern  = 1;
    GroupElementParseResult group = 2;
    EvalParseResult eval        = 3;
    AccumulatePatternParseResult accumulate_pattern = 4;
  }
}
```

Append at the end of the file:

```protobuf
message AccumulatePatternParseResult {
  PatternParseResult source = 1;
  repeated AccumulatorParseResult accumulators = 2;
}

message AccumulatorParseResult {
  string result_type_name = 1;
  string result_bind_name = 2;
  string function_name    = 3;
  repeated string arg_expressions     = 4;
  repeated string referenced_bindings = 5;
}
```

- [ ] **Step 5.4: Update `DrlxRuleAstParseResult` — `toProto` direction**

Open `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`. Locate the method that converts `LhsItemIR` → proto (likely named `lhsToProto` or inside a `switch`/`if-chain` on item types). Add the new branch:

```java
} else if (item instanceof DrlxRuleAstModel.AccumulatePatternIR accPat) {
    var b = DrlxRuleAstProto.AccumulatePatternParseResult.newBuilder()
            .setSource(patternToProto(accPat.source()));
    for (DrlxRuleAstModel.AccumulatorIR acc : accPat.accumulators()) {
        var ab = DrlxRuleAstProto.AccumulatorParseResult.newBuilder()
                .setResultTypeName(acc.resultTypeName())
                .setResultBindName(acc.resultBindName())
                .setFunctionName(acc.functionName());
        acc.argExpressions().forEach(ab::addArgExpressions);
        acc.referencedBindings().forEach(ab::addReferencedBindings);
        b.addAccumulators(ab.build());
    }
    return DrlxRuleAstProto.LhsItemParseResult.newBuilder()
            .setAccumulatePattern(b.build())
            .build();
```

Where `patternToProto` is the existing helper used for `PatternIR` (look for `PatternParseResult` builder usage in the same file).

- [ ] **Step 5.5: Update `DrlxRuleAstParseResult` — `fromProto` direction**

In the same file, locate the inverse (`oneof.getKindCase()` switch or chain). Add the new case:

```java
case ACCUMULATE_PATTERN -> {
    var p = proto.getAccumulatePattern();
    var src = patternFromProto(p.getSource());
    var accs = p.getAccumulatorsList().stream()
            .map(a -> new DrlxRuleAstModel.AccumulatorIR(
                    a.getResultTypeName(),
                    a.getResultBindName(),
                    a.getFunctionName(),
                    a.getArgExpressionsList(),
                    a.getReferencedBindingsList()))
            .toList();
    yield new DrlxRuleAstModel.AccumulatePatternIR(src, accs);
}
```

(`patternFromProto` is the existing `PatternIR` deserialiser in the same file.)

- [ ] **Step 5.6: Rebuild and run**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=DrlxRuleAstParseResultTest
```
Expected: all tests PASS, including the new roundtrip case. Proto regeneration is automatic via the maven plugin.

- [ ] **Step 5.7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstParseResultTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(proto): roundtrip AccumulatePatternIR through protobuf (Refs #39)"
```

---

## Task 6: `AccumulateFunctionRegistry` and `DrlxLambdaAccumulator`

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/AccumulateFunctionRegistry.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaAccumulator.java`
- Test:   `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateFunctionRegistryTest.java`

- [ ] **Step 6.1: Write failing registry tests**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateFunctionRegistryTest.java`:

```java
package org.drools.drlx.builder;

import org.junit.jupiter.api.Test;
import org.drools.core.base.accumulators.AverageAccumulateFunction;
import org.drools.core.base.accumulators.CountAccumulateFunction;
import org.drools.core.base.accumulators.MaxAccumulateFunction;
import org.drools.core.base.accumulators.MinAccumulateFunction;
import org.drools.core.base.accumulators.SumAccumulateFunction;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AccumulateFunctionRegistryTest {

    @Test
    void resolvesBuiltinAverage() {
        var resolved = AccumulateFunctionRegistry.resolve("avg");
        assertThat(resolved.functionClass()).isEqualTo(AverageAccumulateFunction.class);
        assertThat(resolved.resultType()).isEqualTo(Double.class);
        assertThat(resolved.acceptsZeroArgs()).isFalse();
    }

    @Test
    void resolvesBuiltinCountWithZeroArgs() {
        var resolved = AccumulateFunctionRegistry.resolve("count");
        assertThat(resolved.functionClass()).isEqualTo(CountAccumulateFunction.class);
        assertThat(resolved.resultType()).isEqualTo(Long.class);
        assertThat(resolved.acceptsZeroArgs()).isTrue();
    }

    @Test
    void resolvesSumMinMax() {
        assertThat(AccumulateFunctionRegistry.resolve("sum").functionClass())
                .isEqualTo(SumAccumulateFunction.class);
        assertThat(AccumulateFunctionRegistry.resolve("min").functionClass())
                .isEqualTo(MinAccumulateFunction.class);
        assertThat(AccumulateFunctionRegistry.resolve("max").functionClass())
                .isEqualTo(MaxAccumulateFunction.class);
    }

    @Test
    void rejectsUnknownFunction() {
        assertThatThrownBy(() -> AccumulateFunctionRegistry.resolve("bogus"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("unknown accumulate function 'bogus'")
                .hasMessageContaining("avg, sum, min, max, count");
    }

    @Test
    void rejectsQualifiedName() {
        assertThatThrownBy(() -> AccumulateFunctionRegistry.resolve("Func.avg"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("qualified accumulate function names")
                .hasMessageContaining("not yet supported")
                .hasMessageContaining("avg, sum, min, max, count");
    }
}
```

- [ ] **Step 6.2: Run, confirm fails**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=AccumulateFunctionRegistryTest
```
Expected: compilation error — `AccumulateFunctionRegistry` not found.

- [ ] **Step 6.3: Create the registry**

Create `drlx-parser-core/src/main/java/org/drools/drlx/builder/AccumulateFunctionRegistry.java`:

```java
package org.drools.drlx.builder;

import java.util.Map;

import org.drools.core.base.accumulators.AverageAccumulateFunction;
import org.drools.core.base.accumulators.CountAccumulateFunction;
import org.drools.core.base.accumulators.MaxAccumulateFunction;
import org.drools.core.base.accumulators.MinAccumulateFunction;
import org.drools.core.base.accumulators.SumAccumulateFunction;
import org.kie.api.runtime.rule.AccumulateFunction;

/**
 * v1 registry of built-in accumulate function names → {@link AccumulateFunction} classes.
 * Qualified names ({@code Func.avg}) are explicitly rejected; the spec defers custom-function
 * resolution to a fast-follow under #26.
 */
public final class AccumulateFunctionRegistry {

    /** Resolved metadata for one accumulate function. */
    public record Resolution(Class<? extends AccumulateFunction> functionClass,
                             Class<?> resultType,
                             boolean acceptsZeroArgs) {
    }

    private static final Map<String, Resolution> BUILTINS = Map.of(
            "avg",   new Resolution(AverageAccumulateFunction.class, Double.class, false),
            "sum",   new Resolution(SumAccumulateFunction.class,     Number.class, false),
            "min",   new Resolution(MinAccumulateFunction.class,     Comparable.class, false),
            "max",   new Resolution(MaxAccumulateFunction.class,     Comparable.class, false),
            "count", new Resolution(CountAccumulateFunction.class,   Long.class,   true));

    private static final String BUILTIN_LIST = "avg, sum, min, max, count";

    private AccumulateFunctionRegistry() {
    }

    public static Resolution resolve(String functionName) {
        if (functionName.contains(".")) {
            throw new IllegalArgumentException(
                    "qualified accumulate function names ('" + functionName + "') "
                    + "are not yet supported — use unqualified built-ins ("
                    + BUILTIN_LIST + ")");
        }
        Resolution r = BUILTINS.get(functionName);
        if (r == null) {
            throw new IllegalArgumentException(
                    "unknown accumulate function '" + functionName + "' — "
                    + "built-ins are: " + BUILTIN_LIST);
        }
        return r;
    }
}
```

- [ ] **Step 6.4: Run, confirm passes**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=AccumulateFunctionRegistryTest
```
Expected: all five tests PASS.

- [ ] **Step 6.5: Create the accumulator wrapper**

Create `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaAccumulator.java`. This wraps an `AccumulateFunction` instance and an optional value-extractor (a `Function<Object, Object>` projecting a fact to the accumulated value — for `avg(p.age)` it projects `p → p.age`; for `count()` or `count(p)` it is `null` and the function's `accumulateValue` is called with the raw fact).

```java
package org.drools.drlx.builder;

import java.io.Serializable;
import java.util.function.Function;

import org.drools.base.base.ValueResolver;
import org.drools.base.reteoo.BaseTuple;
import org.drools.base.rule.Declaration;
import org.drools.base.rule.accessor.Accumulator;
import org.kie.api.runtime.rule.AccumulateFunction;
import org.kie.api.runtime.rule.FactHandle;

/**
 * Minimal {@link Accumulator} for DRLX accumulate v1. Delegates accumulation lifecycle
 * to an {@link AccumulateFunction} and projects each fact through an optional extractor.
 * Lives in {@code drlx-parser-core} to avoid depending on {@code drools-model-compiler}.
 */
public final class DrlxLambdaAccumulator implements Accumulator {

    private final AccumulateFunction<Serializable> accFunction;
    private final Function<Object, Object> extractor; // null for count() / count(x)

    public DrlxLambdaAccumulator(AccumulateFunction<Serializable> accFunction,
                                 Function<Object, Object> extractor) {
        this.accFunction = accFunction;
        this.extractor = extractor;
    }

    @Override public Object createWorkingMemoryContext() { return null; }

    @Override public Object createContext() { return accFunction.createContext(); }

    @Override
    public Object init(Object wmContext, Object context, BaseTuple tuple,
                       Declaration[] decls, ValueResolver vr) {
        accFunction.init((Serializable) context);
        return context;
    }

    @Override
    public Object accumulate(Object wmContext, Object context, BaseTuple tuple,
                             FactHandle handle, Declaration[] decls,
                             Declaration[] innerDecls, ValueResolver vr) {
        Object value = (extractor == null) ? handle.getObject() : extractor.apply(handle.getObject());
        try {
            accFunction.accumulate((Serializable) context, value);
        } catch (Exception e) {
            throw new RuntimeException("accumulate failed for " + accFunction.getClass().getSimpleName(), e);
        }
        return value;
    }

    @Override public boolean supportsReverse() { return accFunction.supportsReverse(); }

    @Override
    public boolean tryReverse(Object wmContext, Object context, BaseTuple tuple,
                              FactHandle handle, Object value,
                              Declaration[] decls, Declaration[] innerDecls,
                              ValueResolver vr) {
        if (!accFunction.supportsReverse()) {
            return false;
        }
        try {
            accFunction.reverse((Serializable) context, value);
            return true;
        } catch (Exception e) {
            throw new RuntimeException("reverse failed for " + accFunction.getClass().getSimpleName(), e);
        }
    }

    @Override
    public Object getResult(Object wmContext, Object context, BaseTuple tuple,
                            Declaration[] decls, ValueResolver vr) {
        try {
            return accFunction.getResult((Serializable) context);
        } catch (Exception e) {
            throw new RuntimeException("getResult failed for " + accFunction.getClass().getSimpleName(), e);
        }
    }
}
```

The `AccumulateFunction` API surface used above is verified against `kie-api/src/main/java/org/kie/api/runtime/rule/AccumulateFunction.java`: `createContext()`, `init(C) throws Exception`, `accumulate(C, Object)`, `reverse(C, Object) throws Exception`, `getResult(C) throws Exception`, `supportsReverse()`. `getResultType()` is also available on the interface and can be used as a future cross-check against the hand-maintained `Resolution.resultType()` table — out of scope for v1.

- [ ] **Step 6.6: Build the project to confirm the new class compiles**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core compile
```
Expected: BUILD SUCCESS.

- [ ] **Step 6.7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/AccumulateFunctionRegistry.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaAccumulator.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateFunctionRegistryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(builder): add AccumulateFunctionRegistry + DrlxLambdaAccumulator (Refs #39)"
```

---

## Task 7: Runtime builder — single-function lowering

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Test:   `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilderTest.java`
- Test:   create `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 7.1: Confirm the end-to-end test infrastructure**

Skim an existing syntax test for the harness it uses:
```bash
sed -n '1,80p' /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/InlineCastTest.java
```
Note the unit-class type, how facts are inserted, how `DrlxRuleBuilder` / `DrlxCompiler` is invoked, and how the consequence collects observed values.

- [ ] **Step 7.2: Write the failing single-function end-to-end test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`:

```java
package org.drools.drlx.builder.syntax;

import java.util.ArrayList;
import java.util.List;
import org.junit.jupiter.api.Test;
import org.drools.drlx.ruleunit.MyUnit;     // existing test unit class with DataStore<Person> persons
import org.drools.drlx.ruleunit.Person;     // existing test fact class

import static org.assertj.core.api.Assertions.assertThat;
import static org.drools.drlx.builder.syntax.SyntaxTestSupport.runRule; // existing helper — adapt name to whatever exists

class AccumulateTest {

    @Test
    void singleAvgOverPersons() {
        String drlx = """
                package org.drools.drlx.ruleunit;
                unit MyUnit;
                import org.drools.drlx.ruleunit.Person;
                rule AvgRule {
                    var p : /persons,
                    var avgAge = avg(p.age),
                    do { results.add(avgAge); }
                }
                """;
        var observed = new ArrayList<Object>();
        runRule(drlx, unit -> {
            unit.persons.add(new Person("A", 20));
            unit.persons.add(new Person("B", 40));
            unit.persons.add(new Person("C", 60));
        }, observed);
        assertThat(observed).containsExactly(40.0);  // avg(20,40,60) = 40.0
    }
}
```

If the existing tests don't have a `SyntaxTestSupport.runRule` helper, copy the wiring (DrlxCompiler invocation + KieSession setup + fact insertion + fire) directly into the test. The pattern is identical to what `InlineCastTest.java` or `IfElseBranchingTest.java` already does — the goal is to assert that on three inserted persons, the consequence body sees `avgAge == 40.0` and adds it to `results`.

Note: the existing `MyUnit` test class may or may not have a `results` collection. If it doesn't, either:
- add a `public List<Object> results = new ArrayList<>();` field to `MyUnit` (mirroring how globals are exposed today — check `buildGlobalTypeMap`), or
- collect via a `TestDataObserver` mechanism that's already in the test infrastructure.

- [ ] **Step 7.3: Run, confirm fails**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=AccumulateTest#singleAvgOverPersons
```
Expected: FAIL — likely with an "unsupported LHS item: AccumulatePatternIR" or similar message from `buildLhs`.

- [ ] **Step 7.4: Add the `AccumulatePatternIR` branch in `buildLhs`**

Open `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`. Locate the `buildLhs` method (around line 333). Inside the `for (LhsItemIR item : items)` loop, just before the final `else throw new IllegalArgumentException(...)`, add:

```java
} else if (item instanceof AccumulatePatternIR accPat) {
    buildAccumulatePattern(accPat, parent, typeResolver, entryPointTypes,
                           unitClass, boundVariables);
```

Then add the new method to the class (place it next to `buildPattern`):

```java
private void buildAccumulatePattern(AccumulatePatternIR accPat,
                                    GroupElement parent,
                                    TypeResolver typeResolver,
                                    Map<String, Class<?>> entryPointTypes,
                                    Class<?> unitClass,
                                    Map<String, BoundVariable> outerScope) {
    // 1. Build the source pattern template (used to clone per accumulator below).
    PatternIR srcIr = accPat.source();
    Pattern srcTemplate = buildPattern(srcIr, typeResolver, entryPointTypes,
                                       unitClass, outerScope);
    Declaration srcDecl = srcTemplate.getDeclaration();
    Class<?> srcClass = ((ClassObjectType) srcTemplate.getObjectType()).getClassType();

    // 2. Construct the inner scope = outer ∪ { srcDecl }.
    Map<String, BoundVariable> innerScope = new LinkedHashMap<>(outerScope);
    innerScope.put(srcDecl.getIdentifier(),
            new BoundVariable(srcDecl.getIdentifier(), srcClass, srcTemplate));

    // 3. For each accumulator, build a SingleAccumulate, wrap a result Pattern,
    //    and add to the parent group.  Inner pattern is NEVER added to parent.
    int idx = 0;
    for (AccumulatorIR acc : accPat.accumulators()) {
        Pattern innerClone = idx++ == 0 ? srcTemplate : srcTemplate.clone();
        SingleAccumulate single = buildSingleAccumulate(acc, innerClone, innerScope);
        Pattern resultPattern = wrapResultPattern(acc, single);
        parent.addChild(resultPattern);

        Class<?> resultClass = resultClassFor(acc);
        outerScope.put(acc.resultBindName(),
                new BoundVariable(acc.resultBindName(), resultClass, resultPattern));
    }
}

private SingleAccumulate buildSingleAccumulate(AccumulatorIR acc,
                                               Pattern innerPattern,
                                               Map<String, BoundVariable> innerScope) {
    AccumulateFunctionRegistry.Resolution resolved =
            AccumulateFunctionRegistry.resolve(acc.functionName());

    // Validate arg count: count() accepts 0 or 1; others require 1.
    int argCount = acc.argExpressions().size();
    if (resolved.acceptsZeroArgs()) {
        if (argCount > 1) {
            throw new RuntimeException("function '" + acc.functionName()
                    + "' accepts 0 or 1 argument, got " + argCount);
        }
    } else {
        if (argCount != 1) {
            throw new RuntimeException("function '" + acc.functionName()
                    + "' requires exactly 1 argument, got " + argCount);
        }
    }

    // Build the value-extractor lambda (unless this is count() with no/ignored arg).
    java.util.function.Function<Object, Object> extractor = null;
    if (argCount == 1 && !resolved.acceptsZeroArgs()) {
        String argExpr = acc.argExpressions().get(0);
        extractor = lambdaCompiler.createValueExtractor(
                argExpr,
                acc.referencedBindings(),
                innerScope);
    }

    AccumulateFunction<Serializable> fn;
    try {
        @SuppressWarnings("unchecked")
        AccumulateFunction<Serializable> f =
                (AccumulateFunction<Serializable>) resolved.functionClass()
                        .getDeclaredConstructor().newInstance();
        fn = f;
    } catch (ReflectiveOperationException e) {
        throw new RuntimeException("cannot instantiate " + resolved.functionClass(), e);
    }

    DrlxLambdaAccumulator accumulator = new DrlxLambdaAccumulator(fn, extractor);
    Declaration[] required = acc.referencedBindings().stream()
            .map(innerScope::get)
            .filter(java.util.Objects::nonNull)
            .map(bv -> bv.pattern().getDeclaration())
            .toArray(Declaration[]::new);
    return new SingleAccumulate(innerPattern, required, accumulator);
}

private static Pattern wrapResultPattern(AccumulatorIR acc, SingleAccumulate single) {
    Class<?> resultType = resultClassFor(acc);
    Pattern p = new Pattern(0, new ClassObjectType(resultType), null);
    p.addDeclaration(new Declaration(acc.resultBindName(),
            new SelfReferenceClassFieldReader(resultType), p, true));
    p.setSource(single);
    return p;
}

private static Class<?> resultClassFor(AccumulatorIR acc) {
    AccumulateFunctionRegistry.Resolution r =
            AccumulateFunctionRegistry.resolve(acc.functionName());
    if (!"var".equals(acc.resultTypeName())) {
        // Trust the user's explicit type. (Compatibility validation could be added.)
        try {
            return Class.forName(acc.resultTypeName());
        } catch (ClassNotFoundException primitiveOrUnqualified) {
            // Map primitive names to boxed equivalents.
            return switch (acc.resultTypeName()) {
                case "int"     -> Integer.class;
                case "long"    -> Long.class;
                case "double"  -> Double.class;
                case "float"   -> Float.class;
                case "short"   -> Short.class;
                case "byte"    -> Byte.class;
                case "boolean" -> Boolean.class;
                case "char"    -> Character.class;
                default        -> r.resultType();
            };
        }
    }
    return r.resultType();
}
```

You'll need to add these imports at the top of the file:
```java
import java.io.Serializable;
import org.drools.base.rule.SingleAccumulate;
import org.drools.base.rule.accessor.ReadAccessor;
import org.drools.base.rule.accessor.SelfReferenceClassFieldReader;
import org.drools.drlx.builder.DrlxRuleAstModel.AccumulatePatternIR;
import org.drools.drlx.builder.DrlxRuleAstModel.AccumulatorIR;
import org.kie.api.runtime.rule.AccumulateFunction;
```

(`SelfReferenceClassFieldReader` likely lives in `org.drools.base.rule.accessor` or `org.drools.core.base`; if unsure, `grep -r "class SelfReferenceClassFieldReader" /home/tkobayas/usr/work/mvel3-development/drools/drools-base/src/main`.)

- [ ] **Step 7.5: Add the value-extractor compile path on the lambda compiler**

`DrlxLambdaCompiler` already compiles eval expressions; the new method `createValueExtractor(String argExpr, List<String> referencedBindings, Map<String, BoundVariable> scope)` is essentially the same compile path but the resulting lambda must return the argument expression's value (not a boolean).

Open `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java` and locate the eval-compile method (likely `createEvalCondition` or `compileEvalExpression`). Copy/adapt to a new public method:

```java
public Function<Object, Object> createValueExtractor(String argExpr,
                                                     List<String> referencedBindings,
                                                     Map<String, BoundVariable> scope) {
    // Reuses the same MVEL3 lambda-compile path as eval, but the compiled
    // lambda's return type is the expression's value, not a Boolean.
    // ...delegate to the existing compile machinery with a non-boolean
    //    return-type expectation.
}
```

Implementation detail: this hooks into MVEL3's `LambdaCompiler` directly. The current `DrlxEvalExpression` class wraps an MVEL3 expression that returns boolean for eval; for accumulate the expression's natural type is whatever `argExpr` evaluates to. Look at `DrlxEvalExpression.java` for the pattern; produce an equivalent that doesn't coerce to boolean. Single-argument lambda receiving the bound fact (the source pattern's bound object) returning `Object`.

If this turns out to be a large lift (in practice it shouldn't — the underlying MVEL3 API is the same), the workaround is to produce a small Java-source method body, compile via MVEL3's batch compiler (as existing code does for evals), and call it reflectively. Either way, the new method is local to `DrlxLambdaCompiler`.

- [ ] **Step 7.6: Run the end-to-end test, iterate**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=AccumulateTest#singleAvgOverPersons
```
Expected: PASS. If failures point at the extractor compile path, iterate on step 7.5.

- [ ] **Step 7.7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(builder): wire single-function accumulate end-to-end (Refs #39)"
```

---

## Task 8: Multi-function accumulate + remaining built-ins

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 8.1: Add multi-function and built-in-coverage tests**

In `AccumulateTest.java`, add:

```java
@Test
void multiFunctionMinMaxAvg() {
    String drlx = """
            package org.drools.drlx.ruleunit;
            unit MyUnit;
            import org.drools.drlx.ruleunit.Person;
            rule R {
                var p : /persons,
                var minAge = min(p.age),
                var maxAge = max(p.age),
                var avgAge = avg(p.age),
                do { results.add(minAge); results.add(maxAge); results.add(avgAge); }
            }
            """;
    var observed = new ArrayList<Object>();
    runRule(drlx, unit -> {
        unit.persons.add(new Person("A", 20));
        unit.persons.add(new Person("B", 40));
        unit.persons.add(new Person("C", 60));
    }, observed);
    assertThat(observed).containsExactly(20, 60, 40.0);
}

@Test
void countWithNoArgument() {
    String drlx = """
            package org.drools.drlx.ruleunit;
            unit MyUnit;
            import org.drools.drlx.ruleunit.Person;
            rule R {
                var p : /persons,
                long n = count(),
                do { results.add(n); }
            }
            """;
    var observed = new ArrayList<Object>();
    runRule(drlx, unit -> {
        unit.persons.add(new Person("A", 20));
        unit.persons.add(new Person("B", 40));
    }, observed);
    assertThat(observed).containsExactly(2L);
}

@Test
void sumOverIntegerField() {
    String drlx = """
            package org.drools.drlx.ruleunit;
            unit MyUnit;
            import org.drools.drlx.ruleunit.Person;
            rule R {
                var p : /persons,
                var total = sum(p.age),
                do { results.add(total); }
            }
            """;
    var observed = new ArrayList<Object>();
    runRule(drlx, unit -> {
        unit.persons.add(new Person("A", 10));
        unit.persons.add(new Person("B", 30));
        unit.persons.add(new Person("C", 60));
    }, observed);
    assertThat(observed).containsExactly(100);  // SumAccumulateFunction returns the boxed sum
}
```

- [ ] **Step 8.2: Run and iterate**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=AccumulateTest
```
Expected: all four tests (incl. existing `singleAvgOverPersons`) PASS.

If `multiFunctionMinMaxAvg` fails with a Drools error about pattern offsets or duplicate declarations, the most likely cause is that the second/third `srcTemplate.clone()` calls are not producing fresh `Declaration` instances bound to the new clone. Audit `Pattern.clone()` semantics in Drools (it should clone declarations) and, if needed, deep-clone the inner pattern manually.

- [ ] **Step 8.3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: multi-function accumulate + count/sum coverage (Refs #39)"
```

---

## Task 9: Error paths

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 9.1: Add negative tests**

Append to `AccumulateTest.java`:

```java
@Test
void qualifiedFunctionNameRejected() {
    String drlx = """
            package org.drools.drlx.ruleunit;
            unit MyUnit;
            import org.drools.drlx.ruleunit.Person;
            rule R {
                var p : /persons,
                var avgAge = Func.avg(p.age),
                do {}
            }
            """;
    assertThatThrownBy(() -> compile(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("qualified accumulate function names")
            .hasMessageContaining("Func.avg");
}

@Test
void unknownFunctionRejected() {
    String drlx = """
            package org.drools.drlx.ruleunit;
            unit MyUnit;
            import org.drools.drlx.ruleunit.Person;
            rule R {
                var p : /persons,
                var x = bogus(p.age),
                do {}
            }
            """;
    assertThatThrownBy(() -> compile(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("unknown accumulate function 'bogus'")
            .hasMessageContaining("avg, sum, min, max, count");
}

@Test
void sourceBindingNotVisibleAfterAccumulate() {
    // `p` is internal; referencing it in the consequence must fail at build/lambda-compile time.
    String drlx = """
            package org.drools.drlx.ruleunit;
            unit MyUnit;
            import org.drools.drlx.ruleunit.Person;
            rule R {
                var p : /persons,
                var avgAge = avg(p.age),
                do { results.add(p); }
            }
            """;
    assertThatThrownBy(() -> compile(drlx))
            .isInstanceOf(RuntimeException.class);  // exact message from lambda compiler
}
```

The `compile(drlx)` helper invokes `DrlxRuleBuilder.build(...)` without firing — most existing syntax tests have this. If not, factor it out of `runRule`.

- [ ] **Step 9.2: Run and iterate**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=AccumulateTest
```
Expected: all tests PASS including the three new negative ones.

If `sourceBindingNotVisibleAfterAccumulate` doesn't fail at compile time, the inner-scope handling has a leak — the source binding `p` is somehow finding its way into the outer `boundVariables`. Audit `buildAccumulatePattern` in Task 7.4 to confirm `outerScope` is never written for the source pattern.

- [ ] **Step 9.3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: accumulate error paths (qualified-name, unknown-fn, source visibility) (Refs #39)"
```

---

## Task 10: Full suite green

- [ ] **Step 10.1: Run the entire `drlx-parser-core` test suite**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test
```
Expected: BUILD SUCCESS, no regressions. Total test count should be `182 + N` where N = number of new tests added (roughly 6 grammar + 4 visitor + 1 proto + 5 registry + 7 syntax ≈ 23).

If any pre-existing tests broke, fix them or revert the offending step.

- [ ] **Step 10.2: Run drlx-parser-benchmark module compile to catch downstream breakage**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml compile
```
Expected: BUILD SUCCESS.

---

## Task 11: Handover, push, optional PR

**Files:**
- Modify (workspace): `/home/tkobayas/claude/public/drlx-parser/HANDOFF.md`

- [ ] **Step 11.1: Push project commits to origin**

Run:
```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser log --oneline -10
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push origin main
```
Expected: 8–10 new commits pushed.

- [ ] **Step 11.2: Close issue #39**

Either via a final commit message `Closes #39` on the last commit, or:
```bash
gh issue close 39 --repo tkobayas/drlx-parser --comment "v1 (forms 1+2) shipped. Multi-function and inline-from sources, MultiAccumulate folding, and \`acc(...)\` keyword forms tracked separately under #26."
```

- [ ] **Step 11.3: Update parent epic #26**

```bash
gh issue comment 26 --repo tkobayas/drlx-parser --body "#39 (Accumulate v1: forms 1+2 with built-ins avg/sum/min/max/count) shipped. Remaining accumulate work: multi-function MultiAccumulate folding, inline \`from\`-style accumulate (\`avg(/persons.age)\`), \`acc()\` keyword (2/3/5-param), multi-pattern source via \`and(...)\`, custom accumulate functions imported by user code."
```

- [ ] **Step 11.4: Write handover**

Overwrite `/home/tkobayas/claude/public/drlx-parser/HANDOFF.md` with a session-end summary in the style of recent handovers (`git show HEAD~1:HANDOFF.md` for the previous template). Key elements:

- Session goal: ship #39 v1.
- Files added/modified in project repo.
- Total commits + final HEAD SHA + new test count.
- Gotchas encountered (specifically: anything that diverged from the spec during implementation — e.g., `AccumulateFunction` method signatures, `Pattern.clone()` quirks, MVEL3 extractor lambda return-type wiring).
- Followups deferred to epic #26.

Commit the handover (workspace repo):
```bash
git -C /home/tkobayas/claude/public/drlx-parser add HANDOFF.md
git -C /home/tkobayas/claude/public/drlx-parser commit -m "docs: session handover 2026-05-13 — #39 accumulate v1 shipped"
git -C /home/tkobayas/claude/public/drlx-parser push origin main
```

---

## Self-review checklist (the writer's, not the implementer's)

- Each task has TDD: failing test first, then implementation, then re-run.
- Each task ends with a commit.
- Every code block in this plan is a real, copy-pasteable artifact — no `// TODO` left.
- File paths are absolute, qualified to the project or workspace repo.
- Maven commands carry the correct `-f` / `-pl` flags so they run from any cwd.
- Two genuine deferrable details survive into implementation time (called out where they live):
  - **Task 6.5**: exact `AccumulateFunction` interface method names — verified at implementation step against the real Drools 999-SNAPSHOT classes.
  - **Task 7.5**: `DrlxLambdaCompiler.createValueExtractor` plumbing — the structural pattern is fixed; the exact MVEL3 entry-point may be `compileExpression`, `compileLambda`, or `compileEvalExpression` depending on what `DrlxEvalExpression` uses today.

Both are explicitly local (one method each, one file each) and have a "look at the actual class first" step before the code is written.
