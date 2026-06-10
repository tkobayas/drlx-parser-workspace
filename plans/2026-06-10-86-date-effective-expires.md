# @DateEffective/@DateExpires Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `@DateEffective("yyyy-MM-dd")` and `@DateExpires("yyyy-MM-dd")` annotations so rules only fire within calendar-based activation windows.

**Architecture:** Follows the existing annotation pipeline — annotation class → `Kind` enum entry → visitor FQN mapping → `applyAnnotations()` switch case → `RuleImpl` setter. Date strings are parsed via `LocalDate.parse()` (ISO-8601) and converted to `Calendar` for the Drools API.

**Tech Stack:** Java 17, Drools `RuleImpl.setDateEffective()`/`setDateExpires()`, `java.time.LocalDate`

---

### Task 1: Create annotation classes and IR model entries

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/DateEffective.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/DateExpires.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:40`

- [ ] **Step 1: Create DateEffective annotation**

Create `drlx-parser-core/src/main/java/org/drools/drlx/annotations/DateEffective.java`:

```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DateEffective {
    String value();
}
```

- [ ] **Step 2: Create DateExpires annotation**

Create `drlx-parser-core/src/main/java/org/drools/drlx/annotations/DateExpires.java`:

```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DateExpires {
    String value();
}
```

- [ ] **Step 3: Add Kind enum entries**

In `DrlxRuleAstModel.java`, add two new entries to the `Kind` enum after `DURATION`:

```java
DURATION(ArgShape.STRING),
DATE_EFFECTIVE(ArgShape.STRING),
DATE_EXPIRES(ArgShape.STRING);
```

Change the semicolon after `DURATION(ArgShape.STRING)` to a comma, then add the two new entries.

- [ ] **Step 4: Compile to verify**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`

Expected: BUILD SUCCESS

---

### Task 2: Wire visitor to recognize the new annotations

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:37-56,277-281`

- [ ] **Step 1: Add FQN constants**

In `DrlxToRuleAstVisitor.java`, after the `DURATION_FQN` constant (line 45), add:

```java
    private static final String DATE_EFFECTIVE_FQN = "org.drools.drlx.annotations.DateEffective";
    private static final String DATE_EXPIRES_FQN = "org.drools.drlx.annotations.DateExpires";
```

- [ ] **Step 2: Add map entries**

In the `SUPPORTED_ANNOTATION_KINDS` map (after the `DURATION` entry at line 56), add:

```java
            Map.entry(DATE_EFFECTIVE_FQN, Kind.DATE_EFFECTIVE),
            Map.entry(DATE_EXPIRES_FQN, Kind.DATE_EXPIRES));
```

Change the closing `);` after the `DURATION` entry to a comma `,` before adding these.

- [ ] **Step 3: Update error message in resolveKind()**

In `resolveKind()` (line 279), update the error message to include the new annotations:

```java
            throw new RuntimeException(
                    "unsupported DRLX rule annotation '@" + nameText + "' at "
                    + line + ":" + col + " — supported: @Salience, @Description, @DataSource, "
                    + "@NoLoop, @LockOnActive, @Disabled, "
                    + "@ActivationGroup, @Timer, @Duration, "
                    + "@DateEffective, @DateExpires");
```

- [ ] **Step 4: Compile to verify**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`

Expected: BUILD SUCCESS

---

### Task 3: Write parse-level and error tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

- [ ] **Step 1: Write parse test for @DateEffective**

Add to `RuleAnnotationsTest.java`:

```java
    @Test
    void testDateEffectiveParsed() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("2020-01-01")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        final KieBase kieBase = newBuilder().build(rule);
        Rule r = kieBase.getRule("org.drools.drlx.parser", "R1");
        assertThat(r).isNotNull();
        RuleImpl impl = (RuleImpl) r;
        assertThat(impl.getDateEffective()).isNotNull();
        assertThat(impl.getDateEffective().get(Calendar.YEAR)).isEqualTo(2020);
        assertThat(impl.getDateEffective().get(Calendar.MONTH)).isEqualTo(Calendar.JANUARY);
        assertThat(impl.getDateEffective().get(Calendar.DAY_OF_MONTH)).isEqualTo(1);
    }
```

- [ ] **Step 2: Write parse test for @DateExpires**

Add to `RuleAnnotationsTest.java`:

