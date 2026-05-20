# #51 Accumulate acc() Keyword Forms Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the `acc(...)` keyword syntax for accumulate in DRLX — 2-param (function sugar), 3-param (custom w/o reverse), and 5-param (custom w/ reverse).

**Architecture:** New grammar rules parse `acc(source, ...)` contextually (identifier == "acc"). 2-param lowers to existing `AccumulatePatternIR`. 3/5-param introduce `CustomAccumulateIR` + `InitVarIR` records, a new `DrlxCustomAccumulator` runtime class using MVEL3 map-based evaluators, and batch compilation via `DrlxLambdaCompiler.createCustomAccumulator()`.

**Tech Stack:** ANTLR4 grammar, Java 17 records/sealed interfaces, MVEL3 batch compiler, Drools base `Accumulator`/`SingleAccumulate`, protobuf persistence.

**Spec:** `specs/2026-05-19-51-acc-keyword-forms-design.md`

---

## File Structure

| File | Action | Responsibility |
|------|--------|----------------|
| `DrlxParser.g4` | Modify | Add `accKeywordItem`, `accSource`, `accBody`, `accFunctionList`, `accInitVars`, `accInitVar`, `accActionBlock`, `accResultBinding` rules; add `accKeywordItem ','` to `ruleItem` |
| `DrlxRuleAstModel.java` | Modify | Add `CustomAccumulateIR` and `InitVarIR` records; update `LhsItemIR` permits list |
| `DrlxToRuleAstVisitor.java` | Modify | Add `accKeywordItem` dispatch in `buildRule()`; add `buildAccKeywordItem()` with 2/3/5-param handling and all semantic validations |
| `DrlxCustomAccumulator.java` | Create | `Accumulator` implementation using MVEL3 map evaluators for action/reverse/result |
| `DrlxLambdaCompiler.java` | Modify | Add `createCustomAccumulator()` method with batch lambda registration for action/reverse/result slots |
| `DrlxRuleAstRuntimeBuilder.java` | Modify | Add `CustomAccumulateIR` branch in `buildLhs()` and `collectPatternClasses()`; add `buildCustomAccumulatePattern()` method; add `resolveTypeName()` helper |
| `drlx_rule_ast.proto` | Modify | Add `CustomAccumulateParseResult` and `InitVarParseResult` messages; add to `LhsItemParseResult` oneof |
| `DrlxRuleAstParseResult.java` | Modify | Add `CustomAccumulateIR` serialization/deserialization branches |
| `AccumulateVisitorTest.java` | Modify | Add visitor-level tests for all acc() forms |
| `AccumulateTest.java` | Modify | Add runtime integration tests for all acc() forms |

---

### Task 1: Grammar — add accKeywordItem rules to DrlxParser.g4

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:52-62` (ruleItem)
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:201-215` (append new rules after accumulateCall)

- [ ] **Step 1: Add accKeywordItem to ruleItem rule**

In `DrlxParser.g4`, add `accKeywordItem ','` as a new alternative in the `ruleItem` rule, after `accumulateItem ','`:

```antlr
ruleItem
    : rulePattern
    | accumulateItem ','
    | accKeywordItem ','
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','
    | conditionalBranch ','
    | ruleConsequence
    ;
```

- [ ] **Step 2: Add the new grammar rules after the existing accumulateCall rule**

Append these rules after the `inlineFromOopath` rule at the end of the grammar file:

```antlr
// acc(...) keyword forms — DRLXXXX §Accumulate.
// `acc` is contextual: parsed as an identifier, validated at visitor level.
accKeywordItem
    : identifier '(' accSource ',' accBody ')'
    ;

accSource
    : boundOopath
    ;

accBody
    : accFunctionList
    | accInitVars ',' accActionBlock ',' accResultBinding
    | accInitVars ',' accActionBlock ',' accActionBlock ',' accResultBinding
    ;

accFunctionList
    : accumulateItem
    | '(' accumulateItem (',' accumulateItem)* ')'
    ;

accInitVars
    : accInitVar
    | '{' accInitVar+ '}'
    ;

accInitVar
    : localVariableDeclaration ';'
    ;

accActionBlock
    : expression
    | '(' expression ',' expression ')'
    | '{' statement+ '}'
    ;

accResultBinding
    : typeType identifier '=' expression
    ;
```

- [ ] **Step 3: Regenerate the ANTLR parser**

Run:
```bash
mvn -f drlx-parser-core/pom.xml generate-sources -q
```
Expected: BUILD SUCCESS, generated parser contains `AccKeywordItemContext`, `AccSourceContext`, `AccBodyContext`, etc.

- [ ] **Step 4: Verify grammar compiles without ambiguity warnings**

Run:
```bash
mvn -f drlx-parser-core/pom.xml compile -q
```
Expected: BUILD SUCCESS with no ANTLR warnings about ambiguity.

- [ ] **Step 5: Commit**

```bash
git add drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4
git commit -m "feat(grammar): add acc() keyword rules to DrlxParser.g4

Refs #51"
```

---

### Task 2: IR — add CustomAccumulateIR and InitVarIR records

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:32` (LhsItemIR permits)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:79` (append new records)

- [ ] **Step 1: Write the failing test**

In `AccumulateVisitorTest.java`, add a test that parses a 3-param `acc(...)` and asserts it produces a `CustomAccumulateIR`:

```java
@Test
void accKeyword3ParamProducesCustomAccumulateIR() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int s = 0;,
                    s = s + p.age,
                    int sum = s),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    assertThat(rule.lhs()).hasSize(1);
    assertThat(rule.lhs().get(0)).isInstanceOf(DrlxRuleAstModel.CustomAccumulateIR.class);
    var custom = (DrlxRuleAstModel.CustomAccumulateIR) rule.lhs().get(0);
    assertThat(custom.source().bindName()).isEqualTo("p");
    assertThat(custom.initVars()).hasSize(1);
    assertThat(custom.initVars().get(0).typeName()).isEqualTo("int");
    assertThat(custom.initVars().get(0).name()).isEqualTo("s");
    assertThat(custom.initVars().get(0).initializer()).isEqualTo("0");
    assertThat(custom.actionBlock()).isEqualTo("s = s + p.age");
    assertThat(custom.reverseBlock()).isNull();
    assertThat(custom.resultTypeName()).isEqualTo("int");
    assertThat(custom.resultBindName()).isEqualTo("sum");
    assertThat(custom.resultExpression()).isEqualTo("s");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -Dtest=AccumulateVisitorTest#accKeyword3ParamProducesCustomAccumulateIR -q
```
Expected: FAIL — `CustomAccumulateIR` class does not exist.

- [ ] **Step 3: Add InitVarIR and CustomAccumulateIR records to DrlxRuleAstModel.java**

Add `CustomAccumulateIR` to the `LhsItemIR` sealed interface permits list:

```java
public sealed interface LhsItemIR permits PatternIR, GroupElementIR, EvalIR, AccumulatePatternIR, CustomAccumulateIR {
}
```

Add the new records after the existing `AccumulatorIR` record (after line 79):

