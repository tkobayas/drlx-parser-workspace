# Accumulate Inline-From Implementation Plan (#50)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the inline-from shorthand `avg(/persons.age)` to DRLX accumulate, desugaring it at visitor time into a synthetic source pattern + rewritten extractor — no IR model or runtime builder changes.

**Architecture:** Extend the `accumulateCall` grammar rule with a new `inlineFromOopath` alternative that ANTLR dispatches on the `/` prefix. In the visitor, detect this alternative, flush any pending pattern, synthesise a `PatternIR` with a `$inlineN` binding, and emit an `AccumulatePatternIR` immediately (no fold). The runtime builder sees identical IR shapes to the bound form.

**Tech Stack:** ANTLR4 grammar (DrlxParser.g4), Java visitor (DrlxToRuleAstVisitor.java), JUnit 5 + AssertJ tests, Maven build.

---

### Task 1: Grammar — add `inlineFromOopath` rule and extend `accumulateCall`

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:201-209`

- [ ] **Step 1: Write the failing parse test**

In `AccumulateVisitorTest.java`, add a test that parses `avg(/persons.age)` and expects an `AccumulatePatternIR` with a synthetic `$inline`-prefixed source binding:

```java
@Test
void inlineFromParsesToAccumulatePatternWithSyntheticSource() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                var avgAge = avg(/persons.age),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    assertThat(rule.lhs()).hasSize(1);
    var item = rule.lhs().get(0);
    assertThat(item).isInstanceOf(DrlxRuleAstModel.AccumulatePatternIR.class);
    var accPat = (DrlxRuleAstModel.AccumulatePatternIR) item;
    assertThat(accPat.source().bindName()).startsWith("$inline");
    assertThat(accPat.source().entryPoint()).isEqualTo("persons");
    assertThat(accPat.accumulators()).hasSize(1);
    var acc = accPat.accumulators().get(0);
    assertThat(acc.functionName()).isEqualTo("avg");
    assertThat(acc.argExpressions()).containsExactly("$inline0.age");
    assertThat(acc.referencedBindings()).containsExactly("$inline0");
}
```

Add this test to: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`

- [ ] **Step 2: Run the test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateVisitorTest#inlineFromParsesToAccumulatePatternWithSyntheticSource -pl .`
Expected: FAIL — ANTLR can't parse the inline-from syntax yet.

- [ ] **Step 3: Add the `inlineFromOopath` rule and modify `accumulateCall`**

In `DrlxParser.g4`, replace the current `accumulateCall` rule (lines 207-209) with:

```antlr
accumulateCall
    : qualifiedName '(' inlineFromOopath ')'
    | qualifiedName '(' (expression (',' expression)*)? ')'
    ;

inlineFromOopath
    : oopathExpression ('.' identifier)?
    ;
```

The inline-from alternative comes first because its prefix (`/` or `?/`) is disjoint from any Java `expression` start — ANTLR adaptive LL(\*) picks alternative 1 on `/`, alternative 2 otherwise.

- [ ] **Step 4: Regenerate the ANTLR parser**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources`
Expected: Clean generation, no ambiguity warnings.

- [ ] **Step 5: Commit grammar change**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "grammar: add inlineFromOopath rule for #50 accumulate inline-from

Refs #50"
```

---

### Task 2: Visitor — handle inline-from in `buildRule` loop

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:88-141` (buildRule method)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:265-282` (add new buildAccumulator overload)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:413-421` (add new buildPatternFromOopath overload)
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`

- [ ] **Step 1: Add `buildPatternFromOopath` overload with synthetic bind name**

Add a new method below the existing `buildPatternFromOopath(OopathExpressionContext)` at line ~413:

```java
private PatternIR buildPatternFromOopath(DrlxParser.OopathExpressionContext oopathCtx,
                                          String syntheticBindName) {
    String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
    String castTypeName = extractCastType(oopathCtx);
    List<String> conditions = extractConditions(oopathCtx);
    List<String> positionalArgs = extractPositionalArgs(oopathCtx);
    boolean passive = oopathCtx.QUESTION() != null;
    List<String> watchedProperties = extractWatchedProperties(oopathCtx);
    return new PatternIR("", syntheticBindName, entryPoint, conditions, castTypeName,
                          positionalArgs, passive, watchedProperties);
}
```

