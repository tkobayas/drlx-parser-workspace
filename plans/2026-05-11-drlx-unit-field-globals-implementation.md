# DRLX Unit-Field Globals Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make DRLX's `DrlxRuleAstRuntimeBuilder` register every public, non-static unit-class field as a global on the `KiePackage` and merge those symbols into the MVEL3 consequence type map, so rule consequences can reference unit fields like `do { alerts.add(...); }`. Unblocks the disabled `DataStoreAddProbeTest` and adds happy-path tests for `add` and `remove(T)`.

**Architecture:** Add a `buildGlobalTypeMap(unitClass)` helper symmetric to the existing `buildEntryPointTypeMap`. In `build()`, register each entry via `pkg.addGlobal(name, type)`. In `buildRule()`, merge the unit-field types (as `Type.type(rawClass)`) into the map already passed to `lambdaCompiler.createLambdaConsequence`. No grammar, IR, proto, or runtime API change.

**Tech Stack:** Java 17, JUnit 5, AssertJ, MVEL3 (`org.mvel3.Type`), Drools 10.1 (`org.drools.base.definitions.impl.KnowledgePackageImpl#addGlobal(String, java.lang.reflect.Type)`).

**Spec:** `specs/2026-05-11-drlx-unit-field-globals-design.md`
**Issue:** https://github.com/tkobayas/drlx-parser/issues/37
**Epic:** #26

---

## File Structure

**Create — test (1 file):**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilderTest.java` — isolated unit test for the new `buildGlobalTypeMap` helper

**Modify — production (1 file):**
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` — add `buildGlobalTypeMap` + `erasure` helpers; register globals on the package; merge into consequence type map

**Modify — test (1 file):**
- `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreAddProbeTest.java` — drop `@Disabled`, refresh Javadoc, rename class to `DataStoreCrudTest`, add `removeByObjectViaDataStore` and `consequenceCanReferenceMultipleUnitFields`

**Out of scope:** any other file. No changes to `DrlxLambdaCompiler`, `MyUnit`, the wrapper, or drools-ruleunits.

---

## Test command reference

```bash
# Single test class
mvn -pl drlx-parser-core -am test -Dtest=ClassName

# Full module suite
mvn -pl drlx-parser-core -am test

# Install (run once after src/main changes complete)
mvn -pl drlx-parser-core -am install -DskipTests
```

---

## Task 1: `buildGlobalTypeMap` helper (TDD in isolation)

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilderTest.java` (new)

- [ ] **Step 1: Write the failing test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilderTest.java`:

```java
package org.drools.drlx.builder;

import java.lang.reflect.ParameterizedType;
import java.util.Map;

import org.drools.drlx.domain.Address;
import org.drools.drlx.domain.Person;
import org.drools.ruleunits.api.DataStore;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DrlxRuleAstRuntimeBuilderTest {

    public static class Fixture {
        public DataStore<Person> persons;
        public int counter;
        public static String IGNORED;
        @SuppressWarnings("unused")
        private DataStore<Address> hidden;
    }

    @Test
    void buildGlobalTypeMapIncludesPublicNonStaticFields() {
        Map<String, java.lang.reflect.Type> map =
                DrlxRuleAstRuntimeBuilder.buildGlobalTypeMap(Fixture.class);

        assertThat(map).containsOnlyKeys("persons", "counter");
    }

    @Test
    void buildGlobalTypeMapPreservesGenericTypeForDataSourceField() {
        Map<String, java.lang.reflect.Type> map =
                DrlxRuleAstRuntimeBuilder.buildGlobalTypeMap(Fixture.class);

        java.lang.reflect.Type personsType = map.get("persons");
        assertThat(personsType).isInstanceOf(ParameterizedType.class);
        ParameterizedType pt = (ParameterizedType) personsType;
        assertThat(pt.getRawType()).isEqualTo(DataStore.class);
        assertThat(pt.getActualTypeArguments()).containsExactly(Person.class);
    }

    @Test
    void buildGlobalTypeMapPreservesPrimitiveType() {
        Map<String, java.lang.reflect.Type> map =
                DrlxRuleAstRuntimeBuilder.buildGlobalTypeMap(Fixture.class);

        assertThat(map.get("counter")).isEqualTo(int.class);
    }
}
```

- [ ] **Step 2: Run the test, verify it fails**

```bash
mvn -pl drlx-parser-core -am test -Dtest=DrlxRuleAstRuntimeBuilderTest
```

Expected: compilation failure — `DrlxRuleAstRuntimeBuilder.buildGlobalTypeMap` does not exist.

- [ ] **Step 3: Add `buildGlobalTypeMap` to `DrlxRuleAstRuntimeBuilder`**

In `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`, add the import for `Modifier` near the existing reflection imports:

```java
import java.lang.reflect.Modifier;
```

