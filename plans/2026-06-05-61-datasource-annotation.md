# @DataSource annotation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Support `@DataSource("name")` annotation on DRLX query rules to override the implicit entry point name.

**Architecture:** Extends the existing annotation infrastructure (Salience/Description pattern). New `DataSource.java` annotation class, `DATASOURCE` enum in IR model, visitor validation, and runtime builder entry point name override. No grammar changes needed.

**Tech Stack:** Java 17, ANTLR4, drools-core, JUnit 5, AssertJ

**Spec:** `specs/2026-06-05-61-datasource-annotation-design.md`

**Project repo:** `/home/tkobayas/usr/work/mvel3-development/drlx-parser`

---

### Task 1: Create DataSource annotation class and extend IR model

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/DataSource.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:31`

- [ ] **Step 1: Create `DataSource.java` annotation**

```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DataSource {
    String value();
}
```

- [ ] **Step 2: Add `DATASOURCE` to `RuleAnnotationIR.Kind` enum**

In `DrlxRuleAstModel.java` line 31, change:
```java
public enum Kind { SALIENCE, DESCRIPTION }
```
to:
```java
public enum Kind { SALIENCE, DESCRIPTION, DATASOURCE }
```

- [ ] **Step 3: Compile to verify**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/annotations/DataSource.java \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "Add DataSource annotation class and DATASOURCE IR kind

Refs #61"
```

---

