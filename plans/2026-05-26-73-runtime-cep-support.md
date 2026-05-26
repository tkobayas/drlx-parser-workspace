# #73 Runtime CEP Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable runtime CEP window behavior by accepting external `KieBaseConfiguration` (for STREAM mode) and deriving `ClassObjectType.isEvent` from `@Role` annotation.

**Architecture:** Two surgical fixes — `DrlxRuleBuilder` gains config-accepting overloads; `DrlxRuleAstRuntimeBuilder.buildPattern()` replaces window-based `isEvent` with annotation-based. One integration test validates the full runtime path.

**Tech Stack:** drools-core (`RuleBaseFactory`, `EventProcessingOption`), kie-api (`KieBaseConfiguration`, `@Role`)

---

### Task 1: Add `KieBaseConfiguration` overloads to `DrlxRuleBuilder`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleBuilder.java`

- [ ] **Step 1: Add config-accepting `createKieBase` overload**

Add a new import and a new overloaded method. Change the existing no-arg overload to delegate.

Add import:

```java
import org.kie.api.KieBaseConfiguration;
```

Replace the existing `createKieBase` method (lines 42-46):

```java
    public KieBase createKieBase(List<KiePackage> kiePackages) {
        RuleBase kBase = RuleBaseFactory.newRuleBase("myKBase", RuleBaseFactory.newKnowledgeBaseConfiguration());
        kBase.addPackages(kiePackages);
        return KnowledgeBaseFactory.newKnowledgeBase(kBase);
    }
```

With:

```java
    public KieBase createKieBase(List<KiePackage> kiePackages) {
        return createKieBase(kiePackages, RuleBaseFactory.newKnowledgeBaseConfiguration());
    }

    public KieBase createKieBase(List<KiePackage> kiePackages, KieBaseConfiguration config) {
        RuleBase kBase = RuleBaseFactory.newRuleBase("myKBase", config);
        kBase.addPackages(kiePackages);
        return KnowledgeBaseFactory.newKnowledgeBase(kBase);
    }
```

- [ ] **Step 2: Add config-accepting `build` overload**

Add a new `build` overload after the existing `build(String)` method (after line 54):

```java
    public KieBase build(String drlxSource, KieBaseConfiguration config) {
        List<KiePackage> kiePackages = parse(drlxSource);
        return createKieBase(kiePackages, config);
    }
```

- [ ] **Step 3: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run existing tests to confirm no regression**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -q`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleBuilder.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(builder): accept KieBaseConfiguration in createKieBase/build

Adds overloaded createKieBase(packages, config) and build(source, config)
so callers can set EventProcessingOption.STREAM for CEP windows.
Existing no-arg methods delegate with default config.

Refs #73"
```

---

### Task 2: Fix `ClassObjectType.isEvent` derivation in `buildPattern()`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Add import and fix `isEvent` derivation**

Add import:

```java
import org.kie.api.definition.type.Role;
```

In `buildPattern()` method, replace line 1221:

```java
        boolean isEvent = parseResult.windowType() != null;
```

With:

```java
        Role roleAnnotation = type.getAnnotation(Role.class);
        boolean isEvent = roleAnnotation != null && roleAnnotation.value() == Role.Type.EVENT;
```

- [ ] **Step 2: Run existing tests to confirm no regression**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -q`
Expected: All tests pass (existing window tests in `WindowVisitorTest` still pass because `Withdrawal` has `@Role(EVENT)`)

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "fix(builder): derive ClassObjectType.isEvent from @Role annotation

Previously isEvent was set based on windowType != null. Now it checks
the @Role(EVENT) annotation on the domain class, which is consistent
with how TypeDeclaration.createTypeDeclarationForBean() determines
event types.

Refs #73"
```

---

### Task 3: Add session-level integration test

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowTest.java`

- [ ] **Step 1: Write the integration test**

Create `WindowTest.java`:

```java
package org.drools.drlx.builder.syntax;

import java.util.concurrent.atomic.AtomicInteger;

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

import static org.assertj.core.api.Assertions.assertThat;

@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")
class WindowTest {

    @Test
    void lengthWindowRuleFiresAtSessionLevel() {
        String drlx = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Withdrawal;
                import org.drools.drlx.ruleunit.WithdrawalUnit;
                unit WithdrawalUnit;
                rule R1 {
                    var w : /withdrawals | length[5],
                    do {}
                }
                """;

        KieBaseConfiguration config = RuleBaseFactory.newKnowledgeBaseConfiguration();
        config.setOption(EventProcessingOption.STREAM);
        KieBase kieBase = new DrlxRuleBuilder().build(drlx, config);

        WithdrawalUnit unit = new WithdrawalUnit();
        unit.withdrawals.add(new Withdrawal("A1", 100.0));
        unit.withdrawals.add(new Withdrawal("A2", 200.0));

        try (DrlxRuleUnitInstance<WithdrawalUnit> instance =
                     DrlxRuleUnitInstance.create(kieBase, unit)) {
            int fired = instance.fire();
            assertThat(fired).isEqualTo(2);
        }
    }
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=WindowTest -q`
Expected: PASS — two `Withdrawal` events inserted within a length-5 window, rule fires twice.

- [ ] **Step 3: Run full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -q`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/WindowTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(syntax): add session-level integration test for CEP windows

Verifies the full runtime path: STREAM-mode KieBase, @Role(EVENT)
domain class, DrlxRuleUnitInstance, length window, fact insertion
and rule firing without ClassCastException.

Closes #73"
```
