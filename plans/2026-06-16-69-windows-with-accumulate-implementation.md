# #69 Windows with Accumulate — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Verify and test that DRLX correctly combines CEP sliding windows with accumulate patterns (issue #69).

**Architecture:** The existing pipeline already handles this combination — `buildPatternFromBoundOopath()` extracts window info into `PatternIR`, and `buildPattern()` applies window behaviors regardless of whether the pattern is an accumulate source. This plan adds visitor-level tests (V1–V8) and session-level tests (S1–S4) to confirm end-to-end correctness.

**Tech Stack:** Java 17, JUnit 5, AssertJ, ANTLR4 (DRLX grammar), Drools CEP (SlidingTimeWindow, SlidingLengthWindow)

---

### Task 1: Add `results` field to `WithdrawalUnit`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/WithdrawalUnit.java`

The session-level tests need to capture accumulated values. `WithdrawalUnit` currently has only `DataStream<Withdrawal> withdrawals`. Add a `results` list matching the pattern in `MyUnit`.

- [ ] **Step 1: Add the results field**

In `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/WithdrawalUnit.java`, add a `results` field:

```java
package org.drools.drlx.ruleunit;

import java.util.ArrayList;
import java.util.List;

import org.drools.drlx.domain.Withdrawal;
import org.drools.ruleunits.api.DataSource;
import org.drools.ruleunits.api.DataStream;
import org.drools.ruleunits.api.RuleUnitData;

public class WithdrawalUnit implements RuleUnitData {
    public DataStream<Withdrawal> withdrawals = DataSource.createStream();
    public List<Object> results = new ArrayList<>();
}
```

- [ ] **Step 2: Verify existing window tests still pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.WindowTest" -pl .`

Expected: All 6 tests pass. Adding `results` doesn't affect rules that don't reference it.

- [ ] **Step 3: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "test: add results field to WithdrawalUnit for accumulate tests

Refs #69"
```

---

### Task 2: Add visitor-level tests V1–V4 (inline form + acc single function)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`

Add tests that parse window+accumulate rules and assert the IR structure. These use the existing `parseRule()` helper in `AccumulateVisitorTest`.

- [ ] **Step 1: Add V1 — inline time window with single avg**

Add this test method to `AccumulateVisitorTest.java`, before the `parseRule()` helper at the bottom of the class:

```java
    // --- window + accumulate tests ---

    @Test
    void inlineTimeWindowWithSingleAccumulate() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals | time[5s],
                    var a = avg(w.amount),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        assertThat(rule.lhs()).hasSize(1);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        var src = (DrlxRuleAstModel.PatternIR) accPat.source();
        assertThat(src.bindName()).isEqualTo("w");
        assertThat(src.windowType()).isEqualTo("time");
        assertThat(src.windowParameter()).isEqualTo("5s");
        assertThat(accPat.accumulators()).hasSize(1);
        assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("avg");
        assertThat(accPat.accumulators().get(0).resultBindName()).isEqualTo("a");
        assertThat(accPat.accumulators().get(0).argExpressions()).containsExactly("w.amount");
    }
```

- [ ] **Step 2: Add V2 — inline length window with single avg**

```java
    @Test
    void inlineLengthWindowWithSingleAccumulate() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals | length[3],
                    var a = avg(w.amount),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        var src = (DrlxRuleAstModel.PatternIR) accPat.source();
        assertThat(src.windowType()).isEqualTo("length");
        assertThat(src.windowParameter()).isEqualTo("3");
        assertThat(accPat.accumulators()).hasSize(1);
        assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("avg");
    }
```

- [ ] **Step 3: Add V3 — inline time window with multiple accumulators**

```java
    @Test
    void inlineTimeWindowWithMultipleAccumulators() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals | time[5s],
                    var a = avg(w.amount),
                    long n = count(),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        var src = (DrlxRuleAstModel.PatternIR) accPat.source();
        assertThat(src.windowType()).isEqualTo("time");
        assertThat(src.windowParameter()).isEqualTo("5s");
        assertThat(accPat.accumulators()).hasSize(2);
        assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("avg");
        assertThat(accPat.accumulators().get(1).functionName()).isEqualTo("count");
    }
```

