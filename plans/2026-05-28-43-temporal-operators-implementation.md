# #43 Temporal Operators Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add temporal CEP operators (`after`, `before`, `coincides`, etc.) to DRLX pattern constraints so rules can correlate events by timing.

**Architecture:** Grammar adds a generic `customConstraint` rule to `DrlxParser.g4`. The visitor extracts temporal conditions into a new `TemporalConditionIR` list in `PatternIR`. At build time, `DrlxRuleAstRuntimeBuilder` creates `DrlxTemporalConstraint` instances that delegate to drools `TemporalPredicate` implementations for timestamp evaluation. The constraint implements `IntervalProviderConstraint` so the RETE network handles event expiration correctly.

**Tech Stack:** ANTLR4, drools-base (`IntervalProviderConstraint`, `Interval`), drools-canonical-model (`TemporalPredicate`, `AfterPredicate`, etc.), kie-api (`EventHandle`)

---

## File Map

**Modified:**
- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` — add `customConstraint`, `customOp`, `customOpParams` rules
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` — add `TemporalConditionIR` record, extend `PatternIR`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` — handle `customConstraint` in condition extraction
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` — process temporal conditions
- `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` — add `TemporalConditionParseResult` message, extend `PatternParseResult`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` — temporal condition proto serialization

**Created:**
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxTemporalConstraint.java`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/TemporalPredicateFactory.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/TemporalVisitorTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/TemporalPredicateFactoryTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TemporalOperatorTest.java`

**Existing PatternIR call sites requiring `List.of()` for new `temporalConditions` parameter (9 sites):**
- `DrlxToRuleAstVisitor.java:775,786,797`
- `DrlxRuleAstParseResult.java:163`
- `DrlxRuleAstModelTest.java:44,57`
- `DrlxRuleAstParseResultTest.java:49,88,156`
- `NotTest.java:95`
- `EvalIRBuilderTest.java:42`

---

### Task 1: Grammar — add customConstraint rules

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:221-226`

- [ ] **Step 1: Add the grammar rules**

In `DrlxParser.g4`, replace the `drlxExpression` rule (lines 222-226) with:

```antlr
// DRLX expression used inside oopathChunk conditions
// Allows optional binding (name: expression), custom constraint (temporal/pluggable),
// or a plain expression
drlxExpression
    : identifier ':' expression
    | customConstraint
    | expression
    ;

// Custom constraint — generic rule for temporal/pluggable operators.
// The operator identifier is validated at visitor level (avoids lexer keywords).
// Grammar: (THIS | identifier) NOT? operatorName[params]? expression
customConstraint
    : (THIS | identifier) NOT? customOp expression
    ;

customOp
    : identifier ('[' customOpParams ']')?
    ;

customOpParams
    : ~']'+
    ;
```

- [ ] **Step 2: Regenerate the parser and verify it compiles**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources -q
```
Expected: BUILD SUCCESS, generated `DrlxParser.java` contains `customConstraint`, `customOp`, `customOpParams` methods.

- [ ] **Step 3: Commit**

```
feat(grammar): add customConstraint rule for temporal/pluggable operators

Refs #43
```

---

### Task 2: IR — add TemporalConditionIR and extend PatternIR

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`
- Modify: all 9 existing `PatternIR` constructor call sites (see File Map above)

- [ ] **Step 1: Write failing test for the new IR record**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/TemporalVisitorTest.java`:

```java
package org.drools.drlx.builder;

import java.util.List;

