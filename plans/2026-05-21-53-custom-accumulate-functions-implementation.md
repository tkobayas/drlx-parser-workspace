# #53 Custom Accumulate Functions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable user-defined accumulate functions via class-qualified names (`Container.fieldName(expr)`), resolved from `public static final AccumulateFunction` fields on imported container classes.

**Architecture:** Split resolution — built-in functions (no dot) continue through `AccumulateFunctionRegistry`; qualified names (with dot) are resolved by the runtime builder using `TypeResolver` + reflective field lookup. The parser already preserves qualified names verbatim.

**Tech Stack:** Java 17, JUnit 5, AssertJ, Maven (drlx-parser-core module)

**Spec:** `specs/2026-05-21-53-custom-accumulate-functions-design.md`

---

## File Map

| Action | File | Purpose |
|--------|------|---------|
| Create | `drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/TestAccFuncs.java` | Test container class with custom `AccumulateFunction` fields |
| Create | `drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/SumSquaresAccumulateFunction.java` | Test `AccumulateFunction` implementation (sum of squares) |
| Create | `drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/DoubleCountAccumulateFunction.java` | Second test `AccumulateFunction` (count × 2, for multi-accumulate tests) |
| Create | `drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/BadContainer.java` | Container with a non-AccumulateFunction static field (for error tests) |
| Modify | `drlx-parser-core/src/main/java/org/drools/drlx/builder/AccumulateFunctionRegistry.java` | Remove qualified-name rejection from `resolve()` |
| Modify | `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateFunctionRegistryTest.java` | Remove `rejectsQualifiedName` test |
| Modify | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Skip `acceptsZeroArgs` check for qualified names in inline-from path |
| Modify | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` | Add custom function resolution via `TypeResolver` + reflection |
| Modify | `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java` | Replace `qualifiedFunctionNameRejected` with positive tests |

---

### Task 1: Create Test AccumulateFunction Implementations and Container

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/SumSquaresAccumulateFunction.java`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/DoubleCountAccumulateFunction.java`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/TestAccFuncs.java`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/BadContainer.java`

- [ ] **Step 1: Create `SumSquaresAccumulateFunction`**

This function computes the sum of squares of its input values. Result type is `Double`.

```java
package org.drools.drlx.domain.acc;

import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;
import java.io.Serializable;

import org.kie.api.runtime.rule.AccumulateFunction;

public class SumSquaresAccumulateFunction implements AccumulateFunction<SumSquaresAccumulateFunction.SumSqData> {

    public static class SumSqData implements Serializable {
        private double total;
    }

    @Override public SumSqData createContext() { return new SumSqData(); }

    @Override public void init(SumSqData ctx) { ctx.total = 0; }

    @Override public void accumulate(SumSqData ctx, Object value) {
        double v = ((Number) value).doubleValue();
        ctx.total += v * v;
    }

    @Override public void reverse(SumSqData ctx, Object value) {
        double v = ((Number) value).doubleValue();
        ctx.total -= v * v;
    }

    @Override public boolean supportsReverse() { return true; }

    @Override public Object getResult(SumSqData ctx) { return ctx.total; }

    @Override public Class<?> getResultType() { return Double.class; }

    @Override public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {}
    @Override public void writeExternal(ObjectOutput out) throws IOException {}
}
```

- [ ] **Step 2: Create `DoubleCountAccumulateFunction`**

Counts items and returns count × 2 (trivially distinct from built-in `count`). Result type is `Long`.

```java
package org.drools.drlx.domain.acc;

import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;
import java.io.Serializable;

import org.kie.api.runtime.rule.AccumulateFunction;

public class DoubleCountAccumulateFunction implements AccumulateFunction<DoubleCountAccumulateFunction.DblCountData> {

    public static class DblCountData implements Serializable {
        private long count;
    }

    @Override public DblCountData createContext() { return new DblCountData(); }

    @Override public void init(DblCountData ctx) { ctx.count = 0; }

    @Override public void accumulate(DblCountData ctx, Object value) { ctx.count++; }

    @Override public void reverse(DblCountData ctx, Object value) { ctx.count--; }

    @Override public boolean supportsReverse() { return true; }

