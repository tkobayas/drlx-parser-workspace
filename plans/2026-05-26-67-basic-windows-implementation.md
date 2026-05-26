# #67 Basic Length/Time Windows Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Parse `| length[N]` and `| time[Xs]` DRLX window syntax and transpile to Drools `SlidingLengthWindow` / `SlidingTimeWindow` behaviors on `Pattern`.

**Architecture:** The pipe `|` window filter is parsed as a suffix on `boundOopath` in the ANTLR grammar. The visitor extracts window type and parameter into two new fields on `PatternIR`. The runtime builder calls `pattern.addBehavior(new SlidingTimeWindow(...))` or `new SlidingLengthWindow(...)` directly. Duration strings are parsed via Drools' `TimeUtils.parseTimeString()`.

**Tech Stack:** ANTLR4 grammar, Java 17 records, Protocol Buffers, Drools CEP runtime (`SlidingTimeWindow`, `SlidingLengthWindow`, `TimeUtils`)

**Project repo:** `/home/tkobayas/usr/work/mvel3-development/drlx-parser`

**IMPORTANT:** Use `git -C <path>` for git and `mvn -f <pom>` for maven. Never `cd` into directories.

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` | Modify | Add `windowFilter` rule, extend `boundOopath` |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` | Modify | Add `windowType`, `windowParameter` fields to `PatternIR` |
| `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` | Modify | Add fields 9-10 to `PatternParseResult` |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` | Modify | Serialize/deserialize window fields |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Modify | Extract window filter in visitor |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` | Modify | Add window behavior to Pattern |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/WindowVisitorTest.java` | Create | Visitor-level tests for window parsing |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstParseResultTest.java` | Modify | Proto round-trip test for window fields |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java` | Modify | Update existing PatternIR constructor calls |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java` | Modify | Update PatternIR constructor call |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java` | Modify | Update PatternIR constructor call |

---

### Task 1: Extend PatternIR with window fields

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:38-46`

- [ ] **Step 1: Add windowType and windowParameter to PatternIR**

Replace the `PatternIR` record at line 38-46:

```java
    public record PatternIR(String typeName,
                            String bindName,
                            String entryPoint,
                            List<String> conditions,
                            String castTypeName,
                            List<String> positionalArgs,
                            boolean passive,
                            List<String> watchedProperties,
                            String windowType,
                            String windowParameter) implements LhsItemIR {
    }
```

- [ ] **Step 2: Fix all PatternIR constructor call sites to add null, null**

There are 7 call sites that need updating. Add `, null, null` to each:

**`DrlxToRuleAstVisitor.java:765`** — `buildPatternFromBoundOopath`:
```java
        return new PatternIR(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs, passive, watchedProperties, null, null);
```

**`DrlxToRuleAstVisitor.java:775`** — `buildPatternFromOopath` (no bind):
```java
        return new PatternIR("", "", entryPoint, conditions, castTypeName, positionalArgs, passive, watchedProperties, null, null);
```

**`DrlxToRuleAstVisitor.java:786-787`** — `buildPatternFromOopath` (synthetic bind):
```java
        return new PatternIR("", syntheticBindName, entryPoint, conditions, castTypeName,
                              positionalArgs, passive, watchedProperties, null, null);
```

**`DrlxRuleAstParseResult.java:161-169`** — `patternFromProto`:
```java
        return new PatternIR(
                pattern.getTypeName(),
                pattern.getBindName(),
                pattern.getEntryPoint(),
                List.copyOf(pattern.getConditionsList()),
                castTypeName,
                List.copyOf(pattern.getPositionalArgsList()),
                pattern.getPassive(),
                List.copyOf(pattern.getWatchedPropertiesList()),
                null, null);
```
(The `null, null` here is temporary — Task 4 will wire in the actual proto fields.)

**`DrlxRuleAstParseResultTest.java:49-55`** — `passiveFlagRoundTripsThroughProto`:
```java
        PatternIR ir = new PatternIR(
                "Person", "p", "persons",
                List.of("age > 18"),
                null,
                List.of(),
                true,
                List.of(),
                null, null);
```