- [ ] **Step 2: Add `buildAccumulator` overload for inline-from**

Add a new method below the existing `buildAccumulator(AccumulateItemContext)` at line ~265:

```java
private AccumulatorIR buildAccumulator(DrlxParser.AccumulateItemContext ctx,
                                        String srcBindName,
                                        String finalDotIdent) {
    String typeName = ctx.VAR() != null
            ? "var"
            : ctx.typeType().getText();
    String bindName = ctx.identifier().getText();
    String functionName = ctx.accumulateCall().qualifiedName().getText();
    List<String> args;
    List<String> refs;
    if (finalDotIdent != null) {
        args = List.of(srcBindName + "." + finalDotIdent);
        refs = List.of(srcBindName);
    } else {
        args = List.of();
        refs = List.of();
    }
    return new AccumulatorIR(typeName, bindName, functionName, args, refs);
}
```

- [ ] **Step 3: Modify `buildRule` to dispatch inline-from**

In `buildRule`, add an `int inlineCounter = 0;` after line 98 (`List<AccumulatorIR> pendingAccs = new ArrayList<>();`).

Then replace the `if (itemCtx.accumulateItem() != null)` block (lines 101-108) with:

```java
if (itemCtx.accumulateItem() != null) {
    DrlxParser.AccumulateCallContext call = itemCtx.accumulateItem().accumulateCall();
    if (call.inlineFromOopath() != null) {
        // Inline-from: flush any prior pending, synthesise source, emit immediately.
        DrlxParser.InlineFromOopathContext inlineCtx = call.inlineFromOopath();
        String functionName = call.qualifiedName().getText();
        DrlxParser.OopathExpressionContext oopathCtx = inlineCtx.oopathExpression();
        String finalDotIdent = inlineCtx.identifier() != null
                ? inlineCtx.identifier().getText() : null;

        // Zero-arg-function guard: count(/persons.age) silently drops .age
        if (finalDotIdent != null) {
            AccumulateFunctionRegistry.Resolution resolved =
                    AccumulateFunctionRegistry.resolve(functionName);
            if (resolved.acceptsZeroArgs()) {
                throw new RuntimeException(
                        "function '" + functionName
                        + "' does not accept a final-dot extractor in rule '"
                        + name + "'; use '" + functionName + "("
                        + getText(oopathCtx) + ")' instead");
            }
        }

        flushPending(lhs, pendingPattern, pendingAccs);
        pendingPattern = null;
        pendingAccs = new ArrayList<>();

        String synthName = "$inline" + inlineCounter++;
        PatternIR synthSrc = buildPatternFromOopath(oopathCtx, synthName);
        AccumulatorIR accIr = buildAccumulator(
                itemCtx.accumulateItem(), synthName, finalDotIdent);

        lhs.add(new AccumulatePatternIR(synthSrc, List.of(accIr)));
    } else {
        // Regular form: requires a preceding bound pattern.
        if (pendingPattern == null) {
            throw new RuntimeException(
                    "accumulate item without a preceding pattern in rule '" + name + "'");
        }
        pendingAccs.add(buildAccumulator(itemCtx.accumulateItem()));
    }
    continue;
}
```

- [ ] **Step 4: Run the parse test from Task 1 to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateVisitorTest#inlineFromParsesToAccumulatePatternWithSyntheticSource -pl .`
Expected: PASS

- [ ] **Step 5: Run all existing AccumulateVisitorTest tests to verify no regression**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateVisitorTest -pl .`
Expected: All PASS

- [ ] **Step 6: Commit visitor changes**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: visitor handles inline-from accumulate desugaring (#50)

Refs #50"
```

---

