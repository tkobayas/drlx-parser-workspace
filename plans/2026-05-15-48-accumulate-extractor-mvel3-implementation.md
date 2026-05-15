# Accumulate value-extractor — MVEL3 compile path (#48) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Lift the v1 accumulate restriction by compiling arbitrary single-binding extractor expressions (e.g. `sum(p.age + 1)`) through MVEL3 instead of reflection.

**Architecture:** A new `DrlxValueExtractor` class — `Function<Object, Object>` + `EvaluatorSink` — wraps a late-bound MVEL3 `Evaluator<Map<String, Object>, Void, Object>`. A new `createValueExtractor` method on `DrlxLambdaCompiler` mirrors `createEvalExpression` but emits an `Object` output. `DrlxRuleAstRuntimeBuilder.buildSingleAccumulate` switches to the new path; the reflection helpers are deleted in full.

**Tech Stack:** Java 17+, MVEL3 (`MVELBatchCompiler`, `MVEL.<Object>map(...)`), Drools `Accumulator` SPI, JUnit 5, AssertJ.

**Spec:** `specs/2026-05-15-48-accumulate-extractor-mvel3-design.md`

---

## File Structure

- **Create:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxValueExtractor.java` — Function<Object, Object> + EvaluatorSink wrapper around an MVEL3 evaluator.
- **Modify:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java` — add public `createValueExtractor` (around line 245, after `createEvalExpression`'s batch helper) + private `createBatchValueExtractor`.
- **Modify:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` — swap the call at line 449; delete `buildSimpleExtractor` (480-505), `isIdentifier` (507-517), `findGetter` (519-530), and the now-unused `import java.lang.reflect.Method;` at line 17.
- **Modify:** `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java` — flip `complexExtractorExpressionRejectedAsV1Limitation` (lines 213-235); add four new tests at the end of the class.

---

## Task 1: Add `DrlxValueExtractor` class

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxValueExtractor.java`

The class compiles in isolation; it is not yet referenced by anything in the production tree. The full suite must stay green.

- [ ] **Step 1: Create the class file**

Write the following content verbatim:

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 */

package org.drools.drlx.builder;

import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

import org.mvel3.Evaluator;

/**
 * Value-extractor lambda for an accumulate function argument. Produced by
 * {@link DrlxLambdaCompiler#createValueExtractor} and consumed by
 * {@link DrlxLambdaAccumulator}. Holds a late-bound MVEL3 evaluator and
 * threads the source fact through under its binding name.
 *
 * <p>The evaluator is {@code null} on the batch-compile path until
 * {@link DrlxLambdaCompiler#compileBatch(ClassLoader)} resolves all
 * pending handles and calls {@link #bindEvaluator}.
 */
public final class DrlxValueExtractor implements Function<Object, Object>, EvaluatorSink {

    private final String expression;
    private final String sourceBindingName;
    private Evaluator<Map<String, Object>, Void, Object> evaluator;

    public DrlxValueExtractor(String expression, String sourceBindingName,
                              Evaluator<Map<String, Object>, Void, Object> evaluator) {
        this.expression = expression;
        this.sourceBindingName = sourceBindingName;
        this.evaluator = evaluator;
    }

    @SuppressWarnings("unchecked")
    @Override
    public void bindEvaluator(Evaluator<?, ?, ?> evaluator) {
        this.evaluator = (Evaluator<Map<String, Object>, Void, Object>) evaluator;
    }

    @Override
    public Object apply(Object fact) {
        Map<String, Object> map = new HashMap<>(1);
        map.put(sourceBindingName, fact);
        try {
            return evaluator.eval(map);
        } catch (Exception e) {
            throw new RuntimeException(
                    "value extractor '" + expression + "' failed at runtime", e);
        }
    }
}
```

- [ ] **Step 2: Compile the module**

Run: `mvn -pl drlx-parser-core -am -q test-compile -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: build success. If compilation fails, fix imports / typos before proceeding.

- [ ] **Step 3: Run the full test suite — regression check**

Run: `mvn -pl drlx-parser-core -q test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: all 209 existing tests green. The new class is unused, so behavior must not change.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxValueExtractor.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ commit -m "feat(builder): add DrlxValueExtractor — MVEL3-backed Function<Object,Object>

Refs #48"
```

---

## Task 2: Add `createValueExtractor` on `DrlxLambdaCompiler`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java` — insert two methods after the existing `createBatchEvalExpression` (around line 246).

The new methods are not yet called from anywhere; the suite must stay green after this task too.

- [ ] **Step 1: Add the public method**

Insert this block immediately after `createBatchEvalExpression` (which ends at line 245) and before `createLambdaConsequence` (line 247):

```java
    /**
     * Compile an MVEL3 expression into a value-extractor lambda for an accumulate
     * argument. Used by {@link DrlxRuleAstRuntimeBuilder} when an accumulate function
     * (sum / avg / min / max) needs to project each fact through an arbitrary
     * expression over the source binding (e.g. {@code sum(p.age + 1)}).
     *
     * <p>Mirrors {@link #createEvalExpression}: tries the pre-compiled cache first,
     * otherwise registers a {@link PendingLambda} with {@link #batchCompiler} for
     * deferred resolution by {@link #compileBatch(ClassLoader)}.
     */
    public DrlxValueExtractor createValueExtractor(String argExpr,
                                                   Class<?> srcClass,
                                                   String sourceBindingName) {
        int counter = lambdaCounter++;

        @SuppressWarnings("unchecked")
        Evaluator<Map<String, Object>, Void, Object> preCompiled =
                (Evaluator<Map<String, Object>, Void, Object>) tryLoadPreCompiled(counter, argExpr, "value extractor");
        if (preCompiled != null) {
            return new DrlxValueExtractor(argExpr, sourceBindingName, preCompiled);
        }

        DrlxValueExtractor deferred = createBatchValueExtractor(argExpr, srcClass, sourceBindingName);
        onLambdaCreated(counter, argExpr);
        return deferred;
    }

    @SuppressWarnings({"unchecked", "rawtypes"})
    private DrlxValueExtractor createBatchValueExtractor(String argExpr,
                                                         Class<?> srcClass,
                                                         String sourceBindingName) {
        CompilerParameters<Map<String, Object>, Void, Object> evalInfo =
                (CompilerParameters) MVEL.<Object>map(
                        org.mvel3.transpiler.context.Declaration.of(sourceBindingName, srcClass))
                        .<Object>out(Object.class)
                        .expression(argExpr)
                        .imports(new HashSet<>(imports))
                        .classManager(batchCompiler.getClassManager())
                        .generatedClassName("GeneratorEvaluator__")
                        .build();
        MVELBatchCompiler.LambdaHandle handle = batchCompiler.add(evalInfo);
        DrlxValueExtractor extractor = new DrlxValueExtractor(argExpr, sourceBindingName, null);
        pendingLambdas.add(new PendingLambda(handle, extractor));
        return extractor;
    }
```

The imports already in the file cover everything used here (`MVEL`, `MVELBatchCompiler`, `CompilerParameters`, `Evaluator`, `HashSet`, `Map`).

- [ ] **Step 2: Compile the module**

Run: `mvn -pl drlx-parser-core -am -q test-compile -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: build success.

- [ ] **Step 3: Run the full test suite — regression check**

Run: `mvn -pl drlx-parser-core -q test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: 209 tests green. The new method has no callers yet.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ commit -m "feat(lambda): add createValueExtractor compile path

Refs #48"
```

---

## Task 3: Wire new path in `DrlxRuleAstRuntimeBuilder` + flip the v1-limit test

This task atomically swaps the call site, deletes the reflection helpers, and updates the contract test. Keeping these three in one commit means every commit on `main` compiles and is internally consistent.

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Replace the extractor build in `buildSingleAccumulate`**

In `DrlxRuleAstRuntimeBuilder.java`, replace the block around line 447-450:

```java
        Function<Object, Object> extractor = null;
        if (argCount == 1 && !resolved.acceptsZeroArgs()) {
            extractor = buildSimpleExtractor(acc.argExpressions().get(0), srcClass);
        }
```

with:

```java
        Function<Object, Object> extractor = null;
        if (argCount == 1 && !resolved.acceptsZeroArgs()) {
            Declaration innerDecl = innerPattern.getDeclaration();
            String sourceBindingName = innerDecl != null ? innerDecl.getIdentifier() : null;
            if (sourceBindingName == null) {
                throw new RuntimeException(
                        "accumulate source must have a binding to use expression argument '"
                                + acc.argExpressions().get(0) + "'");
            }
            extractor = lambdaCompiler.createValueExtractor(
                    acc.argExpressions().get(0), srcClass, sourceBindingName);
        }
```

`Declaration` is already imported (line 11 of the file via `org.drools.base.rule.Declaration`); no new import needed.

- [ ] **Step 2: Delete the reflection helpers and the now-unused import**

In the same file, delete:
- Lines 17 (`import java.lang.reflect.Method;`)
- The entire `buildSimpleExtractor` method (lines 480-505 — the doc-comment block plus the method body)
- The entire `isIdentifier` method (lines 507-517)
- The entire `findGetter` method (lines 519-530)

After deletion, verify with a grep:

```bash
grep -n "buildSimpleExtractor\|isIdentifier\|findGetter\|java.lang.reflect.Method" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
```

Expected: no matches.

- [ ] **Step 3: Flip the contract test in `AccumulateTest`**

Replace the entire `complexExtractorExpressionRejectedAsV1Limitation` test (lines 213-235) with this new positive test. Keep its position as the last test in the class (i.e., still right before the closing brace at line 236):

```java
    @Test
    void sumOfArithmeticExpression() {
        // After #48: arbitrary MVEL3 expressions over the source binding are
        // accepted. This test was the v1-limit contract — flipped here from
        // assertion-of-rejection to assertion-of-success.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var total = sum(p.age + 1),
                    do { results.add(total); }
                }
                """;

        final List<Object> observed = new ArrayList<>();
        withSession(rule, (kieSession, listener) -> {
            kieSession.setGlobal("results", observed);
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("A", 10));
            entryPoint.insert(new Person("B", 30));
            entryPoint.insert(new Person("C", 60));
            kieSession.fireAllRules();
        });
        // sum((10+1) + (30+1) + (60+1)) = 11 + 31 + 61 = 103
        // SumAccumulateFunction normalises to Double.
        assertThat(observed).containsExactly(103.0);
    }
```

- [ ] **Step 4: Compile and run the full test suite**

Run: `mvn -pl drlx-parser-core -q test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: all 209 tests green (the flipped contract test passes via MVEL3; every other test, including `sumOverIntegerField` / `singleAvgOverPersons` / `multiFunctionMinMaxAvg`, must keep passing — they now go through the MVEL3 path instead of reflection).

If `sumOverIntegerField` or another existing test fails, the regression is most likely in `createValueExtractor` (Map mode setup, Declaration name, output type wiring). Re-check Task 2's code block character-for-character.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ commit -m "feat(builder): wire MVEL3 extractor in accumulate; drop reflection path

Replaces the v1 reflection-based simple-extractor in buildSingleAccumulate
with the new DrlxLambdaCompiler.createValueExtractor path. Deletes
buildSimpleExtractor, isIdentifier, findGetter helpers and their reflect
import. Flips complexExtractorExpressionRejectedAsV1Limitation into
sumOfArithmeticExpression — the v1-limit contract becomes the
expression-accepted contract.

Refs #48"
```

---

## Task 4: Add positive coverage for the unlocked forms

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

Add three new tests at the end of the class (after `sumOfArithmeticExpression`, before the closing brace). Each follows the established `withSession` pattern.

- [ ] **Step 1: Add `sumOfMethodCallExpression`**

```java
    @Test
    void sumOfMethodCallExpression() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var total = sum(p.name.length()),
                    do { results.add(total); }
                }
                """;

        final List<Object> observed = new ArrayList<>();
        withSession(rule, (kieSession, listener) -> {
            kieSession.setGlobal("results", observed);
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("AA", 10));    // length 2
            entryPoint.insert(new Person("BBB", 20));   // length 3
            entryPoint.insert(new Person("CCCC", 30));  // length 4
            kieSession.fireAllRules();
        });
        assertThat(observed).containsExactly(9.0);  // 2 + 3 + 4 = 9
    }