### Task 2: Extend visitor annotation resolution for @DataSource

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:37-42, 253-255, 266-290`

- [ ] **Step 1: Write failing test — @DataSource with import resolves**

In `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`, add:

```java
@Test
void testDataSourceEmptyStringFailsLoud() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Trust;
            import org.drools.drlx.annotations.DataSource;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @DataSource("")
            rule Trusts(Object a, Object b) {
                Trust t : /trusts[a == a, b == b],
            }
            """;

    final DrlxRuleBuilder builder = newBuilder();

    assertThatThrownBy(() -> builder.build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Datasource expects non-empty string literal");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=RuleAnnotationsTest#testDataSourceEmptyStringFailsLoud -q`
Expected: FAIL — `@DataSource` is not yet recognized as a supported annotation.

- [ ] **Step 3: Add `DATASOURCE_FQN` and extend `SUPPORTED_ANNOTATION_KINDS`**

In `DrlxToRuleAstVisitor.java`, after line 38:
```java
private static final String DATASOURCE_FQN = "org.drools.drlx.annotations.DataSource";
```

Change the `SUPPORTED_ANNOTATION_KINDS` map (line 40-42) to:
```java
private static final Map<String, Kind> SUPPORTED_ANNOTATION_KINDS = Map.of(
        SALIENCE_FQN, Kind.SALIENCE,
        DESCRIPTION_FQN, Kind.DESCRIPTION,
        DATASOURCE_FQN, Kind.DATASOURCE);
```

- [ ] **Step 4: Update error message in `resolveKind()`**

In `resolveKind()` (line ~253-255), update the error message:
```java
throw new RuntimeException(
        "unsupported DRLX rule annotation '@" + nameText + "' at "
        + line + ":" + col + " — supported: @Salience, @Description, @DataSource");
```

- [ ] **Step 5: Add `case DATASOURCE` in `extractAnnotationLiteral()`**

In `extractAnnotationLiteral()` (line ~266-290), add before the `default` case:
```java
case DATASOURCE -> {
    if (text.length() >= 2 && text.startsWith("\"") && text.endsWith("\"")) {
        String name = text.substring(1, text.length() - 1);
        if (name.isEmpty()) {
            throw new RuntimeException(
                    "@Datasource expects non-empty string literal at " + line + ":" + col);
        }
        return name;
    }
    throw new RuntimeException(
            "@Datasource expects string literal, got '" + text + "' at " + line + ":" + col);
}
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=RuleAnnotationsTest#testDataSourceEmptyStringFailsLoud -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "Extend visitor to resolve and validate @DataSource annotation

Refs #61"
```

---

### Task 3: Override entry point name in runtime builder

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:115-132, 1447-1454`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`

- [ ] **Step 1: Write failing test — @DataSource overrides entry point**

Add a `trustworthy` field to `MyUnit.java`:
```java
public DataStore<Trust> trustworthy = DataSource.createStore();
```

In `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`, add:

```java
@Test
void dataSourceAnnotationOverridesEntryPoint() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Trust;
            import org.drools.drlx.annotations.DataSource;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @DataSource("trustworthy")
            rule Trusts(Object a, Object b) {
                Trust t : /trusts[a == a, b == b],
            }

            rule R1 {
                /trustworthy("Alice", var b),
                do { results.add(b); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.trusts.add(new Trust("Alice", "Bob"));
    unit.trusts.add(new Trust("Alice", "Charlie"));
    unit.trusts.add(new Trust("Dave", "Eve"));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        assertThat(unit.results).containsExactlyInAnyOrder("Bob", "Charlie");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=QueryTest#dataSourceAnnotationOverridesEntryPoint -q`
Expected: FAIL — entry point name is still `trusts`, not `trustworthy`.

- [ ] **Step 3: Implement entry point override in the query-building loop**

In `DrlxRuleAstRuntimeBuilder.java`, replace the query loop (lines ~117-125):

```java
for (RuleIR rule : parseResult.rules()) {
    if (!rule.parameters().isEmpty()) {
        String entryPointName = rule.annotations().stream()
                .filter(a -> a.kind() == Kind.DATASOURCE)
                .map(RuleAnnotationIR::rawValue)
                .findFirst()
                .orElseGet(() -> Character.toLowerCase(rule.name().charAt(0)) + rule.name().substring(1));
        QueryImpl query = new QueryImpl(rule.name());
        queryRegistry.put(entryPointName, query);
        buildQuery(query, rule, pkg.getTypeResolver(), entryPointTypes, unitClass, queryRegistry);
        pkg.addRule(query);
    }
}
```

- [ ] **Step 4: Add @DataSource validation for non-query rules**

In the non-query rule loop (lines ~127-132), add a check before `buildRule`:

```java
for (RuleIR rule : parseResult.rules()) {
    if (rule.parameters().isEmpty()) {
        if (rule.annotations().stream().anyMatch(a -> a.kind() == Kind.DATASOURCE)) {
            throw new RuntimeException(
                    "@DataSource is only allowed on query rules (rules with parameters)"
                    + " — rule '" + rule.name() + "' has no parameters");
        }
        pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass,
                             globalTypes, dataStoreGlobalNames, updateRewriter, queryRegistry));
    }
}
```

- [ ] **Step 5: Add `DATASOURCE` no-op case in `applyAnnotations()`**

In `applyAnnotations()` (line ~1447-1454), add the case:

```java
private static void applyAnnotations(RuleImpl rule, List<RuleAnnotationIR> annotations) {
    for (RuleAnnotationIR ann : annotations) {
        switch (ann.kind()) {
            case SALIENCE -> rule.setSalience(new SalienceInteger(Integer.parseInt(ann.rawValue())));
            case DESCRIPTION -> rule.addMetaAttribute("Description", ann.rawValue());
            case DATASOURCE -> { }
        }
    }
}
```

- [ ] **Step 6: Add missing import**

Add to imports in `DrlxRuleAstRuntimeBuilder.java`:
```java
import org.drools.drlx.builder.DrlxRuleAstModel.RuleAnnotationIR;
import org.drools.drlx.builder.DrlxRuleAstModel.RuleAnnotationIR.Kind;
```

(Check whether these imports already exist; only add if missing.)

- [ ] **Step 7: Run the test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=QueryTest#dataSourceAnnotationOverridesEntryPoint -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java \
  drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "Use @DataSource value as query entry point name

Refs #61"
```

---

### Task 4: Add remaining error-case and combination tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Add test — @DataSource on non-query rule errors**

In `RuleAnnotationsTest.java`, add:

```java
@Test
void testDataSourceOnNonQueryRuleFailsLoud() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.DataSource;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @DataSource("people")
            rule NotAQuery {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    final DrlxRuleBuilder builder = newBuilder();

    assertThatThrownBy(() -> builder.build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@DataSource is only allowed on query rules");
}
```

- [ ] **Step 2: Add test — @DataSource without argument errors**

In `RuleAnnotationsTest.java`, add:

```java
@Test
void testDataSourceWithoutArgumentFailsLoud() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Trust;
            import org.drools.drlx.annotations.DataSource;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @DataSource
            rule Trusts(Object a, Object b) {
                Trust t : /trusts[a == a, b == b],
            }
            """;

    final DrlxRuleBuilder builder = newBuilder();

    assertThatThrownBy(() -> builder.build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Datasource expects one argument");
}
```

- [ ] **Step 3: Add test — @DataSource with non-string argument errors**

In `RuleAnnotationsTest.java`, add:

```java
@Test
void testDataSourceNonStringArgumentFailsLoud() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Trust;
            import org.drools.drlx.annotations.DataSource;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @DataSource(42)
            rule Trusts(Object a, Object b) {
                Trust t : /trusts[a == a, b == b],
            }
            """;

    final DrlxRuleBuilder builder = newBuilder();

    assertThatThrownBy(() -> builder.build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Datasource expects string literal");
}
```

- [ ] **Step 4: Add test — FQN @DataSource resolves without import**

In `QueryTest.java`, add:

```java
@Test
void dataSourceAnnotationFqnWithoutImport() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Trust;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @org.drools.drlx.annotations.DataSource("trustworthy")
            rule Trusts(Object a, Object b) {
                Trust t : /trusts[a == a, b == b],
            }

            rule R1 {
                /trustworthy("Alice", var b),
                do { results.add(b); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.trusts.add(new Trust("Alice", "Bob"));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        assertThat(unit.results).containsExactlyInAnyOrder("Bob");
    }
}
```

- [ ] **Step 5: Add test — @DataSource combined with @Salience on a query**

In `QueryTest.java`, add:

```java
@Test
void dataSourceAnnotationCombinedWithSalience() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Trust;
            import org.drools.drlx.annotations.DataSource;
            import org.drools.drlx.annotations.Salience;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Salience(10)
            @DataSource("trustworthy")
            rule Trusts(Object a, Object b) {
                Trust t : /trusts[a == a, b == b],
            }

            rule R1 {
                /trustworthy("Alice", var b),
                do { results.add(b); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.trusts.add(new Trust("Alice", "Bob"));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        assertThat(unit.results).containsExactlyInAnyOrder("Bob");
    }
}
```

- [ ] **Step 6: Run all new tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest="RuleAnnotationsTest,QueryTest" -q`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "Add error-case and combination tests for @DataSource

Refs #61"
```

---

### Task 5: Full test suite and install

- [ ] **Step 1: Run the full drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -q`
Expected: All tests PASS, no regressions.

- [ ] **Step 2: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -q -DskipTests`
Expected: BUILD SUCCESS