### Task 3: Visitor-level parse tests — bare oopath, multiple inline-from, count with final-dot rejection

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`

- [ ] **Step 1: Add bare-oopath (count) parse test**

```java
@Test
void inlineFromBareOopathCountHasEmptyArgs() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                long n = count(/persons),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    assertThat(rule.lhs()).hasSize(1);
    var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
    assertThat(accPat.source().bindName()).isEqualTo("$inline0");
    assertThat(accPat.source().entryPoint()).isEqualTo("persons");
    var acc = accPat.accumulators().get(0);
    assertThat(acc.functionName()).isEqualTo("count");
    assertThat(acc.argExpressions()).isEmpty();
    assertThat(acc.referencedBindings()).isEmpty();
    assertThat(acc.resultTypeName()).isEqualTo("long");
}
```

- [ ] **Step 2: Add multiple inline-from parse test**

```java
@Test
void multipleInlineFromProduceSeparateAccumulatePatterns() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                var minAge = min(/persons.age),
                var maxAge = max(/persons.age),
                do {}
            }
            """;
    var rule = parseRule(drlx);
    assertThat(rule.lhs()).hasSize(2);

    var accPat0 = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
    assertThat(accPat0.source().bindName()).isEqualTo("$inline0");
    assertThat(accPat0.accumulators().get(0).functionName()).isEqualTo("min");
    assertThat(accPat0.accumulators().get(0).argExpressions()).containsExactly("$inline0.age");

    var accPat1 = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(1);
    assertThat(accPat1.source().bindName()).isEqualTo("$inline1");
    assertThat(accPat1.accumulators().get(0).functionName()).isEqualTo("max");
    assertThat(accPat1.accumulators().get(0).argExpressions()).containsExactly("$inline1.age");
}
```

- [ ] **Step 3: Add count-with-final-dot rejection parse test**

```java
@Test
void inlineFromCountWithFinalDotRejectedAtVisitor() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R1 {
                long n = count(/persons.age),
                do {}
            }
            """;
    assertThatThrownBy(() -> parseRule(drlx))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("function 'count' does not accept a final-dot extractor");
}
```

Add this import at the top of the file if not already present:
```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
```

- [ ] **Step 4: Run all new visitor tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateVisitorTest -pl .`
Expected: All PASS

