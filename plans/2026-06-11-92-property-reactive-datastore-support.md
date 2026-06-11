# #92 Plain Property Reactivity via DataStoreSupport — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable property reactivity for `@PropertyReactive` classes in DRLX by threading property names through `DataStoreSupport`, for both external updates and consequence-side updates.

**Architecture:** `DataStoreSupport` (in drlx-parser) becomes the facade for property-aware updates. No upstream drools `DataStore` API changes. `DrlxRuleUnitInstance` exposes `InternalRuleBase` for BitMask calculation. The `DataStoreUpdateRewriter` is enhanced to pass `__ruleBase__` and extract property names from `CompactWithExpression`.

**Tech Stack:** Java 17, drools-ruleunits (runtime), MVEL3 (CompactWithExpression AST), JUnit 5 + AssertJ

**Spec:** `specs/2026-06-11-92-property-reactive-datastore-support-design.md`

---

## File Map

| File | Role |
|------|------|
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveTest.java` (new) | 6 runtime tests for property reactivity |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/DataStoreUpdateRewriterTest.java` (modify) | Add rewriter tests for `__ruleBase__` and property extraction |
| `drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java` (modify) | Add `getRuleBase()` accessor |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreSupport.java` (modify) | New external update method; enhance consequence method |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaConsequence.java` (modify) | Add `__ruleBase__` to consequence vars |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` (modify) | Register `__ruleBase__` type |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreUpdateRewriter.java` (modify) | Pass `__ruleBase__`; extract properties from CompactWithExpression |

---

