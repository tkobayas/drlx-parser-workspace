# DrlxRuleUnitInstance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `DrlxRuleUnitInstance<T extends RuleUnitData>` wrapper to drlx-parser-core that runs a DRLX-built `KieBase` against a populated `RuleUnitData`, mirroring the upstream `org.drools.ruleunits.api.RuleUnitInstance` surface without depending on the upstream `RuleUnitProvider` codegen path. Promote `MyUnit` to `RuleUnitData` so it can carry initialised `DataStore`s into the wrapper. Add a `TestDataObserver` utility for verifying consequence-side DataStore mutations in future #37 tests.

**Architecture:** The wrapper constructs a `RuleUnitExecutorImpl` (an internal `ReteEvaluator`) from the cast `InternalRuleBase`, then walks public `DataSource`-typed fields on the `RuleUnitData` to (a) `subscribe` an `EntryPointDataProcessor` to the matching named entry point, and (b) set the same DataSource as a global of the same name. `ListDataStore.subscribe` replays existing facts to new subscribers, so the standard "populate before construct" pattern works.

**Tech Stack:** Java 17, JUnit 5, AssertJ, Drools 10.1 (`drools-ruleunits-api`, `drools-ruleunits-impl`, `drools-core` — already compile-scope deps of `drlx-parser-core`).

**Spec:** `specs/2026-05-11-drlx-rule-unit-instance-design.md`
**Issue:** https://github.com/tkobayas/drlx-parser/issues/37
**Epic:** #26

---

## File Structure

**Create — production (1 file):**
- `drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java` — wrapper implementing `org.drools.ruleunits.api.RuleUnitInstance<T>`

**Create — test (2 files):**
- `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/TestDataObserver.java` — `DataProcessor<T>` that records inserts/updates/removes
- `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstanceTest.java` — end-to-end validation: pre-populate input DataStore → wrap → fire → assert fire count

**Modify (1 file):**
- `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java` — `implements RuleUnitData`, all 14 fields initialised via `DataSource.createStore()`