**`DrlxRuleAstParseResultTest.java:87-93`** — `watchedPropertiesRoundTripThroughProto`:
```java
        PatternIR ir = new PatternIR(
                "ReactiveEmployee", "e", "reactiveEmployees",
                List.of("salary > 0"),
                null,
                List.of(),
                false,
                List.of("basePay", "!bonusPay", "*"),
                null, null);
```

**`DrlxRuleAstModelTest.java:43-45`** — `accumulatePatternIrCopiesAccumulators`:
```java
        var src = new DrlxRuleAstModel.PatternIR(
                "var", "p", "persons",
                List.of(), null, List.of(), false, List.of(), null, null);
```

**`DrlxRuleAstModelTest.java:57-59`** — `accumulatePatternIrIsLhsItem`:
```java
        var src = new DrlxRuleAstModel.PatternIR(
                "var", "p", "persons",
                List.of(), null, List.of(), false, List.of(), null, null);
```

**`syntax/EvalIRBuilderTest.java:42`** — find the PatternIR constructor call and add `, null, null`.

**`syntax/NotTest.java:95`** — the inline constructor call:
```java
        PatternIR inner = new PatternIR("", "", "persons", List.of("age < 18"), null, List.of(), false, List.of(), null, null);
```

- [ ] **Step 3: Compile to verify**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core compile test-compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -q`
Expected: BUILD SUCCESS, all existing tests pass

- [ ] **Step 5: Commit**

```
feat(model): add windowType and windowParameter to PatternIR

Refs #67
```

---

### Task 2: Add windowFilter grammar rule

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:155-166`

- [ ] **Step 1: Add windowFilter rule and extend boundOopath**

Replace lines 155-160 with:

```antlr
// Bound pattern body without trailing `,`. Reused by `rulePattern`
// (top-level, with terminator `,`) and by `groupChild` (inside a CE
// paren form, where `,` is the sibling separator).
// Optional `windowFilter` suffix for CEP windows (DRLXXXX §Windows).
boundOopath
    : identifier identifier (':' | '=') oopathExpression windowFilter?
    ;
```

Add the `windowFilter` rule after `boundOopath` (before `rulePattern`):

```antlr
// Window filter — CEP sliding window applied to a pattern.
// Uses BITOR (`|`) already defined in JavaLexer. Window type is an
// identifier validated by the visitor (avoids reserving `length`/`time`).
windowFilter
    : BITOR identifier '[' expression ']'
    ;
```

- [ ] **Step 2: Compile to regenerate the parser**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core compile -q`
Expected: BUILD SUCCESS (ANTLR generates new parser with `WindowFilterContext`)

- [ ] **Step 3: Run tests to verify no regressions**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
feat(grammar): add windowFilter rule for | length[N] and | time[Xs]

Refs #67
```

---

### Task 3: Extract window filter in visitor

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:755-788`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/WindowVisitorTest.java`

- [ ] **Step 1: Write failing visitor test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/WindowVisitorTest.java`:

```java
package org.drools.drlx.builder;

import org.antlr.v4.runtime.CharStreams;
import org.antlr.v4.runtime.CommonTokenStream;
import org.drools.drlx.parser.DrlxLexer;
import org.drools.drlx.parser.DrlxParser;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.DisabledIfSystemProperty;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")
class WindowVisitorTest {

