# #82 Query Result Handle Access Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable `t.handles()[0]` (indexed) and `t.handles().a` (named) FactHandle access on query result bindings.

**Architecture:** `QueryResultRow.handles()` returns a new `QueryResultHandleRow` (an `AbstractMap<String, Object>` sibling) that lazily resolves FactHandles by searching all entry points via `ReteEvaluator`. The `ValueResolver` is threaded from `QueryResultRowReader` through to `QueryResultRow`. Zero Drools changes.

**Tech Stack:** Java 17+, drools-core `ReteEvaluator` / `InternalFactHandle`, JUnit 5, AssertJ

---

### File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultHandleRow.java` | Create | `AbstractMap` wrapper that resolves FactHandles lazily via `ReteEvaluator` entry point search |
| `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRow.java` | Modify | Add `valueResolver` field, implement `handles()` method |
| `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRowReader.java` | Modify | Pass `valueResolver` to `QueryResultRow` constructor |
| `drlx-parser-core/src/test/java/org/drools/drlx/runtime/QueryResultRowTest.java` | Modify | Update `createRow()` for new constructor, replace `handlesThrowsUnsupported` with null-valueResolver test |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java` | Modify | Add E2E tests for `t.handles()[0]` and `t.handles().a` |

---

### Task 1: Create `QueryResultHandleRow`

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultHandleRow.java`

- [ ] **Step 1: Create the class**

```java
package org.drools.drlx.runtime;

import java.util.AbstractMap;
import java.util.Collection;
import java.util.LinkedHashSet;
import java.util.Map;
import java.util.Set;

import org.drools.core.common.InternalFactHandle;
import org.drools.core.common.ReteEvaluator;
import org.kie.api.runtime.rule.EntryPoint;

public final class QueryResultHandleRow extends AbstractMap<String, Object> {

    private final Object[] values;
    private final Map<String, Integer> nameToIndex;
    private final ReteEvaluator reteEvaluator;

    QueryResultHandleRow(Object[] values, Map<String, Integer> nameToIndex, ReteEvaluator reteEvaluator) {
        this.values = values;
        this.nameToIndex = nameToIndex;
        this.reteEvaluator = reteEvaluator;
    }

    @Override
    public Object get(Object key) {
        Object value;
        if (key instanceof String s) {
            Integer idx = nameToIndex.get(s);
            value = idx != null ? values[idx] : null;
        } else if (key instanceof Integer i) {
            value = (i >= 0 && i < values.length) ? values[i] : null;
        } else {
            return null;
        }
        return value != null ? findFactHandle(value) : null;
    }

    // Searches all entry points because objects may live in named entry points
    // (e.g. "persons", "trusts"), not just the default. Could be improved in the
    // future by threading the entry point name through to avoid linear search.
    private InternalFactHandle findFactHandle(Object object) {
        for (EntryPoint ep : reteEvaluator.getEntryPoints()) {
            InternalFactHandle fh = (InternalFactHandle) ep.getFactHandle(object);
            if (fh != null) {
                return fh;
            }
        }
        return null;
    }

    @Override
    public int size() {
        return nameToIndex.size();
    }

    @Override
    public Set<Entry<String, Object>> entrySet() {
        Set<Entry<String, Object>> entries = new LinkedHashSet<>();
        for (Map.Entry<String, Integer> e : nameToIndex.entrySet()) {
            Object value = values[e.getValue()];
            entries.add(new SimpleImmutableEntry<>(e.getKey(), value != null ? findFactHandle(value) : null));
        }
        return entries;
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultHandleRow.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#82): add QueryResultHandleRow — lazy FactHandle resolution via entry point search"
```

---

### Task 2: Modify `QueryResultRow` to support `handles()`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRow.java`

- [ ] **Step 1: Add `valueResolver` field and update constructor**

Add a `ValueResolver` field. The constructor accepts it as a third parameter. The existing `handles()` method uses it instead of throwing.

Change the constructor from:
```java
public QueryResultRow(Object[] values, Map<String, Integer> nameToIndex) {
    this.values = values;
    this.nameToIndex = nameToIndex;
}
```
to:
```java
private final ValueResolver valueResolver;

public QueryResultRow(Object[] values, Map<String, Integer> nameToIndex, ValueResolver valueResolver) {
    this.values = values;
    this.nameToIndex = nameToIndex;
    this.valueResolver = valueResolver;
}
```

Add import: `import org.drools.base.base.ValueResolver;`

- [ ] **Step 2: Implement `handles()` method**

Replace the existing `handles()` method:
```java
public Object handles() {
    throw new UnsupportedOperationException(
            "Handle access is not yet supported — see issue #82");
}
```
with:
```java
public QueryResultHandleRow handles() {
    if (valueResolver == null) {
        throw new UnsupportedOperationException(
                "Handle access requires a ValueResolver — not available in this context");
    }
    return new QueryResultHandleRow(values, nameToIndex, (ReteEvaluator) valueResolver);
}
```

Add import: `import org.drools.core.common.ReteEvaluator;`

- [ ] **Step 3: Verify it compiles**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: Compilation failure — `QueryResultRowTest.createRow()` uses the old 2-arg constructor. Fix in next task.

---

