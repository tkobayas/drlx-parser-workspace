# Named Windows Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Support named window declarations at the package level and window references from rules in DRLX, mapping to existing drools `WindowDeclaration` / `WindowReference` runtime objects.

**Architecture:** Grammar adds a `window` keyword and `windowDeclaration` rule to `drlxCompilationUnit`. The visitor produces a new `WindowDeclarationIR` record in `CompilationUnitIR`. The runtime builder compiles declarations into `WindowDeclaration` objects registered in the package, and detects window references in rule patterns by checking `entryPoint` against declared window names — setting `WindowReference` as the pattern source with the declaration's type.

**Tech Stack:** ANTLR4 grammar, Java 17 records, protobuf, drools-base runtime objects (`WindowDeclaration`, `WindowReference`, `SlidingTimeWindow`, `SlidingLengthWindow`)

---

### Task 1: Grammar & Lexer — add `WINDOW` keyword and `windowDeclaration` rule

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4`
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`

- [ ] **Step 1: Add WINDOW keyword to lexer**

In `DrlxLexer.g4`, add after the `MATCH` line:

```
WINDOW : 'window';
```

- [ ] **Step 2: Add `windowDeclaration` rule and update `drlxCompilationUnit` in parser**

In `DrlxParser.g4`, replace:

```
drlxCompilationUnit
    : packageDeclaration? importDeclaration* unitDeclaration ruleDeclaration*
    ;
```

with:

```
drlxCompilationUnit
    : packageDeclaration? importDeclaration* unitDeclaration windowDeclaration* ruleDeclaration*
    ;
```

Add the new rule after `unitDeclaration`:

```
// Named window declaration — DRLXXXX §Windows.
// Declares a reusable windowed pattern at the package level.
// Body is an OOPath expression with a mandatory window filter.
windowDeclaration
    : WINDOW identifier '{' oopathExpression windowFilter '}'
    ;
```

- [ ] **Step 3: Rebuild ANTLR sources and verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources compile -q`

Expected: BUILD SUCCESS, generated parser includes `windowDeclaration` methods.

- [ ] **Step 4: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "feat(grammar): add WINDOW keyword and windowDeclaration rule

Refs #72"
```

---

### Task 2: IR model — add `WindowDeclarationIR` and update `CompilationUnitIR`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`

- [ ] **Step 1: Add `WindowDeclarationIR` record**

Add after the `CompilationUnitIR` record (around line 19):

```java
public record WindowDeclarationIR(String name, PatternIR pattern) { }
```

- [ ] **Step 2: Update `CompilationUnitIR` to include window declarations**

Change:

```java
public record CompilationUnitIR(String packageName,
                                String unitName,
                                List<String> imports,
                                List<RuleIR> rules) {
}
```

to:

```java
public record CompilationUnitIR(String packageName,
                                String unitName,
                                List<String> imports,
                                List<WindowDeclarationIR> windowDeclarations,
                                List<RuleIR> rules) {
}
```

- [ ] **Step 3: Fix all compilation errors from the new `CompilationUnitIR` parameter**

Every call site that constructs `CompilationUnitIR` needs an additional `List<WindowDeclarationIR>` argument. There are two:

**`DrlxToRuleAstVisitor.visitDrlxCompilationUnit()` (~line 115):**

Change:

```java
return new CompilationUnitIR(packageName, unitName, List.copyOf(imports), List.copyOf(rules));
```

to:

```java
return new CompilationUnitIR(packageName, unitName, List.copyOf(imports), List.of(), List.copyOf(rules));
```

(Temporary empty list — Task 3 will populate it.)

**`DrlxRuleAstParseResult.load()` (~line 104):**

Change:

```java
return new CompilationUnitIR(parseResult.getPackageName(),
        parseResult.getUnitName(),
        List.copyOf(parseResult.getImportsList()),
        List.copyOf(rules));
```

to:

```java
return new CompilationUnitIR(parseResult.getPackageName(),
        parseResult.getUnitName(),
        List.copyOf(parseResult.getImportsList()),
        List.of(),
        List.copyOf(rules));
```

(Temporary empty list — Task 5 will populate it with deserialized window declarations.)