    @Test
    void parsesLengthWindow() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals | length[5],
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        assertThat(rule.lhs()).hasSize(1);
        var pattern = (DrlxRuleAstModel.PatternIR) rule.lhs().get(0);
        assertThat(pattern.entryPoint()).isEqualTo("withdrawals");
        assertThat(pattern.windowType()).isEqualTo("length");
        assertThat(pattern.windowParameter()).isEqualTo("5");
    }

    @Test
    void parsesTimeWindow() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals | time[5s],
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var pattern = (DrlxRuleAstModel.PatternIR) rule.lhs().get(0);
        assertThat(pattern.windowType()).isEqualTo("time");
        assertThat(pattern.windowParameter()).isEqualTo("5s");
    }

    @Test
    void parsesCompoundTimeWindow() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals | time[4d6h5m6s],
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var pattern = (DrlxRuleAstModel.PatternIR) rule.lhs().get(0);
        assertThat(pattern.windowType()).isEqualTo("time");
        assertThat(pattern.windowParameter()).isEqualTo("4d6h5m6s");
    }

    @Test
    void patternWithoutWindowHasNullFields() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals,
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var pattern = (DrlxRuleAstModel.PatternIR) rule.lhs().get(0);
        assertThat(pattern.windowType()).isNull();
        assertThat(pattern.windowParameter()).isNull();
    }

    @Test
    void windowWithConditionBeforeIt() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals[amount > 100] | length[5],
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var pattern = (DrlxRuleAstModel.PatternIR) rule.lhs().get(0);
        assertThat(pattern.conditions()).containsExactly("amount > 100");
        assertThat(pattern.windowType()).isEqualTo("length");
        assertThat(pattern.windowParameter()).isEqualTo("5");
    }

    @Test
    void rejectsUnknownWindowType() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals | count[5],
                    do {}
                }
                """;
        assertThatThrownBy(() -> parseRule(drlx))
                .hasMessageContaining("Unknown window type");
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

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=WindowVisitorTest -q`
Expected: FAIL — `windowType()` returns null because the visitor doesn't extract it yet

- [ ] **Step 3: Implement window extraction in the visitor**

In `DrlxToRuleAstVisitor.java`, modify `buildPatternFromBoundOopath()` (line 755) to extract the window filter and pass it to PatternIR:

```java
    private PatternIR buildPatternFromBoundOopath(DrlxParser.BoundOopathContext ctx) {
        String typeName = ctx.identifier(0).getText();
        String bindName = ctx.identifier(1).getText();
        DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
        String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
        String castTypeName = extractCastType(oopathCtx);
        List<String> conditions = extractConditions(oopathCtx);
        List<String> positionalArgs = extractPositionalArgs(oopathCtx);
        boolean passive = ctx.oopathExpression().QUESTION() != null;
        List<String> watchedProperties = extractWatchedProperties(ctx.oopathExpression());
        String windowType = null;
        String windowParameter = null;
        if (ctx.windowFilter() != null) {
            windowType = ctx.windowFilter().identifier().getText();
            if (!"length".equals(windowType) && !"time".equals(windowType)) {
                throw new DrlxCompilerException("Unknown window type: " + windowType
                        + ". Expected 'length' or 'time'.");
            }
            windowParameter = getText(ctx.windowFilter().expression());
        }
        return new PatternIR(typeName, bindName, entryPoint, conditions, castTypeName,
                              positionalArgs, passive, watchedProperties, windowType, windowParameter);
    }
```

Also verify `DrlxCompilerException` is imported. If not, check what exception class is used in the visitor for similar validation errors and use the same one.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=WindowVisitorTest -q`
Expected: PASS

- [ ] **Step 5: Run all tests for regressions**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
feat(visitor): extract window filter from boundOopath

Refs #67
```

---

### Task 4: Proto serialization for window fields

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto:50-59`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java:159-170, 247-259`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstParseResultTest.java`

- [ ] **Step 1: Write failing proto round-trip test**

Add to `DrlxRuleAstParseResultTest.java`:

```java
    @Test
    void windowFieldsRoundTripThroughProto() {
        PatternIR ir = new PatternIR(
                "Withdrawal", "w", "withdrawals",
                List.of(),
                null,
                List.of(),
                false,
                List.of(),
                "time", "5s");

        DrlxRuleAstProto.LhsItemParseResult lhsItem = DrlxRuleAstParseResult.toProtoLhs(ir);
        DrlxRuleAstProto.PatternParseResult proto = lhsItem.getPattern();
        assertThat(proto.getWindowType()).isEqualTo("time");
        assertThat(proto.getWindowParameter()).isEqualTo("5s");

        PatternIR back = (PatternIR) DrlxRuleAstParseResult.fromProtoLhs(lhsItem, Path.of("test"));
        assertThat(back.windowType()).isEqualTo("time");
        assertThat(back.windowParameter()).isEqualTo("5s");
    }

    @Test
    void missingWindowFieldsDeserialiseToNull() {
        DrlxRuleAstProto.PatternParseResult proto =
                DrlxRuleAstProto.PatternParseResult.newBuilder()
                        .setTypeName("Person")
                        .setBindName("p")
                        .setEntryPoint("persons")
                        .build();

        DrlxRuleAstProto.LhsItemParseResult lhsItem =
                DrlxRuleAstProto.LhsItemParseResult.newBuilder()
                        .setPattern(proto)
                        .build();

        PatternIR back = (PatternIR) DrlxRuleAstParseResult.fromProtoLhs(lhsItem, Path.of("test"));
        assertThat(back.windowType()).isNull();
        assertThat(back.windowParameter()).isNull();
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest='DrlxRuleAstParseResultTest#windowFieldsRoundTripThroughProto+missingWindowFieldsDeserialiseToNull' -q`
Expected: FAIL — `getWindowType()` method does not exist yet

- [ ] **Step 3: Add proto fields**

In `drlx_rule_ast.proto`, add to `PatternParseResult`:

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
}
```

- [ ] **Step 4: Regenerate protobuf Java code**

Run the protobuf compiler. Check the project's maven build for the protoc plugin, or use the local protoc binary:

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core compile -q`
Expected: BUILD SUCCESS — proto stubs regenerated with `getWindowType()` / `setWindowType()`

- [ ] **Step 5: Update patternToProto()**

In `DrlxRuleAstParseResult.java`, modify `patternToProto()` (line 247) to serialize window fields:

```java
    private static DrlxRuleAstProto.PatternParseResult patternToProto(PatternIR p) {
        DrlxRuleAstProto.PatternParseResult.Builder pb = DrlxRuleAstProto.PatternParseResult.newBuilder()
                .setTypeName(p.typeName())
                .setBindName(p.bindName())
                .setEntryPoint(p.entryPoint())
                .setPassive(p.passive());
        if (p.castTypeName() != null) {
            pb.setCastTypeName(p.castTypeName());
        }
        p.conditions().forEach(pb::addConditions);
        p.positionalArgs().forEach(pb::addPositionalArgs);
        p.watchedProperties().forEach(pb::addWatchedProperties);
        if (p.windowType() != null) {
            pb.setWindowType(p.windowType());
            pb.setWindowParameter(p.windowParameter());
        }
        return pb.build();
    }
```

- [ ] **Step 6: Update patternFromProto()**

In `DrlxRuleAstParseResult.java`, modify `patternFromProto()` (line 159) to deserialize window fields:

```java
    private static PatternIR patternFromProto(DrlxRuleAstProto.PatternParseResult pattern) {
        String castTypeName = pattern.getCastTypeName().isEmpty() ? null : pattern.getCastTypeName();
        String windowType = pattern.getWindowType().isEmpty() ? null : pattern.getWindowType();
        String windowParameter = pattern.getWindowParameter().isEmpty() ? null : pattern.getWindowParameter();
        return new PatternIR(
                pattern.getTypeName(),
                pattern.getBindName(),
                pattern.getEntryPoint(),
                List.copyOf(pattern.getConditionsList()),
                castTypeName,
                List.copyOf(pattern.getPositionalArgsList()),
                pattern.getPassive(),
                List.copyOf(pattern.getWatchedPropertiesList()),
                windowType,
                windowParameter);
    }
```

- [ ] **Step 7: Run tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=DrlxRuleAstParseResultTest -q`
Expected: PASS

- [ ] **Step 8: Run all tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```
feat(proto): add window_type and window_parameter to PatternParseResult

Refs #67
```

---

### Task 5: Wire window behavior into runtime builder

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:1212-1264`

- [ ] **Step 1: Write failing integration test**

Add to `WindowVisitorTest.java` (which already has the `parseRule` helper):

```java
    @Test
    void lengthWindowProducesPatternWithSlidingLengthBehavior() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var p : /persons | length[5],
                    do {}
                }
                """;
        org.kie.api.KieBase kieBase = new org.drools.drlx.builder.DrlxRuleBuilder().build(drlx);
        org.kie.api.definition.rule.Rule rule = kieBase.getRule("org.drools.drlx.parser", "R1");
        assertThat(rule).isNotNull();
        org.drools.base.rule.RuleImpl ruleImpl = (org.drools.base.rule.RuleImpl) rule;
        org.drools.base.rule.Pattern pattern =
                (org.drools.base.rule.Pattern) ruleImpl.getLhs().getChildren().get(0);
        assertThat(pattern.getBehaviors()).hasSize(1);
        assertThat(pattern.getBehaviors().get(0))
                .isInstanceOf(org.drools.core.rule.SlidingLengthWindow.class);
        org.drools.core.rule.SlidingLengthWindow window =
                (org.drools.core.rule.SlidingLengthWindow) pattern.getBehaviors().get(0);
        assertThat(window.getSize()).isEqualTo(5);
    }

    @Test
    void timeWindowProducesPatternWithSlidingTimeBehavior() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var p : /persons | time[5s],
                    do {}
                }
                """;
        org.kie.api.KieBase kieBase = new org.drools.drlx.builder.DrlxRuleBuilder().build(drlx);
        org.drools.base.rule.RuleImpl ruleImpl =
                (org.drools.base.rule.RuleImpl) kieBase.getRule("org.drools.drlx.parser", "R1");
        org.drools.base.rule.Pattern pattern =
                (org.drools.base.rule.Pattern) ruleImpl.getLhs().getChildren().get(0);
        assertThat(pattern.getBehaviors()).hasSize(1);
        assertThat(pattern.getBehaviors().get(0))
                .isInstanceOf(org.drools.core.rule.SlidingTimeWindow.class);
        org.drools.core.rule.SlidingTimeWindow window =
                (org.drools.core.rule.SlidingTimeWindow) pattern.getBehaviors().get(0);
        assertThat(window.getSize()).isEqualTo(5000L);
    }

    @Test
    void compoundTimeWindowParsesCorrectly() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var p : /persons | time[4d6h5m6s],
                    do {}
                }
                """;
        org.kie.api.KieBase kieBase = new org.drools.drlx.builder.DrlxRuleBuilder().build(drlx);
        org.drools.base.rule.RuleImpl ruleImpl =
                (org.drools.base.rule.RuleImpl) kieBase.getRule("org.drools.drlx.parser", "R1");
        org.drools.base.rule.Pattern pattern =
                (org.drools.base.rule.Pattern) ruleImpl.getLhs().getChildren().get(0);
        org.drools.core.rule.SlidingTimeWindow window =
                (org.drools.core.rule.SlidingTimeWindow) pattern.getBehaviors().get(0);
        long expected = 4L * 86400000 + 6L * 3600000 + 5L * 60000 + 6L * 1000;
        assertThat(window.getSize()).isEqualTo(expected);
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest='WindowVisitorTest#lengthWindowProducesPatternWithSlidingLengthBehavior' -q`
Expected: FAIL — `getBehaviors()` returns empty list

- [ ] **Step 3: Add window behavior in buildPattern()**

In `DrlxRuleAstRuntimeBuilder.java`, at the end of `buildPattern()` (after the watch-properties block, around line 1261), add:

```java
        if (parseResult.windowType() != null) {
            switch (parseResult.windowType()) {
                case "time" -> pattern.addBehavior(
                        new SlidingTimeWindow(TimeUtils.parseTimeString(parseResult.windowParameter())));
                case "length" -> pattern.addBehavior(
                        new SlidingLengthWindow(Integer.parseInt(parseResult.windowParameter())));
            }
        }
```

Add imports at the top of the file:

```java
import org.drools.core.rule.SlidingTimeWindow;
import org.drools.core.rule.SlidingLengthWindow;
import org.drools.base.time.TimeUtils;
```

- [ ] **Step 4: Run window tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -Dtest=WindowVisitorTest -q`
Expected: PASS

- [ ] **Step 5: Run all tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core test -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
feat(builder): wire SlidingTimeWindow/SlidingLengthWindow into Pattern

Refs #67
```

---

### Task 6: Install and final verification

- [ ] **Step 1: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install -q`
Expected: BUILD SUCCESS

- [ ] **Step 2: Run full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -q`
Expected: BUILD SUCCESS (all modules pass)

- [ ] **Step 3: Commit and close issue**

No additional commit needed if all prior commits are clean. Close #67:

```bash
gh issue close 67 --repo tkobayas/drlx-parser --comment "Implemented: basic length/time window syntax (| length[N], | time[Xs]) with SlidingLengthWindow/SlidingTimeWindow behavior mapping."
```
