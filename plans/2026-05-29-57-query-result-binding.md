# Query Result Binding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Support `var t : /queryName(a, var b)` to bind the entire query result row to a variable, enabling named access (`t.a`), indexed access (`t[0]`), and method access (`t.objects()`).

**Architecture:** `QueryResultRow` extends `AbstractMap<String, Object>` so MVEL3's existing Map-property rewriter handles `t.a` → `t.get("a")` and `t[0]` → `t.get(0)` without any MVEL3 changes. A `QueryResultRowReader` (implementing `ReadAccessor`) wraps the result `Object[]` into a `QueryResultRow` at runtime. The runtime builder creates an additional Declaration for the bind-name when a query invocation has one.

**Tech Stack:** Java 17, Drools 10 runtime APIs, MVEL3, JUnit 5, AssertJ

**Spec:** `specs/2026-05-29-57-query-result-binding-design.md`

---

## File Structure

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRow.java` | Map-based wrapper over query result Object[] |
| Create | `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRowReader.java` | ReadAccessor that produces QueryResultRow from Object[] |
| Create | `drlx-parser-core/src/test/java/org/drools/drlx/runtime/QueryResultRowTest.java` | Unit tests for QueryResultRow |
| Modify | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:472-494` | Add result binding declaration when bindName is set |
| Modify | `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java` | End-to-end integration tests |

---