```

- [ ] **Step 2: Add `sumOfMultipleBindingRefs`**

```java
    @Test
    void sumOfMultipleBindingRefs() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var total = sum(p.age * p.age),
                    do { results.add(total); }
                }
                """;

        final List<Object> observed = new ArrayList<>();
        withSession(rule, (kieSession, listener) -> {
            kieSession.setGlobal("results", observed);
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("A", 2));
            entryPoint.insert(new Person("B", 3));
            entryPoint.insert(new Person("C", 4));
            kieSession.fireAllRules();
        });
        assertThat(observed).containsExactly(29.0);  // 4 + 9 + 16 = 29
    }
```

- [ ] **Step 3: Add `avgOfExpression`**

```java
    @Test
    void avgOfExpression() {
        // Confirms non-sum functions also pick up the new MVEL3 extractor path.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var avgPlus = avg(p.age + 1),
                    do { results.add(avgPlus); }
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
        assertThat(observed).containsExactly(41.0);  // avg(21, 41, 61) = 41.0
    }
```

- [ ] **Step 4: Run the new tests**

Run: `mvn -pl drlx-parser-core -q test -Dtest=AccumulateTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: all `AccumulateTest` tests green, including the three new ones.

- [ ] **Step 5: Run the full suite to confirm no regression**

Run: `mvn -pl drlx-parser-core -q test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: 209 + 3 = 212 tests green.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ commit -m "test: cover arbitrary-expression accumulate args

sum(p.name.length()), sum(p.age*p.age), avg(p.age+1).

Refs #48"
```

---

## Task 5: Add negative test for compile-time error

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Add `extractorExpressionWithUnknownPropertyFailsAtBuild`**

Append at the end of the class:

```java
    @Test
    void extractorExpressionWithUnknownPropertyFailsAtBuild() {
        // An extractor expression that references a non-existent property must
        // fail at build time (MVEL3 raises during batch compile). We assert on
        // RuntimeException only — not on MVEL3's specific message, which would
        // be brittle across MVEL3 versions.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var total = sum(p.notAField),
                    do {}
                }
                """;
        assertThatThrownBy(() -> new DrlxRuleBuilder().build(rule))
                .isInstanceOf(RuntimeException.class);
    }
