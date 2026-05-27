# #68 Constraint Before/After Window Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add integration tests proving the semantic difference between constraint-before-window and constraint-after-window in DRLX.

**Architecture:** The grammar, visitor, and builder already handle both forms. We add a `customer` field to the `Withdrawal` event class, then write two tests that insert identical event sequences through a `length[3]` window — one with the inline OOPath constraint (filter-before-window), one with a `test` element (filter-after-window). Different fire counts prove the semantic difference.

**Tech Stack:** Java 17, JUnit 5, AssertJ, Drools STREAM mode, DrlxRuleUnitInstance

**Issue:** https://github.com/tkobayas/drlx-parser/issues/68

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `drlx-parser-core/src/test/java/org/drools/drlx/domain/Withdrawal.java` | Modify | Add `customer` field, 3-arg constructor, 2-arg backward-compat overload, getter, update `toString` |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowTest.java` | Modify | Add two integration tests |

---

## Task 1: Add `customer` field to `Withdrawal`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/domain/Withdrawal.java`

- [ ] **Step 1: Add the `customer` field, 3-arg constructor, getter, and backward-compatible 2-arg overload**

Replace the entire `Withdrawal.java` with:

```java
package org.drools.drlx.domain;

import org.kie.api.definition.type.Role;

@Role(Role.Type.EVENT)
public class Withdrawal {

    private final String accountId;
    private final double amount;
    private final String customer;

    public Withdrawal(String accountId, double amount, String customer) {
        this.accountId = accountId;
        this.amount = amount;
        this.customer = customer;
    }

    public Withdrawal(String accountId, double amount) {
        this(accountId, amount, null);
    }

    public String getAccountId() {
        return accountId;
    }

    public double getAmount() {
        return amount;
    }

    public String getCustomer() {
        return customer;
    }

    @Override
    public String toString() {
        return "Withdrawal[accountId=" + accountId + ", amount=" + amount
                + ", customer=" + customer + "]";
    }
}
```

Key points:
- The 2-arg constructor delegates to the 3-arg one with `customer = null`, so all existing tests compile and behave identically.
- The `customer` field is `final` like the other fields.

- [ ] **Step 2: Run existing tests to verify nothing breaks**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="WindowTest" -pl .
```
Expected: All 4 existing `WindowTest` tests PASS.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/domain/Withdrawal.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: add customer field to Withdrawal domain class

Refs #68"
```

---

## Task 2: Add `constraintBeforeWindowFiltersEntries` test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowTest.java`

- [ ] **Step 1: Write the test**

Add this test method to `WindowTest.java` after the existing tests:

```java
@Test
void constraintBeforeWindowFiltersEntries() {
    String drlx = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Withdrawal;
            import org.drools.drlx.ruleunit.WithdrawalUnit;
            unit WithdrawalUnit;
            rule R1 {
                var w : /withdrawals[customer == "GOLD"] | length[3],
                do {}
            }
            """;
    KieBase kieBase = buildWithStreamMode(drlx);

    WithdrawalUnit unit = new WithdrawalUnit();
    unit.withdrawals.add(new Withdrawal("A1", 100.0, "GOLD"));
    unit.withdrawals.add(new Withdrawal("A2", 200.0, "STANDARD"));
    unit.withdrawals.add(new Withdrawal("A3", 300.0, "GOLD"));
    unit.withdrawals.add(new Withdrawal("A4", 400.0, "STANDARD"));
    unit.withdrawals.add(new Withdrawal("A5", 500.0, "GOLD"));

    try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                 DrlxRuleUnitInstance.create(kieBase, unit)) {
        assertThat(instance.fire()).isEqualTo(3);
    }
}
```

Explanation:
- The inline constraint `[customer == "GOLD"]` filters before the window.
- Only A1, A3, A5 (all GOLD) enter the `length[3]` window → exactly 3 fit → fire count = **3**.

- [ ] **Step 2: Run the test**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="WindowTest#constraintBeforeWindowFiltersEntries" -pl .
```
Expected: PASS with fire count 3.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: verify constraint before window filters entries

Inline OOPath constraint [customer == \"GOLD\"] on a length[3] window
filters facts before they enter the window. Only 3 GOLD events enter,
all fit in the window, fire count = 3.

Refs #68"
```

---

## Task 3: Add `constraintAfterWindowFiltersContents` test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowTest.java`

- [ ] **Step 1: Write the test**

Add this test method to `WindowTest.java` after `constraintBeforeWindowFiltersEntries`:

```java
@Test
void constraintAfterWindowFiltersContents() {
    String drlx = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Withdrawal;
            import org.drools.drlx.ruleunit.WithdrawalUnit;
            unit WithdrawalUnit;
            rule R1 {
                var w : /withdrawals | length[3],
                test w.customer == "GOLD",
                do {}
            }
            """;
    KieBase kieBase = buildWithStreamMode(drlx);

    WithdrawalUnit unit = new WithdrawalUnit();
    unit.withdrawals.add(new Withdrawal("A1", 100.0, "GOLD"));
    unit.withdrawals.add(new Withdrawal("A2", 200.0, "STANDARD"));
    unit.withdrawals.add(new Withdrawal("A3", 300.0, "GOLD"));
    unit.withdrawals.add(new Withdrawal("A4", 400.0, "STANDARD"));
    unit.withdrawals.add(new Withdrawal("A5", 500.0, "GOLD"));

    try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                 DrlxRuleUnitInstance.create(kieBase, unit)) {
        assertThat(instance.fire()).isEqualTo(2);
    }
}
```

Explanation:
- All 5 events enter the unfiltered `length[3]` window.
- After all insertions, the window holds the last 3: A3 (GOLD), A4 (STANDARD), A5 (GOLD).
- The `test w.customer == "GOLD"` filters the window contents, leaving A3 and A5 → fire count = **2**.
- The difference (3 vs 2) proves the before/after semantic distinction.

- [ ] **Step 2: Run the test**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="WindowTest#constraintAfterWindowFiltersContents" -pl .
```
Expected: PASS with fire count 2.

- [ ] **Step 3: Run all WindowTest tests together**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="WindowTest" -pl .
```
Expected: All 6 tests PASS (4 existing + 2 new).

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: verify constraint after window filters contents

test element w.customer == \"GOLD\" on a length[3] window filters facts
after the window. All 5 events enter; window holds last 3 (A3, A4, A5);
test filters to 2 GOLD events. Fire count 3 vs 2 proves the semantic
difference between before-window and after-window constraints.

Closes #68"
```