import org.antlr.v4.runtime.CharStreams;
import org.antlr.v4.runtime.CommonTokenStream;
import org.drools.drlx.builder.DrlxRuleAstModel.PatternIR;
import org.drools.drlx.builder.DrlxRuleAstModel.TemporalConditionIR;
import org.drools.drlx.parser.DrlxLexer;
import org.drools.drlx.parser.DrlxParser;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TemporalVisitorTest {

    @Test
    void parsesThisAfterBinding() {
        var rule = parseRule("""
                package p;
                unit MyUnit;
                rule R1 {
                    var a : /as,
                    var b : /bs[this after a],
                    do {}
                }
                """);
        var pattern = (PatternIR) rule.lhs().get(1);
        assertThat(pattern.temporalConditions()).hasSize(1);
        TemporalConditionIR tc = pattern.temporalConditions().get(0);
        assertThat(tc.operator()).isEqualTo("after");
        assertThat(tc.negated()).isFalse();
        assertThat(tc.parameters()).isEmpty();
        assertThat(tc.rightBinding()).isEqualTo("a");
        assertThat(pattern.conditions()).isEmpty();
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

- [ ] **Step 2: Run the test to verify it fails**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TemporalVisitorTest#parsesThisAfterBinding -pl . -q
```
Expected: FAIL — `TemporalConditionIR` does not exist yet.

- [ ] **Step 3: Add TemporalConditionIR record and extend PatternIR**

In `DrlxRuleAstModel.java`, add the new record after `PatternIR` (around line 48):

```java
public record TemporalConditionIR(
    String operator,
    boolean negated,
    List<String> parameters,
    String rightBinding
) {
    public TemporalConditionIR {
        parameters = List.copyOf(parameters);
    }
}
```

Extend `PatternIR` to add `temporalConditions` between `conditions` and `castTypeName`:

```java
public record PatternIR(String typeName,
                        String bindName,
                        String entryPoint,
                        List<String> conditions,
                        List<TemporalConditionIR> temporalConditions,
                        String castTypeName,
                        List<String> positionalArgs,
                        boolean passive,
                        List<String> watchedProperties,
                        String windowType,
                        String windowParameter) implements LhsItemIR {
}
```

- [ ] **Step 4: Fix all existing PatternIR constructor call sites**

Add `List.of()` for the new `temporalConditions` parameter at each site:

**`DrlxToRuleAstVisitor.java:775`** — `buildPatternFromBoundOopath`:
```java
return new PatternIR(typeName, bindName, entryPoint, conditions, List.of(), castTypeName,
                      positionalArgs, passive, watchedProperties, windowType, windowParameter);
```

**`DrlxToRuleAstVisitor.java:786`** — `buildPatternFromOopath` (no bind):
```java
return new PatternIR("", "", entryPoint, conditions, List.of(), castTypeName, positionalArgs, passive, watchedProperties, null, null);
```

**`DrlxToRuleAstVisitor.java:797`** — `buildPatternFromOopath` (synthetic bind):
```java
return new PatternIR("", syntheticBindName, entryPoint, conditions, List.of(), castTypeName,
                      positionalArgs, passive, watchedProperties, null, null);
```

**`DrlxRuleAstParseResult.java:163`** — `patternFromProto`:
```java
return new PatternIR(
        pattern.getTypeName(),
        pattern.getBindName(),
        pattern.getEntryPoint(),
        List.copyOf(pattern.getConditionsList()),
        List.of(),  // temporal conditions — populated from proto in Task 7
        castTypeName,
        ...);
```

**Test files** — `DrlxRuleAstModelTest.java`, `DrlxRuleAstParseResultTest.java`, `NotTest.java`, `EvalIRBuilderTest.java`: add `List.of()` after the `conditions` argument in every `new PatternIR(...)` call.

- [ ] **Step 5: Verify compilation**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile test-compile -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```
feat(ir): add TemporalConditionIR and extend PatternIR

Refs #43
```

---

### Task 3: Visitor — extract temporal conditions from customConstraint

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:755-777,858-873`

- [ ] **Step 1: Add more visitor tests**

Append to `TemporalVisitorTest.java`:

```java
@Test
void parsesAfterWithParams() {
    var rule = parseRule("""
            package p;
            unit MyUnit;
            rule R1 {
                var a : /as,
                var b : /bs[this after[3m, 4m] a],
                do {}
            }
            """);
    var pattern = (PatternIR) rule.lhs().get(1);
    TemporalConditionIR tc = pattern.temporalConditions().get(0);
    assertThat(tc.operator()).isEqualTo("after");
    assertThat(tc.parameters()).containsExactly("3m", "4m");
    assertThat(tc.rightBinding()).isEqualTo("a");
}

@Test
void parsesNegatedBefore() {
    var rule = parseRule("""
            package p;
            unit MyUnit;
            rule R1 {
                var a : /as,
                var b : /bs[this not before a],
                do {}
            }
            """);
    var pattern = (PatternIR) rule.lhs().get(1);
    TemporalConditionIR tc = pattern.temporalConditions().get(0);
    assertThat(tc.operator()).isEqualTo("before");
    assertThat(tc.negated()).isTrue();
    assertThat(tc.rightBinding()).isEqualTo("a");
}

@Test
void mixedTemporalAndRegularConditions() {
    var rule = parseRule("""
            package p;
            unit MyUnit;
            rule R1 {
                var a : /as,
                var b : /bs[this after a, amount > 100],
                do {}
            }
            """);
    var pattern = (PatternIR) rule.lhs().get(1);
    assertThat(pattern.temporalConditions()).hasSize(1);
    assertThat(pattern.temporalConditions().get(0).operator()).isEqualTo("after");
    assertThat(pattern.conditions()).containsExactly("amount > 100");
}

@Test
void patternWithoutTemporalHasEmptyList() {
    var rule = parseRule("""
            package p;
            unit MyUnit;
            rule R1 {
                var w : /withdrawals[amount > 100],
                do {}
            }
            """);
    var pattern = (PatternIR) rule.lhs().get(0);
    assertThat(pattern.temporalConditions()).isEmpty();
    assertThat(pattern.conditions()).containsExactly("amount > 100");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TemporalVisitorTest -pl . -q
```
Expected: FAIL — visitor doesn't handle `customConstraint` yet; temporal conditions list is always empty.

- [ ] **Step 3: Implement visitor changes**

In `DrlxToRuleAstVisitor.java`, add the temporal operator set as a class constant:

```java
private static final java.util.Set<String> TEMPORAL_OPERATORS = java.util.Set.of(
    "after", "before", "coincides", "during",
    "finishes", "finishedby", "includes",
    "meets", "metby", "overlaps", "overlappedby",
    "starts", "startedby"
);
```

Replace the `extractConditions` method (lines 858-873) to handle `customConstraint`:

```java
private List<String> extractConditions(DrlxParser.OopathExpressionContext ctx) {
    List<DrlxParser.DrlxExpressionContext> drlxExprs = collectDrlxExpressions(ctx);
    return drlxExprs.stream()
            .filter(de -> de.customConstraint() == null)
            .map(this::getText)
            .toList();
}

private List<TemporalConditionIR> extractTemporalConditions(DrlxParser.OopathExpressionContext ctx) {
    List<DrlxParser.DrlxExpressionContext> drlxExprs = collectDrlxExpressions(ctx);
    List<TemporalConditionIR> result = new java.util.ArrayList<>();
    for (var de : drlxExprs) {
        if (de.customConstraint() != null) {
            result.add(buildTemporalCondition(de.customConstraint()));
        }
    }
    return List.copyOf(result);
}

private List<DrlxParser.DrlxExpressionContext> collectDrlxExpressions(
        DrlxParser.OopathExpressionContext ctx) {
    List<DrlxParser.OopathChunkContext> chunks = ctx.oopathChunk();
    if (!chunks.isEmpty()) {
        return chunks.get(chunks.size() - 1).drlxExpression();
    }
    DrlxParser.OopathRootContext root = ctx.oopathRoot();
    if (root == null) {
        return List.of();
    }
    return root.drlxExpression();
}

private TemporalConditionIR buildTemporalCondition(DrlxParser.CustomConstraintContext ctx) {
    String operatorName = ctx.customOp().identifier().getText();
    if (!TEMPORAL_OPERATORS.contains(operatorName)) {
        throw new IllegalArgumentException(
            "Unknown custom operator '" + operatorName
            + "'. Supported temporal operators: " + TEMPORAL_OPERATORS);
    }
    boolean negated = ctx.NOT() != null;
    List<String> params = List.of();
    if (ctx.customOp().customOpParams() != null) {
        String raw = getText(ctx.customOp().customOpParams());
        params = java.util.Arrays.stream(raw.split(","))
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .toList();
    }
    String rightBinding = getText(ctx.expression());
    return new TemporalConditionIR(operatorName, negated, params, rightBinding);
}
```

Update `buildPatternFromBoundOopath` (line 755-777) to extract temporal conditions and pass them to `PatternIR`:

```java
private PatternIR buildPatternFromBoundOopath(DrlxParser.BoundOopathContext ctx) {
    String typeName = ctx.identifier(0).getText();
    String bindName = ctx.identifier(1).getText();
    DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
    String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
    String castTypeName = extractCastType(oopathCtx);
    List<String> conditions = extractConditions(oopathCtx);
    List<TemporalConditionIR> temporalConditions = extractTemporalConditions(oopathCtx);
    List<String> positionalArgs = extractPositionalArgs(oopathCtx);
    boolean passive = ctx.oopathExpression().QUESTION() != null;
    List<String> watchedProperties = extractWatchedProperties(ctx.oopathExpression());
    String windowType = null;
    String windowParameter = null;
    if (ctx.windowFilter() != null) {
        windowType = ctx.windowFilter().identifier().getText();
        if (!"length".equals(windowType) && !"time".equals(windowType)) {
            throw new IllegalArgumentException("Unknown window type: " + windowType
                    + ". Expected 'length' or 'time'.");
        }
        windowParameter = ctx.windowFilter().windowParam().getText();
    }
    return new PatternIR(typeName, bindName, entryPoint, conditions, temporalConditions,
                          castTypeName, positionalArgs, passive, watchedProperties,
                          windowType, windowParameter);
}
```

Similarly update both `buildPatternFromOopath` overloads to call `extractTemporalConditions` and pass the result.

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TemporalVisitorTest -pl . -q
```
Expected: all 5 tests PASS.

- [ ] **Step 5: Run the full test suite to check for regressions**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -q
```
Expected: all existing tests PASS.

- [ ] **Step 6: Commit**

```
feat(visitor): extract temporal conditions from customConstraint

Refs #43
```

---

### Task 4: TemporalPredicateFactory

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/TemporalPredicateFactory.java`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/TemporalPredicateFactoryTest.java`

- [ ] **Step 1: Write failing tests**

Create `TemporalPredicateFactoryTest.java`:

```java
package org.drools.drlx.builder;

import java.util.List;

import org.drools.model.functions.temporal.AfterPredicate;
import org.drools.model.functions.temporal.BeforePredicate;
import org.drools.model.functions.temporal.MeetsPredicate;
import org.drools.model.functions.temporal.TemporalPredicate;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TemporalPredicateFactoryTest {

    @Test
    void afterNoParams() {
        TemporalPredicate p = TemporalPredicateFactory.create("after", false, List.of());
        assertThat(p).isInstanceOf(AfterPredicate.class);
        assertThat(p.evaluate(2000, 0, 2000, 0, 0, 0)).isTrue();
        assertThat(p.isNegated()).isFalse();
    }

    @Test
    void afterWithTwoParams() {
        TemporalPredicate p = TemporalPredicateFactory.create("after", false, List.of("3s", "5s"));
        assertThat(p.evaluate(4000, 0, 4000, 0, 0, 0)).isTrue();
        assertThat(p.evaluate(1000, 0, 1000, 0, 0, 0)).isFalse();
    }

    @Test
    void beforeNoParams() {
        TemporalPredicate p = TemporalPredicateFactory.create("before", false, List.of());
        assertThat(p).isInstanceOf(BeforePredicate.class);
    }

    @Test
    void negatedAfter() {
        TemporalPredicate p = TemporalPredicateFactory.create("after", true, List.of());
        assertThat(p.isNegated()).isTrue();
    }

    @Test
    void meetsRejectsTwoParams() {
        assertThatThrownBy(() ->
            TemporalPredicateFactory.create("meets", false, List.of("1s", "2s")))
            .hasMessageContaining("accepts 0 or 1 parameters");
    }

    @Test
    void unknownOperatorThrows() {
        assertThatThrownBy(() ->
            TemporalPredicateFactory.create("bogus", false, List.of()))
            .hasMessageContaining("Unknown temporal operator");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TemporalPredicateFactoryTest -pl . -q
```
Expected: FAIL — `TemporalPredicateFactory` does not exist.

- [ ] **Step 3: Implement TemporalPredicateFactory**

Create `drlx-parser-core/src/main/java/org/drools/drlx/builder/TemporalPredicateFactory.java`:

```java
package org.drools.drlx.builder;

import java.util.List;
import java.util.Set;
import java.util.concurrent.TimeUnit;

import org.drools.base.time.TimeUtils;
import org.drools.model.functions.temporal.*;

public final class TemporalPredicateFactory {

    private static final Set<String> THRESHOLD_ONLY = Set.of(
            "finishes", "finishedby", "meets", "metby", "starts", "startedby");

    private TemporalPredicateFactory() {}

    public static TemporalPredicate create(String operator, boolean negated, List<String> params) {
        if (THRESHOLD_ONLY.contains(operator) && params.size() > 1) {
            throw new IllegalArgumentException(
                    "Operator '" + operator + "' accepts 0 or 1 parameters, but "
                    + params.size() + " were given");
        }

        TemporalPredicate pred = switch (operator) {
            case "after"        -> createRange(params, AfterPredicate::new,
                                       AfterPredicate::new, AfterPredicate::new);
            case "before"       -> createRange(params, BeforePredicate::new,
                                       BeforePredicate::new, BeforePredicate::new);
            case "coincides"    -> createRange(params,
                                       () -> new CoincidesPredicate(0, TimeUnit.MILLISECONDS),
                                       CoincidesPredicate::new, CoincidesPredicate::new);
            case "during"       -> createRange(params, DuringPredicate::new,
                                       DuringPredicate::new, DuringPredicate::new);
            case "finishes"     -> createThreshold(params, FinishesPredicate::new,
                                       FinishesPredicate::new);
            case "finishedby"   -> createThreshold(params, FinishedbyPredicate::new,
                                       FinishedbyPredicate::new);
            case "includes"     -> createRange(params, IncludesPredicate::new,
                                       IncludesPredicate::new, IncludesPredicate::new);
            case "meets"        -> createThreshold(params, MeetsPredicate::new,
                                       MeetsPredicate::new);
            case "metby"        -> createThreshold(params, MetbyPredicate::new,
                                       MetbyPredicate::new);
            case "overlaps"     -> createRange(params, OverlapsPredicate::new,
                                       OverlapsPredicate::new, OverlapsPredicate::new);
            case "overlappedby" -> createRange(params, OverlappedbyPredicate::new,
                                       OverlappedbyPredicate::new, OverlappedbyPredicate::new);
            case "starts"       -> createThreshold(params, StartsPredicate::new,
                                       StartsPredicate::new);
            case "startedby"    -> createThreshold(params, StartedbyPredicate::new,
                                       StartedbyPredicate::new);
            default -> throw new IllegalArgumentException(
                           "Unknown temporal operator: " + operator);
        };
        return negated ? pred.negate() : pred;
    }

    @FunctionalInterface
    interface NoArgFactory { TemporalPredicate create(); }
    @FunctionalInterface
    interface OneArgFactory { TemporalPredicate create(long v, TimeUnit u); }
    @FunctionalInterface
    interface TwoArgFactory { TemporalPredicate create(long v1, TimeUnit u1, long v2, TimeUnit u2); }

    private static TemporalPredicate createRange(List<String> params,
                                                  NoArgFactory f0,
                                                  OneArgFactory f1,
                                                  TwoArgFactory f2) {
        return switch (params.size()) {
            case 0 -> f0.create();
            case 1 -> f1.create(parseMs(params.get(0)), TimeUnit.MILLISECONDS);
            default -> f2.create(parseMs(params.get(0)), TimeUnit.MILLISECONDS,
                                 parseMs(params.get(1)), TimeUnit.MILLISECONDS);
        };
    }

    private static TemporalPredicate createThreshold(List<String> params,
                                                      NoArgFactory f0,
                                                      OneArgFactory f1) {
        return switch (params.size()) {
            case 0 -> f0.create();
            default -> f1.create(parseMs(params.get(0)), TimeUnit.MILLISECONDS);
        };
    }

    private static long parseMs(String duration) {
        return TimeUtils.parseTimeString(duration);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TemporalPredicateFactoryTest -pl . -q
```
Expected: all 6 tests PASS.

- [ ] **Step 5: Commit**

```
feat: add TemporalPredicateFactory for 13 Allen interval operators

Refs #43
```

---

### Task 5: DrlxTemporalConstraint

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxTemporalConstraint.java`

- [ ] **Step 1: Create the constraint class**

Create `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxTemporalConstraint.java`:

```java
package org.drools.drlx.builder;

import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;

import org.drools.base.base.ValueResolver;
import org.drools.base.reteoo.BaseTuple;
import org.drools.base.rule.ContextEntry;
import org.drools.base.rule.Declaration;
import org.drools.base.rule.IntervalProviderConstraint;
import org.drools.base.rule.MutableTypeConstraint;
import org.drools.base.time.Interval;
import org.drools.model.functions.temporal.TemporalPredicate;
import org.kie.api.runtime.rule.EventHandle;
import org.kie.api.runtime.rule.FactHandle;

public class DrlxTemporalConstraint
        extends MutableTypeConstraint<ContextEntry>
        implements IntervalProviderConstraint {

    private final TemporalPredicate temporalPredicate;
    private final Declaration[] requiredDeclarations;
    private final Interval interval;

    public DrlxTemporalConstraint(TemporalPredicate predicate, Declaration[] decls) {
        this.temporalPredicate = predicate;
        this.requiredDeclarations = decls;
        var modelInterval = predicate.getInterval();
        this.interval = new Interval(
                modelInterval.getLowerBound(), modelInterval.getUpperBound());
    }

    @Override
    public boolean isTemporal() {
        return true;
    }

    @Override
    public ConstraintType getType() {
        return ConstraintType.BETA;
    }

    @Override
    public Interval getInterval() {
        return interval;
    }

    @Override
    public Declaration[] getRequiredDeclarations() {
        return requiredDeclarations;
    }

    @Override
    public void replaceDeclaration(Declaration oldDecl, Declaration newDecl) {
        for (int i = 0; i < requiredDeclarations.length; i++) {
            if (requiredDeclarations[i].equals(oldDecl)) {
                requiredDeclarations[i] = newDecl;
            }
        }
    }

    @Override
    public DrlxTemporalConstraint clone() {
        return new DrlxTemporalConstraint(temporalPredicate, requiredDeclarations.clone());
    }

    @Override
    public boolean isAllowed(FactHandle handle, ValueResolver valueResolver) {
        throw new UnsupportedOperationException(
                "Temporal constraint should not be evaluated as alpha");
    }

    @Override
    public boolean isAllowedCachedLeft(ContextEntry context, FactHandle handle) {
        DrlxLambdaBetaConstraint.DrlxBetaContextEntry ctx =
                (DrlxLambdaBetaConstraint.DrlxBetaContextEntry) context;
        EventHandle thisEvent = (EventHandle) handle;
        EventHandle otherEvent = (EventHandle) ctx.tuple.get(requiredDeclarations[0]);
        return evaluateTemporal(thisEvent, otherEvent);
    }

    @Override
    public boolean isAllowedCachedRight(BaseTuple tuple, ContextEntry context) {
        DrlxLambdaBetaConstraint.DrlxBetaContextEntry ctx =
                (DrlxLambdaBetaConstraint.DrlxBetaContextEntry) context;
        EventHandle thisEvent = (EventHandle) ctx.handle;
        EventHandle otherEvent = (EventHandle) tuple.get(requiredDeclarations[0]);
        return evaluateTemporal(thisEvent, otherEvent);
    }

    private boolean evaluateTemporal(EventHandle thisEvent, EventHandle otherEvent) {
        long start1 = thisEvent.getStartTimestamp();
        long dur1   = thisEvent.getDuration();
        long end1   = thisEvent.getEndTimestamp();
        long start2 = otherEvent.getStartTimestamp();
        long dur2   = otherEvent.getDuration();
        long end2   = otherEvent.getEndTimestamp();
        if (temporalPredicate.isThisOnRight()) {
            return temporalPredicate.evaluate(start2, dur2, end2, start1, dur1, end1);
        }
        return temporalPredicate.evaluate(start1, dur1, end1, start2, dur2, end2);
    }

    @Override
    public ContextEntry createContext() {
        return new DrlxLambdaBetaConstraint.DrlxBetaContextEntry();
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        throw new UnsupportedOperationException("Not supported yet.");
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        throw new UnsupportedOperationException("Not supported yet.");
    }

    @Override
    public String toString() {
        return "temporal:" + temporalPredicate;
    }
}
```

- [ ] **Step 2: Verify compilation**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```
feat: add DrlxTemporalConstraint with IntervalProviderConstraint

Refs #43
```

---

### Task 6: Runtime builder — wire temporal constraints into Pattern

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:1255-1276`

- [ ] **Step 1: Write the failing integration test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TemporalOperatorTest.java`:

```java
package org.drools.drlx.builder.syntax;

import java.util.concurrent.TimeUnit;

import org.drools.core.ClockType;
import org.drools.core.SessionConfiguration;
import org.drools.core.impl.RuleBaseFactory;
import org.drools.drlx.builder.DrlxRuleBuilder;
import org.drools.drlx.domain.Withdrawal;
import org.drools.drlx.ruleunit.DrlxRuleUnitInstance;
import org.drools.drlx.ruleunit.WithdrawalUnit;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.DisabledIfSystemProperty;
import org.kie.api.KieBase;
import org.kie.api.KieBaseConfiguration;
import org.kie.api.conf.EventProcessingOption;
import org.kie.api.time.SessionPseudoClock;

import static org.assertj.core.api.Assertions.assertThat;

@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")
class TemporalOperatorTest {

    @Test
    void afterOperatorFiresWhenSecondEventComesLater() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var a : /withdrawals[customer == "A"],
                    var b : /withdrawals[this after a, customer == "B"],
                    do {}
                }
                """;
        try (var instance = createStreamInstance(drlx)) {
            SessionPseudoClock clock = instance.getClock();
            instance.unit().withdrawals.add(new Withdrawal("X", 100.0, "A"));
            clock.advanceTime(1, TimeUnit.SECONDS);
            instance.unit().withdrawals.add(new Withdrawal("Y", 200.0, "B"));
            assertThat(instance.fire()).isEqualTo(1);
        }
    }

    @Test
    void afterOperatorDoesNotFireWhenSimultaneous() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var a : /withdrawals[customer == "A"],
                    var b : /withdrawals[this after a, customer == "B"],
                    do {}
                }
                """;
        try (var instance = createStreamInstance(drlx)) {
            instance.unit().withdrawals.add(new Withdrawal("X", 100.0, "A"));
            instance.unit().withdrawals.add(new Withdrawal("Y", 200.0, "B"));
            assertThat(instance.fire()).isEqualTo(0);
        }
    }

    @Test
    void afterWithRangeFiresInRange() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var a : /withdrawals[customer == "A"],
                    var b : /withdrawals[this after[2s, 5s] a, customer == "B"],
                    do {}
                }
                """;
        try (var instance = createStreamInstance(drlx)) {
            SessionPseudoClock clock = instance.getClock();
            instance.unit().withdrawals.add(new Withdrawal("X", 100.0, "A"));
            clock.advanceTime(3, TimeUnit.SECONDS);
            instance.unit().withdrawals.add(new Withdrawal("Y", 200.0, "B"));
            assertThat(instance.fire()).isEqualTo(1);
        }
    }

    @Test
    void afterWithRangeDoesNotFireOutOfRange() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var a : /withdrawals[customer == "A"],
                    var b : /withdrawals[this after[2s, 5s] a, customer == "B"],
                    do {}
                }
                """;
        try (var instance = createStreamInstance(drlx)) {
            SessionPseudoClock clock = instance.getClock();
            instance.unit().withdrawals.add(new Withdrawal("X", 100.0, "A"));
            clock.advanceTime(10, TimeUnit.SECONDS);
            instance.unit().withdrawals.add(new Withdrawal("Y", 200.0, "B"));
            assertThat(instance.fire()).isEqualTo(0);
        }
    }

    @Test
    void beforeOperatorFires() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var a : /withdrawals[customer == "A"],
                    var b : /withdrawals[this before a, customer == "B"],
                    do {}
                }
                """;
        try (var instance = createStreamInstance(drlx)) {
            SessionPseudoClock clock = instance.getClock();
            instance.unit().withdrawals.add(new Withdrawal("Y", 200.0, "B"));
            clock.advanceTime(1, TimeUnit.SECONDS);
            instance.unit().withdrawals.add(new Withdrawal("X", 100.0, "A"));
            assertThat(instance.fire()).isEqualTo(1);
        }
    }

    @Test
    void notAfterFiresWhenSimultaneous() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var a : /withdrawals[customer == "A"],
                    var b : /withdrawals[this not after a, customer == "B"],
                    do {}
                }
                """;
        try (var instance = createStreamInstance(drlx)) {
            instance.unit().withdrawals.add(new Withdrawal("X", 100.0, "A"));
            instance.unit().withdrawals.add(new Withdrawal("Y", 200.0, "B"));
            assertThat(instance.fire()).isEqualTo(1);
        }
    }

    private record StreamInstance<U extends org.drools.ruleunits.api.RuleUnitData>(
            DrlxRuleUnitInstance<U> delegate, U unit) implements AutoCloseable {
        SessionPseudoClock getClock() { return delegate.getClock(); }
        int fire() { return delegate.fire(); }
        @Override public void close() { delegate.close(); }
    }

    private static StreamInstance<WithdrawalUnit> createStreamInstance(String drlx) {
        KieBaseConfiguration kbConfig = RuleBaseFactory.newKnowledgeBaseConfiguration();
        kbConfig.setOption(EventProcessingOption.STREAM);
        KieBase kieBase = new DrlxRuleBuilder().build(drlx, kbConfig);

        SessionConfiguration sessionConfig = RuleBaseFactory.newKnowledgeSessionConfiguration()
                .as(SessionConfiguration.KEY);
        sessionConfig.setClockType(ClockType.PSEUDO_CLOCK);

        WithdrawalUnit unit = new WithdrawalUnit();
        DrlxRuleUnitInstance<WithdrawalUnit> instance =
                DrlxRuleUnitInstance.create(kieBase, unit, sessionConfig);
        return new StreamInstance<>(instance, unit);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TemporalOperatorTest -pl . -q
```
Expected: FAIL — runtime builder doesn't process temporal conditions.

- [ ] **Step 3: Implement runtime builder integration**

In `DrlxRuleAstRuntimeBuilder.java`, in the `buildPattern` method, add temporal constraint processing after the regular conditions loop (after line 1261) and before the watched-properties block (line 1263):

```java
        // Temporal constraints
        for (DrlxRuleAstModel.TemporalConditionIR tc : parseResult.temporalConditions()) {
            DrlxLambdaCompiler.BoundVariable ref = boundVariables.get(tc.rightBinding());
            if (ref == null) {
                throw new RuntimeException(
                    "Temporal constraint references unknown binding '" + tc.rightBinding() + "'");
            }
            if (!pattern.getObjectType().isEvent()) {
                throw new RuntimeException(
                    "Temporal operator '" + tc.operator()
                    + "' requires event types (@Role(Type.EVENT)) but '"
                    + parseResult.typeName() + "' is not an event");
            }
            if (!ref.pattern().getObjectType().isEvent()) {
                throw new RuntimeException(
                    "Temporal operator '" + tc.operator()
                    + "' requires event types (@Role(Type.EVENT)) but the referenced binding '"
                    + tc.rightBinding() + "' is not an event");
            }
            org.drools.model.functions.temporal.TemporalPredicate predicate =
                TemporalPredicateFactory.create(tc.operator(), tc.negated(), tc.parameters());
            pattern.addConstraint(
                new DrlxTemporalConstraint(predicate, new Declaration[] { ref.declaration() }));
        }
```

- [ ] **Step 4: Run the integration tests**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TemporalOperatorTest -pl . -q
```
Expected: all 6 tests PASS.

- [ ] **Step 5: Run the full test suite**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -q
```
Expected: all tests PASS.

- [ ] **Step 6: Commit**

```
feat: wire temporal constraints in DrlxRuleAstRuntimeBuilder

Refs #43
```

---

### Task 7: Proto serialization for temporal conditions

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`

- [ ] **Step 1: Add the proto message**

In `drlx_rule_ast.proto`, add after the `PatternParseResult` message (line 61):

```protobuf
message TemporalConditionParseResult {
  string operator = 1;
  bool negated = 2;
  repeated string parameters = 3;
  string right_binding = 4;
}
```

And extend `PatternParseResult` with field 11:

```protobuf
message PatternParseResult {
  string type_name = 1;
  string bind_name = 2;
  string entry_point = 3;
  repeated string conditions = 4;
  string cast_type_name = 5;
  repeated string positional_args = 6;
  bool passive = 7;
  repeated string watched_properties = 8;
  string window_type = 9;
  string window_parameter = 10;
  repeated TemporalConditionParseResult temporal_conditions = 11;
}
```

- [ ] **Step 2: Regenerate proto classes**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Update serialization code**

In `DrlxRuleAstParseResult.java`, update `patternToProto` (line 251) to serialize temporal conditions:

```java
for (DrlxRuleAstModel.TemporalConditionIR tc : p.temporalConditions()) {
    pb.addTemporalConditions(DrlxRuleAstProto.TemporalConditionParseResult.newBuilder()
            .setOperator(tc.operator())
            .setNegated(tc.negated())
            .addAllParameters(tc.parameters())
            .setRightBinding(tc.rightBinding())
            .build());
}
```

Update `patternFromProto` (line 159) to deserialize temporal conditions:

```java
List<DrlxRuleAstModel.TemporalConditionIR> temporalConditions =
        pattern.getTemporalConditionsList().stream()
                .map(tc -> new DrlxRuleAstModel.TemporalConditionIR(
                        tc.getOperator(), tc.getNegated(),
                        List.copyOf(tc.getParametersList()),
                        tc.getRightBinding()))
                .toList();
```

And update the `new PatternIR(...)` call in `patternFromProto` to pass `temporalConditions` instead of `List.of()`.

- [ ] **Step 4: Run proto round-trip tests**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstParseResultTest -pl . -q
```
Expected: all tests PASS.

- [ ] **Step 5: Commit**

```
feat(proto): serialize temporal conditions in PatternParseResult

Refs #43
```

---

### Task 8: Install module and run full test suite

- [ ] **Step 1: Install the module**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 2: Run the full drlx-parser test suite**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -q
```
Expected: all tests PASS across all modules.

- [ ] **Step 3: Final commit if any fixups needed**

Only if test failures required fixes.