```

- [ ] **Step 2: Run the new test**

Run: `mvn -pl drlx-parser-core -q test -Dtest=AccumulateTest#extractorExpressionWithUnknownPropertyFailsAtBuild -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: PASS (MVEL3 throws during build).

If this test fails (e.g. because the build succeeds and the error only surfaces at fact-insertion time), that's a real signal: MVEL3 may not validate property references at compile time for `Map<String, Object>`-mode declarations. In that case:
- Document the deferred-validation behavior in the test body (rename to `extractorExpressionWithUnknownPropertyFailsAtRuntime`, change the assertion to wrap the rule in `withSession` and assert the throw on `kieSession.fireAllRules()` after inserting a Person).
- Note the finding in `HANDOFF.md` for follow-up — eager validation is a separate concern from #48's scope.

- [ ] **Step 3: Run the full suite**

Run: `mvn -pl drlx-parser-core -q test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: 213 tests green.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ commit -m "test: extractor with unknown property fails at build

Refs #48"
```

---

## Task 6: Reactor-wide regression sweep

**Files:** none modified.

- [ ] **Step 1: Build and test the full reactor**

Run: `mvn clean install -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml`

Expected: BUILD SUCCESS across all three reactor modules; full reactor test suite green.

- [ ] **Step 2: Push and close**