- [ ] **Step 4: Add V4 — acc keyword 2-param time window**

```java
    @Test
    void accKeywordTimeWindowSingleFunction() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    acc(var w : /withdrawals | time[5s],
                        var a = avg(w.amount)),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        assertThat(rule.lhs()).hasSize(1);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        var src = (DrlxRuleAstModel.PatternIR) accPat.source();
        assertThat(src.bindName()).isEqualTo("w");
        assertThat(src.windowType()).isEqualTo("time");
        assertThat(src.windowParameter()).isEqualTo("5s");
        assertThat(accPat.accumulators()).hasSize(1);
        assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("avg");
        assertThat(accPat.accumulators().get(0).resultBindName()).isEqualTo("a");
    }
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.AccumulateVisitorTest" -pl .`

Expected: All tests pass (existing + V1–V4).

- [ ] **Step 6: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "test: add visitor tests for inline/acc window+accumulate (V1-V4)

Refs #69"
```

---

### Task 3: Add visitor-level tests V5–V8 (acc length, acc multi, custom, constraint)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`

- [ ] **Step 1: Add V5 — acc keyword 2-param length window**

```java
    @Test
    void accKeywordLengthWindowSingleFunction() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    acc(var w : /withdrawals | length[3],
                        var a = avg(w.amount)),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        var src = (DrlxRuleAstModel.PatternIR) accPat.source();
        assertThat(src.windowType()).isEqualTo("length");
        assertThat(src.windowParameter()).isEqualTo("3");
        assertThat(accPat.accumulators()).hasSize(1);
        assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("avg");
    }
```

- [ ] **Step 2: Add V6 — acc keyword 2-param time window with multiple functions**

```java
    @Test
    void accKeywordTimeWindowGroupedFunctions() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    acc(var w : /withdrawals | time[5s],
                        (var a = avg(w.amount),
                         long n = count())),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        var src = (DrlxRuleAstModel.PatternIR) accPat.source();
        assertThat(src.windowType()).isEqualTo("time");
        assertThat(src.windowParameter()).isEqualTo("5s");
        assertThat(accPat.accumulators()).hasSize(2);
        assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("avg");
        assertThat(accPat.accumulators().get(1).functionName()).isEqualTo("count");
    }
```

- [ ] **Step 3: Add V7 — acc keyword custom 3-param with time window**

```java
    @Test
    void accKeywordCustom3ParamWithTimeWindow() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    acc(var w : /withdrawals | time[5s],
                        int s = 0;,
                        s = (int)(s + w.amount),
                        int sum = s),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        assertThat(rule.lhs()).hasSize(1);
        var custom = (DrlxRuleAstModel.CustomAccumulateIR) rule.lhs().get(0);
        var src = (DrlxRuleAstModel.PatternIR) custom.source();
        assertThat(src.windowType()).isEqualTo("time");
        assertThat(src.windowParameter()).isEqualTo("5s");
        assertThat(custom.initVars()).hasSize(1);
        assertThat(custom.initVars().get(0).typeName()).isEqualTo("int");
        assertThat(custom.initVars().get(0).name()).isEqualTo("s");
        assertThat(custom.resultBindName()).isEqualTo("sum");
        assertThat(custom.resultExpression()).isEqualTo("s");
    }
```

- [ ] **Step 4: Add V8 — inline with constraint before window**