**Out of scope (not touched):**
- `MyUnitWithRawField.java` (parser-test fixture, never instantiated at runtime)
- `drlx-parser-benchmark/src/main/java/org/drools/drlx/ruleunit/MyUnit.java` (benchmark copy — not exercised by #37)
- Any of the 146 existing tests

---

## Test command reference

All test commands use the module-scoped form so dependent modules resolve transitively:

```bash
mvn -pl drlx-parser-core -am test -Dtest=ClassName
```

For a single test method:

```bash
mvn -pl drlx-parser-core -am test -Dtest=ClassName#methodName
```

---

## Task 1: Promote `MyUnit` to `RuleUnitData`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnitTest.java` (new)

- [ ] **Step 1: Write the failing test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnitTest.java`:

```java
package org.drools.drlx.ruleunit;

import org.drools.ruleunits.api.RuleUnitData;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MyUnitTest {

    @Test
    void implementsRuleUnitData() {
        assertThat(new MyUnit()).isInstanceOf(RuleUnitData.class);
    }

    @Test
    void dataStoresAreInitialised() {
        MyUnit unit = new MyUnit();
        assertThat(unit.persons).isNotNull();
        assertThat(unit.addresses).isNotNull();
        assertThat(unit.seniors).isNotNull();
        assertThat(unit.juniors).isNotNull();
        assertThat(unit.locations).isNotNull();
        assertThat(unit.persons1).isNotNull();
        assertThat(unit.persons2).isNotNull();
        assertThat(unit.persons3).isNotNull();
        assertThat(unit.childPositionedThings).isNotNull();
        assertThat(unit.duplicateLocations).isNotNull();
        assertThat(unit.plainLocations).isNotNull();
        assertThat(unit.objects).isNotNull();
        assertThat(unit.orders).isNotNull();
        assertThat(unit.reactiveEmployees).isNotNull();
    }
}
```

- [ ] **Step 2: Run the test, verify it fails**

```bash
mvn -pl drlx-parser-core -am test -Dtest=MyUnitTest
```

Expected: 2 failures. `implementsRuleUnitData` fails because `MyUnit` does not implement `RuleUnitData`. `dataStoresAreInitialised` fails on the first assertion because `unit.persons` is `null`.

- [ ] **Step 3: Update `MyUnit`**

Replace the contents of `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java` with:

```java
package org.drools.drlx.ruleunit;

import org.drools.drlx.domain.Address;
import org.drools.drlx.domain.ChildPositioned;
import org.drools.drlx.domain.DuplicatePositionLocation;
import org.drools.drlx.domain.Location;
import org.drools.drlx.domain.Order;
import org.drools.drlx.domain.Person;
import org.drools.drlx.domain.PlainLocation;
import org.drools.drlx.domain.ReactiveEmployee;
import org.drools.drlx.domain.Vehicle;
import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.DataStore;
import org.drools.ruleunits.api.RuleUnitData;

public class MyUnit implements RuleUnitData {
    public DataStore<Person> persons = DataSource.createStore();
    public DataStore<Address> addresses = DataSource.createStore();
    public DataStore<Person> seniors = DataSource.createStore();
    public DataStore<Person> juniors = DataSource.createStore();
    public DataStore<Location> locations = DataSource.createStore();
    public DataStore<Person> persons1 = DataSource.createStore();
    public DataStore<Person> persons2 = DataSource.createStore();
    public DataStore<Person> persons3 = DataSource.createStore();
    public DataStore<ChildPositioned> childPositionedThings = DataSource.createStore();
    public DataStore<DuplicatePositionLocation> duplicateLocations = DataSource.createStore();
    public DataStore<PlainLocation> plainLocations = DataSource.createStore();
    public DataStore<Vehicle> objects = DataSource.createStore();
    public DataStore<Order> orders = DataSource.createStore();
    public DataStore<ReactiveEmployee> reactiveEmployees = DataSource.createStore();
}
```

- [ ] **Step 4: Run the new test, verify it passes**

```bash
mvn -pl drlx-parser-core -am test -Dtest=MyUnitTest
```

Expected: 2 tests pass.

- [ ] **Step 5: Run the full test suite, verify nothing regressed**

```bash
mvn -pl drlx-parser-core -am test
```

Expected: all tests pass (146 existing + 2 new = 148). If any existing test fails, the change to `MyUnit` broke a parse-time assumption — investigate before continuing. Existing tests reference `MyUnit` only via DRLX source `import` / `unit` statements, so failures here are unexpected.

- [ ] **Step 6: Commit**

```bash
git add drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java \
        drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnitTest.java
git commit -m "$(cat <<'EOF'
test: promote MyUnit to RuleUnitData with initialised DataStores

Required for DrlxRuleUnitInstance to walk MyUnit's DataSource fields at
construction time and bind them to entry points / globals. Field types
and visibility unchanged; existing tests continue to reference MyUnit
purely as a parse-time symbol and are unaffected.

Refs #37
EOF
)"
```

---

## Task 2: Add `TestDataObserver` utility

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/TestDataObserver.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/TestDataObserverTest.java` (new)

- [ ] **Step 1: Write the failing test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/TestDataObserverTest.java`:

```java
package org.drools.drlx.ruleunit;

import org.drools.ruleunits.api.DataHandle;
import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.DataStore;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TestDataObserverTest {

    @Test
    void capturesInserts() {
        DataStore<String> store = DataSource.createStore();
        TestDataObserver<String> obs = TestDataObserver.subscribeTo(store);

        store.add("a");
        store.add("b");

        assertThat(obs.inserted()).containsExactly("a", "b");
        assertThat(obs.updated()).isEmpty();
        assertThat(obs.removed()).isEmpty();
    }

    @Test
    void replaysExistingItemsOnSubscribe() {
        DataStore<String> store = DataSource.createStore();
        store.add("pre-existing");

        TestDataObserver<String> obs = TestDataObserver.subscribeTo(store);

        assertThat(obs.inserted()).containsExactly("pre-existing");
    }

    @Test
    void capturesUpdate() {
        DataStore<String> store = DataSource.createStore();
        DataHandle handle = store.add("v1");

        TestDataObserver<String> obs = TestDataObserver.subscribeTo(store);
        store.update(handle, "v2");

        assertThat(obs.inserted()).containsExactly("v1");
        assertThat(obs.updated()).containsExactly("v2");
    }

    @Test
    void capturesRemove() {
        DataStore<String> store = DataSource.createStore();
        DataHandle handle = store.add("doomed");

        TestDataObserver<String> obs = TestDataObserver.subscribeTo(store);
        store.remove(handle);

        assertThat(obs.removed()).containsExactly(handle);
    }
}
```

- [ ] **Step 2: Run the test, verify it fails**

```bash
mvn -pl drlx-parser-core -am test -Dtest=TestDataObserverTest
```

Expected: compilation failure — `TestDataObserver` does not exist.

- [ ] **Step 3: Implement `TestDataObserver`**

Create `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/TestDataObserver.java`:

```java
package org.drools.drlx.ruleunit;