    @Override public Object getResult(DblCountData ctx) { return ctx.count * 2L; }

    @Override public Class<?> getResultType() { return Long.class; }

    @Override public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {}
    @Override public void writeExternal(ObjectOutput out) throws IOException {}
}
```

- [ ] **Step 3: Create `TestAccFuncs` container class**

```java
package org.drools.drlx.domain.acc;

import org.kie.api.runtime.rule.AccumulateFunction;

public class TestAccFuncs {
    public static final AccumulateFunction sumSquares = new SumSquaresAccumulateFunction();
    public static final AccumulateFunction doubleCount = new DoubleCountAccumulateFunction();
}
```

- [ ] **Step 4: Create `BadContainer` for error tests**

```java
package org.drools.drlx.domain.acc;

public class BadContainer {
    public static final String notAFunction = "hello";
}
```

- [ ] **Step 5: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test-compile`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/domain/acc/
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: add custom AccumulateFunction test fixtures for #53

Refs #53"
```

---

### Task 2: Clean Up AccumulateFunctionRegistry

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/AccumulateFunctionRegistry.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateFunctionRegistryTest.java`

- [ ] **Step 1: Write failing test — registry no longer rejects qualified names**

In `AccumulateFunctionRegistryTest.java`, replace the `rejectsQualifiedName` test:

```java
@Test
void qualifiedNameReturnsNull() {
    var resolved = AccumulateFunctionRegistry.resolve("Func.avg");
    assertThat(resolved).isNull();
}
```

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateFunctionRegistryTest#qualifiedNameReturnsNull -pl .`
Expected: FAIL — currently throws `IllegalArgumentException`

- [ ] **Step 2: Update `AccumulateFunctionRegistry.resolve()`**

Remove the qualified-name rejection block. When the name contains a dot, return `null` instead of throwing (signals "not a built-in, resolve externally"). Update `resolve()` in `AccumulateFunctionRegistry.java`:

Replace:
```java
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
```

With:
```java
    public static Resolution resolve(String functionName) {
        if (functionName.contains(".")) {
            return null;
        }
        Resolution r = BUILTINS.get(functionName);
        if (r == null) {
            throw new IllegalArgumentException(
                    "unknown accumulate function '" + functionName + "' — "
                    + "built-ins are: " + BUILTIN_LIST);
        }
        return r;
    }
```

- [ ] **Step 3: Run all registry tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateFunctionRegistryTest -pl .`
Expected: All 5 tests pass (including the new `qualifiedNameReturnsNull`)

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/AccumulateFunctionRegistry.java \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateFunctionRegistryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "refactor: AccumulateFunctionRegistry returns null for qualified names

Instead of throwing for qualified function names, resolve() now returns
null — callers use this as the signal to resolve via imports instead.

Refs #53"
```

---

### Task 3: Update DrlxToRuleAstVisitor — Qualified Name Pass-through

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

- [ ] **Step 1: Update inline-from validation to skip qualified names**

In `DrlxToRuleAstVisitor.java`, around line 113, change:

```java
                        if (finalDotIdent != null) {
                            AccumulateFunctionRegistry.Resolution resolved =
                                    AccumulateFunctionRegistry.resolve(functionName);
                            if (resolved.acceptsZeroArgs()) {
```

To:

```java
                        if (finalDotIdent != null && !functionName.contains(".")) {
                            AccumulateFunctionRegistry.Resolution resolved =
                                    AccumulateFunctionRegistry.resolve(functionName);
                            if (resolved.acceptsZeroArgs()) {
```

This skips the `acceptsZeroArgs` check for qualified names. Custom functions always require arguments, so the `finalDotIdent` extractor is always valid for them.

- [ ] **Step 2: Run existing accumulate tests to verify no regressions**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest -pl .`
Expected: All tests pass except `qualifiedFunctionNameRejected` (which now fails because qualified names are no longer rejected at the visitor level — the error now moves to the runtime builder where `resolve()` returns `null`). Other tests must remain green.

Note: `qualifiedFunctionNameRejected` may still fail at the runtime builder level since we haven't added custom resolution yet. This is expected — Task 4 will handle the runtime builder, and Task 5 will replace this test.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "refactor: visitor passes qualified accumulate names through to runtime builder

Skip the acceptsZeroArgs check for qualified function names in the
inline-from path. Custom functions always require arguments.

Refs #53"
```

