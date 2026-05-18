# Accumulate MultiAccumulate folding (#49) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the v1 multi-function lowering (N×`SingleAccumulate`, N source-pattern clones) with a single `MultiAccumulate` over one source pattern, mirroring Drools' `KiePackagesBuilder` convention.

**Architecture:** `buildAccumulatePattern` becomes a uniform per-accumulator builder followed by a single dispatch on accumulator count. N=1 stays on `SingleAccumulate` with the typed result Pattern from v1. N>1 emits one `MultiAccumulate` whose result is projected through an unnamed `Object[]` Pattern carrying N `ArrayElementReader` declarations. The `BoundVariable` record is extended to carry a per-binding `Declaration` so result lookups don't depend on `Pattern.getDeclaration()` (which is `null` on an unnamed wrap Pattern); `collectPatternTypes` is widened to iterate every declaration on each Pattern.

**Tech Stack:** Java 17+, Drools `MultiAccumulate` / `SingleAccumulate` / `ArrayElementReader`, JUnit 5, AssertJ.

**Spec:** `specs/2026-05-18-49-multiaccumulate-folding-design.md`

---

## File Structure

- **Modify:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java` — extend `BoundVariable` record (line 113) to add `Declaration declaration`; rewrite `collectPatternTypes` (line 468) to iterate `p.getDeclarations().values()`.
- **Modify:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` — three `new BoundVariable(...)` call sites (lines 357, 413, 424) gain a fourth `Declaration` argument; two `bv.pattern().getDeclaration()` lookups (lines 472, 525) become `bv.declaration()`; `buildAccumulatePattern` (line 397) is refactored to call new helpers `buildSingleAccumulator` and `requiredFor`, and gains a `MultiAccumulate` branch with a new `wrapMultiResultPattern` helper.
- **Modify:** `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java` — add two structural tests (`singleFunctionEmitsSingleAccumulate`, `multiFunctionEmitsOneMultiAccumulate`) and two behavioural tests (`multiFunctionCountAndSum`, `multiFunctionWithExpressionExtractors`). Existing `multiFunctionMinMaxAvg` assertion is unchanged.

No new production files. No deletions.

---

## Task 1: Extend `BoundVariable` record + migrate callers

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java:113`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:357,413,424,469-475,520-526`

Pure mechanical refactor: add a fourth `Declaration` field to `BoundVariable`, supply it at every construction site, and swap the two `bv.pattern().getDeclaration()` lookups to `bv.declaration()`. No behaviour change; the suite must stay green.

- [ ] **Step 1: Extend the `BoundVariable` record**

In `DrlxLambdaCompiler.java`, replace line 113:

```java
public record BoundVariable(String name, Class<?> type, Pattern pattern) {}
```

with:

```java
public record BoundVariable(String name, Class<?> type, Pattern pattern, Declaration declaration) {}
```

Ensure `org.drools.base.rule.Declaration` is imported (the surrounding file already imports it).

- [ ] **Step 2: Update the three construction sites in `DrlxRuleAstRuntimeBuilder.java`**

Line 357 (regular pattern in `buildLhs`):

```java
boundVariables.put(declaration.getIdentifier(),
        new BoundVariable(declaration.getIdentifier(), patternClass, pattern, declaration));
```

Line 413 (accumulate inner scope — source binding):

```java
innerScope.put(srcDecl.getIdentifier(),
        new BoundVariable(srcDecl.getIdentifier(), srcClass, srcTemplate, srcDecl));
```

Line 424 (accumulate result binding) — note: use `resultPattern.getDeclarations().get(acc.resultBindName())` rather than `resultPattern.getDeclaration()`. `wrapResultPattern` constructs the Pattern with an identifier (which seeds `this.declaration` with a `PatternExtractor`-backed declaration), then `addDeclaration(...)` overwrites the entry in `getDeclarations()` with a `SelfReferenceClassFieldReader`-backed declaration but does NOT touch `this.declaration`. The runtime-resolvable one is the SelfReference variant in the map.

```java
outerScope.put(acc.resultBindName(),
        new BoundVariable(acc.resultBindName(),
                          resultClass,
                          resultPattern,
                          resultPattern.getDeclarations().get(acc.resultBindName())));
```

- [ ] **Step 3: Update the two lookup sites in `DrlxRuleAstRuntimeBuilder.java`**

Line 469-474 in `buildSingleAccumulate`:

```java
        Declaration[] required = acc.referencedBindings().stream()
                .map(innerScope::get)
                .filter(java.util.Objects::nonNull)
                .map(BoundVariable::declaration)
                .filter(java.util.Objects::nonNull)
                .toArray(Declaration[]::new);
```

Line 525 in `buildEvalCondition`:

```java
                declarations.add(bv.declaration());
```

- [ ] **Step 4: Audit the rest of the module for `bv.pattern().getDeclaration()` and `new BoundVariable(`**

Run:

```bash
grep -rn "\.pattern()\.getDeclaration()\|new BoundVariable(" \
    /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java
```

Expected hits after Steps 1-3:
- Three `new BoundVariable(...)` calls — all in `DrlxRuleAstRuntimeBuilder.java`, all four-arg now.
- Zero `.pattern().getDeclaration()` references.

If any other module references either form, update them the same way.

- [ ] **Step 5: Compile the module**

Run:

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Run the full test suite — regression check**

Run:

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```

Expected: all 213 tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "refactor: carry Declaration on BoundVariable (#49 prep)

Multi-decl wrapping Patterns require each binding to carry its own
Declaration; the existing Pattern.getDeclaration() lookup returns null
on unnamed Patterns with N declarations.

Refs #49"
```

---

## Task 2: Widen `collectPatternTypes` to iterate every declaration

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java:468-480`

Existing patterns are single-decl, so per-declaration iteration is equivalent today. The change must not regress the suite.

- [ ] **Step 1: Replace the body of `collectPatternTypes`**

In `DrlxLambdaCompiler.java`, replace lines 468-480 (the `private static void collectPatternTypes` method body):

```java
    private static void collectPatternTypes(GroupElement ge, Map<String, Type<?>> types) {
        for (Object child : ge.getChildren()) {
            if (child instanceof Pattern p) {
                for (Declaration d : p.getDeclarations().values()) {
                    types.put(d.getIdentifier(),
                              Type.type(d.getExtractor().getExtractToClass()));
                }
            } else if (child instanceof GroupElement nested) {
                collectPatternTypes(nested, types);
            }
        }
    }
```

The unused `((ClassObjectType) p.getObjectType()).getClassType()` lookup goes away with the old body.

- [ ] **Step 2: Compile the module**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Run the full test suite — regression check**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```

Expected: all 213 tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "refactor: type every Pattern declaration in collectPatternTypes (#49 prep)

Iterate p.getDeclarations() and use each declaration's extractor type
instead of the Pattern's object type. Equivalent for single-decl
Patterns; correct for the multi-result wrapping Pattern landed by #49.

Refs #49"
```

---

## Task 3: Extract `buildSingleAccumulator` and `requiredFor` helpers

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:397-485`

Refactor the existing `buildSingleAccumulate` (line 428) into three pieces without changing behaviour: a per-accumulator constructor that returns an `Accumulator`, a required-decl computation, and the existing `SingleAccumulate` allocation. `buildAccumulatePattern` is rewritten to call the helpers but still emits N×`SingleAccumulate` — the dispatch in Task 5 is what flips to `MultiAccumulate`.

- [ ] **Step 1: Add `buildSingleAccumulator` and `requiredFor`**

Insert the following two methods into `DrlxRuleAstRuntimeBuilder.java`, immediately above the existing `buildSingleAccumulate` (line 428):

```java
    /**
     * Per-function construction shared by SingleAccumulate (N=1) and
     * MultiAccumulate (N>1) paths. Validates arity, builds the extractor,
     * instantiates the AccumulateFunction.
     *
     * @param srcBindingName the source binding name (e.g. {@code "p"}); may
     *                       be {@code null}, in which case any expression
     *                       argument is rejected.
     */
    private org.drools.base.rule.accessor.Accumulator buildSingleAccumulator(
            AccumulatorIR acc,
            Class<?> srcClass,
            String srcBindingName) {
        AccumulateFunctionRegistry.Resolution resolved =
                AccumulateFunctionRegistry.resolve(acc.functionName());

        int argCount = acc.argExpressions().size();
        if (resolved.acceptsZeroArgs()) {
            if (argCount > 1) {
                throw new RuntimeException("function '" + acc.functionName()
                        + "' accepts 0 or 1 argument, got " + argCount);
            }
        } else if (argCount != 1) {
            throw new RuntimeException("function '" + acc.functionName()
                    + "' requires exactly 1 argument, got " + argCount);
        }

        Function<Object, Object> extractor = null;
        if (argCount == 1 && !resolved.acceptsZeroArgs()) {
            if (srcBindingName == null) {
                throw new RuntimeException(
                        "accumulate source must have a binding to use expression argument '"
                                + acc.argExpressions().get(0) + "'");
            }
            extractor = lambdaCompiler.createValueExtractor(
                    acc.argExpressions().get(0), srcClass, srcBindingName);
        }

        @SuppressWarnings("unchecked")
        AccumulateFunction<Serializable> fn;
        try {
            fn = (AccumulateFunction<Serializable>) resolved.functionClass()
                    .getDeclaredConstructor().newInstance();
        } catch (ReflectiveOperationException e) {
            throw new RuntimeException("cannot instantiate " + resolved.functionClass(), e);
        }

        return new DrlxLambdaAccumulator(fn, extractor);
    }

    /** Map referenced bindings through the inner scope to a Declaration[] for SingleAccumulate. */
    private static Declaration[] requiredFor(AccumulatorIR acc,
                                             Map<String, BoundVariable> innerScope) {
        return acc.referencedBindings().stream()
                .map(innerScope::get)
                .filter(java.util.Objects::nonNull)
                .map(BoundVariable::declaration)
                .filter(java.util.Objects::nonNull)
                .toArray(Declaration[]::new);
    }
```