```java
public record CustomAccumulateIR(
    PatternIR source,
    List<InitVarIR> initVars,
    String actionBlock,
    String reverseBlock,
    String resultTypeName,
    String resultBindName,
    String resultExpression,
    List<String> referencedBindings
) implements LhsItemIR {
    public CustomAccumulateIR {
        initVars = List.copyOf(initVars);
        referencedBindings = List.copyOf(referencedBindings);
    }
}

public record InitVarIR(
    String typeName,
    String name,
    String initializer
) {
}
```

- [ ] **Step 4: Verify compilation**

Run:
```bash
mvn -f drlx-parser-core/pom.xml compile -q
```
Expected: BUILD SUCCESS. The sealed interface now permits `CustomAccumulateIR`. Existing consumers (`buildLhs()`, `collectPatternClasses()`, `toProtoLhs()`, `fromProtoLhs()`) use if/else if chains — not switch expressions — so there is no exhaustiveness compilation failure. The new branches for `CustomAccumulateIR` are added in Tasks 4 (protobuf) and 7 (runtime builder), before any `CustomAccumulateIR` instance can reach those paths at runtime.

- [ ] **Step 5: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java
git commit -m "feat(ir): add CustomAccumulateIR and InitVarIR records

Refs #51"
```

---

### Task 3: Visitor — handle accKeywordItem in DrlxToRuleAstVisitor

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:88-174` (buildRule method)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` (append new methods)
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java` (add test from Task 2 Step 1 + more)

This task is the largest and most complex. It implements the full visitor logic including all semantic validations specified in the design spec.

- [ ] **Step 1: Add accKeywordItem dispatch to buildRule()**

In `DrlxToRuleAstVisitor.buildRule()`, add a new branch after the `accumulateItem` handling (after line 141, before the comment "Any non-accumulate item..."):

```java
                if (itemCtx.accKeywordItem() != null) {
                    flushPending(lhs, pendingPattern, pendingAccs);
                    pendingPattern = null;
                    pendingAccs = new ArrayList<>();
                    lhs.add(buildAccKeywordItem(itemCtx.accKeywordItem()));
                    continue;
                }
```

Also add the import for the new IR types at the top of the file:

```java
import org.drools.drlx.builder.DrlxRuleAstModel.CustomAccumulateIR;
import org.drools.drlx.builder.DrlxRuleAstModel.InitVarIR;
```

- [ ] **Step 2: Add buildAccKeywordItem() method**

Add the full method to `DrlxToRuleAstVisitor`. This method handles:
- Contextual `acc` validation (identifier text must be "acc")
- 2-param dispatch to existing `AccumulatePatternIR`
- 3-param and 5-param dispatch to `CustomAccumulateIR`
- All semantic validations (var rejection, init var names, literal validation, reference validation, paired block rejection in 5-param)

```java
private LhsItemIR buildAccKeywordItem(DrlxParser.AccKeywordItemContext ctx) {
    String keyword = ctx.identifier().getText();
    if (!"acc".equals(keyword)) {
        throw new RuntimeException(
                "expected 'acc' keyword but found '" + keyword + "' at "
                + ctx.getStart().getLine() + ":" + ctx.getStart().getCharPositionInLine());
    }

    PatternIR source = buildPatternFromBoundOopath(ctx.accSource().boundOopath());
    String srcBindName = source.bindName();
    DrlxParser.AccBodyContext body = ctx.accBody();

    if (body.accFunctionList() != null) {
        return buildAccKeyword2Param(source, body.accFunctionList());
    }

    List<DrlxParser.AccActionBlockContext> actionBlocks = body.accActionBlock();
    boolean is5Param = actionBlocks.size() == 2;

    List<InitVarIR> initVars = buildInitVars(body.accInitVars(), srcBindName);

    String actionBlock;
    String reverseBlock;

    if (is5Param) {
        actionBlock = extractActionBlockText(actionBlocks.get(0), true);
        reverseBlock = extractActionBlockText(actionBlocks.get(1), true);
    } else {
        DrlxParser.AccActionBlockContext actionCtx = actionBlocks.get(0);
        if (actionCtx.expression().size() == 2 && actionCtx.getChild(0).getText().equals("(")) {
            actionBlock = getText(actionCtx.expression(0));
            reverseBlock = getText(actionCtx.expression(1));
        } else {
            actionBlock = extractActionBlockText(actionCtx, false);
            reverseBlock = null;
        }
    }

    DrlxParser.AccResultBindingContext resultCtx = body.accResultBinding();
    String resultTypeName = resultCtx.typeType().getText();
    String resultBindName = resultCtx.identifier().getText();
    String resultExpression = getText(resultCtx.expression());

    validateResultExpression(resultExpression, srcBindName, initVars);

    java.util.LinkedHashSet<String> refs = new java.util.LinkedHashSet<>();
    refs.addAll(extractIdentifiers(actionBlock));
    if (reverseBlock != null) {
        refs.addAll(extractIdentifiers(reverseBlock));
    }
    refs.addAll(extractIdentifiers(resultExpression));

    return new CustomAccumulateIR(source, initVars, actionBlock, reverseBlock,
            resultTypeName, resultBindName, resultExpression, List.copyOf(refs));
}
```

- [ ] **Step 3: Add buildAccKeyword2Param() helper**

```java
private LhsItemIR buildAccKeyword2Param(PatternIR source,
                                         DrlxParser.AccFunctionListContext funcListCtx) {
    List<AccumulatorIR> accumulators = new ArrayList<>();
    for (DrlxParser.AccumulateItemContext accItemCtx : funcListCtx.accumulateItem()) {
        accumulators.add(buildAccumulator(accItemCtx));
    }
    return new AccumulatePatternIR(source, accumulators);
}
```

- [ ] **Step 4: Add buildInitVars() helper with semantic validations**

```java
private List<InitVarIR> buildInitVars(DrlxParser.AccInitVarsContext ctx,
                                       String srcBindName) {
    List<InitVarIR> result = new ArrayList<>();
    java.util.Set<String> seenNames = new java.util.LinkedHashSet<>();

    for (DrlxParser.AccInitVarContext initVarCtx : ctx.accInitVar()) {
        DrlxParser.LocalVariableDeclarationContext localVarCtx = initVarCtx.localVariableDeclaration();

        if (localVarCtx.VAR() != null) {
            throw new RuntimeException(
                    "custom accumulate init vars require explicit types — 'var' is not permitted");
        }

        String typeName = localVarCtx.typeType().getText();
        DrlxParser.VariableDeclaratorsContext declsCtx = localVarCtx.variableDeclarators();

        for (DrlxParser.VariableDeclaratorContext declCtx : declsCtx.variableDeclarator()) {
            String varName = declCtx.variableDeclaratorId().getText();

            if (varName.equals(srcBindName)) {
                throw new RuntimeException(
                        "init var '" + varName + "' conflicts with source binding name");
            }
            if (!seenNames.add(varName)) {
                throw new RuntimeException(
                        "duplicate init var name '" + varName + "'");
            }

            String initializer;
            if (declCtx.variableInitializer() != null) {
                initializer = getText(declCtx.variableInitializer());
                validateLiteralInitializer(initializer, typeName, varName);
            } else {
                initializer = defaultValueFor(typeName);
            }

            result.add(new InitVarIR(typeName, varName, initializer));
        }
    }
    return result;
}
```