import java.util.ArrayList;
import java.util.List;

import org.drools.ruleunits.api.DataHandle;
import org.drools.ruleunits.api.DataProcessor;
import org.drools.ruleunits.api.DataSource;
import org.kie.api.runtime.rule.FactHandle;

/**
 * Test-only {@link DataProcessor} that records every insert / update / remove
 * notification it receives from a {@link DataSource}. Use {@link #subscribeTo}
 * to attach a fresh observer to a DataSource before firing rules, then read
 * back {@link #inserted()} / {@link #updated()} / {@link #removed()} to verify
 * what the rule consequence did.
 *
 * <p>Subscribing to a non-empty DataSource will replay its current contents
 * through {@link #insert(DataHandle, Object)} — that is the upstream
 * {@code ListDataStore} contract, not behaviour added here.
 */
public final class TestDataObserver<T> implements DataProcessor<T> {

    private final List<T> inserted = new ArrayList<>();
    private final List<T> updated = new ArrayList<>();
    private final List<DataHandle> removed = new ArrayList<>();

    public static <T> TestDataObserver<T> subscribeTo(DataSource<T> source) {
        TestDataObserver<T> observer = new TestDataObserver<>();
        source.subscribe(observer);
        return observer;
    }

    @Override
    public FactHandle insert(DataHandle handle, T object) {
        inserted.add(object);
        return null;
    }

    @Override
    public void update(DataHandle handle, T object) {
        updated.add(object);
    }

    @Override
    public void delete(DataHandle handle) {
        removed.add(handle);
    }

    public List<T> inserted() {
        return List.copyOf(inserted);
    }

    public List<T> updated() {
        return List.copyOf(updated);
    }

    public List<DataHandle> removed() {
        return List.copyOf(removed);
    }
}
```

- [ ] **Step 4: Run the test, verify it passes**

```bash
mvn -pl drlx-parser-core -am test -Dtest=TestDataObserverTest
```

Expected: 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/TestDataObserver.java \
        drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/TestDataObserverTest.java
git commit -m "$(cat <<'EOF'
test: add TestDataObserver utility for DataStore mutation assertions

Records inserts, updates, and removes on a DataSource so tests can verify
what a rule consequence did, not just the final fact set. Used by upcoming
DrlxRuleUnitInstance integration tests and #37 DataStore CRUD tests.

Refs #37
EOF
)"
```

---

## Task 3: Add `DrlxRuleUnitInstance` wrapper

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstanceTest.java` (new)

- [ ] **Step 1: Write the failing test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstanceTest.java`:

```java
package org.drools.drlx.ruleunit;

import org.drools.core.event.TrackingAgendaEventListener;
import org.drools.drlx.builder.DrlxRuleBuilder;
import org.drools.drlx.domain.Person;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.DisabledIfSystemProperty;
import org.kie.api.KieBase;

import static org.assertj.core.api.Assertions.assertThat;

@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")
class DrlxRuleUnitInstanceTest {

    private static final String RULE =
            """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule AdultMatch {
                Person p : /persons[ age > 30 ],
                do { System.out.println(p); }
            }
            """;

    @Test
    void firesRuleAgainstPrePopulatedDataStore() {
        KieBase kieBase = new DrlxRuleBuilder().build(RULE);

        MyUnit unit = new MyUnit();
        unit.persons.add(new Person("Alice", 40));
        unit.persons.add(new Person("Bob", 20));

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            assertThat(instance.fire()).isEqualTo(1);
        }
    }

    @Test
    void exposesRuleUnitData() {
        KieBase kieBase = new DrlxRuleBuilder().build(RULE);
        MyUnit unit = new MyUnit();

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            assertThat(instance.ruleUnitData()).isSameAs(unit);
        }
    }

    @Test
    void factsAddedAfterConstructionAreVisibleToRules() {
        KieBase kieBase = new DrlxRuleBuilder().build(RULE);
        MyUnit unit = new MyUnit();

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            unit.persons.add(new Person("Alice", 40));
            assertThat(instance.fire()).isEqualTo(1);
        }
    }

    @Test
    void closeIsIdempotentViaTryWithResources() {
        KieBase kieBase = new DrlxRuleBuilder().build(RULE);
        MyUnit unit = new MyUnit();
        DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit);
        instance.close();
        // second close must not throw — try-with-resources may call it again under exception paths
        instance.close();
    }
}
```