Then add the helper next to the existing `buildEntryPointTypeMap` method (immediately after line 172, the closing brace of that method). Make it **package-private** (no modifier) so the test in the same package can call it directly:

```java
static Map<String, java.lang.reflect.Type> buildGlobalTypeMap(Class<?> unitClass) {
    Map<String, java.lang.reflect.Type> map = new LinkedHashMap<>();
    for (Field field : unitClass.getDeclaredFields()) {
        int mods = field.getModifiers();
        if (!Modifier.isPublic(mods) || Modifier.isStatic(mods)) {
            continue;
        }
        map.put(field.getName(), field.getGenericType());
    }
    return map;
}
```

- [ ] **Step 4: Run the test, verify it passes**

```bash
mvn -pl drlx-parser-core -am test -Dtest=DrlxRuleAstRuntimeBuilderTest
```

Expected: 3 tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilderTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ commit -m "$(cat <<'EOF'
feat: introduce buildGlobalTypeMap to enumerate unit-class fields

Pure helper symmetric to the existing buildEntryPointTypeMap: walks public,
non-static fields of the unit class and returns a name -> reflect Type map.
No call sites yet — wired into build()/buildRule() in the next commit.

Refs #37
EOF
)"
```

---

## Task 2: Wire globals end-to-end, re-enable the disabled probe

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreAddProbeTest.java` (rename to `DataStoreCrudTest.java`)

- [ ] **Step 1: Rename probe file and re-enable the test**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ mv \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreAddProbeTest.java \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java
```

Replace the **entire contents** of the renamed file with the following (rename the class, drop `@Disabled` on the existing method, refresh the Javadoc, keep the existing test as-is otherwise):

```java
package org.drools.drlx.ruleunit;

import org.drools.drlx.builder.DrlxRuleBuilder;
import org.drools.drlx.domain.Person;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.DisabledIfSystemProperty;
import org.kie.api.KieBase;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * End-to-end tests for issue #37 (DataStore CRUD): each rule consequence
 * calls a {@code DataStore} method on a unit-field reference (e.g.
 * {@code persons1.add(p)}), and {@link DrlxRuleUnitInstance} provides the
 * runtime surface. The class is intentionally named after the broader
 * #37 scope; it will grow as additional sub-pieces (update, with-block) land.
 */
@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")
class DataStoreCrudTest {

    @Test
    void consequenceCanCallDataStoreAdd() {
        String rule =
                """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule CopyAdults {
                    Person p : /persons[ age > 30 ],
                    do { persons1.add(p); }
                }
                """;
        KieBase kieBase = new DrlxRuleBuilder().build(rule);

        MyUnit unit = new MyUnit();
        Person alice = new Person("Alice", 40);
        Person bob = new Person("Bob", 20);
        unit.persons.add(alice);
        unit.persons.add(bob);

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            TestDataObserver<Person> obs = TestDataObserver.subscribeTo(unit.persons1);

            assertThat(instance.fire()).isEqualTo(1);
            assertThat(obs.inserted()).containsExactly(alice);
        }
    }
}
```

- [ ] **Step 2: Run the renamed test, verify it fails with UnsolvedSymbol**

```bash
mvn -pl drlx-parser-core -am test -Dtest=DataStoreCrudTest
```

Expected: 1 test errors with `UnsolvedSymbol Unsolved symbol in persons1 : Solving persons1`. The probe is no longer disabled, the parser still can't resolve the unit-field reference.

- [ ] **Step 3: Wire globals through `build()` and `buildRule()`**

In `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`:

**3a.** Add the `erasure` helper near `buildGlobalTypeMap` (it converts a `java.lang.reflect.Type` to its raw `Class<?>` for the MVEL3 type map):

```java
private static Class<?> erasure(java.lang.reflect.Type t) {
    if (t instanceof Class<?> c) {
        return c;
    }
    if (t instanceof ParameterizedType pt && pt.getRawType() instanceof Class<?> c) {
        return c;
    }
    return null;
}
```

**3b.** In `build(CompilationUnitIR parseResult)` — the existing method around line 55 — change the rule-build section so that:

1. A global type map is computed once.
2. Every entry is registered on the package.
3. The map is passed into `buildRule`.

Replace this existing block:

```java
        Map<String, Class<?>> entryPointTypes = buildEntryPointTypeMap(unitClass);

        Map<String, KnowledgePackageImpl> typeDeclPackages = new LinkedHashMap<>();
        registerTypeDeclarations(typeDeclPackages, parseResult, pkg.getTypeResolver(), entryPointTypes, unitClass);

        parseResult.rules().forEach(rule ->
                pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass)));