- [ ] **Step 4: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "feat(ir): add WindowDeclarationIR and update CompilationUnitIR

Refs #72"
```

---

### Task 3: Visitor — parse window declarations into `WindowDeclarationIR`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/WindowVisitorTest.java`

- [ ] **Step 1: Write failing visitor-level tests**

Add these tests to `WindowVisitorTest.java`:

```java
@Test
void parsesNamedWindowDeclaration() {
    String drlx = """
            package p;
            unit MyUnit;
            window WithdrawalWindow {
                /withdrawals | time[10s]
            }
            rule R1 {
                var w : /withdrawals,
                do {}
            }
            """;
    var unit = parseUnit(drlx);
    assertThat(unit.windowDeclarations()).hasSize(1);
    var windowDecl = unit.windowDeclarations().get(0);
    assertThat(windowDecl.name()).isEqualTo("WithdrawalWindow");
    assertThat(windowDecl.pattern().entryPoint()).isEqualTo("withdrawals");
    assertThat(windowDecl.pattern().windowType()).isEqualTo("time");
    assertThat(windowDecl.pattern().windowParameter()).isEqualTo("10s");
}

@Test
void parsesNamedWindowWithConstraints() {
    String drlx = """
            package p;
            unit MyUnit;
            window GoldWithdrawalWindow {
                /withdrawals[customer == "GOLD"] | time[5s]
            }
            rule R1 {
                var w : /withdrawals,
                do {}
            }
            """;
    var unit = parseUnit(drlx);
    var windowDecl = unit.windowDeclarations().get(0);
    assertThat(windowDecl.name()).isEqualTo("GoldWithdrawalWindow");
    assertThat(windowDecl.pattern().conditions()).containsExactly("customer == \"GOLD\"");
    assertThat(windowDecl.pattern().windowType()).isEqualTo("time");
    assertThat(windowDecl.pattern().windowParameter()).isEqualTo("5s");
}

@Test
void parsesNamedLengthWindowDeclaration() {
    String drlx = """
            package p;
            unit MyUnit;
            window RecentWithdrawals {
                /withdrawals | length[5]
            }
            rule R1 {
                var w : /withdrawals,
                do {}
            }
            """;
    var unit = parseUnit(drlx);
    var windowDecl = unit.windowDeclarations().get(0);
    assertThat(windowDecl.name()).isEqualTo("RecentWithdrawals");
    assertThat(windowDecl.pattern().windowType()).isEqualTo("length");
    assertThat(windowDecl.pattern().windowParameter()).isEqualTo("5");
}
```

Also add the `parseUnit` helper method:

```java
private static DrlxRuleAstModel.CompilationUnitIR parseUnit(String drlx) {
    DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(drlx));
    CommonTokenStream tokens = new CommonTokenStream(lexer);
    DrlxParser parser = new DrlxParser(tokens);
    DrlxParser.DrlxCompilationUnitContext ctx = parser.drlxStart().drlxCompilationUnit();
    return new DrlxToRuleAstVisitor(tokens).visitDrlxCompilationUnit(ctx);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=WindowVisitorTest -q`

Expected: FAIL — `windowDeclarations()` returns empty list.

- [ ] **Step 3: Implement `visitWindowDeclaration` in `DrlxToRuleAstVisitor`**

Add the import:

```java
import org.drools.drlx.builder.DrlxRuleAstModel.WindowDeclarationIR;
```

Add a new method:

```java
private WindowDeclarationIR visitWindowDeclaration(DrlxParser.WindowDeclarationContext ctx) {
    String name = ctx.identifier().getText();
    DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
    String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
    String castTypeName = extractCastType(oopathCtx);
    List<String> conditions = extractConditions(oopathCtx);
    List<TemporalConditionIR> temporalConditions = extractTemporalConditions(oopathCtx);
    List<String> positionalArgs = extractPositionalArgs(oopathCtx);
    boolean passive = oopathCtx.QUESTION() != null;
    List<String> watchedProperties = extractWatchedProperties(oopathCtx);
    String windowType = ctx.windowFilter().identifier().getText();
    if (!"length".equals(windowType) && !"time".equals(windowType)) {
        throw new IllegalArgumentException("Unknown window type: " + windowType
                + ". Expected 'length' or 'time'.");
    }
    String windowParameter = ctx.windowFilter().windowParam().getText();
    PatternIR pattern = new PatternIR("", "", entryPoint, conditions, temporalConditions,
                                       castTypeName, positionalArgs, passive, watchedProperties,
                                       windowType, windowParameter);
    return new WindowDeclarationIR(name, pattern);
}
```