- [ ] **Step 2: Run the test, verify it fails**

```bash
mvn -pl drlx-parser-core -am test -Dtest=DrlxRuleUnitInstanceTest
```

Expected: compilation failure — `DrlxRuleUnitInstance` does not exist.

- [ ] **Step 3: Implement `DrlxRuleUnitInstance`**

Create `drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java`:

```java
package org.drools.drlx.ruleunit;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;

import org.drools.core.common.ReteEvaluator;
import org.drools.core.impl.InternalRuleBase;
import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.RuleUnit;
import org.drools.ruleunits.api.RuleUnitData;
import org.drools.ruleunits.api.RuleUnitInstance;
import org.drools.ruleunits.impl.EntryPointDataProcessor;
import org.drools.ruleunits.impl.sessions.RuleUnitExecutorImpl;
import org.kie.api.KieBase;
import org.kie.api.runtime.rule.AgendaFilter;
import org.kie.api.runtime.rule.QueryResults;
import org.kie.api.time.SessionClock;

/**
 * Runs a DRLX-built {@link KieBase} against a populated {@link RuleUnitData},
 * bridging DRLX (which compiles rules from a string at runtime) and the
 * {@link RuleUnitInstance} surface from {@code drools-ruleunits-api}.
 *
 * <p>The upstream {@code RuleUnitProvider.createRuleUnitInstance(unit)} path
 * is not usable for DRLX: it requires a generated {@code RuleUnit<T>} class
 * registered via {@link java.util.ServiceLoader}, produced by scanning {@code
 * .drl} files on the classpath. DRLX has no equivalent codegen, so this
 * wrapper performs the bind step directly against the {@link KieBase}.
 *
 * <p>For each public, non-static {@link DataSource} field declared on {@code
 * T}, the constructor (a) subscribes an
 * {@link EntryPointDataProcessor} to the same-named entry point on the
 * underlying {@link ReteEvaluator}, and (b) sets the DataSource as a global
 * of the same name. Globals that the rule unit did not declare are silently
 * skipped — same convention as the upstream
 * {@code AbstractRuleUnitInstance.bind}.
 *
 * <p>{@link #unit()} returns {@code null}: there is no upstream
 * {@code RuleUnit<T>} for a DRLX-built KieBase. Tests that need a
 * {@code RuleUnit} reference cannot use this wrapper.
 */
public final class DrlxRuleUnitInstance<T extends RuleUnitData> implements RuleUnitInstance<T> {

    private final T unitData;
    private final ReteEvaluator reteEvaluator;

    public static <T extends RuleUnitData> DrlxRuleUnitInstance<T> create(KieBase kieBase, T unitData) {
        return new DrlxRuleUnitInstance<>(kieBase, unitData);
    }

    private DrlxRuleUnitInstance(KieBase kieBase, T unitData) {
        this.unitData = unitData;
        this.reteEvaluator = new RuleUnitExecutorImpl((InternalRuleBase) kieBase);
        bind();
    }

    private void bind() {
        for (Field field : unitData.getClass().getDeclaredFields()) {
            int mods = field.getModifiers();
            if (!Modifier.isPublic(mods) || Modifier.isStatic(mods)) {
                continue;
            }
            Object value;
            try {
                value = field.get(unitData);
            } catch (IllegalAccessException e) {
                throw new IllegalStateException("Cannot read field " + field.getName(), e);
            }
            if (value == null) {
                continue;
            }
            String name = field.getName();
            if (value instanceof DataSource<?> ds) {
                ds.subscribe(new EntryPointDataProcessor(reteEvaluator.getEntryPoint(name)));
            }
            try {
                reteEvaluator.setGlobal(name, value);
            } catch (RuntimeException ignored) {
                // Global not declared in this rule unit — same convention as upstream bind.
            }
        }
    }

    @Override
    public RuleUnit<T> unit() {
        return null;
    }

    @Override
    public T ruleUnitData() {
        return unitData;
    }

    @Override
    public int fire() {
        return reteEvaluator.fireAllRules();
    }

    @Override
    public int fire(AgendaFilter agendaFilter) {
        return reteEvaluator.fireAllRules(agendaFilter);
    }

    @Override
    public QueryResults executeQuery(String query, Object... arguments) {
        fire();
        return reteEvaluator.getQueryResults(query, arguments);
    }

    @Override
    @SuppressWarnings("unchecked")
    public <C extends SessionClock> C getClock() {
        return (C) reteEvaluator.getSessionClock();
    }

    @Override
    public void close() {
        reteEvaluator.dispose();
    }
}
```