---

### Task 4: Add Custom Function Resolution in DrlxRuleAstRuntimeBuilder

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Add `ResolvedFunction` record and `resolveFunction` helper**

Add these inside `DrlxRuleAstRuntimeBuilder`, after the existing private methods (around line 760, after `resultClassFor`):

```java
    private record ResolvedFunction(AccumulateFunction<Serializable> instance,
                                    Class<?> resultType,
                                    boolean acceptsZeroArgs) {
    }

    @SuppressWarnings("unchecked")
    private ResolvedFunction resolveFunction(String functionName, TypeResolver typeResolver) {
        AccumulateFunctionRegistry.Resolution builtIn = AccumulateFunctionRegistry.resolve(functionName);
        if (builtIn != null) {
            try {
                AccumulateFunction<Serializable> fn =
                        (AccumulateFunction<Serializable>) builtIn.functionClass()
                                .getDeclaredConstructor().newInstance();
                return new ResolvedFunction(fn, builtIn.resultType(), builtIn.acceptsZeroArgs());
            } catch (ReflectiveOperationException e) {
                throw new RuntimeException("cannot instantiate " + builtIn.functionClass(), e);
            }
        }

        int dot = functionName.lastIndexOf('.');
        String className = functionName.substring(0, dot);
        String fieldName = functionName.substring(dot + 1);

        Class<?> containerClass;
        try {
            containerClass = typeResolver.resolveType(className);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(
                    "cannot resolve accumulate function class '" + className
                    + "' — ensure it is imported");
        }

        java.lang.reflect.Field field;
        try {
            field = containerClass.getField(fieldName);
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(
                    "class '" + className + "' has no static AccumulateFunction field named '"
                    + fieldName + "'");
        }

        if (!java.lang.reflect.Modifier.isStatic(field.getModifiers())) {
            throw new RuntimeException(
                    "class '" + className + "' has no static AccumulateFunction field named '"
                    + fieldName + "'");
        }

        Object value;
        try {
            value = field.get(null);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(
                    "cannot access field '" + className + "." + fieldName + "'", e);
        }

        if (!(value instanceof AccumulateFunction)) {
            throw new RuntimeException(
                    "field '" + className + "." + fieldName + "' is not an AccumulateFunction");
        }

        AccumulateFunction<Serializable> fn = (AccumulateFunction<Serializable>) value;
        return new ResolvedFunction(fn, fn.getResultType(), false);
    }
```

- [ ] **Step 2: Update `buildSingleAccumulator` to use `resolveFunction`**

Add `TypeResolver typeResolver` parameter and replace the resolution + instantiation logic. Replace the entire method (around line 630-669):

```java
    private org.drools.base.rule.accessor.Accumulator buildSingleAccumulator(
            AccumulatorIR acc,
            Class<?> srcClass,
            String srcBindingName,
            TypeResolver typeResolver) {
        ResolvedFunction resolved = resolveFunction(acc.functionName(), typeResolver);

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

        return new DrlxLambdaAccumulator(resolved.instance(), extractor);
    }
```

- [ ] **Step 3: Update `buildSingleAccumulatorMulti` to use `resolveFunction`**

Add `TypeResolver typeResolver` parameter and replace the resolution + instantiation logic. Replace the entire method (around line 671-704):

```java
    private org.drools.base.rule.accessor.Accumulator buildSingleAccumulatorMulti(
            AccumulatorIR acc,
            Map<String, BoundVariable> sourceScope,
            TypeResolver typeResolver) {
        ResolvedFunction resolved = resolveFunction(acc.functionName(), typeResolver);

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

        DrlxValueExtractor multiExtractor = null;
        if (argCount == 1 && !resolved.acceptsZeroArgs()) {
            multiExtractor = lambdaCompiler.createValueExtractor(
                    acc.argExpressions().get(0), sourceScope);
        }

        return new DrlxLambdaAccumulator(resolved.instance(), multiExtractor, true);
    }
```

- [ ] **Step 4: Update `resultClassFor` to use `resolveFunction`**