Update `visitDrlxCompilationUnit` — change the return statement section (around line 105–115):

Replace:

```java
return new CompilationUnitIR(packageName, unitName, List.copyOf(imports), List.of(), List.copyOf(rules));
```

with:

```java
List<WindowDeclarationIR> windowDeclarations = new ArrayList<>();
if (ctx.windowDeclaration() != null) {
    ctx.windowDeclaration().forEach(wCtx -> windowDeclarations.add(visitWindowDeclaration(wCtx)));
}

return new CompilationUnitIR(packageName, unitName, List.copyOf(imports),
                              List.copyOf(windowDeclarations), List.copyOf(rules));
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=WindowVisitorTest -q`

Expected: PASS

- [ ] **Step 5: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "feat(visitor): parse window declarations into WindowDeclarationIR

Refs #72"
```

---

### Task 4: Runtime builder — compile window declarations and resolve references

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/NamedWindowTest.java`

- [ ] **Step 1: Write failing session-level tests**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/NamedWindowTest.java`:

```java
package org.drools.drlx.builder;

import java.util.concurrent.TimeUnit;

import org.drools.core.ClockType;
import org.drools.core.SessionConfiguration;
import org.drools.core.impl.RuleBaseFactory;
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
class NamedWindowTest {

    private static KieBase buildWithStreamMode(String drlx) {
        KieBaseConfiguration config = RuleBaseFactory.newKnowledgeBaseConfiguration();
        config.setOption(EventProcessingOption.STREAM);
        return new DrlxRuleBuilder().build(drlx, config);
    }

    @Test
    void namedTimeWindowExpiresOldEvents() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                window WithdrawalWindow {
                    /withdrawals | time[5s]
                }
                rule R1 {
                    var w : /withdrawalWindow,
                    do {}
                }
                """;
        KieBase kieBase = buildWithStreamMode(drlx);

        SessionConfiguration sessionConfig = RuleBaseFactory.newKnowledgeSessionConfiguration()
                .as(SessionConfiguration.KEY);
        sessionConfig.setClockType(ClockType.PSEUDO_CLOCK);

        WithdrawalUnit unit = new WithdrawalUnit();

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit, sessionConfig)) {
            SessionPseudoClock clock = instance.getClock();

            unit.withdrawals.append(new Withdrawal("A1", 100.0));
            unit.withdrawals.append(new Withdrawal("A2", 200.0));

            clock.advanceTime(6, TimeUnit.SECONDS);

            unit.withdrawals.append(new Withdrawal("A3", 300.0));
            assertThat(instance.fire()).isEqualTo(1);
        }
    }

    @Test
    void namedLengthWindowLimitsMatches() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                window RecentWithdrawals {
                    /withdrawals | length[3]
                }
                rule R1 {
                    var w : /recentWithdrawals,
                    do {}
                }
                """;
        KieBase kieBase = buildWithStreamMode(drlx);