- [ ] **Step 2: Rewrite `buildSingleAccumulate` to delegate**

Replace the body of `buildSingleAccumulate` (lines 428-476) with:

```java
    private SingleAccumulate buildSingleAccumulate(AccumulatorIR acc,
                                                   Pattern innerPattern,
                                                   Class<?> srcClass,
                                                   Map<String, BoundVariable> innerScope) {
        Declaration innerDecl = innerPattern.getDeclaration();
        String srcBindingName = innerDecl != null ? innerDecl.getIdentifier() : null;
        org.drools.base.rule.accessor.Accumulator accumulator =
                buildSingleAccumulator(acc, srcClass, srcBindingName);
        Declaration[] required = requiredFor(acc, innerScope);
        return new SingleAccumulate(innerPattern, required, accumulator);
    }
```

Delete the original validate/extractor/instantiate code now that it lives in `buildSingleAccumulator`.

- [ ] **Step 3: Compile the module**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Run the full test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```

Expected: all 213 tests pass — including every accumulate test (multi-function still N×SingleAccumulate after this refactor).

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "refactor: split buildSingleAccumulate into Accumulator + required helpers (#49 prep)

Extract per-function Accumulator construction and required-decl
computation so the MultiAccumulate path can share them.

Refs #49"
```

---

## Task 4: Add `singleFunctionEmitsSingleAccumulate` structural test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

Lock in the baseline so the dispatch landing in Task 5 cannot drift the N=1 shape silently. The test passes from the current state.

- [ ] **Step 1: Add imports at the top of `AccumulateTest.java`**