### Task 1: Write external update tests (failing)

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveTest.java`

- [ ] **Step 1: Create `PropertyReactiveTest` with 3 external update tests**

```java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.builder.DataStoreSupport;
import org.drools.drlx.domain.ReactiveEmployee;
import org.drools.ruleunits.api.DataHandle;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PropertyReactiveTest extends DrlxBuilderTestSupport {

    private static final String SALARY_CONSTRAINT_RULE = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.ReactiveEmployee;
            import org.drools.drlx.ruleunit.MyUnit;

            unit MyUnit;

            rule R1 {
                var e : /reactiveEmployees[salary > 5000],
                do { }
            }
            """;

    @Test
    void externalUpdate_firesOnConstraintProperty() {
        withMyUnitInstance(SALARY_CONSTRAINT_RULE, (instance, unit, listener) -> {
            ReactiveEmployee emp = new ReactiveEmployee(6000, 4000, 1000);
            DataHandle dh = unit.reactiveEmployees.add(emp);

            assertThat(instance.fire()).isEqualTo(1);
            listener.getAfterMatchFired().clear();

            // salary IS used in the constraint → re-fire expected
            emp.setSalary(7000);
            DataStoreSupport.update(unit.reactiveEmployees, dh, emp,
                    instance.getRuleBase(), "salary");
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }

    @Test
    void externalUpdate_doesNotFireOnUnrelatedProperty() {
        withMyUnitInstance(SALARY_CONSTRAINT_RULE, (instance, unit, listener) -> {
            ReactiveEmployee emp = new ReactiveEmployee(6000, 4000, 1000);
            DataHandle dh = unit.reactiveEmployees.add(emp);

            assertThat(instance.fire()).isEqualTo(1);
            listener.getAfterMatchFired().clear();

            // bonusPay NOT used in the constraint → no re-fire
            emp.setBonusPay(2000);
            DataStoreSupport.update(unit.reactiveEmployees, dh, emp,
                    instance.getRuleBase(), "bonusPay");
            assertThat(instance.fire()).isEqualTo(0);
            assertThat(listener.getAfterMatchFired()).isEmpty();
        });
    }

    @Test
    void externalUpdate_withoutPropertyNames_firesAlways() {
        withMyUnitInstance(SALARY_CONSTRAINT_RULE, (instance, unit, listener) -> {
            ReactiveEmployee emp = new ReactiveEmployee(6000, 4000, 1000);
            DataHandle dh = unit.reactiveEmployees.add(emp);

            assertThat(instance.fire()).isEqualTo(1);
            listener.getAfterMatchFired().clear();

            // No property names → AllSetBitMask → re-fire (backward compat)
            emp.setBonusPay(2000);
            unit.reactiveEmployees.update(dh, emp);
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=PropertyReactiveTest -pl . 2>&1 | tail -20`

Expected: Compilation failure — `getRuleBase()` and `DataStoreSupport.update(DataStore, DataHandle, Object, InternalRuleBase, String...)` do not exist yet.

- [ ] **Step 3: Commit failing tests**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#92): add external update property reactivity tests (red)

Refs #92"
```

---

### Task 2: Implement external update path

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java:47`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreSupport.java:19-34`

- [ ] **Step 1: Add `getRuleBase()` to `DrlxRuleUnitInstance`**

Add the following method after the `reteEvaluator` field declaration (after line 47):

```java
public InternalRuleBase getRuleBase() {
    return (InternalRuleBase) reteEvaluator.getKnowledgeBase();
}
```

No new imports needed — `InternalRuleBase` is already imported (line 8).

- [ ] **Step 2: Add external update method to `DataStoreSupport`**

Add the following method to `DataStoreSupport.java` after the existing `update` method (after line 33):

```java
public static void update(DataStore<?> store, DataHandle handle, Object fact,
                          InternalRuleBase ruleBase, String... modifiedProperties) {
    InternalStoreCallback callback = (InternalStoreCallback) store;
    BitMask mask = modifiedProperties.length == 0
            ? AllSetBitMask.get()
            : calculateUpdateBitMask(ruleBase, fact, modifiedProperties);
    callback.update(handle, fact, mask, fact.getClass(), null);
}
```

Add these imports to `DataStoreSupport.java`:

```java
import org.drools.core.impl.InternalRuleBase;
import org.drools.ruleunits.api.DataHandle;
import org.drools.util.bitmask.BitMask;

import static org.drools.kiesession.entrypoints.NamedEntryPoint.calculateUpdateBitMask;
```

Note: `AllSetBitMask` and `InternalStoreCallback` are already imported. `DataHandle` is already imported.

- [ ] **Step 3: Run external update tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=PropertyReactiveTest -pl .`

Expected: All 3 tests pass.

- [ ] **Step 4: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreSupport.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#92): add property-name-aware external update via DataStoreSupport

Refs #92"
```

---

### Task 3: Write consequence-side tests (failing)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveTest.java`

- [ ] **Step 1: Add 3 consequence-side tests to `PropertyReactiveTest`**

Append to the `PropertyReactiveTest` class:

```java
    @Test
    void consequenceUpdate_compactWith_firesOnConstraintProperty() {
        // R1 modifies salary (read by R2's constraint) via CompactWithExpression
        String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[basePay > 3000],
                    do { reactiveEmployees.update(e{salary = 9999}); }
                }

                rule R2 {
                    var e : /reactiveEmployees[salary > 9000],
                    do { }
                }
                """;

        withMyUnitInstance(rule, (instance, unit, listener) -> {
            ReactiveEmployee emp = new ReactiveEmployee(5000, 4000, 1000);
            unit.reactiveEmployees.add(emp);

            // R1 fires (basePay 4000 > 3000), sets salary to 9999.
            // R2 should fire because salary (modified by R1) is in R2's constraint.
            instance.fire();
            assertThat(listener.getAfterMatchFired()).contains("R1", "R2");
        });
    }

    @Test
    void consequenceUpdate_compactWith_doesNotFireOnUnrelatedProperty() {
        // R1 modifies bonusPay (NOT read by R2's constraint) via CompactWithExpression
        String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[basePay > 3000],
                    do { reactiveEmployees.update(e{bonusPay = 9999}); }
                }

                rule R2 {
                    var e : /reactiveEmployees[salary > 9000],
                    do { }
                }
                """;

        withMyUnitInstance(rule, (instance, unit, listener) -> {
            ReactiveEmployee emp = new ReactiveEmployee(5000, 4000, 1000);
            unit.reactiveEmployees.add(emp);

            // R1 fires (basePay 4000 > 3000), sets bonusPay to 9999.
            // R2 should NOT fire because bonusPay is not in R2's constraint
            // and salary (5000) does not match salary > 9000.
            instance.fire();
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }

    @Test
    void consequenceUpdate_plainSetter_treatsAllPropertiesAsChanged() {
        // R1 modifies bonusPay (NOT read by R2's constraint) via plain setter + update.
        // Because the rewriter cannot detect which properties were modified in
        // arbitrary control flow preceding the update() call, AllSetBitMask is used.
        // This means R2 re-fires even though only bonusPay changed — a safe default
        // that preserves correctness at the cost of unnecessary re-evaluations.
        String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[basePay > 3000],
                    do { e.setBonusPay(9999); reactiveEmployees.update(e); }
                }

                rule R2 {
                    var e : /reactiveEmployees[salary > 4000],
                    do { }
                }
                """;

        withMyUnitInstance(rule, (instance, unit, listener) -> {
            ReactiveEmployee emp = new ReactiveEmployee(5000, 4000, 1000);
            unit.reactiveEmployees.add(emp);

            // R1 fires (basePay 4000 > 3000) and calls update with AllSetBitMask.
            // R2 fires because AllSetBitMask treats all properties (including salary)
            // as changed, even though only bonusPay was actually modified.
            instance.fire();
            assertThat(listener.getAfterMatchFired()).contains("R1", "R2");
        });
    }