### Task 1: Create QueryResultRow with unit tests

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/runtime/QueryResultRowTest.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRow.java`

- [ ] **Step 1: Write failing unit tests**

```java
package org.drools.drlx.runtime;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class QueryResultRowTest {

    private static QueryResultRow createRow() {
        Object[] values = {"Alice", 30};
        Map<String, Integer> nameToIndex = new LinkedHashMap<>();
        nameToIndex.put("name", 0);
        nameToIndex.put("age", 1);
        return new QueryResultRow(values, nameToIndex);
    }

    @Test
    void namedAccess() {
        QueryResultRow row = createRow();
        assertThat(row.get("name")).isEqualTo("Alice");
        assertThat(row.get("age")).isEqualTo(30);
        assertThat(row.get("missing")).isNull();
    }

    @Test
    void indexedAccess() {
        QueryResultRow row = createRow();
        assertThat(row.get(0)).isEqualTo("Alice");
        assertThat(row.get(1)).isEqualTo(30);
    }

    @Test
    void objectsMethod() {
        QueryResultRow row = createRow();
        assertThat(row.objects()).containsExactly("Alice", 30);
    }

    @Test
    void handlesThrowsUnsupported() {
        QueryResultRow row = createRow();
        assertThatThrownBy(row::handles)
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void iterableOverValues() {
        QueryResultRow row = createRow();
        List<Object> collected = new ArrayList<>();
        for (Object v : row) {
            collected.add(v);
        }
        assertThat(collected).containsExactly("Alice", 30);
    }

    @Test
    void mapEntrySet() {
        QueryResultRow row = createRow();
        assertThat(row).containsEntry("name", "Alice").containsEntry("age", 30);
        assertThat(row).hasSize(2);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.runtime.QueryResultRowTest" -pl .`
Expected: Compilation failure — `QueryResultRow` class does not exist.

- [ ] **Step 3: Implement QueryResultRow**

```java
package org.drools.drlx.runtime;

import java.util.AbstractMap;
import java.util.Arrays;
import java.util.Iterator;
import java.util.LinkedHashSet;
import java.util.Map;
import java.util.Set;

public final class QueryResultRow extends AbstractMap<String, Object> implements Iterable<Object> {

    private final Object[] values;
    private final Map<String, Integer> nameToIndex;

    public QueryResultRow(Object[] values, Map<String, Integer> nameToIndex) {
        this.values = values;
        this.nameToIndex = nameToIndex;
    }

    @Override
    public Object get(Object key) {
        if (key instanceof String s) {
            Integer idx = nameToIndex.get(s);
            return idx != null ? values[idx] : null;
        } else if (key instanceof Integer i) {
            return values[i];
        }
        return null;
    }

    public Object[] objects() {
        return values;
    }

    public Object handles() {
        throw new UnsupportedOperationException(
                "Handle access is not yet supported — see issue #82");
    }

    @Override
    public int size() {
        return nameToIndex.size();
    }

    @Override
    public Set<Entry<String, Object>> entrySet() {
        Set<Entry<String, Object>> entries = new LinkedHashSet<>();
        for (Map.Entry<String, Integer> e : nameToIndex.entrySet()) {
            entries.add(new SimpleImmutableEntry<>(e.getKey(), values[e.getValue()]));
        }
        return entries;
    }

    @Override
    public Iterator<Object> iterator() {
        return Arrays.asList(values).iterator();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.runtime.QueryResultRowTest" -pl .`
Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRow.java \
  drlx-parser-core/src/test/java/org/drools/drlx/runtime/QueryResultRowTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#57): add QueryResultRow — Map-based wrapper for query result tuples

Refs #57"
```

---

### Task 2: Create QueryResultRowReader

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRowReader.java`

- [ ] **Step 1: Implement QueryResultRowReader**

This class follows the same `ReadAccessor` pattern as `DroolsQueryElementsReader` (in `org.drools.drlx.builder`). It receives the `Object[]` fact from the result pattern and wraps it in a `QueryResultRow`.

```java
package org.drools.drlx.runtime;

import java.lang.reflect.Method;
import java.util.Map;

import org.drools.base.base.ValueResolver;
import org.drools.base.base.ValueType;
import org.drools.base.rule.accessor.ReadAccessor;

public final class QueryResultRowReader implements ReadAccessor {

    private final Map<String, Integer> nameToIndex;

    public QueryResultRowReader(Map<String, Integer> nameToIndex) {
        this.nameToIndex = nameToIndex;
    }

    @Override
    public Object getValue(Object object) {
        return new QueryResultRow((Object[]) object, nameToIndex);
    }

    @Override
    public Object getValue(ValueResolver valueResolver, Object object) {
        return new QueryResultRow((Object[]) object, nameToIndex);
    }

    @Override public int getIndex() { return -1; }
    @Override public Class<?> getExtractToClass() { return QueryResultRow.class; }
    @Override public String getExtractToClassName() { return QueryResultRow.class.getName(); }
    @Override public ValueType getValueType() { return ValueType.OBJECT_TYPE; }
    @Override public boolean isSelfReference() { return false; }
    @Override public boolean isGlobal() { return false; }
    @Override public Method getNativeReadMethod() { return null; }
    @Override public String getNativeReadMethodName() { return "getValue"; }
    @Override public boolean isNullValue(ValueResolver vr, Object o) { return getValue(vr, o) == null; }

    @Override public int getHashCode(Object o) {
        Object v = getValue(o);
        return v != null ? v.hashCode() : 0;
    }

    @Override public int getHashCode(ValueResolver vr, Object o) {
        Object v = getValue(vr, o);
        return v != null ? v.hashCode() : 0;
    }

    @Override public boolean getBooleanValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public byte getByteValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public char getCharValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public double getDoubleValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public float getFloatValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public int getIntValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public long getLongValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public short getShortValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
}
```

- [ ] **Step 2: Compile to verify it builds**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -pl .`
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRowReader.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#57): add QueryResultRowReader — ReadAccessor producing QueryResultRow

Refs #57"
```

---

### Task 3: Integrate result binding into runtime builder

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:472-494`

- [ ] **Step 1: Write failing end-to-end test for named access**

Add to `QueryTest.java`:

```java
@Test
void queryResultBindingNamedAccess() {
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
                do { results.add(t.result); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));
    unit.persons.add(new Person("Bob", 20));
    unit.persons.add(new Person("Charlie", 40));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        List<String> names = unit.results.stream()
                .map(o -> ((Person) o).getName())
                .toList();
        assertThat(names).containsExactlyInAnyOrder("Alice", "Charlie");
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.QueryTest#queryResultBindingNamedAccess" -pl .`
Expected: FAIL — `t` is not bound as a variable (bindName is ignored on query element path).

- [ ] **Step 3: Modify DrlxRuleAstRuntimeBuilder to add result binding**

In `DrlxRuleAstRuntimeBuilder.java`, after the output variable binding loop (after line 494, before the `continue;` statement), add the result binding block:

```java
                    // Bind the entire query result row if a bind name is present
                    if (patternIr.bindName() != null) {
                        Map<String, Integer> nameToIndex = new LinkedHashMap<>();
                        Declaration[] qParams = targetQuery.getParameters();
                        for (int i = 0; i < qParams.length; i++) {
                            nameToIndex.put(qParams[i].getIdentifier(), i);
                        }
                        QueryResultRowReader rowReader = new QueryResultRowReader(nameToIndex);
                        Pattern resultPattern = queryElement.getResultPattern();
                        Declaration rowDecl = new Declaration(patternIr.bindName(), rowReader, resultPattern);
                        resultPattern.addDeclaration(rowDecl);
                        boundVariables.put(patternIr.bindName(),
                                new BoundVariable(patternIr.bindName(), QueryResultRow.class, resultPattern, rowDecl));
                    }
```

Add these imports at the top of the file:

```java
import org.drools.drlx.runtime.QueryResultRow;
import org.drools.drlx.runtime.QueryResultRowReader;
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.QueryTest#queryResultBindingNamedAccess" -pl .`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#57): wire query result binding into runtime builder

Refs #57"
```

---

### Task 4: Add indexed access and equivalence tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write indexed access test**

```java
@Test
void queryResultBindingIndexedAccess() {
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
                do { results.add(t[1]); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));
    unit.persons.add(new Person("Bob", 20));
    unit.persons.add(new Person("Charlie", 40));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        List<String> names = unit.results.stream()
                .map(o -> ((Person) o).getName())
                .toList();
        assertThat(names).containsExactlyInAnyOrder("Alice", "Charlie");
    }
}
```

- [ ] **Step 2: Write equivalence eval test (named == indexed)**

```java
@Test
void queryResultBindingNamedEqualsIndexed() {
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
                eval(t.result == t[1]),
                do { results.add(p); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));
    unit.persons.add(new Person("Bob", 20));
    unit.persons.add(new Person("Charlie", 40));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        List<String> names = unit.results.stream()
                .map(o -> ((Person) o).getName())
                .toList();
        // eval(t.result == t[1]) must pass — same Person object
        assertThat(names).containsExactlyInAnyOrder("Alice", "Charlie");
    }
}
```

- [ ] **Step 3: Run both new tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.QueryTest#queryResultBindingIndexedAccess+queryResultBindingNamedEqualsIndexed" -pl .`
Expected: Both PASS.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#57): add indexed access and named==indexed equivalence tests