Add these imports next to the existing block (preserving alphabetical order with the file's conventions):

```java
import org.drools.base.base.ClassObjectType;
import org.drools.base.base.extractors.ArrayElementReader;
import org.drools.base.base.extractors.SelfReferenceClassFieldReader;
import org.drools.base.definitions.rule.impl.RuleImpl;
import org.drools.base.rule.Declaration;
import org.drools.base.rule.GroupElement;
import org.drools.base.rule.MultiAccumulate;
import org.drools.base.rule.Pattern;
import org.drools.base.rule.SingleAccumulate;
import org.kie.api.KieBase;
```

- [ ] **Step 2: Add a private `accumulateResultPattern` helper to the test class**

Insert below the existing `@Test` methods, above the closing class brace:

```java
    /** Walk the rule's LHS to find the single Pattern whose source is an Accumulate. */
    private static Pattern accumulateResultPattern(KieBase kieBase, String pkg, String ruleName) {
        RuleImpl impl = (RuleImpl) kieBase.getKiePackage(pkg).getRules().stream()
                .filter(r -> r.getName().equals(ruleName))
                .findFirst()
                .orElseThrow();
        GroupElement lhs = impl.getLhs();
        return lhs.getChildren().stream()
                .filter(Pattern.class::isInstance)
                .map(Pattern.class::cast)
                .filter(p -> p.getSource() instanceof org.drools.base.rule.Accumulate)
                .findFirst()
                .orElseThrow(() -> new AssertionError("no accumulate result Pattern under rule " + ruleName));
    }
```

- [ ] **Step 3: Add the structural test**

Append:

```java
    @Test
    void singleFunctionEmitsSingleAccumulate() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var minAge = min(p.age),
                    do {}
                }
                """;
        final KieBase kieBase = new DrlxRuleBuilder().build(rule);
        final Pattern wrap = accumulateResultPattern(kieBase, "org.drools.drlx.parser", "R");

        assertThat(wrap.getSource()).isInstanceOf(SingleAccumulate.class);
        // resultClassFor(min) returns the registry's resultType for var-bindings;
        // AccumulateFunctionRegistry maps min/max → Comparable.class.
        assertThat(((ClassObjectType) wrap.getObjectType()).getClassType())
                .isEqualTo(Comparable.class);
        assertThat(wrap.getDeclarations()).hasSize(1).containsKey("minAge");
        final Declaration d = wrap.getDeclarations().get("minAge");
        assertThat(d.getExtractor()).isInstanceOf(SelfReferenceClassFieldReader.class);
        assertThat(d.getExtractor().getExtractToClass()).isEqualTo(Comparable.class);
    }
```

- [ ] **Step 4: Run the new test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml \
    test -Dtest='AccumulateTest#singleFunctionEmitsSingleAccumulate'
```

Expected: PASS.

- [ ] **Step 5: Run the full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```

Expected: all 214 tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: structural baseline for SingleAccumulate (N=1) shape (#49)

Asserts that var x = min(p.age) emits a Pattern of type Integer
wrapping SingleAccumulate with one SelfReferenceClassFieldReader
declaration. Prevents N=1 drift when the dispatch lands.

Refs #49"
```

---

## Task 5: Implement `MultiAccumulate` dispatch (red-green-refactor)

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

Write the failing structural test first; then add the helper + dispatch; then watch it go green.

- [ ] **Step 1: Add the failing `multiFunctionEmitsOneMultiAccumulate` structural test**

Append in `AccumulateTest.java` after `singleFunctionEmitsSingleAccumulate`:

```java
    @Test
    void multiFunctionEmitsOneMultiAccumulate() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var minAge = min(p.age),
                    var maxAge = max(p.age),
                    var avgAge = avg(p.age),
                    do {}
                }
                """;
        final KieBase kieBase = new DrlxRuleBuilder().build(rule);
        final java.util.List<Pattern> patterns = ((RuleImpl) kieBase.getKiePackage("org.drools.drlx.parser")
                .getRules().stream().filter(r -> r.getName().equals("R")).findFirst().orElseThrow())
                .getLhs().getChildren().stream()
                .filter(Pattern.class::isInstance)
                .map(Pattern.class::cast)
                .filter(p -> p.getSource() instanceof org.drools.base.rule.Accumulate)
                .toList();
        assertThat(patterns).hasSize(1);
        final Pattern wrap = patterns.get(0);

        assertThat(wrap.getSource()).isInstanceOf(MultiAccumulate.class);
        final MultiAccumulate multi = (MultiAccumulate) wrap.getSource();
        assertThat(multi.getAccumulators()).hasSize(3);
        assertThat(((ClassObjectType) wrap.getObjectType()).getClassType())
                .isEqualTo(Object[].class);
        assertThat(wrap.getDeclarations()).hasSize(3)
                .containsKeys("minAge", "maxAge", "avgAge");

        final Declaration minDecl = wrap.getDeclarations().get("minAge");
        assertThat(minDecl.getExtractor()).isInstanceOf(ArrayElementReader.class);
        // Registry resultType for min/max is Comparable.class; avg is Double.class.
        assertThat(minDecl.getExtractor().getExtractToClass()).isEqualTo(Comparable.class);
        final Declaration maxDecl = wrap.getDeclarations().get("maxAge");
        assertThat(maxDecl.getExtractor().getExtractToClass()).isEqualTo(Comparable.class);
        final Declaration avgDecl = wrap.getDeclarations().get("avgAge");
        assertThat(avgDecl.getExtractor().getExtractToClass()).isEqualTo(Double.class);
    }
```

- [ ] **Step 2: Run the new test and confirm it fails**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml \
    test -Dtest='AccumulateTest#multiFunctionEmitsOneMultiAccumulate'
```

Expected: FAIL — under the current baseline the LHS still contains 3 result Patterns, each wrapping a `SingleAccumulate`. The first assertion `assertThat(patterns).hasSize(1)` fails because `patterns.size() == 3`.

- [ ] **Step 3: Add the `wrapMultiResultPattern` helper in `DrlxRuleAstRuntimeBuilder.java`**

Add these imports near the top (mirroring how Drools' own ArrayElementReader imports its accessor type):

```java
import org.drools.base.base.extractors.ArrayElementReader;
import org.drools.base.rule.MultiAccumulate;
import org.drools.base.rule.accessor.ReadAccessor;
```

Insert this method below `wrapResultPattern` (around line 485):

```java
    private static Pattern wrapMultiResultPattern(java.util.List<AccumulatorIR> accs,
                                                  MultiAccumulate multi) {
        ReadAccessor selfReader = new SelfReferenceClassFieldReader(Object[].class);
        Pattern p = new Pattern(0, new ClassObjectType(Object[].class));
        for (int i = 0; i < accs.size(); i++) {
            Class<?> rType = resultClassFor(accs.get(i));
            p.addDeclaration(new Declaration(
                    accs.get(i).resultBindName(),
                    new ArrayElementReader(selfReader, i, rType),
                    p,
                    true));
        }
        p.setSource(multi);
        return p;
    }
```

- [ ] **Step 4: Rewrite `buildAccumulatePattern` with the dispatch**

Replace the body of `buildAccumulatePattern` (lines 397-426) with:

```java
    private void buildAccumulatePattern(AccumulatePatternIR accPat,
                                        GroupElement parent,
                                        TypeResolver typeResolver,
                                        Map<String, Class<?>> entryPointTypes,
                                        Class<?> unitClass,
                                        Map<String, BoundVariable> outerScope) {
        PatternIR srcIr = accPat.source();
        Pattern srcPattern = buildPattern(srcIr, typeResolver, entryPointTypes, unitClass, outerScope);
        Class<?> srcClass = ((ClassObjectType) srcPattern.getObjectType()).getClassType();
        Declaration srcDecl = srcPattern.getDeclaration();

        Map<String, BoundVariable> innerScope = new LinkedHashMap<>(outerScope);
        if (srcDecl != null) {
            innerScope.put(srcDecl.getIdentifier(),
                    new BoundVariable(srcDecl.getIdentifier(), srcClass, srcPattern, srcDecl));
        }

        java.util.List<AccumulatorIR> accumulators = accPat.accumulators();
        int n = accumulators.size();
        String srcBindingName = srcDecl != null ? srcDecl.getIdentifier() : null;

        org.drools.base.rule.accessor.Accumulator[] accs =
                new org.drools.base.rule.accessor.Accumulator[n];
        for (int i = 0; i < n; i++) {
            accs[i] = buildSingleAccumulator(accumulators.get(i), srcClass, srcBindingName);
        }

        Pattern wrap;
        if (n == 1) {
            Declaration[] required = requiredFor(accumulators.get(0), innerScope);
            SingleAccumulate single = new SingleAccumulate(srcPattern, required, accs[0]);
            wrap = wrapResultPattern(accumulators.get(0), single);
        } else {
            MultiAccumulate multi = new MultiAccumulate(srcPattern, new Declaration[0], accs, n);
            wrap = wrapMultiResultPattern(accumulators, multi);
        }

        parent.addChild(wrap);

        for (int i = 0; i < n; i++) {
            AccumulatorIR acc = accumulators.get(i);
            Class<?> resultClass = resultClassFor(acc);
            Declaration decl = wrap.getDeclarations().get(acc.resultBindName());
            outerScope.put(acc.resultBindName(),
                    new BoundVariable(acc.resultBindName(), resultClass, wrap, decl));
        }
    }
```

`buildSingleAccumulate` (the one introduced in Task 3 step 2) is now unused in this file. Delete it.

- [ ] **Step 5: Compile the module**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Run the failing test and confirm it now passes**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml \
    test -Dtest='AccumulateTest#multiFunctionEmitsOneMultiAccumulate'
```

Expected: PASS.

- [ ] **Step 7: Run the full suite — must catch any consequence-side regressions**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```

Expected: all 215 tests pass. In particular `multiFunctionMinMaxAvg` keeps its `containsExactly(20, 60, 40.0)` assertion — the consequence resolves `minAge` / `maxAge` / `avgAge` through the per-declaration `collectPatternTypes` change from Task 2.

- [ ] **Step 8: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: fold multi-function accumulate into a single MultiAccumulate (#49)

Mirrors Drools KiePackagesBuilder convention: one source pattern,
one MultiAccumulate with Accumulator[N], one unnamed result Pattern
of Object[].class with N ArrayElementReader declarations. N=1 stays
on SingleAccumulate unchanged.

Closes #49"
```

---

## Task 6: Add `multiFunctionCountAndSum` behavioural test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

Covers the null-extractor + non-null-extractor mix inside one `MultiAccumulate`.

- [ ] **Step 1: Add the test**

Append:

```java
    @Test
    void multiFunctionCountAndSum() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    long n = count(),
                    var total = sum(p.age),
                    do { results.add(n); results.add(total); }
                }
                """;

        final List<Object> observed = new ArrayList<>();
        withSession(rule, (kieSession, listener) -> {
            kieSession.setGlobal("results", observed);
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("A", 10));
            entryPoint.insert(new Person("B", 20));
            entryPoint.insert(new Person("C", 30));
            kieSession.fireAllRules();
        });
        assertThat(observed).containsExactly(3L, 60.0);
    }