- [ ] **Step 4: Run the test, verify it passes**

```bash
mvn -pl drlx-parser-core -am test -Dtest=DrlxRuleUnitInstanceTest
```

Expected: 4 tests pass.

Common failure modes to debug if it does not:

1. `ClassCastException` casting `KieBase` to `InternalRuleBase` — `DrlxRuleBuilder` returned something other than the expected runtime type. Inspect what `DrlxRuleBuilder.build` returns.
2. `firesRuleAgainstPrePopulatedDataStore` returns 0 fires — either subscribe-replay isn't happening (unlikely; verified in `ListDataStore.subscribe`), or the entry-point name doesn't match the field name. Add a `System.out.println(name)` in `bind` to confirm `persons` is processed.
3. `setGlobal` throws but isn't caught — verify the catch block is present and that the runtime exception is `RuntimeException`-typed, not an `Error`.
4. `closeIsIdempotentViaTryWithResources` throws on second close — `ReteEvaluator.dispose` may not be idempotent. If so, guard with a `boolean closed` flag and short-circuit the second call. Update the test comment accordingly.

- [ ] **Step 5: Run the full test suite, verify nothing regressed**

```bash
mvn -pl drlx-parser-core -am test
```

Expected: all tests pass (152 from Tasks 1+2 + 4 new = 156).

- [ ] **Step 6: Install the module so downstream consumers (benchmark) resolve the new class**

```bash
mvn -pl drlx-parser-core -am install -DskipTests
```

Expected: BUILD SUCCESS. Required by the project Maven rule whenever production code changes in a multi-module project.

- [ ] **Step 7: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java \
        drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstanceTest.java
git commit -m "$(cat <<'EOF'
feat: add DrlxRuleUnitInstance wrapper for KieBase + RuleUnitData

Implements org.drools.ruleunits.api.RuleUnitInstance directly against a
DRLX-built KieBase, bypassing the upstream RuleUnitProvider codegen path
that DRLX cannot satisfy. Walks public DataSource fields on the unit and
binds each to a same-named entry point + global. Required precondition
for #37 tests whose consequences call DataStore.add / remove / update.

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

Expected: 156 tests pass, 0 failures, 0 errors. (146 baseline + 2 MyUnitTest + 4 TestDataObserverTest + 4 DrlxRuleUnitInstanceTest.)

- [ ] **Full module install (so downstream modules pick up the new wrapper)**

```bash
mvn -pl drlx-parser-core -am install
```

Expected: BUILD SUCCESS across the reactor.

- [ ] **Confirm spec → tasks alignment**

| Spec section            | Plan task |
|-------------------------|-----------|
| `DrlxRuleUnitInstance` component | Task 3 |
| `MyUnit` promotion       | Task 1 |
| `TestDataObserver`       | Task 2 |
| Bind protocol (public fields, replay on subscribe) | Task 3 step 3 + Task 2 replay test |
| Coupling cost (impl-package imports) | Task 3 step 3 — class Javadoc & imports |
| `MyUnitWithRawField` NOT promoted | Out-of-scope note above + Task 1 leaves it untouched |
| Validation test using existing rule pattern | Task 3 step 1 — `Person p : /persons[ age > 30 ]` |

If any spec requirement lacks a task, stop and add it.
