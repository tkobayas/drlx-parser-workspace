# @Duration/@Timer Rule Annotations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `@Timer(String)` and `@Duration(String)` rule annotations to DRLX, supporting interval and cron timer scheduling via drools' existing `RuleBuilder.buildTimer()` and `DurationTimer`.

**Architecture:** Follow the established annotation pipeline: annotation class → visitor resolution (with `@Timer`+`@Duration` mutual exclusion check) → IR model (`Kind.TIMER`/`Kind.DURATION` with `ArgShape.STRING`) → protobuf serialisation → runtime application in `DrlxRuleAstRuntimeBuilder` (delegating to `RuleBuilder.buildTimer()` for `@Timer`, constructing `DurationTimer` directly for `@Duration`). Timer tests use pseudo clock for deterministic assertions.

**Tech Stack:** Java 17, drools 10.1.0 (`RuleBuilder`, `TimeUtils`, `IntervalTimer`, `CronTimer`, `DurationTimer`, `PseudoClockScheduler`), protobuf 3.25.5, JUnit 5, AssertJ

---

### Task 1: Annotation Classes

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/Timer.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/Duration.java`

- [ ] **Step 1: Create Timer annotation**

Create `drlx-parser-core/src/main/java/org/drools/drlx/annotations/Timer.java`:

```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Timer {
    String value();
}
```

- [ ] **Step 2: Create Duration annotation**

Create `drlx-parser-core/src/main/java/org/drools/drlx/annotations/Duration.java`:

```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Duration {
    String value();
}
```

- [ ] **Step 3: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add drlx-parser-core/src/main/java/org/drools/drlx/annotations/Timer.java \
       drlx-parser-core/src/main/java/org/drools/drlx/annotations/Duration.java
git commit -m "feat(#85): add @Timer and @Duration annotation classes"
```

---

### Task 2: IR Model + Protobuf

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:30-43`
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto:80-91`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java:286-309`

- [ ] **Step 1: Add Kind entries to DrlxRuleAstModel**

In `DrlxRuleAstModel.java`, add `TIMER` and `DURATION` to the `Kind` enum (lines 37-38). Change:

```java
            ACTIVATION_GROUP(ArgShape.STRING);
```

to:

```java
            ACTIVATION_GROUP(ArgShape.STRING),
            TIMER(ArgShape.STRING),
            DURATION(ArgShape.STRING);
```

- [ ] **Step 2: Add protobuf enum values**

In `drlx_rule_ast.proto`, add two new entries to `AnnotationKind` (after line 90, before the closing `}`):

```protobuf
  ANNOTATION_KIND_TIMER = 11;
  ANNOTATION_KIND_DURATION = 12;
```

- [ ] **Step 3: Add proto ↔ Kind mapping in DrlxRuleAstParseResult**

In `DrlxRuleAstParseResult.java`, add to the `fromProtoKind` switch (lines 286-297, before the `UNSPECIFIED` case):

```java
            case ANNOTATION_KIND_TIMER -> RuleAnnotationIR.Kind.TIMER;
            case ANNOTATION_KIND_DURATION -> RuleAnnotationIR.Kind.DURATION;
```

Add to the `toProtoKind` switch (lines 300-309, before the closing `};`):

```java
            case TIMER -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_TIMER;
            case DURATION -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_DURATION;
```

- [ ] **Step 4: Verify compilation (protobuf regeneration)**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS (protobuf plugin regenerates Java classes from .proto)

- [ ] **Step 5: Commit**

```
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
       drlx-parser-core/src/main/proto/drlx_rule_ast.proto \
       drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java
git commit -m "feat(#85): add TIMER/DURATION Kind entries and protobuf mapping"
```

---

### Task 3: Visitor — Resolution + Mutual Exclusion

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:37-52` (FQN constants and SUPPORTED map)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:211-237` (buildRuleAnnotations — mutual exclusion)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:259-280` (resolveKind — error message)
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

- [ ] **Step 1: Write validation tests for @Timer and @Duration argument shapes**

Add to `RuleAnnotationsTest.java`:

```java
@Test
void testTimerWithoutArgumentFails() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Timer;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Timer
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Timer expects one argument");
}

@Test
void testDurationWithoutArgumentFails() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Duration;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Duration
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Duration expects one argument");
}

@Test
void testTimerEmptyStringFails() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Timer;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Timer("")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Timer expects non-empty string literal");
}

@Test
void testDurationEmptyStringFails() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Duration;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Duration("")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Duration expects non-empty string literal");
}

@Test
void testTimerAndDurationTogetherFails() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Timer;
            import org.drools.drlx.annotations.Duration;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Timer("int: 1s")
            @Duration("5s")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Timer and @Duration cannot be used together");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=RuleAnnotationsTest#testTimerWithoutArgumentFails+testDurationWithoutArgumentFails+testTimerEmptyStringFails+testDurationEmptyStringFails+testTimerAndDurationTogetherFails -q`
Expected: FAIL (annotations not yet recognized)

- [ ] **Step 3: Add FQN constants and SUPPORTED_ANNOTATION_KINDS entries**

In `DrlxToRuleAstVisitor.java`, add constants after line 43:

```java
    private static final String TIMER_FQN = "org.drools.drlx.annotations.Timer";
    private static final String DURATION_FQN = "org.drools.drlx.annotations.Duration";
```

Add entries to the `SUPPORTED_ANNOTATION_KINDS` map (line 52, before the closing `)`):

```java
            Map.entry(TIMER_FQN, Kind.TIMER),
            Map.entry(DURATION_FQN, Kind.DURATION));
```

(Remove the `)` after `ACTIVATION_GROUP` entry and add a comma instead.)

- [ ] **Step 4: Update error message in resolveKind**

In `resolveKind()` (line 269-271), update the supported annotations list in the error message to include `@Timer` and `@Duration`:

```java
                    + "@ActivationGroup, @Timer, @Duration");
```

- [ ] **Step 5: Add mutual exclusion check in buildRuleAnnotations**

In `buildRuleAnnotations()`, after the duplicate check block (after line 230), add:

```java
            if ((kind == Kind.TIMER && seen.contains(Kind.DURATION))
                    || (kind == Kind.DURATION && seen.contains(Kind.TIMER))) {
                throw new RuntimeException(
                        "@Timer and @Duration cannot be used together at " + line + ":" + col);
            }
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=RuleAnnotationsTest#testTimerWithoutArgumentFails+testDurationWithoutArgumentFails+testTimerEmptyStringFails+testDurationEmptyStringFails+testTimerAndDurationTogetherFails -q`
Expected: PASS

- [ ] **Step 7: Commit**

```
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
       drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git commit -m "feat(#85): visitor resolution and mutual exclusion for @Timer/@Duration"
```

---

### Task 4: Runtime Builder — Timer Construction

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:0-55` (imports)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:1465-1477` (applyAnnotations)
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

- [ ] **Step 1: Write validation tests for invalid timer/duration specs**

Add to `RuleAnnotationsTest.java`:

```java
@Test
void testTimerInvalidProtocolFails() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Timer;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Timer("xyz: 1s")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class);
}

@Test
void testTimerMissingColonFails() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Timer;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Timer("int 1s")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class);
}

@Test
void testDurationInvalidTimeStringFails() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Duration;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Duration("abc")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=RuleAnnotationsTest#testTimerInvalidProtocolFails+testTimerMissingColonFails+testDurationInvalidTimeStringFails -q`
Expected: FAIL (runtime builder doesn't handle TIMER/DURATION yet)

- [ ] **Step 3: Add imports to DrlxRuleAstRuntimeBuilder**

Add these imports to `DrlxRuleAstRuntimeBuilder.java`:

```java
import org.drools.base.time.impl.Timer;
import org.drools.core.time.impl.DurationTimer;
import org.drools.core.time.TimerExpression;
import org.drools.base.base.ValueResolver;
import org.drools.base.reteoo.BaseTuple;
import org.drools.base.rule.Declaration;
import org.drools.compiler.rule.builder.RuleBuilder;
```

Note: `Timer` here is the `org.drools.base.time.impl.Timer` interface, not `java.util.Timer`. The existing `TimeUtils` import at line 38 is already present.

- [ ] **Step 4: Add TIMER and DURATION cases to applyAnnotations**

In `applyAnnotations()` (lines 1465-1477), add two new cases to the switch before the closing `}`:

```java
                case TIMER -> {
                    org.drools.base.time.impl.Timer timer = RuleBuilder.buildTimer(
                            ann.rawValue(),
                            null,
                            s -> {
                                if (s == null) return null;
                                long ms = TimeUtils.parseTimeString(s);
                                return new TimerExpression() {
                                    @Override public Declaration[] getDeclarations() { return new Declaration[0]; }
                                    @Override public Object getValue(BaseTuple leftTuple, Declaration[] declrs, ValueResolver valueResolver) { return ms; }
                                };
                            },
                            err -> { throw new RuntimeException(err); }
                    );
                    if (timer == null) {
                        throw new RuntimeException("Failed to build timer from: " + ann.rawValue());
                    }
                    rule.setTimer(timer);
                }
                case DURATION -> {
                    long ms = TimeUtils.parseTimeString(ann.rawValue());
                    rule.setTimer(new DurationTimer(ms));
                }
```

- [ ] **Step 5: Run validation tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=RuleAnnotationsTest#testTimerInvalidProtocolFails+testTimerMissingColonFails+testDurationInvalidTimeStringFails -q`
Expected: PASS

- [ ] **Step 6: Commit**

```
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
       drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git commit -m "feat(#85): runtime builder applies @Timer and @Duration to RuleImpl"
```

---

### Task 5: Functional Tests with Pseudo Clock

**Files:**
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

These tests use `PseudoClockScheduler` for deterministic time control — no real-time waiting, no Awaitility needed.

- [ ] **Step 1: Write interval timer test**

Add to `RuleAnnotationsTest.java` (add necessary imports at top):

```java
import org.drools.core.ClockType;
import org.drools.core.SessionConfiguration;
import org.drools.core.time.impl.PseudoClockScheduler;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```

```java
@Test
void testIntervalTimerFiresRepeatedly() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Timer;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Timer("int: 1s 1s")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println("timer fired for " + p.getName()); }
            }
            """;

    final KieBase kieBase = newBuilder().build(rule);
    SessionConfiguration config = SessionConfiguration.newInstance();
    config.setClockType(ClockType.PSEUDO_CLOCK);

    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit, config)) {
        AtomicInteger fireCount = new AtomicInteger();
        instance.addEventListener(new DefaultAgendaEventListener() {
            @Override
            public void afterMatchFired(AfterMatchFiredEvent event) {
                fireCount.incrementAndGet();
            }
        });

        PseudoClockScheduler clock = instance.getClock();

        // Initial fire — rule matches but timer hasn't elapsed yet
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(0);

        // Advance past the 1s delay
        clock.advanceTime(1, TimeUnit.SECONDS);
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(1);

        // Advance another 1s (period)
        clock.advanceTime(1, TimeUnit.SECONDS);
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(2);

        // Advance another 1s (period)
        clock.advanceTime(1, TimeUnit.SECONDS);
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(3);
    }
}
```

- [ ] **Step 2: Write duration timer test**

```java
@Test
void testDurationDelayedFire() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Duration;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Duration("5s")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println("duration fired for " + p.getName()); }
            }
            """;

    final KieBase kieBase = newBuilder().build(rule);
    SessionConfiguration config = SessionConfiguration.newInstance();
    config.setClockType(ClockType.PSEUDO_CLOCK);

    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit, config)) {
        AtomicInteger fireCount = new AtomicInteger();
        instance.addEventListener(new DefaultAgendaEventListener() {
            @Override
            public void afterMatchFired(AfterMatchFiredEvent event) {
                fireCount.incrementAndGet();
            }
        });

        PseudoClockScheduler clock = instance.getClock();

        // Initial fire — timer hasn't elapsed
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(0);

        // Advance 3s — still not enough
        clock.advanceTime(3, TimeUnit.SECONDS);
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(0);

        // Advance another 3s — past the 5s duration
        clock.advanceTime(3, TimeUnit.SECONDS);
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(1);

        // Duration is one-shot — no more fires
        clock.advanceTime(10, TimeUnit.SECONDS);
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(1);
    }
}
```

- [ ] **Step 3: Write cron timer test**

```java
@Test
void testCronTimerFires() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Timer;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Timer("cron: 0/5 * * * * ?")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println("cron fired for " + p.getName()); }
            }
            """;

    final KieBase kieBase = newBuilder().build(rule);
    SessionConfiguration config = SessionConfiguration.newInstance();
    config.setClockType(ClockType.PSEUDO_CLOCK);

    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit, config)) {
        AtomicInteger fireCount = new AtomicInteger();
        instance.addEventListener(new DefaultAgendaEventListener() {
            @Override
            public void afterMatchFired(AfterMatchFiredEvent event) {
                fireCount.incrementAndGet();
            }
        });

        PseudoClockScheduler clock = instance.getClock();

        // Initial fire — cron hasn't ticked yet
        instance.fire();
        int initial = fireCount.get();

        // Advance 10s — should trigger at least one cron tick (every 5s)
        clock.advanceTime(10, TimeUnit.SECONDS);
        instance.fire();
        assertThat(fireCount.get()).isGreaterThan(initial);
    }
}
```

- [ ] **Step 4: Write default-protocol timer test (no "int:" prefix)**

```java
@Test
void testTimerDefaultProtocol() {
    // "500ms" with no protocol prefix defaults to "int" protocol (delay-only)
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Timer;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Timer("1s")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println("default protocol fired"); }
            }
            """;

    final KieBase kieBase = newBuilder().build(rule);
    SessionConfiguration config = SessionConfiguration.newInstance();
    config.setClockType(ClockType.PSEUDO_CLOCK);

    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit, config)) {
        AtomicInteger fireCount = new AtomicInteger();
        instance.addEventListener(new DefaultAgendaEventListener() {
            @Override
            public void afterMatchFired(AfterMatchFiredEvent event) {
                fireCount.incrementAndGet();
            }
        });

        PseudoClockScheduler clock = instance.getClock();

        instance.fire();
        assertThat(fireCount.get()).isEqualTo(0);

        // Advance past 1s delay — should fire once (delay-only, no period)
        clock.advanceTime(1, TimeUnit.SECONDS);
        instance.fire();
        assertThat(fireCount.get()).isEqualTo(1);
    }
}
```

- [ ] **Step 5: Check if addEventListener is available on DrlxRuleUnitInstance (before running tests)**

Before running tests, verify that `DrlxRuleUnitInstance` has `addEventListener`. If not, we need to use the `withSession` pattern or add the method.

Run: `grep -n "addEventListener" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java`

If `addEventListener` is not available, use the `KieSession`-based `withSession` approach instead — build the session with pseudo clock config directly:

```java
SessionConfiguration config = SessionConfiguration.newInstance();
config.setClockType(ClockType.PSEUDO_CLOCK);
KieSession kieSession = kieBase.newKieSession(KieSessionConfiguration.from(config), null);
```

Adapt the tests accordingly. The core timer assertions remain the same regardless of which session API is used.

- [ ] **Step 6: Run functional tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=RuleAnnotationsTest#testIntervalTimerFiresRepeatedly+testDurationDelayedFire+testCronTimerFires+testTimerDefaultProtocol -q`
Expected: PASS

If a test fails due to API availability (e.g., missing `addEventListener`), adapt the test approach per Step 4's fallback and re-run.

- [ ] **Step 7: Commit**

```
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git commit -m "test(#85): functional tests for @Timer and @Duration with pseudo clock"
```

---

### Task 6: Full Test Suite + Final Verification

**Files:**
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java` (run full suite)

- [ ] **Step 1: Run the full RuleAnnotationsTest suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=RuleAnnotationsTest -q`
Expected: All tests PASS (existing + new)

- [ ] **Step 2: Run the full drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -pl drlx-parser-core -q`
Expected: All tests PASS, 0 failures, 0 errors

- [ ] **Step 3: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Final commit (if any fixups were needed)**

If any fixes were made during verification, commit them:

```
git commit -am "fix(#85): test suite fixups for @Timer/@Duration"
```