```

- [ ] **Step 2: Run the new test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml \
    test -Dtest='AccumulateTest#multiFunctionCountAndSum'
```

Expected: PASS — `count` slot has `extractor == null`, `sum(p.age)` slot has the MVEL3 extractor, both ride the same `MultiAccumulate`.

- [ ] **Step 3: Run the full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```

Expected: all 216 tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: mixed null/non-null extractor in MultiAccumulate (#49)

Asserts count()+sum(p.age) coexist in one MultiAccumulate, with the
count slot carrying a null extractor.

Refs #49"
```

---

## Task 7: Add `multiFunctionWithExpressionExtractors` behavioural test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

Confirms every slot of a `MultiAccumulate` can run an MVEL3-compiled extractor (the #48 path).

- [ ] **Step 1: Add the test**

Append:

```java
    @Test
    void multiFunctionWithExpressionExtractors() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var s1 = sum(p.age + 1),
                    var s2 = sum(p.age * 2),
                    do { results.add(s1); results.add(s2); }
                }
                """;

        final List<Object> observed = new ArrayList<>();
        withSession(rule, (kieSession, listener) -> {
            kieSession.setGlobal("results", observed);
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("A", 10));
            entryPoint.insert(new Person("B", 20));
            entryPoint.insert(new Person("C", 30));
            kieSession.fireAllRules();
        });
        // sum(age+1) = 11+21+31 = 63   |   sum(age*2) = 20+40+60 = 120
        // SumAccumulateFunction normalises to Double.
        assertThat(observed).containsExactly(63.0, 120.0);
    }
```

- [ ] **Step 2: Run the new test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml \
    test -Dtest='AccumulateTest#multiFunctionWithExpressionExtractors'
```

Expected: PASS.

- [ ] **Step 3: Run the full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```

Expected: all 217 tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: MVEL3-compiled extractor per slot in MultiAccumulate (#49)

Verifies sum(p.age+1) and sum(p.age*2) share a single MultiAccumulate
and each slot resolves its own MVEL3 evaluator.

Refs #49"
```

---

## Task 8: Reactor-wide regression sweep + push + close

**Files:** none — verification only.

- [ ] **Step 1: Build and test the full reactor**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install
```

Expected: BUILD SUCCESS; all tests pass across modules. Confirms downstream consumers of `BoundVariable` (none expected outside `drlx-parser-core`, but the reactor build will catch surprises).

- [ ] **Step 2: Push the branch and close the issue**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push origin main
gh issue view 49 --repo tkobayas/drlx-parser
```

If the most recent commit message uses `Closes #49`, GitHub closes the issue automatically on push. Otherwise:

```bash
gh issue close 49 --repo tkobayas/drlx-parser \
    --comment "Multi-function accumulate now folded into a single MultiAccumulate node (one source pattern, one Object[] result Pattern with N ArrayElementReader declarations). Existing multiFunctionMinMaxAvg behaviour preserved; structural tests pinned in AccumulateTest."
```

---

## Notes on commit cadence

Eight commits in total, each isolated and revertable:

1. Extend `BoundVariable` + migrate callers (refactor, no behaviour change)
2. Widen `collectPatternTypes` (refactor, no behaviour change)
3. Split `buildSingleAccumulate` into helpers (refactor, no behaviour change)
4. `singleFunctionEmitsSingleAccumulate` structural test (locks N=1 baseline)
5. `MultiAccumulate` dispatch + multi structural test (feature; closes #49)
6. `multiFunctionCountAndSum` behavioural test
7. `multiFunctionWithExpressionExtractors` behavioural test
8. Final reactor sweep (no code; only verification + push/close)

If any commit fails the suite, halt and diagnose — never amend a green commit to absorb a later regression.