- [ ] **Step 5: Add extractActionBlockText() helper**

```java
private String extractActionBlockText(DrlxParser.AccActionBlockContext ctx,
                                       boolean rejectPaired) {
    if (ctx.expression().size() == 2 && ctx.getChild(0).getText().equals("(")) {
        if (rejectPaired) {
            throw new RuntimeException(
                    "paired (action, reverse) block is not valid in 5-param acc — use separate action and reverse positions");
        }
        return getText(ctx.expression(0));
    }
    if (ctx.statement() != null && !ctx.statement().isEmpty()) {
        StringBuilder sb = new StringBuilder();
        for (DrlxParser.StatementContext stCtx : ctx.statement()) {
            sb.append(getText(stCtx));
        }
        return sb.toString().trim();
    }
    return getText(ctx.expression(0));
}
```

- [ ] **Step 6: Add validateLiteralInitializer() helper**

```java
private static void validateLiteralInitializer(String initializer, String typeName, String varName) {
    if (initializer.equals("null")) {
        if (isPrimitiveTypeName(typeName)) {
            throw new RuntimeException(
                    "init var '" + varName + "': cannot assign null to primitive type " + typeName);
        }
        return;
    }
    if (initializer.equals("true") || initializer.equals("false")) {
        if (!"boolean".equals(typeName) && !"Boolean".equals(typeName)) {
            throw new RuntimeException(
                    "init var '" + varName + "': cannot assign boolean literal to " + typeName);
        }
        return;
    }
    if (initializer.startsWith("\"") && initializer.endsWith("\"")) {
        if (!"String".equals(typeName) && !"java.lang.String".equals(typeName)) {
            throw new RuntimeException(
                    "init var '" + varName + "': cannot assign String literal to " + typeName);
        }
        return;
    }
    if (initializer.startsWith("'") && initializer.endsWith("'")) {
        if (!"char".equals(typeName) && !"Character".equals(typeName)) {
            throw new RuntimeException(
                    "init var '" + varName + "': cannot assign char literal to " + typeName);
        }
        return;
    }

    if (isNumericLiteral(initializer)) {
        validateNumericLiteralType(initializer, typeName, varName);
        return;
    }

    throw new RuntimeException(
            "custom accumulate init vars must be literals — complex initializers are not yet supported");
}

private static boolean isPrimitiveTypeName(String typeName) {
    return switch (typeName) {
        case "int", "long", "double", "float", "short", "byte", "boolean", "char" -> true;
        default -> false;
    };
}

private static boolean isNumericLiteral(String value) {
    if (value.isEmpty()) return false;
    String v = value;
    if (v.startsWith("-") || v.startsWith("+")) v = v.substring(1);
    if (v.isEmpty()) return false;
    String lower = v.toLowerCase();
    if (lower.endsWith("l") || lower.endsWith("f") || lower.endsWith("d")) {
        lower = lower.substring(0, lower.length() - 1);
    }
    if (lower.isEmpty()) return false;
    if (lower.contains(".")) {
        try { Double.parseDouble(lower); return true; }
        catch (NumberFormatException e) { return false; }
    }
    try { Long.parseLong(lower); return true; }
    catch (NumberFormatException e) { return false; }
}

private static void validateNumericLiteralType(String initializer, String typeName, String varName) {
    String lower = initializer.toLowerCase();
    boolean isLong = lower.endsWith("l");
    boolean isFloat = lower.endsWith("f");
    boolean isDouble = lower.endsWith("d") || (!isLong && !isFloat && lower.contains("."));

    switch (typeName) {
        case "int", "Integer" -> {
            if (isLong) throw new RuntimeException("init var '" + varName + "': cannot assign long literal to int");
            if (isFloat || isDouble) throw new RuntimeException("init var '" + varName + "': cannot assign floating-point literal to int");
            String numStr = initializer;
            try {
                long val = Long.parseLong(numStr);
                if (val < Integer.MIN_VALUE || val > Integer.MAX_VALUE) {
                    throw new RuntimeException("init var '" + varName + "': literal value out of range for int");
                }
            } catch (NumberFormatException e) {
                throw new RuntimeException("init var '" + varName + "': invalid numeric literal '" + initializer + "'");
            }
        }
        case "long", "Long" -> {
            if (isFloat || isDouble) throw new RuntimeException("init var '" + varName + "': cannot assign floating-point literal to long");
        }
        case "double", "Double" -> { /* any numeric literal widens to double */ }
        case "float", "Float" -> {
            if (isLong) throw new RuntimeException("init var '" + varName + "': cannot assign long literal to float");
        }
        case "short", "Short" -> {
            if (isLong) throw new RuntimeException("init var '" + varName + "': cannot assign long literal to short");
            if (isFloat || isDouble) throw new RuntimeException("init var '" + varName + "': cannot assign floating-point literal to short");
            try {
                long val = Long.parseLong(initializer);
                if (val < Short.MIN_VALUE || val > Short.MAX_VALUE) {
                    throw new RuntimeException("init var '" + varName + "': literal value out of range for short");
                }
            } catch (NumberFormatException e) { /* non-numeric already filtered */ }
        }
        case "byte", "Byte" -> {
            if (isLong) throw new RuntimeException("init var '" + varName + "': cannot assign long literal to byte");
            if (isFloat || isDouble) throw new RuntimeException("init var '" + varName + "': cannot assign floating-point literal to byte");
            try {
                long val = Long.parseLong(initializer);
                if (val < Byte.MIN_VALUE || val > Byte.MAX_VALUE) {
                    throw new RuntimeException("init var '" + varName + "': literal value out of range for byte");
                }
            } catch (NumberFormatException e) { /* non-numeric already filtered */ }
        }
        case "boolean", "Boolean" -> throw new RuntimeException("init var '" + varName + "': cannot assign numeric literal to boolean");
        case "char", "Character" -> throw new RuntimeException("init var '" + varName + "': cannot assign numeric literal to char");
        default -> { /* reference types accept numeric widening at runtime */ }
    }
}
```

- [ ] **Step 7: Add validateResultExpression() and defaultValueFor() helpers**

Outer-binding rejection is deferred to the runtime builder (Task 7, Step 5) where `outerScope` is available. The visitor records `referencedBindings` and the runtime builder validates against live bound variables.

```java
private static void validateResultExpression(String resultExpr, String srcBindName,
                                              List<InitVarIR> initVars) {
    List<String> ids = extractIdentifiers(resultExpr);
    if (ids.contains(srcBindName)) {
        throw new RuntimeException(
                "result expression cannot reference source binding '" + srcBindName
                + "' — only init vars are available");
    }
}

private static String defaultValueFor(String typeName) {
    return switch (typeName) {
        case "int", "Integer", "short", "Short", "byte", "Byte" -> "0";
        case "long", "Long" -> "0";
        case "double", "Double" -> "0.0";
        case "float", "Float" -> "0.0";
        case "boolean", "Boolean" -> "false";
        case "char", "Character" -> "' '";
        default -> "null";
    };
}
```