        WithdrawalUnit unit = new WithdrawalUnit();

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit)) {
            for (int i = 1; i <= 5; i++) {
                unit.withdrawals.append(new Withdrawal("A" + i, i * 100.0));
            }
            assertThat(instance.fire()).isEqualTo(3);
        }
    }

    @Test
    void namedWindowReferenceWithAdditionalConstraints() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                window WithdrawalWindow {
                    /withdrawals | length[5]
                }
                rule R1 {
                    var w : /withdrawalWindow[amount > 200.0],
                    do {}
                }
                """;
        KieBase kieBase = buildWithStreamMode(drlx);

        WithdrawalUnit unit = new WithdrawalUnit();

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit)) {
            unit.withdrawals.append(new Withdrawal("A1", 100.0));
            unit.withdrawals.append(new Withdrawal("A2", 200.0));
            unit.withdrawals.append(new Withdrawal("A3", 300.0));
            unit.withdrawals.append(new Withdrawal("A4", 400.0));
            assertThat(instance.fire()).isEqualTo(2);
        }
    }

    @Test
    void multipleRulesReferenceSameNamedWindow() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                window WithdrawalWindow {
                    /withdrawals | length[3]
                }
                rule R1 {
                    var w : /withdrawalWindow[amount > 200.0],
                    do {}
                }
                rule R2 {
                    var w : /withdrawalWindow[amount <= 200.0],
                    do {}
                }
                """;
        KieBase kieBase = buildWithStreamMode(drlx);

        WithdrawalUnit unit = new WithdrawalUnit();

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit)) {
            unit.withdrawals.append(new Withdrawal("A1", 100.0));
            unit.withdrawals.append(new Withdrawal("A2", 200.0));
            unit.withdrawals.append(new Withdrawal("A3", 300.0));
            assertThat(instance.fire()).isEqualTo(3);
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=NamedWindowTest -q`

Expected: FAIL — window declarations are not compiled, window references are unresolved.

- [ ] **Step 3: Implement declaration compilation in `DrlxRuleAstRuntimeBuilder.build()`**

Add imports at the top of `DrlxRuleAstRuntimeBuilder.java`:

```java
import org.drools.base.rule.WindowDeclaration;
import org.drools.base.rule.WindowReference;
import org.drools.drlx.builder.DrlxRuleAstModel.WindowDeclarationIR;
```

In the `build(CompilationUnitIR parseResult)` method, after the `entryPointTypes` setup (after line 113 `entryPointTypes.keySet().forEach(pkg::addEntryPointId);`) and before the query loop, add window declaration compilation:

```java
Map<String, WindowDeclaration> windowRegistry = new LinkedHashMap<>();
for (WindowDeclarationIR windowIr : parseResult.windowDeclarations()) {
    WindowDeclaration windowDecl = new WindowDeclaration(windowIr.name(), parseResult.packageName());

    Class<?> windowType = entryPointTypes.get(windowIr.pattern().entryPoint());
    if (windowType == null) {
        throw new RuntimeException(
                "window '" + windowIr.name() + "' references unknown entry point '"
                + windowIr.pattern().entryPoint() + "'");
    }

    Role roleAnnotation = windowType.getAnnotation(Role.class);
    boolean isEvent = roleAnnotation != null && roleAnnotation.value() == Role.Type.EVENT;
    ObjectType objectType = new ClassObjectType(windowType, isEvent);
    Pattern windowPattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0, objectType, null, false);
    windowPattern.setSource(new EntryPointId(windowIr.pattern().entryPoint()));

    org.mvel3.transpiler.context.Declaration<?>[] declarations =
            DrlxLambdaCompiler.extractDeclarations(windowType);
    for (String expression : windowIr.pattern().conditions()) {
        Constraint constraint = lambdaCompiler.createLambdaConstraint(expression, windowType, declarations);
        windowPattern.addConstraint(constraint);
    }

    switch (windowIr.pattern().windowType()) {
        case "time" -> windowPattern.addBehavior(
                new SlidingTimeWindow(TimeUtils.parseTimeString(windowIr.pattern().windowParameter())));
        case "length" -> windowPattern.addBehavior(
                new SlidingLengthWindow(Integer.parseInt(windowIr.pattern().windowParameter())));
    }

    windowDecl.setPattern(windowPattern);
    pkg.addWindowDeclaration(windowDecl);

    String refName = Character.toLowerCase(windowIr.name().charAt(0)) + windowIr.name().substring(1);
    windowRegistry.put(refName, windowDecl);
}
```

- [ ] **Step 4: Thread `windowRegistry` through to `buildLhs`, `buildRule`, and `buildQuery`**

Add `Map<String, WindowDeclaration> windowRegistry` parameter to:

1. `buildRule()` method signature — add after `queryRegistry`:

```java
private RuleImpl buildRule(RuleIR parseResult,
                           TypeResolver typeResolver,
                           Map<String, Class<?>> entryPointTypes,
                           Class<?> unitClass,
                           Map<String, java.lang.reflect.Type> globalTypes,
                           Set<String> dataStoreGlobalNames,
                           DataStoreUpdateRewriter updateRewriter,
                           Map<String, QueryImpl> queryRegistry,
                           Map<String, WindowDeclaration> windowRegistry) {
```

Update the call in `build()`:

```java
pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass,
                     globalTypes, dataStoreGlobalNames, updateRewriter, queryRegistry, windowRegistry));