### Task 3: Fix unit tests for new constructor

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/runtime/QueryResultRowTest.java`

- [ ] **Step 1: Update `createRow()` to use 3-arg constructor**

Change:
```java
private static QueryResultRow createRow() {
    Object[] values = {"Alice", 30};
    Map<String, Integer> nameToIndex = new LinkedHashMap<>();
    nameToIndex.put("name", 0);
    nameToIndex.put("age", 1);
    return new QueryResultRow(values, nameToIndex);
}
```
to:
```java
private static QueryResultRow createRow() {
    Object[] values = {"Alice", 30};
    Map<String, Integer> nameToIndex = new LinkedHashMap<>();
    nameToIndex.put("name", 0);
    nameToIndex.put("age", 1);
    return new QueryResultRow(values, nameToIndex, null);
}
```

- [ ] **Step 2: Update `handlesThrowsUnsupported` test**

Replace:
```java
@Test
void handlesThrowsUnsupported() {
    QueryResultRow row = createRow();
    assertThatThrownBy(row::handles)
            .isInstanceOf(UnsupportedOperationException.class);
}
```
with:
```java
@Test
void handlesThrowsWhenNoValueResolver() {
    QueryResultRow row = createRow();
    assertThatThrownBy(row::handles)
            .isInstanceOf(UnsupportedOperationException.class)
            .hasMessageContaining("ValueResolver");
}
```

- [ ] **Step 3: Run unit tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest="QueryResultRowTest" -q`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRow.java drlx-parser-core/src/test/java/org/drools/drlx/runtime/QueryResultRowTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#82): add handles() to QueryResultRow with ValueResolver support"
```

---

### Task 4: Thread `ValueResolver` through `QueryResultRowReader`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRowReader.java`

- [ ] **Step 1: Update `getValue` methods**

Change `getValue(Object)`:
```java
@Override
public Object getValue(Object object) {
    return new QueryResultRow((Object[]) object, nameToIndex);
}
```
to:
```java
@Override
public Object getValue(Object object) {
    return new QueryResultRow((Object[]) object, nameToIndex, null);
}
```

Change `getValue(ValueResolver, Object)`:
```java
@Override
public Object getValue(ValueResolver valueResolver, Object object) {
    return new QueryResultRow((Object[]) object, nameToIndex);
}
```
to:
```java
@Override
public Object getValue(ValueResolver valueResolver, Object object) {
    return new QueryResultRow((Object[]) object, nameToIndex, valueResolver);
}
```

- [ ] **Step 2: Run all existing query tests to verify no regressions**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest="QueryTest,QueryResultRowTest" -q`
Expected: All tests pass

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRowReader.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#82): thread ValueResolver through QueryResultRowReader to QueryResultRow"
```

---

### Task 5: E2E test — indexed handle access `t.handles()[0]`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write the failing test**

Add to `QueryTest.java`:

```java
@Test
void queryResultBindingHandleIndexedAccess() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                var t : /personsByAge(25, var p),
                do { results.add(t.handles()[1].getObject()); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        // t.handles()[1] is the FactHandle for the result (index 1 = Person parameter)
        // .getObject() should return the same Person object
        assertThat(unit.results).hasSize(1);
        assertThat(unit.results.get(0)).isInstanceOf(Person.class);
        assertThat(((Person) unit.results.get(0)).getName()).isEqualTo("Alice");
    }
}
```

- [ ] **Step 2: Run the test — expect it to fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest="QueryTest#queryResultBindingHandleIndexedAccess" -q`
Expected: FAIL — if `handles()` return type is not resolvable by MVEL3, this may fail at compile time. Diagnose and fix.

- [ ] **Step 3: Fix any MVEL3 type resolution issues**

If MVEL3 can't resolve `QueryResultHandleRow` as the return type of `handles()`, the problem is that `QueryResultHandleRow` needs to be on the MVEL3 compilation classpath. Since `QueryResultRow` is already on the classpath and `handles()` returns `QueryResultHandleRow` (which lives in the same package), this should resolve automatically. If not, check whether `QueryResultHandleRow` needs to be registered in the type resolver.

- [ ] **Step 4: Run the test — expect it to pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest="QueryTest#queryResultBindingHandleIndexedAccess" -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#82): add indexed handle access test — t.handles()[1].getObject()"
```

---

### Task 6: E2E test — named handle access `t.handles().a`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write the test**

Add to `QueryTest.java`:

```java
@Test
void queryResultBindingHandleNamedAccess() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                var t : /personsByAge(25, var p),
                do { results.add(t.handles().result.getObject()); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        // t.handles().result is the FactHandle for the "result" parameter
        // .getObject() returns the same Person
        assertThat(unit.results).hasSize(1);
        assertThat(unit.results.get(0)).isInstanceOf(Person.class);
        assertThat(((Person) unit.results.get(0)).getName()).isEqualTo("Alice");
    }
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest="QueryTest#queryResultBindingHandleNamedAccess" -q`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#82): add named handle access test — t.handles().result.getObject()"
```

---

### Task 7: E2E test — handle identity (indexed == named)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write the test**

Add to `QueryTest.java`:

```java
@Test
void queryResultBindingHandleIdentity() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                var t : /personsByAge(25, var p),
                do {
                    results.add(t.handles()[1].getObject() == t.result);
                    results.add(t.handles().result.getObject() == t.result);
                }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        // Both indexed and named handle access should return the same object as t.result
        assertThat(unit.results).allMatch(o -> Boolean.TRUE.equals(o));
        assertThat(unit.results).hasSize(2);
    }
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest="QueryTest#queryResultBindingHandleIdentity" -q`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#82): add handle identity test — indexed and named handle match object"
```

---

### Task 8: Run full test suite

- [ ] **Step 1: Install the modified module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -q`
Expected: BUILD SUCCESS

- [ ] **Step 2: Run all drlx-parser tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -q`
Expected: All tests pass, no regressions

- [ ] **Step 3: Final commit if any fixes needed**

If any test failures were found and fixed in previous steps, commit the fixes referencing #82.