- [ ] **Step 8: Run the test from Task 2 Step 1**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -Dtest=AccumulateVisitorTest#accKeyword3ParamProducesCustomAccumulateIR -q
```
Expected: PASS.

- [ ] **Step 9: Add and run more visitor tests**

Add these tests to `AccumulateVisitorTest.java`:

```java
@Test
void accKeyword2ParamSingleFunctionProducesAccumulatePatternIR() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    var avgAge = avg(p.age)),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    assertThat(rule.lhs()).hasSize(1);
    assertThat(rule.lhs().get(0)).isInstanceOf(DrlxRuleAstModel.AccumulatePatternIR.class);
    var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
    assertThat(accPat.source().bindName()).isEqualTo("p");
    assertThat(accPat.accumulators()).hasSize(1);
    assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("avg");
}

@Test
void accKeyword2ParamGroupedFunctions() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    (var maxAge = max(p.age),
                     var minAge = min(p.age))),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    assertThat(rule.lhs()).hasSize(1);
    var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
    assertThat(accPat.accumulators()).hasSize(2);
    assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("max");
    assertThat(accPat.accumulators().get(1).functionName()).isEqualTo("min");
}

@Test
void accKeyword3ParamWithPairedActionReverse() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int s = 0;,
                    (s = s + p.age, s = s - p.age),
                    int sum = s),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    var custom = (DrlxRuleAstModel.CustomAccumulateIR) rule.lhs().get(0);
    assertThat(custom.actionBlock()).isEqualTo("s = s + p.age");
    assertThat(custom.reverseBlock()).isEqualTo("s = s - p.age");
}

@Test
void accKeyword5ParamWithBracedBlocks() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    { int count = 0; int total = 0; },
                    { total += p.age; count++; },
                    { total -= p.age; count--; },
                    double avgAge = total / count),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    var custom = (DrlxRuleAstModel.CustomAccumulateIR) rule.lhs().get(0);
    assertThat(custom.source().bindName()).isEqualTo("p");
    assertThat(custom.initVars()).hasSize(2);
    assertThat(custom.initVars().get(0)).isEqualTo(new DrlxRuleAstModel.InitVarIR("int", "count", "0"));
    assertThat(custom.initVars().get(1)).isEqualTo(new DrlxRuleAstModel.InitVarIR("int", "total", "0"));
    assertThat(custom.reverseBlock()).isNotNull();
    assertThat(custom.resultTypeName()).isEqualTo("double");
    assertThat(custom.resultBindName()).isEqualTo("avgAge");
}

@Test
void accKeywordRejectsNonAccIdentifier() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                notAcc(var p : /persons,
                    int s = 0;,
                    s = s + p.age,
                    int sum = s),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class);
}

@Test
void accKeywordRejectsVarInInitVars() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    var s = 0;,
                    s = s + p.age,
                    int sum = s),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("var");
}

@Test
void accKeywordRejectsInitVarNameCollision() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int p = 0;,
                    p = p + 1,
                    int sum = p),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("conflicts with source binding name");
}

@Test
void accKeywordRejectsDuplicateInitVarNames() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    { int x = 0; int x = 1; },
                    x = x + p.age,
                    int sum = x),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("duplicate init var name 'x'");
}

@Test
void accKeywordRejectsSourceRefInResult() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int s = 0;,
                    s = s + p.age,
                    int sum = p),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("result expression cannot reference source binding");
}

@Test
void accKeywordRejectsPairedBlockIn5Param() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int s = 0;,
                    (s = s + p.age, s = s - p.age),
                    s = s - p.age,
                    int sum = s),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("paired (action, reverse) block is not valid in 5-param acc");
}

@Test
void accKeywordRejectsNullForPrimitiveInit() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int s = null;,
                    s = s + p.age,
                    int sum = s),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("cannot assign null to primitive type int");
}

@Test
void accKeywordMultiDeclaratorSplitsIntoSeparateInitVars() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int a = 0, b = 1;,
                    a = a + p.age,
                    int sum = a),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    var custom = (DrlxRuleAstModel.CustomAccumulateIR) rule.lhs().get(0);
    assertThat(custom.initVars()).hasSize(2);
    assertThat(custom.initVars().get(0).name()).isEqualTo("a");
    assertThat(custom.initVars().get(0).initializer()).isEqualTo("0");
    assertThat(custom.initVars().get(1).name()).isEqualTo("b");
    assertThat(custom.initVars().get(1).initializer()).isEqualTo("1");
}

@Test
void accKeywordRejectsNarrowingLiteral() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int s = 0L;,
                    s = s + p.age,
                    int sum = s),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("cannot assign long literal to int");
}

@Test
void accKeywordRejectsComplexInitializer() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    String s = new String();,
                    s = p.name,
                    String result = s),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("complex initializers are not yet supported");
}

@Test
void accKeywordMissingInitializerUsesDefault() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                acc(var p : /persons,
                    int a;,
                    a = a + p.age,
                    int sum = a),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    var custom = (DrlxRuleAstModel.CustomAccumulateIR) rule.lhs().get(0);
    assertThat(custom.initVars()).hasSize(1);
    assertThat(custom.initVars().get(0).initializer()).isEqualTo("0");
}
```

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -Dtest=AccumulateVisitorTest -q
```
Expected: ALL PASS.

- [ ] **Step 10: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java
git commit -m "feat(visitor): handle acc() keyword in DrlxToRuleAstVisitor

2-param dispatches to AccumulatePatternIR, 3/5-param to
CustomAccumulateIR with full semantic validation.

Refs #51"
```

---

### Task 4: Protobuf — add CustomAccumulateParseResult and InitVarParseResult

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto:24-31` (LhsItemParseResult oneof)
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` (append new messages)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` (toProtoLhs + fromProtoLhs)

- [ ] **Step 1: Add proto messages**

In `drlx_rule_ast.proto`, add `custom_accumulate = 5` to the `LhsItemParseResult` oneof:

```protobuf
message LhsItemParseResult {
  oneof kind {
    PatternParseResult pattern = 1;
    GroupElementParseResult group = 2;
    EvalParseResult eval = 3;
    AccumulatePatternParseResult accumulate_pattern = 4;
    CustomAccumulateParseResult custom_accumulate = 5;
  }
}
```

Append these new messages at the end of the file:

```protobuf
message CustomAccumulateParseResult {
  PatternParseResult source = 1;
  repeated InitVarParseResult init_vars = 2;
  string action_block = 3;
  string reverse_block = 4;
  string result_type_name = 5;
  string result_bind_name = 6;
  string result_expression = 7;
  repeated string referenced_bindings = 8;
}

message InitVarParseResult {
  string type_name = 1;
  string name = 2;
  string initializer = 3;
}
```

- [ ] **Step 2: Regenerate protobuf**

Run:
```bash
mvn -f drlx-parser-core/pom.xml generate-sources -q
```
Expected: BUILD SUCCESS, generated Java includes `CustomAccumulateParseResult` and `InitVarParseResult`.

- [ ] **Step 3: Add serialization branch to toProtoLhs()**

In `DrlxRuleAstParseResult.toProtoLhs()`, add a new branch after the `AccumulatePatternIR` branch (after line 194):