```

Update `buildRule`'s call to `buildLhs`:

```java
buildLhs(parseResult.lhs(), root, typeResolver, entryPointTypes, unitClass, boundVariables, queryRegistry, null, windowRegistry);
```

2. `buildQuery()` method signature — add after `queryRegistry`:

```java
private void buildQuery(QueryImpl query,
                        RuleIR parseResult,
                        TypeResolver typeResolver,
                        Map<String, Class<?>> entryPointTypes,
                        Class<?> unitClass,
                        Map<String, QueryImpl> queryRegistry,
                        Map<String, WindowDeclaration> windowRegistry) {
```

Update the call in `build()`:

```java
buildQuery(query, rule, pkg.getTypeResolver(), entryPointTypes, unitClass, queryRegistry, windowRegistry);
```

Update `buildQuery`'s call to `buildLhs`:

```java
buildLhs(parseResult.lhs(), root, typeResolver, entryPointTypes, unitClass, boundVariables, queryRegistry, query, windowRegistry);
```

3. `buildLhs()` method signature — add after `currentQuery`:

```java
private void buildLhs(List<LhsItemIR> items,
                      GroupElement parent,
                      TypeResolver typeResolver,
                      Map<String, Class<?>> entryPointTypes,
                      Class<?> unitClass,
                      Map<String, BoundVariable> boundVariables,
                      Map<String, QueryImpl> queryRegistry,
                      QueryImpl currentQuery,
                      Map<String, WindowDeclaration> windowRegistry) {
```

Update all recursive `buildLhs` calls within the method to pass `windowRegistry` as the last argument.

4. Update `buildAccumulatePattern`, `buildCustomAccumulatePattern`, `buildGroupByAccumulatePattern`, `buildGroupByCustomAccumulatePattern` to accept and pass `windowRegistry` the same way (these call `buildLhs` internally).

5. Update `collectPatternClasses()` and `registerTypeDeclarations()` to also accept `windowRegistry`, so they can resolve window reference types:

```java
private static void collectPatternClasses(List<LhsItemIR> items,
                                           Set<Class<?>> classes,
                                           TypeResolver typeResolver,
                                           Map<String, Class<?>> entryPointTypes,
                                           Class<?> unitClass,
                                           Map<String, WindowDeclaration> windowRegistry) {
```

In `collectPatternClasses`, when handling a `PatternIR`, check if it's a window reference before calling `resolvePatternType`:

```java
if (item instanceof PatternIR p) {
    WindowDeclaration windowDecl = windowRegistry.get(p.entryPoint());
    if (windowDecl != null) {
        classes.add(((ClassObjectType) windowDecl.getPattern().getObjectType()).getClassType());
    } else {
        // existing type resolution code
    }
}
```

- [ ] **Step 5: Implement window reference detection in `buildLhs`**

In the `buildLhs` method, inside the `if (item instanceof PatternIR patternIr)` block, after the query detection logic (around line 609) and before the normal `buildPattern` call (line 610), add window reference detection:

```java
WindowDeclaration windowDecl = windowRegistry.get(patternIr.entryPoint());
if (windowDecl != null) {
    Pattern windowSrcPattern = windowDecl.getPattern();
    Class<?> windowPatternClass = ((ClassObjectType) windowSrcPattern.getObjectType()).getClassType();
    Role roleAnnotation = windowPatternClass.getAnnotation(Role.class);
    boolean isEvent = roleAnnotation != null && roleAnnotation.value() == Role.Type.EVENT;
    ObjectType objectType = new ClassObjectType(windowPatternClass, isEvent);

    Pattern pattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0, objectType, patternIr.bindName(), false);
    pattern.setSource(new WindowReference(windowDecl.getName()));

    org.mvel3.transpiler.context.Declaration<?>[] declarations =
            DrlxLambdaCompiler.extractDeclarations(windowPatternClass);
    for (String expression : patternIr.conditions()) {
        List<BoundVariable> referencedBindings = lambdaCompiler.findReferencedBindings(expression, boundVariables);
        Constraint constraint = referencedBindings.isEmpty()
                ? lambdaCompiler.createLambdaConstraint(expression, windowPatternClass, declarations)
                : lambdaCompiler.createBetaLambdaConstraint(expression, windowPatternClass, declarations, referencedBindings);
        pattern.addConstraint(constraint);
    }

    parent.addChild(pattern);
    Declaration declaration = pattern.getDeclaration();
    if (declaration != null) {
        boundVariables.put(declaration.getIdentifier(),
                new BoundVariable(declaration.getIdentifier(), windowPatternClass, pattern, declaration));
    }
    continue;
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=NamedWindowTest -q`

Expected: PASS

- [ ] **Step 7: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "feat(runtime): compile window declarations and resolve references

Refs #72"
```

---

### Task 5: Protobuf — serialize/deserialize window declarations

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`

- [ ] **Step 1: Add `WindowDeclarationParseResult` message to protobuf**

In `drlx_rule_ast.proto`, add after the `CompilationUnitParseResult` message (before `RuleParseResult`):

```protobuf
message WindowDeclarationParseResult {
  string name = 1;
  PatternParseResult pattern = 2;
}
```

Add the new field to `CompilationUnitParseResult`:

```protobuf
message CompilationUnitParseResult {
  string source_hash = 1;
  string package_name = 2;
  repeated string imports = 3;
  repeated RuleParseResult rules = 4;
  string unit_name = 5;
  repeated WindowDeclarationParseResult window_declarations = 6;
}
```

- [ ] **Step 2: Update `DrlxRuleAstParseResult.save()` to serialize window declarations**

In the `save()` method, after `data.rules().forEach(rule -> builder.addRules(toProtoRule(rule)));`, add:

```java
for (WindowDeclarationIR windowDecl : data.windowDeclarations()) {
    builder.addWindowDeclarations(DrlxRuleAstProto.WindowDeclarationParseResult.newBuilder()
            .setName(windowDecl.name())
            .setPattern(patternToProto(windowDecl.pattern()))
            .build());
}
```

Add the import:

```java
import org.drools.drlx.builder.DrlxRuleAstModel.WindowDeclarationIR;
```

- [ ] **Step 3: Update `DrlxRuleAstParseResult.load()` to deserialize window declarations**

In the `load()` method, before the final return statement, add:

```java
List<WindowDeclarationIR> windowDeclarations = new ArrayList<>(parseResult.getWindowDeclarationsCount());
for (DrlxRuleAstProto.WindowDeclarationParseResult wdPR : parseResult.getWindowDeclarationsList()) {
    windowDeclarations.add(new WindowDeclarationIR(wdPR.getName(), patternFromProto(wdPR.getPattern())));
}
```

Update the return statement to pass `windowDeclarations`:

```java
return new CompilationUnitIR(parseResult.getPackageName(),
        parseResult.getUnitName(),
        List.copyOf(parseResult.getImportsList()),
        List.copyOf(windowDeclarations),
        List.copyOf(rules));
```

- [ ] **Step 4: Regenerate protobuf and verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources compile -q`

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "feat(proto): serialize/deserialize window declarations

Refs #72"
```

---

### Task 6: Full test suite and cleanup

**Files:**
- (no new files)

- [ ] **Step 1: Run the full drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml install -q`

Expected: BUILD SUCCESS, all existing tests pass plus the new named window tests.

- [ ] **Step 2: Investigate and fix any failures**

If any existing tests fail due to the `CompilationUnitIR` parameter change or threading changes, fix them.

- [ ] **Step 3: Commit any fixes**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "fix: resolve test failures from named windows integration

Refs #72"
```