Refs #57"
```

---

### Task 5: Add objects() and iteration tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write objects() method test**

```java
@Test
void queryResultBindingObjectsMethod() {
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
                do { results.add(t.objects().length); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        // PersonsByAge has 2 parameters (minAge, result) → objects() length is 2
        assertThat(unit.results).containsExactly(2);
    }
}
```

- [ ] **Step 2: Write individual output variable still works alongside result binding**

```java
@Test
void queryResultBindingCoexistsWithOutputVariables() {
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
                do { results.add(p.name + ":" + ((org.drools.drlx.domain.Person)t.result).name); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        // p and t.result should refer to the same Person
        assertThat(unit.results).containsExactly("Alice:Alice");
    }
}
```

- [ ] **Step 3: Run both new tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.QueryTest#queryResultBindingObjectsMethod+queryResultBindingCoexistsWithOutputVariables" -pl .`
Expected: Both PASS.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#57): add objects() and coexistence tests

Refs #57"
```

---

### Task 6: Full test suite run

- [ ] **Step 1: Run the full drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`
Expected: All tests PASS, including existing query tests (no regressions).

- [ ] **Step 2: If any failures, investigate and fix**

Check that existing query tests still pass — especially `queryInvocationFromRule`, `queryMultipleParameters`, `passiveQueryInvocationDoesNotWakeRule`, `passiveQueryInvocationWakesWhenReactiveSideFires`, and `recursiveQueryTransitiveClosure`.

- [ ] **Step 3: Final commit (if fixes were needed)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add -u
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "fix(#57): address test regressions from result binding

Refs #57"
```