```

- [ ] **Step 2: Run tests to verify consequence-side tests fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=PropertyReactiveTest -pl . 2>&1 | tail -30`

Expected: The 3 external update tests pass. The consequence-side tests may pass or fail depending on current behavior — the key is to verify they compile and run. The `compactWith_doesNotFireOnUnrelatedProperty` test is the one that should fail: without property extraction, the rewriter uses `AllSetBitMask`, so R2 fires when it shouldn't.

- [ ] **Step 3: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#92): add consequence-side property reactivity tests

Refs #92"
```

---

### Task 4: Add `__ruleBase__` to consequence variable context

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaConsequence.java:80-91`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:409-411`

- [ ] **Step 1: Add `__ruleBase__` to `DrlxLambdaConsequence.evaluate()`**

In `DrlxLambdaConsequence.java`, add the `__ruleBase__` variable after `__match__` (line 89):

```java
    @Override
    public void evaluate(KnowledgeHelper knowledgeHelper, ValueResolver valueResolver) throws Exception {
        Map<String, Object> vars = new HashMap<>();
        InternalMatch match = knowledgeHelper.getMatch();
        Map<String, Declaration> declarations = match.getTerminalNode().getSubRule().getOuterDeclarations();
        for (String declarationId : match.getDeclarationIds()) {
            Declaration decl = declarations.get(declarationId);
            vars.put(declarationId, decl.getValue(valueResolver, match.getTuple()));
        }
        globalNames.forEach(name -> vars.put(name, valueResolver.getGlobal(name)));
        vars.put("__match__", match);
        vars.put("__ruleBase__", valueResolver.getRuleBase());

        evaluator.eval(vars);
    }
```

No new imports needed — `ValueResolver` is already imported (line 8), and `getRuleBase()` returns `RuleBase` (the base interface that `InternalRuleBase` extends).

- [ ] **Step 2: Register `__ruleBase__` type in `DrlxRuleAstRuntimeBuilder`**

In `DrlxRuleAstRuntimeBuilder.java`, after the `__match__` type registration (line 410), add:

```java
            if (!dataStoreGlobalNames.isEmpty()) {
                types.put("__match__", Type.type(InternalMatch.class));
                types.put("__ruleBase__", Type.type(InternalRuleBase.class));
            }
```

Add import at the top of `DrlxRuleAstRuntimeBuilder.java`:

```java
import org.drools.core.impl.InternalRuleBase;
```

- [ ] **Step 3: Verify existing tests still pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DataStoreCrudTest -pl .`