Add `TypeResolver typeResolver` parameter. Change the method signature from `static` to instance (it needs `this.resolveFunction`). Replace the registry call:

```java
    private Class<?> resultClassFor(AccumulatorIR acc, TypeResolver typeResolver) {
        ResolvedFunction resolved = resolveFunction(acc.functionName(), typeResolver);
        if ("var".equals(acc.resultTypeName())) {
            return resolved.resultType();
        }
        return switch (acc.resultTypeName()) {
            case "int"     -> Integer.class;
            case "long"    -> Long.class;
            case "double"  -> Double.class;
            case "float"   -> Float.class;
            case "short"   -> Short.class;
            case "byte"    -> Byte.class;
            case "boolean" -> Boolean.class;
            case "char"    -> Character.class;
            default        -> {
```

(The rest of the `default` branch stays unchanged.)

- [ ] **Step 5: Update call sites in `buildAccumulatePattern`**

In `buildAccumulatePattern()`, update the three call sites that pass to the updated methods:

**Call site 1** (~line 475): `buildSingleAccumulatorMulti` — add `typeResolver`:
```java
accs[i] = buildSingleAccumulatorMulti(accumulators.get(i), sourceScope, typeResolver);
```

**Call site 2** (~line 480): `buildSingleAccumulator` — add `typeResolver`:
```java
accs[i] = buildSingleAccumulator(accumulators.get(i), srcClass, srcBindingName, typeResolver);
```

**Call site 3** (~line 500): `resultClassFor` — add `typeResolver`:
```java
Class<?> resultClass = resultClassFor(acc, typeResolver);
```

**Call site 4** (~line 722, `wrapMultiResultPattern`): `resultClassFor` is called from the static `wrapMultiResultPattern` method. This method needs either a `TypeResolver` parameter or the call needs to move. The simplest approach: add `TypeResolver typeResolver` parameter to `wrapMultiResultPattern` and make it non-static:

Update the method signature:
```java
private Pattern wrapMultiResultPattern(List<AccumulatorIR> accs,
                                       MultiAccumulate multi,
                                       TypeResolver typeResolver) {
```

And its `resultClassFor` call:
```java
Class<?> rType = resultClassFor(accs.get(i), typeResolver);
```

Update the call in `buildAccumulatePattern` (~line 493):
```java
wrap = wrapMultiResultPattern(accumulators, multi, typeResolver);
```

**Call site 5** (~line 735, `wrapResultPattern`): Same treatment — add `TypeResolver` parameter, make non-static:

Update the method signature:
```java
private Pattern wrapResultPattern(AccumulatorIR acc, SingleAccumulate single,
                                  TypeResolver typeResolver) {
```

And its `resultClassFor` call:
```java
Class<?> resultType = resultClassFor(acc, typeResolver);
```

Update the call in `buildAccumulatePattern` (~line 490):
```java
wrap = wrapResultPattern(accumulators.get(0), single, typeResolver);
```

- [ ] **Step 6: Run existing accumulate tests (excluding the soon-to-be-replaced rejection test)**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#singleAvgOverPersons+multiFunctionMinMaxAvg+countWithNoArgument+countWithArgument+sumOverPersons+unknownFunctionRejected -pl .`
Expected: All listed tests pass — built-in functions work identically to before.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: resolve custom accumulate functions via TypeResolver + reflection

Add resolveFunction() helper that splits qualified names on the last dot,
resolves the container class via imports, and reads the AccumulateFunction
from a public static field. Built-in path unchanged.