```java
} else if (item instanceof CustomAccumulateIR customAcc) {
    DrlxRuleAstProto.CustomAccumulateParseResult.Builder cab =
            DrlxRuleAstProto.CustomAccumulateParseResult.newBuilder()
                    .setSource(patternToProto(customAcc.source()))
                    .setActionBlock(customAcc.actionBlock())
                    .setReverseBlock(customAcc.reverseBlock() != null ? customAcc.reverseBlock() : "")
                    .setResultTypeName(customAcc.resultTypeName())
                    .setResultBindName(customAcc.resultBindName())
                    .setResultExpression(customAcc.resultExpression());
    for (InitVarIR iv : customAcc.initVars()) {
        cab.addInitVars(DrlxRuleAstProto.InitVarParseResult.newBuilder()
                .setTypeName(iv.typeName())
                .setName(iv.name())
                .setInitializer(iv.initializer()));
    }
    customAcc.referencedBindings().forEach(cab::addReferencedBindings);
    builder.setCustomAccumulate(cab);
```

Add the import:

```java
import org.drools.drlx.builder.DrlxRuleAstModel.CustomAccumulateIR;
import org.drools.drlx.builder.DrlxRuleAstModel.InitVarIR;
```

- [ ] **Step 4: Add deserialization branch to fromProtoLhs()**

In `DrlxRuleAstParseResult.fromProtoLhs()`, add a new case before `KIND_NOT_SET`:

```java
case CUSTOM_ACCUMULATE -> {
    DrlxRuleAstProto.CustomAccumulateParseResult cap = item.getCustomAccumulate();
    PatternIR srcIr = patternFromProto(cap.getSource());
    List<InitVarIR> initVars = new ArrayList<>(cap.getInitVarsCount());
    for (DrlxRuleAstProto.InitVarParseResult iv : cap.getInitVarsList()) {
        initVars.add(new InitVarIR(iv.getTypeName(), iv.getName(), iv.getInitializer()));
    }
    String reverseBlock = cap.getReverseBlock().isEmpty() ? null : cap.getReverseBlock();
    yield new CustomAccumulateIR(srcIr, initVars,
            cap.getActionBlock(), reverseBlock,
            cap.getResultTypeName(), cap.getResultBindName(),
            cap.getResultExpression(),
            List.copyOf(cap.getReferencedBindingsList()));
}
```

- [ ] **Step 5: Verify compilation**

Run:
```bash
mvn -f drlx-parser-core/pom.xml compile -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add drlx-parser-core/src/main/proto/drlx_rule_ast.proto drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java
git commit -m "feat(proto): add CustomAccumulateParseResult for acc() persistence

Refs #51"
```

---

### Task 5: Runtime — create DrlxCustomAccumulator

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxCustomAccumulator.java`

- [ ] **Step 1: Write the failing test**

Add to `AccumulateTest.java`:

```java
@Test
void accKeyword3ParamSumWithoutReverse() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                acc(var p : /persons,
                    int s = 0;,
                    s = s + p.age,
                    int sum = s),
                do { results.add(sum); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("A", 20));
        entryPoint.insert(new Person("B", 40));
        entryPoint.insert(new Person("C", 60));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(120);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -Dtest=AccumulateTest#accKeyword3ParamSumWithoutReverse -q
```
Expected: FAIL — `DrlxCustomAccumulator` class does not exist / no `CustomAccumulateIR` handler in runtime builder.

- [ ] **Step 3: Create DrlxCustomAccumulator.java**

```java
package org.drools.drlx.builder;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.drools.base.base.ValueResolver;
import org.drools.base.reteoo.BaseTuple;
import org.drools.base.rule.Declaration;
import org.drools.base.rule.accessor.Accumulator;
import org.drools.drlx.builder.DrlxRuleAstModel.InitVarIR;
import org.mvel3.Evaluator;
import org.kie.api.runtime.rule.FactHandle;

public final class DrlxCustomAccumulator implements Accumulator {

    private final List<InitVarIR> initVars;
    private final String srcBindingName;
    private final Map<String, Object> initDefaults;

    private Evaluator<Map<String, Object>, Void, ?> actionEval;
    private Evaluator<Map<String, Object>, Void, ?> reverseEval;
    private Evaluator<Map<String, Object>, Void, Object> resultEval;

    public DrlxCustomAccumulator(List<InitVarIR> initVars, String srcBindingName) {
        this.initVars = initVars;
        this.srcBindingName = srcBindingName;
        this.initDefaults = buildDefaults(initVars);
    }

    void setActionEval(Evaluator<Map<String, Object>, Void, ?> eval) { this.actionEval = eval; }
    void setReverseEval(Evaluator<Map<String, Object>, Void, ?> eval) { this.reverseEval = eval; }
    void setResultEval(Evaluator<Map<String, Object>, Void, Object> eval) { this.resultEval = eval; }

    @Override public Object createWorkingMemoryContext() { return null; }

    @Override
    public Object createContext() {
        return new HashMap<String, Object>(initDefaults.size() + 1);
    }

    @Override
    public Object init(Object wmContext, Object context, BaseTuple tuple,
                       Declaration[] decls, ValueResolver vr) {
        @SuppressWarnings("unchecked")
        Map<String, Object> map = (Map<String, Object>) context;
        map.putAll(initDefaults);
        return context;
    }

    @Override
    public Object accumulate(Object wmContext, Object context, BaseTuple tuple,
                             FactHandle handle, Declaration[] decls,
                             Declaration[] innerDecls, ValueResolver vr) {
        @SuppressWarnings("unchecked")
        Map<String, Object> map = (Map<String, Object>) context;
        Object srcFact = handle.getObject();
        map.put(srcBindingName, srcFact);
        try {
            actionEval.eval(map);
        } finally {
            map.remove(srcBindingName);
        }
        return srcFact;
    }

    @Override
    public boolean supportsReverse() { return reverseEval != null; }

    @Override
    public boolean tryReverse(Object wmContext, Object context, BaseTuple tuple,
                              FactHandle handle, Object value,
                              Declaration[] decls, Declaration[] innerDecls,
                              ValueResolver vr) {
        if (reverseEval == null) return false;
        @SuppressWarnings("unchecked")
        Map<String, Object> map = (Map<String, Object>) context;
        map.put(srcBindingName, value);
        try {
            reverseEval.eval(map);
        } finally {
            map.remove(srcBindingName);
        }
        return true;
    }

    @Override
    public Object getResult(Object wmContext, Object context, BaseTuple tuple,
                            Declaration[] decls, ValueResolver vr) {
        @SuppressWarnings("unchecked")
        Map<String, Object> map = (Map<String, Object>) context;
        return resultEval.eval(map);
    }

    static final class ActionSink implements EvaluatorSink {
        private final DrlxCustomAccumulator parent;
        ActionSink(DrlxCustomAccumulator parent) { this.parent = parent; }
        @SuppressWarnings("unchecked")
        @Override public void bindEvaluator(Evaluator<?, ?, ?> evaluator) {
            parent.setActionEval((Evaluator<Map<String, Object>, Void, ?>) evaluator);
        }
    }

    static final class ReverseSink implements EvaluatorSink {
        private final DrlxCustomAccumulator parent;
        ReverseSink(DrlxCustomAccumulator parent) { this.parent = parent; }
        @SuppressWarnings("unchecked")
        @Override public void bindEvaluator(Evaluator<?, ?, ?> evaluator) {
            parent.setReverseEval((Evaluator<Map<String, Object>, Void, ?>) evaluator);
        }
    }