- [ ] **Step 5: Commit parse tests**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: visitor-level parse tests for inline-from (#50)

Refs #50"
```

---

### Task 4: Behavioural test — `inlineFromAvg` (session execution)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write the `inlineFromAvg` test**

```java
@Test
void inlineFromAvg() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                var avgAge = avg(/persons.age),
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

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#inlineFromAvg -pl .`
Expected: PASS — the runtime builder sees the same `AccumulatePatternIR` shape it always has.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: inlineFromAvg behavioural test (#50)

Refs #50"
```

---

### Task 5: Behavioural test — `inlineFromCount` (bare oopath, zero-arg)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write the `inlineFromCount` test**

```java
@Test
void inlineFromCount() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                long n = count(/persons),
                do { results.add(n); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("A", 20));
        entryPoint.insert(new Person("B", 40));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(2L);
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#inlineFromCount -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: inlineFromCount behavioural test (#50)

Refs #50"
```

---

### Task 6: Behavioural test — `inlineFromMultipleSameSource` (three inline-from items)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write the `inlineFromMultipleSameSource` test**

```java
@Test
void inlineFromMultipleSameSource() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                var minAge = min(/persons.age),
                var maxAge = max(/persons.age),
                var avgAge = avg(/persons.age),
                do { results.add(minAge); results.add(maxAge); results.add(avgAge); }
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
    assertThat(observed).containsExactly(20, 60, 40.0);
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#inlineFromMultipleSameSource -pl .`
Expected: PASS — each inline-from produces its own SingleAccumulate.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: inlineFromMultipleSameSource behavioural test (#50)

Refs #50"
```

---

### Task 7: Behavioural test — `inlineFromWithSourceConstraint` (oopath filter)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write the `inlineFromWithSourceConstraint` test**

```java
@Test
void inlineFromWithSourceConstraint() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                var totalSenior = sum(/persons[age >= 40].age),
                do { results.add(totalSenior); }
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
    assertThat(observed).containsExactly(100.0);  // sum(40, 60) = 100.0
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#inlineFromWithSourceConstraint -pl .`
Expected: PASS — oopath `[...]` constraints carry into the synthetic source pattern.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: inlineFromWithSourceConstraint behavioural test (#50)

Refs #50"
```

---

### Task 8: Behavioural test — `inlineFromComposesWithBoundPattern`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write the `inlineFromComposesWithBoundPattern` test**

```java
@Test
void inlineFromComposesWithBoundPattern() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                var p : /persons[age >= 18],
                long n = count(/persons),
                do { results.add(p.getName()); results.add(n); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 20));
        entryPoint.insert(new Person("Bob", 40));
        kieSession.fireAllRules();
    });
    // Both persons pass age>=18. Each fires with count=2.
    // Order of firings may vary — use containsExactlyInAnyOrder for the pairs.
    assertThat(observed).containsExactlyInAnyOrder("Alice", 2L, "Bob", 2L);
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#inlineFromComposesWithBoundPattern -pl .`
Expected: PASS — bound `p` flushes as a normal LhsItem, inline-from count emits a separate AccumulatePatternIR.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: inlineFromComposesWithBoundPattern behavioural test (#50)

Refs #50"
```

---

### Task 9: Structural tests — AST shape verification

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write `inlineFromSynthesisesSourcePattern` structural test**

```java
@Test
void inlineFromSynthesisesSourcePattern() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                var avgAge = avg(/persons.age),
                do {}
            }
            """;
    final KieBase kieBase = new DrlxRuleBuilder().build(rule);
    final Pattern wrap = accumulateResultPattern(kieBase, "org.drools.drlx.parser", "R");

    assertThat(wrap.getSource()).isInstanceOf(SingleAccumulate.class);
    SingleAccumulate single = (SingleAccumulate) wrap.getSource();
    Pattern srcPattern = (Pattern) single.getSource();
    assertThat(srcPattern.getDeclaration().getIdentifier()).startsWith("$inline");
    assertThat(wrap.getDeclarations()).hasSize(1).containsKey("avgAge");
}
```

- [ ] **Step 2: Write `inlineFromMultipleEmitsSeparateSingleAccumulates` structural test**

```java
@Test
void inlineFromMultipleEmitsSeparateSingleAccumulates() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                var minAge = min(/persons.age),
                var maxAge = max(/persons.age),
                do {}
            }
            """;
    final KieBase kieBase = new DrlxRuleBuilder().build(rule);
    RuleImpl impl = (RuleImpl) kieBase.getKiePackage("org.drools.drlx.parser")
            .getRules().stream().filter(r -> r.getName().equals("R")).findFirst().orElseThrow();
    java.util.List<Pattern> accPatterns = impl.getLhs().getChildren().stream()
            .filter(Pattern.class::isInstance)
            .map(Pattern.class::cast)
            .filter(p -> p.getSource() instanceof SingleAccumulate)
            .toList();

    assertThat(accPatterns).hasSize(2);

    SingleAccumulate s0 = (SingleAccumulate) accPatterns.get(0).getSource();
    SingleAccumulate s1 = (SingleAccumulate) accPatterns.get(1).getSource();
    String id0 = ((Pattern) s0.getSource()).getDeclaration().getIdentifier();
    String id1 = ((Pattern) s1.getSource()).getDeclaration().getIdentifier();
    assertThat(id0).isNotEqualTo(id1);
    assertThat(id0).startsWith("$inline");
    assertThat(id1).startsWith("$inline");

    assertThat(accPatterns.get(0).getDeclarations()).containsKey("minAge");
    assertThat(accPatterns.get(1).getDeclarations()).containsKey("maxAge");
}
```

- [ ] **Step 3: Run structural tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#inlineFromSynthesisesSourcePattern+inlineFromMultipleEmitsSeparateSingleAccumulates -pl .`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: structural AST shape tests for inline-from (#50)

Refs #50"
```

---

### Task 10: Negative tests — rejection paths

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write `inlineFromCountWithFinalDotRejected` test**

```java
@Test
void inlineFromCountWithFinalDotRejected() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                long n = count(/persons.age),
                do {}
            }
            """;
    assertThatThrownBy(() -> new DrlxRuleBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("function 'count' does not accept a final-dot extractor");
}
```

- [ ] **Step 2: Write `inlineFromBareOopathRejectedForNonZeroArg` test**

```java
@Test
void inlineFromBareOopathRejectedForNonZeroArg() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                var avgAge = avg(/persons),
                do {}
            }
            """;
    assertThatThrownBy(() -> new DrlxRuleBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("requires exactly 1 argument, got 0");
}
```

- [ ] **Step 3: Write `inlineFromUnknownPropertyFailsAtBuild` test**

```java
@Test
void inlineFromUnknownPropertyFailsAtBuild() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                var avgAge = avg(/persons.nope),
                do {}
            }
            """;
    assertThatThrownBy(() -> new DrlxRuleBuilder().build(rule))
            .isInstanceOf(RuntimeException.class);
}
```

- [ ] **Step 4: Run negative tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#inlineFromCountWithFinalDotRejected+inlineFromBareOopathRejectedForNonZeroArg+inlineFromUnknownPropertyFailsAtBuild -pl .`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: negative tests for inline-from rejection paths (#50)

Refs #50"
```

---

### Task 11: Full regression run and install

- [ ] **Step 1: Run the full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml clean test`
Expected: All tests pass, including all prior AccumulateTest, AccumulateVisitorTest, and all other test classes.

- [ ] **Step 2: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml install -DskipTests`
Expected: BUILD SUCCESS — local Maven cache updated with the inline-from changes.

- [ ] **Step 3: Squash or rebase commits if desired, then final commit message**

If commits are already clean, no action needed. Otherwise:
```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser log --oneline -10
```
Review and confirm all commits reference `#50`.

---

## Spec coverage check

| Spec requirement | Task |
|---|---|
| Grammar: `inlineFromOopath` rule, first alt of `accumulateCall` | Task 1 |
| Visitor: flush pending, synthesise `$inlineN` source, emit `AccumulatePatternIR` | Task 2 |
| Visitor: `buildPatternFromOopath` overload with synthetic bind | Task 2 step 1 |
| Visitor: `buildAccumulator` overload for inline-from | Task 2 step 2 |
| Visitor: zero-arg-function guard (`count(/persons.age)` rejected) | Task 2 step 3 |
| Behavioural: `avg(/persons.age)` | Task 4 |
| Behavioural: `count(/persons)` bare oopath | Task 5 |
| Behavioural: multiple inline-from same source | Task 6 |
| Behavioural: oopath with `[...]` constraints | Task 7 |
| Behavioural: compose with bound pattern | Task 8 |
| Structural: synthetic source pattern shape | Task 9 step 1 |
| Structural: separate SingleAccumulates, different `$inlineN` ids | Task 9 step 2 |
| Negative: `count(/persons.age)` | Task 10 step 1 |
| Negative: `avg(/persons)` arity error | Task 10 step 2 |
| Negative: unknown property at build | Task 10 step 3 |
| Runtime builder unchanged | No code changes — verified by full regression (Task 11) |
| IR model unchanged | No code changes — verified by full regression (Task 11) |

## Notes

- The spec's `inlineFromDifferentSourceTypes` structural test mentions "different ObjectTypes" for `avg(/persons.age)` vs `avg(/seniors.age)`, but `MyUnit.seniors` is `DataStore<Person>` — same type as `persons`. The structural tests in Task 9 already prove separate `SingleAccumulate`s with distinct `$inlineN` identifiers, which is the meaningful assertion. A separate-entry-point behavioural test can be added as follow-up if needed.
- If ANTLR emits an ambiguity warning during `generate-sources` (Task 1 step 4), add a one-token semantic predicate `{_input.LT(1).getType() == DIV || _input.LT(1).getType() == QUESTION}?` in front of the inline-from alternative — the spec anticipates this.