```

with:

```java
        Map<String, Class<?>> entryPointTypes = buildEntryPointTypeMap(unitClass);
        Map<String, java.lang.reflect.Type> globalTypes = buildGlobalTypeMap(unitClass);
        globalTypes.forEach(pkg::addGlobal);

        Map<String, KnowledgePackageImpl> typeDeclPackages = new LinkedHashMap<>();
        registerTypeDeclarations(typeDeclPackages, parseResult, pkg.getTypeResolver(), entryPointTypes, unitClass);

        parseResult.rules().forEach(rule ->
                pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass, globalTypes)));
```

**3c.** Change `buildRule` to accept and use the globals. Replace the existing signature and body of the method that starts:

```java
    private RuleImpl buildRule(RuleIR parseResult,
                               TypeResolver typeResolver,
                               Map<String, Class<?>> entryPointTypes,
                               Class<?> unitClass) {
```

with:

```java
    private RuleImpl buildRule(RuleIR parseResult,
                               TypeResolver typeResolver,
                               Map<String, Class<?>> entryPointTypes,
                               Class<?> unitClass,
                               Map<String, java.lang.reflect.Type> globalTypes) {
        lambdaCompiler.beginRule(parseResult.name());

        RuleImpl rule = new RuleImpl(parseResult.name());
        rule.setResource(rule.getResource());
        applyAnnotations(rule, parseResult.annotations());

        GroupElement root = GroupElementFactory.newAndInstance();
        Map<String, BoundVariable> boundVariables = new LinkedHashMap<>();

        buildLhs(parseResult.lhs(), root, typeResolver, entryPointTypes, unitClass, boundVariables);

        if (parseResult.rhs() != null) {
            Map<String, Type<?>> types = lambdaCompiler.getTypeMap(root);
            for (Map.Entry<String, java.lang.reflect.Type> e : globalTypes.entrySet()) {
                Class<?> raw = erasure(e.getValue());
                if (raw != null) {
                    types.put(e.getKey(), Type.type(raw));
                }
            }
            rule.setConsequence(lambdaCompiler.createLambdaConsequence(parseResult.rhs().block(), types));
        }

        rule.setLhs(root);
        return rule;
    }
```

The only changes from the original are: an extra `globalTypes` parameter, and the four-line `for` loop that merges those globals into the MVEL3 type map after `getTypeMap`.

- [ ] **Step 4: Run the re-enabled test, verify it passes**

```bash
mvn -pl drlx-parser-core -am test -Dtest=DataStoreCrudTest
```

Expected: 1 test passes.

Common failure modes if it does not:

1. **Still `UnsolvedSymbol`** — the merge isn't reaching the consequence type map. Double-check that the `for` loop in `buildRule` is *after* `lambdaCompiler.getTypeMap(root)` and *before* `createLambdaConsequence`, and that the put calls actually go into the same `types` map (not a copy).
2. **NoSuchMethodError on `pkg.addGlobal`** — wrong overload. Confirm that `KnowledgePackageImpl.addGlobal(String, java.lang.reflect.Type)` exists at line 421 of `KnowledgePackageImpl.java` (it does in Drools 10.1.0).
3. **MVEL3 compile error like "method add(Person) not found on raw DataStore"** — generic erasure caused MVEL3 to look for `add(Object)`. If this surfaces, the fix is to encode generics in the MVEL3 `Type` constructor (`new Type(rawClass, "<Person>")`) — but the spec defers this and we should NOT preemptively add it. Re-raise to the user.

- [ ] **Step 5: Run the full module suite, verify nothing regressed**

```bash
mvn -pl drlx-parser-core -am test
```

Expected: all tests pass. The total goes up by 3 compared to the pre-Task-1 baseline (the three new `DrlxRuleAstRuntimeBuilderTest` cases). The probe test count is unchanged — it just transitions from Skipped to Passed in the `default-test` profile. What matters: **0 failures, 0 errors.**

- [ ] **Step 6: Install the module so downstream consumers resolve the changes**

```bash
mvn -pl drlx-parser-core -am install -DskipTests
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreAddProbeTest.java \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ commit -m "$(cat <<'EOF'
feat: register unit-class fields as globals in DRLX-built packages

Mirrors PackageModel.addRuleUnitVariable from upstream exec-model codegen:
every public, non-static field on the unit class becomes a global on the
KiePackage, and the MVEL3 consequence type map gets the field name -> raw
class binding so `do { persons1.add(p); }` resolves end-to-end.

Re-enables DataStoreCrudTest.consequenceCanCallDataStoreAdd (renamed from
DataStoreAddProbeTest) which was the pinned failing test that motivated
this work.

Refs #37
EOF
)"
```

---

## Task 3: Add `remove(T)` and multi-field happy-path tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java`

- [ ] **Step 1: Add the two new tests to the existing file**

Append the following two methods inside the `DataStoreCrudTest` class (after `consequenceCanCallDataStoreAdd`):