    static final class ResultSink implements EvaluatorSink {
        private final DrlxCustomAccumulator parent;
        ResultSink(DrlxCustomAccumulator parent) { this.parent = parent; }
        @SuppressWarnings("unchecked")
        @Override public void bindEvaluator(Evaluator<?, ?, ?> evaluator) {
            parent.setResultEval((Evaluator<Map<String, Object>, Void, Object>) evaluator);
        }
    }

    private static Map<String, Object> buildDefaults(List<InitVarIR> initVars) {
        Map<String, Object> defaults = new HashMap<>(initVars.size());
        for (InitVarIR iv : initVars) {
            defaults.put(iv.name(), parseLiteralValue(iv.initializer(), iv.typeName()));
        }
        return defaults;
    }

    static Object parseLiteralValue(String initializer, String typeName) {
        if (initializer == null || initializer.equals("null")) return null;
        if (initializer.equals("true")) return Boolean.TRUE;
        if (initializer.equals("false")) return Boolean.FALSE;
        if (initializer.startsWith("\"") && initializer.endsWith("\"")) {
            return initializer.substring(1, initializer.length() - 1);
        }
        if (initializer.startsWith("'") && initializer.endsWith("'") && initializer.length() == 3) {
            return initializer.charAt(1);
        }
        return parseNumericValue(initializer, typeName);
    }

    private static Object parseNumericValue(String initializer, String typeName) {
        String lower = initializer.toLowerCase();
        String numStr = lower;
        if (numStr.endsWith("l") || numStr.endsWith("f") || numStr.endsWith("d")) {
            numStr = numStr.substring(0, numStr.length() - 1);
        }
        return switch (typeName) {
            case "int", "Integer" -> Integer.parseInt(numStr);
            case "long", "Long" -> Long.parseLong(numStr);
            case "double", "Double" -> Double.parseDouble(numStr);
            case "float", "Float" -> Float.parseFloat(numStr);
            case "short", "Short" -> Short.parseShort(numStr);
            case "byte", "Byte" -> Byte.parseByte(numStr);
            default -> {
                if (lower.contains(".") || lower.endsWith("d")) yield Double.parseDouble(numStr);
                if (lower.endsWith("f")) yield Float.parseFloat(numStr);
                if (lower.endsWith("l")) yield Long.parseLong(numStr);
                yield Integer.parseInt(numStr);
            }
        };
    }
}
```

- [ ] **Step 4: Verify compilation**

Run:
```bash
mvn -f drlx-parser-core/pom.xml compile -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxCustomAccumulator.java
git commit -m "feat(runtime): add DrlxCustomAccumulator with map-based MVEL3 evaluators

Refs #51"
```

---

### Task 6: Lambda compiler — add createCustomAccumulator() method

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java` (append method)

- [ ] **Step 1: Add createCustomAccumulator() method**

Add this method to `DrlxLambdaCompiler`:

```java
@SuppressWarnings({"unchecked", "rawtypes"})
public DrlxCustomAccumulator createCustomAccumulator(
        DrlxRuleAstModel.CustomAccumulateIR ir,
        Class<?> srcClass,
        String srcBindingName) {

    DrlxCustomAccumulator acc = new DrlxCustomAccumulator(ir.initVars(), srcBindingName);

    // Build MVEL3 declarations for init vars + source binding
    List<org.mvel3.transpiler.context.Declaration<?>> holderDecls = new ArrayList<>();
    for (DrlxRuleAstModel.InitVarIR iv : ir.initVars()) {
        holderDecls.add(org.mvel3.transpiler.context.Declaration.of(iv.name(), resolveInitVarType(iv.typeName())));
    }

    List<org.mvel3.transpiler.context.Declaration<?>> actionDecls = new ArrayList<>(holderDecls);
    actionDecls.add(org.mvel3.transpiler.context.Declaration.of(srcBindingName, srcClass));
    org.mvel3.transpiler.context.Declaration<?>[] actionDeclArray =
            actionDecls.toArray(new org.mvel3.transpiler.context.Declaration[0]);

    // Action block
    {
        int counter = lambdaCounter++;
        String normalizedAction = normalizeBlockText(ir.actionBlock());
        Evaluator<Map<String, Object>, Void, ?> preCompiled =
                (Evaluator) tryLoadPreCompiled(counter, normalizedAction, "custom acc action");
        if (preCompiled != null) {
            acc.setActionEval(preCompiled);
        } else {
            CompilerParameters<Map<String, Object>, Void, String> evalInfo =
                    (CompilerParameters) MVEL.<Object>map(actionDeclArray)
                            .<String>out(String.class)
                            .block(normalizedAction + RETURN_NULL)
                            .imports(new HashSet<>(imports))
                            .classManager(batchCompiler.getClassManager())
                            .generatedClassName("GeneratorEvaluator__")
                            .build();
            MVELBatchCompiler.LambdaHandle handle = batchCompiler.add(evalInfo);
            pendingLambdas.add(new PendingLambda(handle, new DrlxCustomAccumulator.ActionSink(acc)));
            onLambdaCreated(counter, normalizedAction);
        }
    }

    // Reverse block (optional)
    if (ir.reverseBlock() != null) {
        int counter = lambdaCounter++;
        String normalizedReverse = normalizeBlockText(ir.reverseBlock());
        Evaluator<Map<String, Object>, Void, ?> preCompiled =
                (Evaluator) tryLoadPreCompiled(counter, normalizedReverse, "custom acc reverse");
        if (preCompiled != null) {
            acc.setReverseEval(preCompiled);
        } else {
            CompilerParameters<Map<String, Object>, Void, String> evalInfo =
                    (CompilerParameters) MVEL.<Object>map(actionDeclArray)
                            .<String>out(String.class)
                            .block(normalizedReverse + RETURN_NULL)
                            .imports(new HashSet<>(imports))
                            .classManager(batchCompiler.getClassManager())
                            .generatedClassName("GeneratorEvaluator__")
                            .build();
            MVELBatchCompiler.LambdaHandle handle = batchCompiler.add(evalInfo);
            pendingLambdas.add(new PendingLambda(handle, new DrlxCustomAccumulator.ReverseSink(acc)));
            onLambdaCreated(counter, normalizedReverse);
        }
    }

    // Result expression
    {
        int counter = lambdaCounter++;
        org.mvel3.transpiler.context.Declaration<?>[] holderDeclArray =
                holderDecls.toArray(new org.mvel3.transpiler.context.Declaration[0]);
        Class<?> resultClass = resolveInitVarType(ir.resultTypeName());

        Evaluator<Map<String, Object>, Void, Object> preCompiled =
                (Evaluator) tryLoadPreCompiled(counter, ir.resultExpression(), "custom acc result");
        if (preCompiled != null) {
            acc.setResultEval(preCompiled);
        } else {
            CompilerParameters<Map<String, Object>, Void, Object> evalInfo =
                    (CompilerParameters) MVEL.<Object>map(holderDeclArray)
                            .<Object>out(resultClass)
                            .expression(ir.resultExpression())
                            .imports(new HashSet<>(imports))
                            .classManager(batchCompiler.getClassManager())
                            .generatedClassName("GeneratorEvaluator__")
                            .build();
            MVELBatchCompiler.LambdaHandle handle = batchCompiler.add(evalInfo);
            pendingLambdas.add(new PendingLambda(handle, new DrlxCustomAccumulator.ResultSink(acc)));
            onLambdaCreated(counter, ir.resultExpression());
        }
    }

    return acc;
}

private static String normalizeBlockText(String text) {
    if (text == null) return "";
    String trimmed = text.trim();
    if (trimmed.endsWith(";")) {
        return trimmed;
    }
    return trimmed + ";";
}

// Returns primitive classes (int.class, double.class) for MVEL3 compile-time
// declarations. MVEL3 handles boxing internally when the map stores boxed values.
// This is distinct from resolveCustomResultType() in the runtime builder which
// returns boxed classes (Integer.class) for Drools Pattern ObjectType wrappers.
private static Class<?> resolveInitVarType(String typeName) {
    return switch (typeName) {
        case "int"     -> int.class;
        case "long"    -> long.class;
        case "double"  -> double.class;
        case "float"   -> float.class;
        case "short"   -> short.class;
        case "byte"    -> byte.class;
        case "boolean" -> boolean.class;
        case "char"    -> char.class;
        case "Integer" -> Integer.class;
        case "Long"    -> Long.class;
        case "Double"  -> Double.class;
        case "Float"   -> Float.class;
        case "Short"   -> Short.class;
        case "Byte"    -> Byte.class;
        case "Boolean" -> Boolean.class;
        case "Character" -> Character.class;
        case "String"  -> String.class;
        default -> {
            try { yield Class.forName(typeName); }
            catch (ClassNotFoundException e) {
                throw new RuntimeException(
                        "cannot resolve type '" + typeName + "' in custom accumulate — use a fully-qualified name or add an import");
            }
        }
    };
}
```