```java
    @Test
    void inlineConstraintBeforeWindowWithAccumulate() {
        String drlx = """
                package p;
                unit MyUnit;
                rule R1 {
                    var w : /withdrawals[amount > 100] | time[5s],
                    var a = avg(w.amount),
                    do {}
                }
                """;
        var rule = parseRule(drlx);
        var accPat = (DrlxRuleAstModel.AccumulatePatternIR) rule.lhs().get(0);
        var src = (DrlxRuleAstModel.PatternIR) accPat.source();
        assertThat(src.conditions()).isNotEmpty();
        assertThat(src.conditions()).containsExactly("amount > 100");
        assertThat(src.windowType()).isEqualTo("time");
        assertThat(src.windowParameter()).isEqualTo("5s");
        assertThat(accPat.accumulators()).hasSize(1);
        assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("avg");
    }
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.AccumulateVisitorTest" -pl .`

Expected: All tests pass (existing + V1–V8).

- [ ] **Step 6: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "test: add visitor tests for remaining window+accumulate cases (V5-V8)

Refs #69"
```

---

### Task 4: Create `WindowAccumulateTest.java` with session-level tests S1–S2

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowAccumulateTest.java`

Session-level tests verify the full pipeline: parse → compile → execute with real CEP windowing and accumulate function evaluation. These use `WithdrawalUnit` (DataStream + @Role(EVENT)) with pseudo clock for time windows.

- [ ] **Step 1: Create the test class with S1 — inline time window avg**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowAccumulateTest.java`:

```java
package org.drools.drlx.builder.syntax;

import java.util.concurrent.TimeUnit;

import org.drools.core.ClockType;
import org.drools.core.SessionConfiguration;
import org.drools.core.impl.RuleBaseFactory;
import org.drools.drlx.builder.DrlxRuleBuilder;
import org.drools.drlx.domain.Withdrawal;
import org.drools.drlx.ruleunit.DrlxRuleUnitInstance;
import org.drools.drlx.ruleunit.WithdrawalUnit;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.DisabledIfSystemProperty;
import org.kie.api.KieBase;
import org.kie.api.KieBaseConfiguration;
import org.kie.api.conf.EventProcessingOption;
import org.kie.api.time.SessionPseudoClock;

import static org.assertj.core.api.Assertions.assertThat;

@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")
class WindowAccumulateTest {

    private static KieBase buildWithStreamMode(String drlx) {
        KieBaseConfiguration config = RuleBaseFactory.newKnowledgeBaseConfiguration();
        config.setOption(EventProcessingOption.STREAM);
        return new DrlxRuleBuilder().build(drlx, config);
    }

    @Test
    void inlineTimeWindowAvg() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var w : /withdrawals | time[5s],
                    var avgAmount = avg(w.amount),
                    do { results.add(avgAmount); }
                }
                """;
        KieBase kieBase = buildWithStreamMode(drlx);

        SessionConfiguration sessionConfig = RuleBaseFactory.newKnowledgeSessionConfiguration()
                .as(SessionConfiguration.KEY);
        sessionConfig.setClockType(ClockType.PSEUDO_CLOCK);

        WithdrawalUnit unit = new WithdrawalUnit();

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit, sessionConfig)) {
            SessionPseudoClock clock = instance.getClock();

            unit.withdrawals.append(new Withdrawal("A1", 100.0));
            unit.withdrawals.append(new Withdrawal("A2", 200.0));
            unit.withdrawals.append(new Withdrawal("A3", 300.0));

            clock.advanceTime(6, TimeUnit.SECONDS);

            unit.withdrawals.append(new Withdrawal("A4", 400.0));
            instance.fire();

            assertThat(unit.results).containsExactly(400.0);
        }
    }
}
```

- [ ] **Step 2: Run S1 to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.WindowAccumulateTest#inlineTimeWindowAvg" -pl .`

Expected: PASS. If it fails, investigate the error — the pipeline should produce the correct runtime structures.

- [ ] **Step 3: Add S2 — inline length window avg**

Add this test method to `WindowAccumulateTest`:

```java
    @Test
    void inlineLengthWindowAvg() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var w : /withdrawals | length[3],
                    var avgAmount = avg(w.amount),
                    do { results.add(avgAmount); }
                }
                """;
        KieBase kieBase = buildWithStreamMode(drlx);

        WithdrawalUnit unit = new WithdrawalUnit();

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit)) {
            unit.withdrawals.append(new Withdrawal("A1", 100.0));
            unit.withdrawals.append(new Withdrawal("A2", 200.0));
            unit.withdrawals.append(new Withdrawal("A3", 300.0));
            unit.withdrawals.append(new Withdrawal("A4", 400.0));
            unit.withdrawals.append(new Withdrawal("A5", 500.0));
            instance.fire();

            // length[3] keeps only last 3 events: 300, 400, 500 → avg = 400.0
            assertThat(unit.results).containsExactly(400.0);
        }
    }