Expected: All existing DataStoreCrudTest tests pass (no behavioral change yet — `__ruleBase__` is available but not used until the rewriter is updated).

- [ ] **Step 4: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaConsequence.java drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#92): inject __ruleBase__ into consequence variable context

Refs #92"
```

---

### Task 5: Enhance `DataStoreSupport` consequence method and `DataStoreUpdateRewriter`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreSupport.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreUpdateRewriter.java:107-112`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DataStoreUpdateRewriterTest.java`

- [ ] **Step 1: Replace the consequence `update` method in `DataStoreSupport`**

Replace the existing `update(DataStore<?>, Object, InternalMatch, String)` method with:

```java
    public static void update(DataStore<?> store, Object fact, InternalMatch match,
                              InternalRuleBase ruleBase, String storeName,
                              String... modifiedProperties) {
        InternalStoreCallback callback = (InternalStoreCallback) store;
        DataHandle handle = Objects.requireNonNull(callback.lookup(fact),
                "DataStore '" + storeName + "' has no DataHandle for the given fact");
        BitMask mask = modifiedProperties.length == 0
                ? AllSetBitMask.get()
                : calculateUpdateBitMask(ruleBase, fact, modifiedProperties);
        callback.update(handle, fact, mask, fact.getClass(), match);
    }
```

The imports for `InternalRuleBase`, `BitMask`, and `calculateUpdateBitMask` were already added in Task 2.

- [ ] **Step 2: Update `DataStoreUpdateRewriter` to pass `__ruleBase__` and extract compact-with properties**

In `DataStoreUpdateRewriter.java`, replace the `rewriteCallIfMatch` method (lines 73-116):

```java
    private boolean rewriteCallIfMatch(MethodCallExpr call, Set<String> dataStoreGlobalNames) {
        if (!"update".equals(call.getNameAsString())) {
            return false;
        }
        if (call.getArguments().size() != 1) {
            return false;
        }
        Optional<Expression> scope = call.getScope();
        if (scope.isEmpty() || !(scope.get() instanceof NameExpr scopeName)) {
            return false;
        }
        if (!dataStoreGlobalNames.contains(scopeName.getNameAsString())) {
            return false;
        }
        Expression arg = call.getArgument(0);
        List<String> extractedProperties = List.of();
        if (arg instanceof CompactWithExpression compactWith) {
            if (call.getParentNode().orElse(null) instanceof ExpressionStmt exprStmt
                    && exprStmt.getParentNode().orElse(null) instanceof BlockStmt block) {
                int index = block.getStatements().indexOf(exprStmt);
                if (index >= 0) {
                    block.addStatement(index, new ExpressionStmt(compactWith.clone()));
                    extractedProperties = extractPropertyNames(compactWith);
                    arg = compactWith.getTarget().clone();
                    call.setArgument(0, arg);
                } else {
                    return false;
                }
            } else {
                return false;
            }
        }
        if (!(arg instanceof NameExpr) && !(arg instanceof FieldAccessExpr)) {
            return false;
        }

        String globalName = scopeName.getNameAsString();
        String argText = arg.toString();
        StringBuilder sb = new StringBuilder();
        sb.append("org.drools.drlx.builder.DataStoreSupport.update(")
                .append(globalName).append(", ")
                .append(argText).append(", __match__, __ruleBase__, ")
                .append("\"").append(globalName).append("\"");
        for (String prop : extractedProperties) {
            sb.append(", \"").append(prop).append("\"");
        }
        sb.append(")");
        Expression updateCall = StaticJavaParser.parseExpression(sb.toString());

        call.replace(updateCall);
        return true;
    }

    private static List<String> extractPropertyNames(CompactWithExpression compactWith) {
        return compactWith.getAssignments().stream()
                .map(assign -> assign.getTarget().toString())
                .toList();
    }