Add the required import at top of the file:

```java
import java.util.HashSet;
import org.mvel3.CompilerParameters;
```

Note: `HashSet` and `CompilerParameters` may already be imported. Check before adding.

- [ ] **Step 2: Make tryLoadPreCompiled() accessible for the new method**

The `tryLoadPreCompiled()` method is already `private` and accessible within the class. No changes needed since `createCustomAccumulator()` is in the same class.

- [ ] **Step 3: Verify compilation**

Run:
```bash
mvn -f drlx-parser-core/pom.xml compile -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java
git commit -m "feat(compiler): add createCustomAccumulator() to DrlxLambdaCompiler

Registers action/reverse/result as separate PendingLambda slots
for batch compilation.

Refs #51"
```

---

### Task 7: Runtime builder — wire CustomAccumulateIR into DrlxRuleAstRuntimeBuilder

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Add CustomAccumulateIR import**

```java
import org.drools.drlx.builder.DrlxRuleAstModel.CustomAccumulateIR;
import org.drools.drlx.builder.DrlxRuleAstModel.InitVarIR;
```

- [ ] **Step 2: Add CustomAccumulateIR branch to collectPatternClasses()**

In `collectPatternClasses()`, after the `AccumulatePatternIR` branch (after line 154), add:

```java
} else if (item instanceof CustomAccumulateIR customAcc) {
    classes.add(resolvePatternType(customAcc.source(), typeResolver, entryPointTypes, unitClass));
}
```

- [ ] **Step 3: Add CustomAccumulateIR branch to buildLhs()**

In `buildLhs()`, after the `AccumulatePatternIR` branch (after line 387), add:

```java
} else if (item instanceof CustomAccumulateIR customAcc) {
    buildCustomAccumulatePattern(customAcc, parent, typeResolver, entryPointTypes,
                                 unitClass, boundVariables);
```

- [ ] **Step 4: Add buildCustomAccumulatePattern() method**

```java
private void buildCustomAccumulatePattern(CustomAccumulateIR customAcc,
                                           GroupElement parent,
                                           TypeResolver typeResolver,
                                           Map<String, Class<?>> entryPointTypes,
                                           Class<?> unitClass,
                                           Map<String, BoundVariable> outerScope) {
    PatternIR srcIr = customAcc.source();
    Pattern srcPattern = buildPattern(srcIr, typeResolver, entryPointTypes, unitClass, outerScope);
    Class<?> srcClass = ((ClassObjectType) srcPattern.getObjectType()).getClassType();
    Declaration srcDecl = srcPattern.getDeclaration();
    String srcBindingName = srcDecl != null ? srcDecl.getIdentifier() : null;

    DrlxCustomAccumulator accumulator =
            lambdaCompiler.createCustomAccumulator(customAcc, srcClass, srcBindingName);

    Declaration[] required = new Declaration[0];
    SingleAccumulate single = new SingleAccumulate(srcPattern, required, accumulator);

    Class<?> resultClass = resolveCustomResultType(customAcc.resultTypeName(), typeResolver);
    Pattern wrap = new Pattern(lambdaCompiler.nextPatternId(), new ClassObjectType(resultClass),
                               customAcc.resultBindName());
    wrap.addDeclaration(new Declaration(customAcc.resultBindName(),
            new SelfReferenceClassFieldReader(resultClass), wrap, true));
    wrap.setSource(single);

    parent.addChild(wrap);

    Declaration decl = wrap.getDeclarations().get(customAcc.resultBindName());
    outerScope.put(customAcc.resultBindName(),
            new BoundVariable(customAcc.resultBindName(), resultClass, wrap, decl));
}

private static Class<?> resolveCustomResultType(String typeName, TypeResolver typeResolver) {
    return switch (typeName) {
        case "int"     -> Integer.class;
        case "long"    -> Long.class;
        case "double"  -> Double.class;
        case "float"   -> Float.class;
        case "short"   -> Short.class;
        case "byte"    -> Byte.class;
        case "boolean" -> Boolean.class;
        case "char"    -> Character.class;
        default -> {
            try { yield typeResolver.resolveType(typeName); }
            catch (ClassNotFoundException e) {
                throw new RuntimeException(
                        "cannot resolve type '" + typeName + "' in custom accumulate result — use a fully-qualified name or add an import", e);
            }
        }
    };
}
```

- [ ] **Step 5: Add outer-binding rejection in buildCustomAccumulatePattern()**

After building the source pattern and before calling `createCustomAccumulator`, validate that action/reverse/result blocks don't reference outer-scope bindings:

```java
    // Reject outer-binding references in action/reverse/result blocks (#54)
    java.util.Set<String> allowedNames = new java.util.LinkedHashSet<>();
    allowedNames.add(srcBindingName);
    for (InitVarIR iv : customAcc.initVars()) {
        allowedNames.add(iv.name());
    }
    for (String ref : customAcc.referencedBindings()) {
        if (!allowedNames.contains(ref) && outerScope.containsKey(ref)) {
            throw new RuntimeException(
                    "outer-binding reference '" + ref + "' in custom accumulate is not yet supported (see #54)");
        }
    }
```

Insert this block in `buildCustomAccumulatePattern()` after determining `srcBindingName` and before calling `lambdaCompiler.createCustomAccumulator(...)`.

- [ ] **Step 6: Install and run the failing test from Task 5**