If everything is green, push the commits and close #48 with a short note that the v1 extractor limit is lifted.

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ push origin main
gh issue close 48 --repo tkobayas/drlx-parser --comment "Shipped at <HEAD-sha>. Arbitrary expressions over the source binding now compile through DrlxLambdaCompiler.createValueExtractor via MVEL3. Reflection-based v1 path removed. Outer-binding refs in extractor expressions remain out of scope and will be filed as a separate follow-up if needed."
```

(Substitute `<HEAD-sha>` with the actual project HEAD after the push.)

---

## Notes

- **MVEL3 install:** No MVEL3 changes are made in this issue, so no `mvn install` on the MVEL3 module is required.
- **Pre-build path:** the new compile path uses `tryLoadPreCompiled` and `onLambdaCreated` exactly like `createEvalExpression`, so pre-build (`DrlxPreBuildLambdaCompiler`) and pre-built metadata reload (`DrlxLambdaMetadata`) work without any additional changes. If `DrlxPreBuildLambdaCompilerTest` already exercises accumulate rules, the existing tests cover this transitively. If a regression surfaces only on the pre-build path, the fix point is the new method's structure — re-check it parallels `createEvalExpression` exactly.
- **Imports in extractor expressions:** the `imports` field on `DrlxLambdaCompiler` is passed to `CompilerParameters.imports(...)`, so user-imported types (e.g. enum constants) resolve inside extractor expressions, matching eval / beta paths.
- **The `Declaration[] required` plumbing** in `buildSingleAccumulate` (built from `acc.referencedBindings()`) is intentionally left unchanged — it stays in place for when outer-binding extractor refs are tackled in a follow-up issue.