```

Add this import at the top of `DataStoreUpdateRewriter.java`:

```java
import java.util.List;
import org.mvel3.parser.ast.expr.CompactWithExpression;
```

- [ ] **Step 3: Update `DataStoreUpdateRewriterTest` to verify new output**

In `DataStoreUpdateRewriterTest.java`, update all assertions that check for the rewritten output to expect `__ruleBase__`:

Replace the assertion in `simpleUpdateWithNameExprArgIsRewritten` (line 37):

```java
        assertThat(result).contains("DataStoreSupport.update(alerts, t, __match__, __ruleBase__,");
```

Replace the assertion in `updateOnFieldAccessExprArgIsRewritten` (line 46):

```java
        assertThat(result).contains("DataStoreSupport.update(alerts, this.t, __match__, __ruleBase__,");
```

Replace the assertion in `shadowedGlobalIsRewrittenAnyway` (line 105):

```java
        assertThat(result).contains("DataStoreSupport.update(alerts, t, __match__, __ruleBase__,");
```

Replace the assertion in `compactWithPlusUpdateIsRewritten` (line 113):

```java
        assertThat(result).contains("DataStoreSupport.update(alerts, p, __match__, __ruleBase__,");
```

Replace the assertion in `compactWithMultipleAssignmentsPlusUpdateIsRewritten` (line 121):

```java
        assertThat(result).contains("DataStoreSupport.update(alerts, p, __match__, __ruleBase__,");
```

Replace the assertion in `compactWithAsUpdateArgIsRewritten` (line 129):

```java
        assertThat(result).contains("DataStoreSupport.update(alerts, t, __match__, __ruleBase__,");
```

Replace the assertion in `compactWithMultipleAssignmentsAsUpdateArgIsRewritten` (line 138):

```java
        assertThat(result).contains("DataStoreSupport.update(alerts, t, __match__, __ruleBase__,");
```

Add a new test to verify property names are extracted from CompactWithExpression:

```java
    @Test
    void compactWithAsUpdateArgExtractsPropertyNames() {
        String body = "alerts.update(t{salary = 7000});";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        assertThat(result).contains("DataStoreSupport.update(alerts, t, __match__, __ruleBase__, \"alerts\", \"salary\")");
    }

    @Test
    void compactWithMultipleAssignmentsExtractsAllPropertyNames() {
        String body = "alerts.update(t{salary = 7000, basePay = 5000});";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        assertThat(result).contains("\"alerts\", \"salary\", \"basePay\"");
    }

    @Test
    void plainUpdateDoesNotIncludePropertyNames() {
        String body = "alerts.update(t);";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        assertThat(result).contains("DataStoreSupport.update(alerts, t, __match__, __ruleBase__, \"alerts\")");
        // No extra property name arguments after the store name
        assertThat(result).doesNotContain("\"alerts\",");
    }
```

- [ ] **Step 4: Run rewriter unit tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DataStoreUpdateRewriterTest -pl .`

Expected: All tests pass (including existing and new ones).

- [ ] **Step 5: Run all `PropertyReactiveTest` tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=PropertyReactiveTest -pl .`

Expected: All 6 tests pass.

- [ ] **Step 6: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreSupport.java drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreUpdateRewriter.java drlx-parser-core/src/test/java/org/drools/drlx/builder/DataStoreUpdateRewriterTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#92): property-aware consequence updates via DataStoreSupport

Enhanced DataStoreUpdateRewriter to pass __ruleBase__ and extract
property names from CompactWithExpression. Plain setter-before-update
falls back to AllSetBitMask (safe default).

Refs #92"
```

---

### Task 6: Full test suite verification and close

**Files:** None (verification only)

- [ ] **Step 1: Run the full drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`

Expected: All tests pass. No regressions.

- [ ] **Step 2: If any failures, investigate and fix before proceeding**

Pay special attention to:
- `DataStoreCrudTest` — existing compact-with tests must still work with the new `__ruleBase__` parameter
- `PropertyReactiveWatchListTest` — still uses `withSession` (KieSession), unaffected by these changes
- Any test that uses `DataStoreSupport.update` directly

- [ ] **Step 3: Final commit if any fixes were needed**

Only if adjustments were made in Step 2.