Run:
```bash
mvn -f drlx-parser-core/pom.xml install -q && mvn -f drlx-parser-core/pom.xml test -pl . -Dtest=AccumulateTest#accKeyword3ParamSumWithoutReverse -q
```
Expected: PASS — the 3-param custom accumulate computes sum = 120.

- [ ] **Step 7: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
git commit -m "feat(builder): wire CustomAccumulateIR into DrlxRuleAstRuntimeBuilder

Refs #51"
```

---

### Task 8: Integration tests — full acc() runtime tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Add 2-param runtime tests**

```java
@Test
void accKeyword2ParamSingleFunction() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                acc(var p : /persons,
                    var avgAge = avg(p.age)),
                do { results.add(avgAge); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("A", 20));
        entryPoint.insert(new Person("B", 40));
        entryPoint.insert(new Person("C", 60));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(40.0);
}

@Test
void accKeyword2ParamGroupedFunctions() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                acc(var p : /persons,
                    (var maxAge = max(p.age),
                     var minAge = min(p.age))),
                do { results.add(maxAge); results.add(minAge); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("A", 20));
        entryPoint.insert(new Person("B", 40));
        entryPoint.insert(new Person("C", 60));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(60, 20);
}
```

- [ ] **Step 2: Add 3-param with reverse test**

```java
@Test
void accKeyword3ParamSumWithReverse() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                acc(var p : /persons,
                    int s = 0;,
                    (s = s + p.age, s = s - p.age),
                    int sum = s),
                do { results.add(sum); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        FactHandle h1 = entryPoint.insert(new Person("A", 20));
        entryPoint.insert(new Person("B", 40));
        entryPoint.insert(new Person("C", 60));
        kieSession.fireAllRules();

        observed.clear();
        entryPoint.delete(h1);
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(100);
}
```

Add the import for `FactHandle`:

```java
import org.kie.api.runtime.rule.FactHandle;
```

- [ ] **Step 3: Add 5-param avg test**

```java
@Test
void accKeyword5ParamAvg() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                acc(var p : /persons,
                    { int count = 0; int total = 0; },
                    { total += p.age; count++; },
                    { total -= p.age; count--; },
                    double avgAge = (double) total / count),
                do { results.add(avgAge); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("A", 20));
        entryPoint.insert(new Person("B", 40));
        entryPoint.insert(new Person("C", 60));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(40.0);
}
```

- [ ] **Step 4: Add 5-param with retraction test**

```java
@Test
void accKeyword5ParamWithRetraction() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                acc(var p : /persons,
                    { int count = 0; int total = 0; },
                    { total += p.age; count++; },
                    { total -= p.age; count--; },
                    double avgAge = (double) total / count),
                do { results.add(avgAge); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        FactHandle h1 = entryPoint.insert(new Person("A", 20));
        entryPoint.insert(new Person("B", 40));
        entryPoint.insert(new Person("C", 60));
        kieSession.fireAllRules();

        observed.clear();
        entryPoint.delete(h1);
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(50.0);
}
```

- [ ] **Step 5: Run all accumulate tests**

Run:
```bash
mvn -f drlx-parser-core/pom.xml install -q && mvn -f drlx-parser-core/pom.xml test -pl . -Dtest=AccumulateTest -q
```
Expected: ALL PASS.

- [ ] **Step 6: Commit**

```bash
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git commit -m "test: add runtime integration tests for acc() keyword forms

Covers 2-param single/grouped, 3-param with/without reverse,
5-param avg with/without retraction.

Refs #51"
```

---

### Task 9: Protobuf round-trip test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java` (or new test class)

- [ ] **Step 1: Add protobuf round-trip test**

Add to `AccumulateVisitorTest.java`:

```java
@Test
void customAccumulateIRProtobufRoundTrip() {
    var original = new DrlxRuleAstModel.CustomAccumulateIR(
            new DrlxRuleAstModel.PatternIR("var", "p", "persons", List.of(), null, List.of(), false, List.of()),
            List.of(
                    new DrlxRuleAstModel.InitVarIR("int", "count", "0"),
                    new DrlxRuleAstModel.InitVarIR("int", "total", "0")),
            "total += p.age; count++;",
            "total -= p.age; count--;",
            "double",
            "avgAge",
            "(double) total / count",
            List.of("p", "total", "count"));

    var protoLhs = DrlxRuleAstParseResult.toProtoLhs(original);
    var roundTripped = DrlxRuleAstParseResult.fromProtoLhs(protoLhs, java.nio.file.Path.of("test"));

    assertThat(roundTripped).isInstanceOf(DrlxRuleAstModel.CustomAccumulateIR.class);
    var rt = (DrlxRuleAstModel.CustomAccumulateIR) roundTripped;
    assertThat(rt.source().bindName()).isEqualTo("p");
    assertThat(rt.initVars()).hasSize(2);
    assertThat(rt.initVars().get(0).name()).isEqualTo("count");
    assertThat(rt.initVars().get(1).name()).isEqualTo("total");
    assertThat(rt.actionBlock()).isEqualTo("total += p.age; count++;");
    assertThat(rt.reverseBlock()).isEqualTo("total -= p.age; count--;");
    assertThat(rt.resultTypeName()).isEqualTo("double");
    assertThat(rt.resultBindName()).isEqualTo("avgAge");
    assertThat(rt.resultExpression()).isEqualTo("(double) total / count");
    assertThat(rt.referencedBindings()).containsExactly("p", "total", "count");
}

@Test
void customAccumulateIRProtobufRoundTripNullReverse() {
    var original = new DrlxRuleAstModel.CustomAccumulateIR(
            new DrlxRuleAstModel.PatternIR("var", "p", "persons", List.of(), null, List.of(), false, List.of()),
            List.of(new DrlxRuleAstModel.InitVarIR("int", "s", "0")),
            "s = s + p.age",
            null,
            "int",
            "sum",
            "s",
            List.of("p", "s"));

    var protoLhs = DrlxRuleAstParseResult.toProtoLhs(original);
    var roundTripped = DrlxRuleAstParseResult.fromProtoLhs(protoLhs, java.nio.file.Path.of("test"));

    var rt = (DrlxRuleAstModel.CustomAccumulateIR) roundTripped;
    assertThat(rt.reverseBlock()).isNull();
    assertThat(rt.actionBlock()).isEqualTo("s = s + p.age");
}
```

- [ ] **Step 2: Run the round-trip test**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -Dtest=AccumulateVisitorTest#customAccumulateIRProtobufRoundTrip,AccumulateVisitorTest#customAccumulateIRProtobufRoundTripNullReverse -q
```
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java
git commit -m "test: add protobuf round-trip tests for CustomAccumulateIR

Refs #51"
```

---

### Task 10: Full test suite validation

- [ ] **Step 1: Run the full test suite**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml clean install -q
```
Expected: BUILD SUCCESS with all tests passing.

- [ ] **Step 2: Verify no regressions in existing accumulate tests**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -Dtest=AccumulateVisitorTest,AccumulateTest -q
```
Expected: ALL PASS — no existing test breaks.

- [ ] **Step 3: Commit any fixups if needed**

If any test adjustments were needed for compilation (e.g. exhaustive switch fixes), commit them here.

```bash
git add -u
git commit -m "fix: address any compilation/exhaustiveness issues from sealed interface change

Refs #51"
```