```

- [ ] **Step 4: Run S2 to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.WindowAccumulateTest#inlineLengthWindowAvg" -pl .`

Expected: PASS.

- [ ] **Step 5: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "test: add session-level tests for inline window+accumulate (S1-S2)

Refs #69"
```

---

### Task 5: Add session-level tests S3–S4 (acc keyword form)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowAccumulateTest.java`

- [ ] **Step 1: Add S3 — acc keyword time window avg**

```java
    @Test
    void accKeywordTimeWindowAvg() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    acc(var w : /withdrawals | time[5s],
                        var avgAmount = avg(w.amount)),
                    do { results.add(avgAmount); }
                }
                """;
        KieBase kieBase = buildWithStreamMode(drlx);

        SessionConfiguration sessionConfig = RuleBaseFactory.newKnowledgeSessionConfiguration()
                .as(SessionConfiguration.KEY);
        sessionConfig.setClockType(ClockType.PSEUDO_CLOCK);

        WithdrawalUnit unit = new WithdrawalUnit();

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit, sessionConfig)) {
            SessionPseudoClock clock = instance.getClock();

            unit.withdrawals.append(new Withdrawal("A1", 100.0));
            unit.withdrawals.append(new Withdrawal("A2", 200.0));
            unit.withdrawals.append(new Withdrawal("A3", 300.0));

            clock.advanceTime(6, TimeUnit.SECONDS);

            unit.withdrawals.append(new Withdrawal("A4", 400.0));
            instance.fire();

            assertThat(unit.results).containsExactly(400.0);
        }
    }
```

- [ ] **Step 2: Add S4 — acc keyword length window avg**

```java
    @Test
    void accKeywordLengthWindowAvg() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    acc(var w : /withdrawals | length[3],
                        var avgAmount = avg(w.amount)),
                    do { results.add(avgAmount); }
                }
                """;
        KieBase kieBase = buildWithStreamMode(drlx);

        WithdrawalUnit unit = new WithdrawalUnit();

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit)) {
            unit.withdrawals.append(new Withdrawal("A1", 100.0));
            unit.withdrawals.append(new Withdrawal("A2", 200.0));
            unit.withdrawals.append(new Withdrawal("A3", 300.0));
            unit.withdrawals.append(new Withdrawal("A4", 400.0));
            unit.withdrawals.append(new Withdrawal("A5", 500.0));
            instance.fire();

            // length[3] keeps only last 3 events: 300, 400, 500 → avg = 400.0
            assertThat(unit.results).containsExactly(400.0);
        }
    }
```

- [ ] **Step 3: Run all session-level tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.WindowAccumulateTest" -pl .`

Expected: All 4 tests pass (S1–S4).

- [ ] **Step 4: Commit**

```
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -am "test: add session-level tests for acc keyword window+accumulate (S3-S4)

Refs #69"
```

---

### Task 6: Run full test suite and close issue

**Files:** None (verification only)

- [ ] **Step 1: Run the full drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`

Expected: All 461+ tests pass with no regressions.

- [ ] **Step 2: Close issue #69**

If all tests pass:

```
gh issue close 69 --repo tkobayas/drlx-parser --comment "Windows with accumulate verified and tested. Both inline and acc keyword forms work end-to-end — the existing pipeline correctly combines window behaviors with accumulate patterns. Added 8 visitor-level tests (V1-V8) and 4 session-level tests (S1-S4)."
```