Refs #53"
```

---

### Task 5: Replace Rejection Tests with Positive Custom Function Tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Replace `qualifiedFunctionNameRejected` with `customFunctionSumSquares`**

Replace the existing `qualifiedFunctionNameRejected` test with:

```java
    @Test
    void customFunctionSumSquares() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.acc.TestAccFuncs;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var ss = TestAccFuncs.sumSquares(p.age),
                    do { results.add(ss); }
                }
                """;

        final List<Object> observed = new ArrayList<>();
        withSession(rule, (kieSession, listener) -> {
            kieSession.setGlobal("results", observed);
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("A", 3));
            entryPoint.insert(new Person("B", 4));
            kieSession.fireAllRules();
        });
        assertThat(observed).containsExactly(25.0);  // 3² + 4² = 9 + 16 = 25.0
    }
```

- [ ] **Step 2: Run the new test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#customFunctionSumSquares -pl .`
Expected: PASS

- [ ] **Step 3: Add `customFunctionMultiAccumulate` test**

Two custom functions from the same container in one rule:

```java
    @Test
    void customFunctionMultiAccumulate() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.acc.TestAccFuncs;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var ss = TestAccFuncs.sumSquares(p.age),
                    var dc = TestAccFuncs.doubleCount(p.age),
                    do { results.add(ss); results.add(dc); }
                }
                """;

        final List<Object> observed = new ArrayList<>();
        withSession(rule, (kieSession, listener) -> {
            kieSession.setGlobal("results", observed);
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("A", 3));
            entryPoint.insert(new Person("B", 4));
            entryPoint.insert(new Person("C", 5));
            kieSession.fireAllRules();
        });
        assertThat(observed).containsExactly(50.0, 6L);  // 9+16+25=50.0, count=3 × 2=6
    }
```

- [ ] **Step 4: Run the multi-accumulate test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#customFunctionMultiAccumulate -pl .`
Expected: PASS

- [ ] **Step 5: Add `mixedBuiltinAndCustomFunction` test**

Built-in `avg` and custom `TestAccFuncs.sumSquares` in the same rule:

```java
    @Test
    void mixedBuiltinAndCustomFunction() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.acc.TestAccFuncs;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var avgAge = avg(p.age),
                    var ss = TestAccFuncs.sumSquares(p.age),
                    do { results.add(avgAge); results.add(ss); }
                }
                """;

        final List<Object> observed = new ArrayList<>();
        withSession(rule, (kieSession, listener) -> {
            kieSession.setGlobal("results", observed);
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("A", 3));
            entryPoint.insert(new Person("B", 4));
            kieSession.fireAllRules();
        });
        assertThat(observed).containsExactly(3.5, 25.0);  // avg(3,4)=3.5, 9+16=25.0
    }
```

- [ ] **Step 6: Run the mixed test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#mixedBuiltinAndCustomFunction -pl .`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: add positive tests for custom accumulate functions

Replace qualifiedFunctionNameRejected with three positive tests:
single custom function, multi-accumulate with two custom functions,
and mixed built-in + custom in the same rule.

Refs #53"
```

---

### Task 6: Add Error Case Tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Add `customFunctionClassNotFound` test**

```java
    @Test
    void customFunctionClassNotFound() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var ss = NoSuchClass.sumSquares(p.age),
                    do {}
                }
                """;
        assertThatThrownBy(() -> new DrlxRuleBuilder().build(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("cannot resolve accumulate function class")
                .hasMessageContaining("NoSuchClass");
    }
```

- [ ] **Step 2: Add `customFunctionFieldNotFound` test**

```java
    @Test
    void customFunctionFieldNotFound() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.acc.TestAccFuncs;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var ss = TestAccFuncs.noSuchField(p.age),
                    do {}
                }
                """;
        assertThatThrownBy(() -> new DrlxRuleBuilder().build(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("has no static AccumulateFunction field")
                .hasMessageContaining("noSuchField");
    }
```

- [ ] **Step 3: Add `customFunctionFieldWrongType` test**

```java
    @Test
    void customFunctionFieldWrongType() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.acc.BadContainer;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    var p : /persons,
                    var ss = BadContainer.notAFunction(p.age),
                    do {}
                }
                """;
        assertThatThrownBy(() -> new DrlxRuleBuilder().build(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("is not an AccumulateFunction");
    }
```

- [ ] **Step 4: Run all error case tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest#customFunctionClassNotFound+customFunctionFieldNotFound+customFunctionFieldWrongType -pl .`
Expected: All 3 pass

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: add error case tests for custom accumulate function resolution

Tests: class not found, field not found, field wrong type.

Refs #53"
```

---

### Task 7: Full Test Suite Verification

- [ ] **Step 1: Run the full accumulate test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=AccumulateTest -pl .`
Expected: All tests pass (no regressions in built-in function tests)

- [ ] **Step 2: Run the full drlx-parser-core test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 3: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install`
Expected: BUILD SUCCESS