```java
    @Test
    void removeByObjectViaDataStore() {
        String rule =
                """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule RemoveAdults {
                    Person p : /persons[ age > 30 ],
                    do { persons.remove(p); }
                }
                """;
        KieBase kieBase = new DrlxRuleBuilder().build(rule);

        MyUnit unit = new MyUnit();
        Person alice = new Person("Alice", 40);
        unit.persons.add(alice);

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            TestDataObserver<Person> obs = TestDataObserver.subscribeTo(unit.persons);

            assertThat(instance.fire()).isEqualTo(1);
            assertThat(obs.removed()).hasSize(1);
        }
    }

    @Test
    void consequenceCanReferenceMultipleUnitFields() {
        String rule =
                """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule FanOut {
                    Person p : /persons[ age > 30 ],
                    do { persons1.add(p); persons2.add(p); }
                }
                """;
        KieBase kieBase = new DrlxRuleBuilder().build(rule);

        MyUnit unit = new MyUnit();
        Person alice = new Person("Alice", 40);
        unit.persons.add(alice);

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            TestDataObserver<Person> obs1 = TestDataObserver.subscribeTo(unit.persons1);
            TestDataObserver<Person> obs2 = TestDataObserver.subscribeTo(unit.persons2);

            assertThat(instance.fire()).isEqualTo(1);
            assertThat(obs1.inserted()).containsExactly(alice);
            assertThat(obs2.inserted()).containsExactly(alice);
        }
    }
```

- [ ] **Step 2: Run the updated test class, verify all 3 tests pass**

```bash
mvn -pl drlx-parser-core -am test -Dtest=DataStoreCrudTest
```

Expected: 3 tests pass.

Common failure modes:

1. **`removeByObjectViaDataStore` asserts `obs.removed()` has size 1 but it's 0** — the rule may have fired but `persons.remove(p)` resolved against a no-op overload or a wrong method. Inspect the actual fact in `unit.persons` after fire — `persons.iterator().hasNext()` should be `false`. If the fact is still there, the bind protocol is correct but the consequence didn't actually remove it; investigate MVEL3 method resolution.
2. **`consequenceCanReferenceMultipleUnitFields` fires once but only one observer sees the insert** — likely a bug in the merge logic if it dropped one of the globals. Verify by adding a `System.out.println(types.keySet())` in `buildRule` before `createLambdaConsequence`.

- [ ] **Step 3: Run the full module suite, verify nothing regressed**

```bash
mvn -pl drlx-parser-core -am test
```

Expected: all tests pass. The total goes up by 2 compared to the pre-Task-3 count (the two new tests added in this step). What matters: **0 failures, 0 errors.**

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ add \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser/ commit -m "$(cat <<'EOF'
test: cover DataStore.remove(T) and multi-field references in consequence

Locks in two further behaviours unlocked by the global-registration change:
calling DataStore.remove(T) on a pattern-bound fact, and referencing more
than one unit field within the same consequence body.

Refs #37
EOF
)"
```

---

## Verification

After all three tasks land:

- [ ] **Full module test suite**

```bash
mvn -pl drlx-parser-core -am test
```

Expected: all tests pass, **0 failures, 0 errors**. Net delta vs the baseline at the start of this plan: +5 tests (3 from Task 1 + 2 from Task 3). The previously-skipped `consequenceCanCallDataStoreAdd` is now executed and passing in the `default-test` profile.

- [ ] **Full module install (so downstream modules pick up the new globals path)**

```bash
mvn -pl drlx-parser-core -am install
```

Expected: BUILD SUCCESS across the reactor.

- [ ] **Confirm spec → tasks alignment**

| Spec section                                            | Plan task |
|---------------------------------------------------------|-----------|
| `buildGlobalTypeMap` helper                             | Task 1, Step 3 |
| `KnowledgePackageImpl.addGlobal` registration in `build` | Task 2, Step 3b |
| Consequence type-map merge in `buildRule` (after LHS bindings) | Task 2, Step 3c |
| `erasure` helper for `java.lang.reflect.Type` → `Class<?>` | Task 2, Step 3a |
| Re-enable `DataStoreAddProbeTest.consequenceCanCallDataStoreAdd` | Task 2, Step 1 |
| Optional rename to `DataStoreCrudTest`                  | Task 2, Step 1 (rename applied) |
| `removeByObjectViaDataStore` test                       | Task 3, Step 1 |
| `consequenceCanReferenceMultipleUnitFields` test        | Task 3, Step 1 |
| Unit test for `buildGlobalTypeMap`                      | Task 1, Step 1 |
| No changes to `DataStore` API / drools-core / drools-ruleunits | Verified by file list above |
| `update(T)` deferred                                    | Out of scope — no task |
| `with`-block compact update deferred                    | Out of scope — no task |

If any spec requirement lacks a task, stop and add it before starting Task 1.