```java
    @Test
    void testDateExpiresParsed() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateExpires;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateExpires("2099-12-31")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        final KieBase kieBase = newBuilder().build(rule);
        Rule r = kieBase.getRule("org.drools.drlx.parser", "R1");
        assertThat(r).isNotNull();
        RuleImpl impl = (RuleImpl) r;
        assertThat(impl.getDateExpires()).isNotNull();
        assertThat(impl.getDateExpires().get(Calendar.YEAR)).isEqualTo(2099);
        assertThat(impl.getDateExpires().get(Calendar.MONTH)).isEqualTo(Calendar.DECEMBER);
        assertThat(impl.getDateExpires().get(Calendar.DAY_OF_MONTH)).isEqualTo(31);
    }
```

- [ ] **Step 3: Write parse test for both annotations together**

Add to `RuleAnnotationsTest.java`:

```java
    @Test
    void testDateEffectiveAndExpiresTogether() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;
                import org.drools.drlx.annotations.DateExpires;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("2020-01-01")
                @DateExpires("2099-12-31")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        final KieBase kieBase = newBuilder().build(rule);
        Rule r = kieBase.getRule("org.drools.drlx.parser", "R1");
        assertThat(r).isNotNull();
        RuleImpl impl = (RuleImpl) r;
        assertThat(impl.getDateEffective()).isNotNull();
        assertThat(impl.getDateExpires()).isNotNull();
    }
```

- [ ] **Step 4: Write FQN test (no import)**

Add to `RuleAnnotationsTest.java`:

```java
    @Test
    void testDateEffectiveFqnWithoutImport() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @org.drools.drlx.annotations.DateEffective("2020-06-15")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        final KieBase kieBase = newBuilder().build(rule);
        Rule r = kieBase.getRule("org.drools.drlx.parser", "R1");
        assertThat(r).isNotNull();
        RuleImpl impl = (RuleImpl) r;
        assertThat(impl.getDateEffective()).isNotNull();
        assertThat(impl.getDateEffective().get(Calendar.YEAR)).isEqualTo(2020);
    }
```

- [ ] **Step 5: Add Calendar import to test file**

Add at the top of `RuleAnnotationsTest.java`:

```java
import java.util.Calendar;
```

- [ ] **Step 6: Write error tests**

Add to `RuleAnnotationsTest.java`:

```java
    @Test
    void testDateEffectiveWithoutArgumentFails() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> newBuilder().build(rule))
                .isInstanceOf(RuntimeException.class);
    }

    @Test
    void testDateEffectiveEmptyStringFails() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> newBuilder().build(rule))
                .isInstanceOf(RuntimeException.class);
    }

    @Test
    void testDateEffectiveMalformedDateFails() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("not-a-date")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> newBuilder().build(rule))
                .isInstanceOf(RuntimeException.class);
    }

    @Test
    void testDateEffectiveLegacyFormatFails() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("01-Jan-2025")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> newBuilder().build(rule))
                .isInstanceOf(RuntimeException.class);
    }

    @Test
    void testDuplicateDateEffectiveFails() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("2025-01-01")
                @DateEffective("2025-06-01")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> newBuilder().build(rule))
                .isInstanceOf(RuntimeException.class);
    }
```

- [ ] **Step 7: Run error tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="RuleAnnotationsTest#testDateEffectiveWithoutArgumentFails,RuleAnnotationsTest#testDateEffectiveEmptyStringFails,RuleAnnotationsTest#testDuplicateDateEffectiveFails" -pl . -q`

Expected: ALL PASS — these are caught by the existing `ArgShape.STRING` validation in the visitor, no runtime builder changes needed.

---

### Task 4: Wire runtime builder and make parse tests pass

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:1499-1503`

- [ ] **Step 1: Add imports**

At the top of `DrlxRuleAstRuntimeBuilder.java`, add these imports (after the existing `java.util.*` block around line 14):

```java
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.Calendar;
import java.util.GregorianCalendar;
```

Note: check if `Calendar` and `GregorianCalendar` are already imported — if so, skip those.

- [ ] **Step 2: Add switch cases in applyAnnotations()**

In `applyAnnotations()`, after the `DURATION` case (line 1502), add:

```java
                case DATE_EFFECTIVE -> {
                    LocalDate date = LocalDate.parse(ann.rawValue());
                    Calendar cal = GregorianCalendar.from(
                            date.atStartOfDay(ZoneId.systemDefault()));
                    rule.setDateEffective(cal);
                }
                case DATE_EXPIRES -> {
                    LocalDate date = LocalDate.parse(ann.rawValue());
                    Calendar cal = GregorianCalendar.from(
                            date.atStartOfDay(ZoneId.systemDefault()));
                    rule.setDateExpires(cal);
                }
```

- [ ] **Step 3: Run parse tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="RuleAnnotationsTest#testDateEffective*,RuleAnnotationsTest#testDateExpires*,RuleAnnotationsTest#testDuplicateDateEffective*" -pl . -q`

Expected: ALL PASS

---

### Task 5: Write runtime tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

These tests use far-past and far-future dates so the system clock determines whether the rule is inside/outside the activation window. No pseudo clock needed.

- [ ] **Step 1: Write test — rule with past effective date fires**

Add to `RuleAnnotationsTest.java`:

```java
    @Test
    void testDateEffectivePastDateRuleFires() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("2020-01-01")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p.getName()); }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            kieSession.getEntryPoint("persons").insert(new Person("Alice", 30));
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(1);
        });
    }
```

- [ ] **Step 2: Write test — rule with future effective date does NOT fire**

```java
    @Test
    void testDateEffectiveFutureDateRuleDoesNotFire() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("2099-01-01")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p.getName()); }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            kieSession.getEntryPoint("persons").insert(new Person("Alice", 30));
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(0);
        });
    }
```

- [ ] **Step 3: Write test — rule with future expiry date fires**

```java
    @Test
    void testDateExpiresFutureDateRuleFires() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateExpires;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateExpires("2099-01-01")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p.getName()); }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            kieSession.getEntryPoint("persons").insert(new Person("Alice", 30));
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(1);
        });
    }
```

- [ ] **Step 4: Write test — rule with past expiry date does NOT fire**

```java
    @Test
    void testDateExpiresPastDateRuleDoesNotFire() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateExpires;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateExpires("2020-01-01")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p.getName()); }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            kieSession.getEntryPoint("persons").insert(new Person("Alice", 30));
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(0);
        });
    }
```

- [ ] **Step 5: Write test — within date window, rule fires**

```java
    @Test
    void testDateWindowCurrentlyActiveRuleFires() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;
                import org.drools.drlx.annotations.DateExpires;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("2020-01-01")
                @DateExpires("2099-12-31")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p.getName()); }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            kieSession.getEntryPoint("persons").insert(new Person("Alice", 30));
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(1);
        });
    }
```

- [ ] **Step 6: Write test — outside date window (future), rule does NOT fire**

```java
    @Test
    void testDateWindowFutureRuleDoesNotFire() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;
                import org.drools.drlx.annotations.DateExpires;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("2098-01-01")
                @DateExpires("2099-12-31")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p.getName()); }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            kieSession.getEntryPoint("persons").insert(new Person("Alice", 30));
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(0);
        });
    }
```

- [ ] **Step 7: Write test — outside date window (past), rule does NOT fire**

```java
    @Test
    void testDateWindowExpiredRuleDoesNotFire() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.annotations.DateEffective;
                import org.drools.drlx.annotations.DateExpires;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                @DateEffective("2019-01-01")
                @DateExpires("2020-01-01")
                rule R1 {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p.getName()); }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            kieSession.getEntryPoint("persons").insert(new Person("Alice", 30));
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(0);
        });
    }
```

- [ ] **Step 8: Run all new tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="RuleAnnotationsTest#testDateEffective*,RuleAnnotationsTest#testDateExpires*,RuleAnnotationsTest#testDateWindow*,RuleAnnotationsTest#testDuplicateDateEffective*" -pl . -q`

Expected: ALL PASS

---

### Task 6: Full test suite and commit

- [ ] **Step 1: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -q`

Expected: BUILD SUCCESS

- [ ] **Step 2: Run the full drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -q`

Expected: All tests pass (383 existing + new date annotation tests), 0 failures, 0 errors

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/annotations/DateEffective.java \
    drlx-parser-core/src/main/java/org/drools/drlx/annotations/DateExpires.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#86): @DateEffective/@DateExpires rule annotations

ISO-8601 date-only format (yyyy-MM-dd), parse-time validation via
LocalDate.parse(), Calendar conversion for RuleImpl date setters.
Closes #86"
```
