# TrackingAgendaEventListener Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Vendor Drools 10.2.0-rc2 `TrackingAgendaEventListener` into `src/test`, change `DrlxBuilderTestSupport.withSession` to auto-attach it and expose it to test lambdas, then migrate 12 test files from the anon-listener/fire-count pattern to strict `containsExactly("RuleName")` assertions. The remaining 4 single-rule smoke-test files keep bare `fireAllRules() == N`.

**Architecture:** Test-infrastructure refactoring only. No production code changes, no grammar changes, no IR/proto changes. The `withSession` signature changes from `Consumer<KieSession>` to `BiConsumer<KieSession, TrackingAgendaEventListener>`. Every caller updates its lambda shape; tests that need rule names also upgrade their assertions.

**Tech Stack:** Java 17, JUnit 5, AssertJ, Drools 10.1 runtime APIs (`org.kie.api.event.rule.*`).

**Spec:** `specs/2026-04-24-tracking-agenda-event-listener-design.md`
**Issue:** https://github.com/tkobayas/drlx-parser/issues/18

---

## File Structure

**Create (1 file):**
- `drlx-parser-core/src/test/java/org/drools/core/event/TrackingAgendaEventListener.java` — verbatim copy from Drools 10.2.0-rc2 + a class-level Javadoc explaining the temporary-vendor rationale

**Modify — `withSession` signature (1 file):**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/DrlxBuilderTestSupport.java` — `Consumer<KieSession>` → `BiConsumer<KieSession, TrackingAgendaEventListener>`; auto-attach a listener and pass it through

**Modify — callers switching to new lambda shape only, no assertion upgrade (3 files):**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/InlineCastTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PositionalTest.java`

**Modify — full migration (anon listener → `TrackingAgendaEventListener` + `containsExactly`) — 10 files:**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotExistsBindingTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrBindingScopeTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PassivePatternTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AndTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NestedGroupTest.java`

**Modify — inline pattern (non-`withSession` tests that run multi-rule) — 2 files:**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleBuilderTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/tools/DrlxCompilerTest.java`

**Stay as-is — single-rule bare fireCount:**
- `DrlxCompilerNoPersistTest.java` (no changes)

---

## Migration Recipe (referenced by Tasks 3–14)

Each migration task replaces the anonymous `DefaultAgendaEventListener` pattern with direct use of `TrackingAgendaEventListener`. The recipe is mechanical; the per-task sections below spell out the specific rule names and expected lists per method.

**In every migrated file, apply all of these changes:**

1. **Imports — remove:**
   ```java
   import java.util.ArrayList;
   import java.util.List;
   import org.kie.api.event.rule.AfterMatchFiredEvent;
   import org.kie.api.event.rule.DefaultAgendaEventListener;
   ```
   Remove `ArrayList`/`List` imports ONLY if no other code in the file still uses them. (Several files also use `List` for proto tests or multi-element reuse — keep those.)

2. **Inside each affected test method:**

   Before:
   ```java
   withSession(rule, kieSession -> {
       final List<String> fired = new ArrayList<>();
       kieSession.addEventListener(new DefaultAgendaEventListener() {
           @Override
           public void afterMatchFired(AfterMatchFiredEvent event) {
               fired.add(event.getMatch().getRule().getName());
           }
       });
       // ... inserts ...
       assertThat(kieSession.fireAllRules()).isEqualTo(1);
       assertThat(fired).hasSize(1);
   });
   ```

   After:
   ```java
   withSession(rule, (kieSession, listener) -> {
       // ... inserts ...
       assertThat(kieSession.fireAllRules()).isEqualTo(1);
       assertThat(listener.getAfterMatchFired()).containsExactly("RuleName");
   });
   ```

   Concrete transforms:
   - Lambda param `kieSession ->` becomes `(kieSession, listener) ->`.
   - Drop the `List<String> fired = new ArrayList<>()` declaration.
   - Drop the entire `kieSession.addEventListener(new DefaultAgendaEventListener() { ... })` block.
   - Replace `assertThat(fired).hasSize(N)` with `assertThat(listener.getAfterMatchFired()).containsExactly("Name1", "Name2", ...)` using the rule names expected in that test.
   - Replace `assertThat(fired).isEmpty()` with `assertThat(listener.getAfterMatchFired()).isEmpty()`.
   - Replace `assertThat(fired).containsExactly("X")` with `assertThat(listener.getAfterMatchFired()).containsExactly("X")` — simple rename.
   - **Drop `fired.clear()` calls** — the listener accumulates across fires; assert cumulative content at the end of the test. This is the spec's explicit policy for multi-fire tests.
   - Keep all `assertThat(kieSession.fireAllRules()).isEqualTo(N)` / `.isZero()` assertions as-is. Fire count is the cycle boundary.

3. **For parse-error and non-listener methods in the same file:**
   Methods using `assertThatThrownBy(() -> withSession(...))` must update the inner lambda from `kieSession -> { /* unreachable */ }` to `(kieSession, listener) -> { /* unreachable */ }`. No assertion changes.
   Methods that don't call `withSession` (e.g., ones using `newBuilder().build(rule)` directly) are untouched.

4. **For multi-fire tests** (`NotTest`, `NestedGroupTest`, `AndTest`, `ExistsTest` have fire → insert → fire patterns):
   Do NOT call `listener.resetAllEvents()`. Assert cumulative state at the end. Each intermediate `fireAllRules() == N` / `.isZero()` assertion still checks per-cycle count via the `int` return.

---

## Task 1: Vendor TrackingAgendaEventListener class

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/core/event/TrackingAgendaEventListener.java`

- [ ] **Step 1: Create the package directory**

Run:
```bash
mkdir -p /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test/java/org/drools/core/event
```

- [ ] **Step 2: Fetch the upstream source to confirm exact contents**

Run:
```bash
curl -sL "https://raw.githubusercontent.com/apache/incubator-kie-drools/10.2.0-rc2/drools-core/src/main/java/org/drools/core/event/TrackingAgendaEventListener.java" > /tmp/tracking.java
wc -l /tmp/tracking.java
```
Expected: ~130 lines (license header + class).

- [ ] **Step 3: Create the file with upstream content + class-level Javadoc**

Write `drlx-parser-core/src/test/java/org/drools/core/event/TrackingAgendaEventListener.java`. Preserve the upstream license header, package declaration, imports, tab indentation, and field/method order verbatim. Insert the following Javadoc immediately above `public class TrackingAgendaEventListener`:

```java
/**
 * Vendored verbatim from Drools 10.2.0-rc2 (drools-core) into drlx-parser's
 * {@code src/test} tree. Placed under the upstream package path
 * ({@code org.drools.core.event}) intentionally: when drlx-parser upgrades
 * its Drools dependency to 10.2.0 or later, deleting this file is a
 * single-command cleanup — the FQCN resolves from {@code drools-core}
 * automatically with zero import changes in any test.
 *
 * <p><strong>Do not modify the class body.</strong> Keep it verbatim against
 * upstream so the diff stays empty and the eventual swap is trivial. Only
 * this Javadoc is drlx-parser-specific.
 *
 * <p>On upgrade: {@code rm} this file, re-run the test suite, done.
 */
```

The file's full content is:

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package org.drools.core.event;

import java.util.ArrayList;
import java.util.List;

import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.AgendaEventListener;
import org.kie.api.event.rule.AgendaGroupPoppedEvent;
import org.kie.api.event.rule.AgendaGroupPushedEvent;
import org.kie.api.event.rule.BeforeMatchFiredEvent;
import org.kie.api.event.rule.MatchCancelledEvent;
import org.kie.api.event.rule.MatchCreatedEvent;
import org.kie.api.event.rule.RuleFlowGroupActivatedEvent;
import org.kie.api.event.rule.RuleFlowGroupDeactivatedEvent;

/**
 * Vendored verbatim from Drools 10.2.0-rc2 (drools-core) into drlx-parser's
 * {@code src/test} tree. Placed under the upstream package path
 * ({@code org.drools.core.event}) intentionally: when drlx-parser upgrades
 * its Drools dependency to 10.2.0 or later, deleting this file is a
 * single-command cleanup — the FQCN resolves from {@code drools-core}
 * automatically with zero import changes in any test.
 *
 * <p><strong>Do not modify the class body.</strong> Keep it verbatim against
 * upstream so the diff stays empty and the eventual swap is trivial. Only
 * this Javadoc is drlx-parser-specific.
 *
 * <p>On upgrade: {@code rm} this file, re-run the test suite, done.
 */
public class TrackingAgendaEventListener implements AgendaEventListener {
	
	private List<String> matchCreated = new ArrayList<>();
	private List<String> matchCancelled = new ArrayList<>();
	private List<String> beforeMatchFired = new ArrayList<>();
	private List<String> afterMatchFired = new ArrayList<>();
	private List<String> agendaGroupPopped = new ArrayList<>();
	private List<String> agendaGroupPushed = new ArrayList<>();
	private List<String> beforeRuleFlowGroupActivated = new ArrayList<>();
	private List<String> afterRuleFlowGroupActivated = new ArrayList<>();
	private List<String> beforeRuleFlowGroupDeactivated = new ArrayList<>();
	private List<String> afterRuleFlowGroupDeactivated = new ArrayList<>();
	
	public TrackingAgendaEventListener() {
        // intentionally left blank
    }
	
    public List<String> getMatchCreated() {
		return matchCreated;
	}

	public List<String> getMatchCancelled() {
		return matchCancelled;
	}

	public List<String> getBeforeMatchFired() {
		return beforeMatchFired;
	}

	public List<String> getAfterMatchFired() {
		return afterMatchFired;
	}

	public List<String> getAgendaGroupPopped() {
		return agendaGroupPopped;
	}

	public List<String> getAgendaGroupPushed() {
		return agendaGroupPushed;
	}

	public List<String> getBeforeRuleFlowGroupActivated() {
		return beforeRuleFlowGroupActivated;
	}

	public List<String> getAfterRuleFlowGroupActivated() {
		return afterRuleFlowGroupActivated;
	}

	public List<String> getBeforeRuleFlowGroupDeactivated() {
		return beforeRuleFlowGroupDeactivated;
	}

	public List<String> getAfterRuleFlowGroupDeactivated() {
		return afterRuleFlowGroupDeactivated;
	}

    public void matchCreated(MatchCreatedEvent event) {
        matchCreated.add(event.getMatch().getRule().getName());
    }

    public void matchCancelled(MatchCancelledEvent event) {
    	matchCancelled.add(event.getMatch().getRule().getName());
    }

    public void beforeMatchFired(BeforeMatchFiredEvent event) {
    	beforeMatchFired.add(event.getMatch().getRule().getName());
    }

    public void afterMatchFired(AfterMatchFiredEvent event) {
    	afterMatchFired.add(event.getMatch().getRule().getName());
    }

    public void agendaGroupPopped(AgendaGroupPoppedEvent event) {
    	agendaGroupPopped.add(event.getAgendaGroup().getName());
    }

    public void agendaGroupPushed(AgendaGroupPushedEvent event) {
    	agendaGroupPushed.add(event.getAgendaGroup().getName());
    }

    public void beforeRuleFlowGroupActivated(RuleFlowGroupActivatedEvent event) {
    	beforeRuleFlowGroupActivated.add(event.getRuleFlowGroup().getName());
    }

    public void afterRuleFlowGroupActivated(RuleFlowGroupActivatedEvent event) {
    	afterRuleFlowGroupActivated.add(event.getRuleFlowGroup().getName());
    }

    public void beforeRuleFlowGroupDeactivated(RuleFlowGroupDeactivatedEvent event) {
    	beforeRuleFlowGroupDeactivated.add(event.getRuleFlowGroup().getName());
    }

    public void afterRuleFlowGroupDeactivated(RuleFlowGroupDeactivatedEvent event) {
    	afterRuleFlowGroupDeactivated.add(event.getRuleFlowGroup().getName());
    }
    
    public void resetAllEvents() {
    	matchCreated = new ArrayList<>();
    	matchCancelled = new ArrayList<>();
    	beforeMatchFired = new ArrayList<>();
    	afterMatchFired = new ArrayList<>();
    	agendaGroupPopped = new ArrayList<>();
    	agendaGroupPushed = new ArrayList<>();
    	beforeRuleFlowGroupActivated = new ArrayList<>();
    	afterRuleFlowGroupActivated = new ArrayList<>();
    	beforeRuleFlowGroupDeactivated = new ArrayList<>();
    	afterRuleFlowGroupDeactivated = new ArrayList<>();    }
}
```

- [ ] **Step 4: Compile**

Run:
```bash
mvn -pl drlx-parser-core test-compile -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: BUILD SUCCESS. The class is unused at this point but must compile cleanly.

- [ ] **Step 5: Run full test suite (baseline still passes)**

Run:
```bash
mvn -pl drlx-parser-core test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 96 tests run, 0 failures, 0 errors.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/core/event/TrackingAgendaEventListener.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: vendor TrackingAgendaEventListener from Drools 10.2.0-rc2

Copy upstream class verbatim into src/test under its upstream package
(org.drools.core.event). Class is unused in this commit — subsequent
commits wire it into DrlxBuilderTestSupport and migrate call sites.

When drlx-parser upgrades to Drools 10.2.0, deleting this file is a
single rm — the FQCN resolves from drools-core with zero import changes.

Refs #18
EOF
)"
```

---

## Task 2: Change `withSession` signature + update all 13 syntax-package callers (logic unchanged)

This task is a single atomic commit because the signature change breaks compilation of all 13 `withSession` callers simultaneously. Each caller only changes its lambda header from `kieSession ->` to `(kieSession, listener) ->`; the listener parameter is ignored in this commit. Existing anon listeners stay in place. Assertions stay identical. Result: tests compile and pass unchanged; the scaffold is ready for per-file migration.

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/DrlxBuilderTestSupport.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AndTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/InlineCastTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NestedGroupTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotExistsBindingTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrBindingScopeTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PassivePatternTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PositionalTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java`

- [ ] **Step 1: Modify `DrlxBuilderTestSupport`**

Replace its full content with:

```java
package org.drools.drlx.builder.syntax;

import java.util.function.BiConsumer;

import org.drools.core.event.TrackingAgendaEventListener;
import org.drools.drlx.builder.DrlxRuleBuilder;
import org.junit.jupiter.api.condition.DisabledIfSystemProperty;
import org.kie.api.KieBase;
import org.kie.api.runtime.KieSession;

@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")
abstract class DrlxBuilderTestSupport {

    protected static void withSession(final String rule,
                                      final BiConsumer<KieSession, TrackingAgendaEventListener> test) {
        final KieBase kieBase = new DrlxRuleBuilder().build(rule);
        final KieSession kieSession = kieBase.newKieSession();
        final TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);
        try {
            test.accept(kieSession, listener);
        } finally {
            kieSession.dispose();
        }
    }

    protected static DrlxRuleBuilder newBuilder() {
        return new DrlxRuleBuilder();
    }
}
```

- [ ] **Step 2: Rename lambda headers in all 13 callers (mechanical)**

In every `.java` file in `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/`, replace each occurrence of:
```java
withSession(rule, kieSession -> {
```
with:
```java
withSession(rule, (kieSession, listener) -> {
```

And each occurrence of:
```java
withSession(rule, ks -> { /* never runs */ }))
```
with:
```java
withSession(rule, (ks, listener) -> { /* never runs */ }))
```

And each occurrence of:
```java
withSession(rule, kieSession -> { /* unreachable */ })
```
with:
```java
withSession(rule, (kieSession, listener) -> { /* unreachable */ })
```

Do NOT remove the existing anonymous `DefaultAgendaEventListener` blocks or change any assertion — those are upgraded per-file in Tasks 3–12.

Run this grep to confirm no stale `kieSession ->` lambda headers remain:
```bash
grep -rn "withSession(rule, kieSession ->\|withSession(rule, ks ->" \
  /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test/java/
```
Expected: empty output.

- [ ] **Step 3: Compile**

Run:
```bash
mvn -pl drlx-parser-core test-compile -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: BUILD SUCCESS. If any file fails to compile, the grep in Step 2 missed a caller — find it via the compile error and rename.

- [ ] **Step 4: Run full test suite (should pass unchanged)**

Run:
```bash
mvn -pl drlx-parser-core test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 96 tests run, 0 failures, 0 errors. Behavior is unchanged — every test still uses its own anon listener; the new `listener` parameter is ignored.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: change withSession to pass TrackingAgendaEventListener to lambda

DrlxBuilderTestSupport.withSession signature changes from
Consumer<KieSession> to BiConsumer<KieSession, TrackingAgendaEventListener>.
The listener is always attached, always passed through. All 13 callers
update their lambda headers; anon listeners and assertions stay in place
for now — per-file assertion upgrades follow in subsequent commits.

Refs #18
EOF
)"
```

---

## Task 3: Migrate `BindingInGroupTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java`

Apply the Migration Recipe to all 4 test methods in this file. Specifics per method:

| Method | Expected `containsExactly(...)` |
|--------|----------------------------------|
| `andBindingVisibleToRightSibling` | `"JoinByName"` |
| `andBindingExplicitType` | `"JoinByNameExplicitType"` |
| `andBothChildrenBound` | `"BothBound"` |
| `bareOopathStillWorks` | `"BareOopathsInAnd"` |

Each test fires exactly one rule once.

- [ ] **Step 1: Read the file to identify all rule names**

```bash
grep -n "rule [A-Z]" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java
```
Note the rule name(s) used by each `@Test` method (one rule per method, identified by its `rule XYZ {` header).

- [ ] **Step 2: Remove the obsolete imports**

Remove these three imports from the top of the file:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```

- [ ] **Step 3: For each test method in the file, apply the Migration Recipe transform**

In each `@Test` method:
1. Delete the `final List<String> fired = new ArrayList<>();` line.
2. Delete the full `kieSession.addEventListener(new DefaultAgendaEventListener() { ... });` block.
3. Replace `assertThat(fired).hasSize(N)` with `assertThat(listener.getAfterMatchFired()).containsExactly("<RuleName>")` — where `<RuleName>` is the rule named in that method's DSL. Each test in this file fires exactly one rule once.

Example — `andBindingVisibleToRightSibling` before/after:

Before (lines 36–54):
```java
withSession(rule, (kieSession, listener) -> {
    final List<String> fired = new ArrayList<>();
    kieSession.addEventListener(new DefaultAgendaEventListener() {
        @Override
        public void afterMatchFired(AfterMatchFiredEvent event) {
            fired.add(event.getMatch().getRule().getName());
        }
    });

    final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
    final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

    persons1.insert(new Person("Alice", 30));
    persons1.insert(new Person("Bob", 40));
    persons2.insert(new Person("Alice", 25));
    persons2.insert(new Person("Carol", 50));

    assertThat(kieSession.fireAllRules()).isEqualTo(1);
    assertThat(fired).hasSize(1);
});
```

After:
```java
withSession(rule, (kieSession, listener) -> {
    final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
    final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

    persons1.insert(new Person("Alice", 30));
    persons1.insert(new Person("Bob", 40));
    persons2.insert(new Person("Alice", 25));
    persons2.insert(new Person("Carol", 50));

    assertThat(kieSession.fireAllRules()).isEqualTo(1);
    assertThat(listener.getAfterMatchFired()).containsExactly("JoinByName");
});
```

Apply the same transformation to every other `@Test` method in the file, using that method's rule name.

- [ ] **Step 4: Run only this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=BindingInGroupTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: all tests in the file pass.

**If a test fails on `containsExactly("Foo")`:** the rule name you used doesn't match what Drools reports. Copy the actual name from the assertion diff and update.

- [ ] **Step 5: Run the full suite to check for collateral damage**

```bash
mvn -pl drlx-parser-core test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 96 passing.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate BindingInGroupTest to TrackingAgendaEventListener

Drop the anonymous DefaultAgendaEventListener boilerplate; upgrade
assertions from hasSize(N) to containsExactly(\"RuleName\") for strict
rule-name verification.

Refs #18
EOF
)"
```

---

## Task 4: Migrate `NotExistsBindingTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotExistsBindingTest.java`

Two test methods, both fire exactly once per test:

| Method | `containsExactly(...)` |
|--------|--------------------------|
| `notBindingWithinGroup` | `"NoJoinExists"` |
| `existsBindingWithinGroup` | `"JoinExists"` |

- [ ] **Step 1: Remove obsolete imports**

Delete:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```

- [ ] **Step 2: Apply the Migration Recipe to `notBindingWithinGroup`**

Replace lines 36–53 with:

```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            // Alice in persons1, no Alice in persons2 → no join exists → fires.
            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Bob", 40));

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("NoJoinExists");
        });
```

- [ ] **Step 3: Apply the Migration Recipe to `existsBindingWithinGroup`**

Replace lines 74–92 with:

```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            // Alice in both → join exists → fires.
            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Alice", 25));

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("JoinExists");
        });
```

- [ ] **Step 4: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=NotExistsBindingTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 2 passing.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotExistsBindingTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate NotExistsBindingTest to TrackingAgendaEventListener

Refs #18
EOF
)"
```

---

## Task 5: Migrate `OrBindingScopeTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrBindingScopeTest.java`

Three test methods; the third uses `assertThatThrownBy` and has no assertion to upgrade (only its lambda header was changed in Task 2).

| Method | `containsExactly(...)` |
|--------|--------------------------|
| `orBranchBindingLocal` | `"BranchLocalJoin", "BranchLocalJoin"` (fires twice — one per OR branch expanded by LogicTransformer) |
| `orDirectBindingBranchLocal` | `"DirectBranchLocal", "DirectBranchLocal"` (same — two OR branches expand) |
| `orBindingNotVisibleAfterGroup` | N/A — the test asserts build-time exception; no listener assertion to change |

**Note on OR-expansion rule names:** Drools' `LogicTransformer` splits top-level OR into two internal rule instances that both carry the ORIGINAL rule name. The listener therefore records the same name twice. If this assumption is wrong, the test will fail with a diff like `expecting ["BranchLocalJoin_0", "BranchLocalJoin_1"] but was ["BranchLocalJoin", "BranchLocalJoin"]` or similar — copy actuals from the diff and update.

- [ ] **Step 1: Remove obsolete imports**

Delete:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```
Keep `assertThatThrownBy` import — still used by `orBindingNotVisibleAfterGroup`.

- [ ] **Step 2: Apply Migration Recipe to `orBranchBindingLocal` (lines 39–62)**

Replace the `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");
            final EntryPoint persons3 = kieSession.getEntryPoint("persons3");

            // First branch satisfied: Alice in persons1 and persons2.
            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Alice", 25));
            // Second branch satisfied: Bob in persons2 and persons3.
            persons2.insert(new Person("Bob", 40));
            persons3.insert(new Person("Bob", 50));

            // Each branch fires once (Drools OR-expansion).
            assertThat(kieSession.fireAllRules()).isEqualTo(2);
            assertThat(listener.getAfterMatchFired()).containsExactly("BranchLocalJoin", "BranchLocalJoin");
        });
```

- [ ] **Step 3: Apply Migration Recipe to `orDirectBindingBranchLocal` (lines 84–102)**

Replace the `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Bob", 40));

            // Each branch fires once — both matched.
            assertThat(kieSession.fireAllRules()).isEqualTo(2);
            assertThat(listener.getAfterMatchFired()).containsExactly("DirectBranchLocal", "DirectBranchLocal");
        });
```

- [ ] **Step 4: Leave `orBindingNotVisibleAfterGroup` alone**

Its lambda header was already updated in Task 2. The `assertThatThrownBy` assertion does not use the listener.

- [ ] **Step 5: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=OrBindingScopeTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 3 passing. **If one of the two `containsExactly` assertions fails:** copy the actual rule names from the diff and update the `containsExactly` argument.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrBindingScopeTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate OrBindingScopeTest to TrackingAgendaEventListener

Refs #18
EOF
)"
```

---

## Task 6: Migrate `RuleAnnotationsTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

Only one method in this file uses `withSession` + a listener: `testSalienceAffectsFiringOrder`. All other methods use `newBuilder().build(rule)` directly and are untouched.

| Method | `containsExactly(...)` |
|--------|--------------------------|
| `testSalienceAffectsFiringOrder` | `"HighSalience", "LowSalience"` (order matters — higher salience fires first) |

- [ ] **Step 1: Remove obsolete imports**

Delete:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```
Keep all other imports — the remaining tests still use `Rule`, `KieBase`, `assertThatThrownBy`, etc.

- [ ] **Step 2: Apply Migration Recipe to `testSalienceAffectsFiringOrder` (lines 46–62)**

Replace the full `withSession` body with:

```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
            entryPoint.insert(new Person("Alice", 30));

            final int firedCount = kieSession.fireAllRules();

            assertThat(firedCount).isEqualTo(2);
            assertThat(listener.getAfterMatchFired()).containsExactly("HighSalience", "LowSalience");
        });
```

This test already used `containsExactly("HighSalience", "LowSalience")` on the local `fired` list — just rename the subject.

- [ ] **Step 3: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=RuleAnnotationsTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate RuleAnnotationsTest to TrackingAgendaEventListener

Refs #18
EOF
)"
```

---

## Task 7: Migrate `OrTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrTest.java`

Four test methods. One (`orEmpty_failsParse`) uses `assertThatThrownBy` — no assertion changes. Three use the listener pattern.

| Method | Notes | Final `listener.getAfterMatchFired()` |
|--------|-------|----------------------------------------|
| `orFirstBranchFires` | multi-fire (isZero then isEqualTo(1)); one rule fires once | `containsExactly("SeniorOrJunior")` |
| `orBothBranchesFire` | one fire cycle, both OR branches match | `containsExactly("SeniorOrJunior", "SeniorOrJunior")` (see OR-expansion note in Task 5) |
| `orBlocksWhenNeither` | one fire cycle, zero matches | `isEmpty()` |

- [ ] **Step 1: Remove obsolete imports**

Delete:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```

- [ ] **Step 2: Apply Migration Recipe to `orFirstBranchFires` (lines 35–51)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint seniors = kieSession.getEntryPoint("seniors");

            assertThat(kieSession.fireAllRules()).isZero();

            seniors.insert(new Person("Grandpa", 75));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("SeniorOrJunior");
        });
```

- [ ] **Step 3: Apply Migration Recipe to `orBothBranchesFire` (lines 73–93)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint seniors = kieSession.getEntryPoint("seniors");
            final EntryPoint juniors = kieSession.getEntryPoint("juniors");

            seniors.insert(new Person("Grandpa", 75));
            juniors.insert(new Person("Kid", 10));

            // Two firings — one per branch — since LogicTransformer splits OR
            // into two rule instances. Each instance's conditions are
            // satisfied once by the single matching fact on its branch.
            assertThat(kieSession.fireAllRules()).isEqualTo(2);
            assertThat(listener.getAfterMatchFired()).containsExactly("SeniorOrJunior", "SeniorOrJunior");
        });
```

- [ ] **Step 4: Apply Migration Recipe to `orBlocksWhenNeither` (lines 113–131)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint seniors = kieSession.getEntryPoint("seniors");
            final EntryPoint juniors = kieSession.getEntryPoint("juniors");

            // Middle-aged → neither branch.
            seniors.insert(new Person("Middle1", 40));
            juniors.insert(new Person("Middle2", 40));

            assertThat(kieSession.fireAllRules()).isZero();
            assertThat(listener.getAfterMatchFired()).isEmpty();
        });
```

- [ ] **Step 5: Leave `orEmpty_failsParse` alone**

Its lambda header was already updated in Task 2.

- [ ] **Step 6: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=OrTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 4 passing. **If `orBothBranchesFire` fails on the double-name assertion:** copy the actual names from the diff and update.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate OrTest to TrackingAgendaEventListener

Refs #18
EOF
)"
```

---

## Task 8: Migrate `ExistsTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java`

Five test methods. Two use `assertThatThrownBy` — no assertion changes. Three use the listener, and each has a multi-fire pattern (isZero → insert → isZero → insert → isEqualTo(1)).

| Method | Final `listener.getAfterMatchFired()` |
|--------|----------------------------------------|
| `existsAllowsMatch` | `containsExactly("HasAdult")` |
| `existsWithOuterBinding` | `containsExactly("ConfirmedPerson")` |
| `existsMultiElement_crossProduct` | `containsExactly("HasAdultWithHighValueOrder")` |

These are already using `containsExactly("RuleName")` on the local `fired` list in some places. Migration is mainly boilerplate removal + subject rename.

- [ ] **Step 1: Remove obsolete imports**

Delete:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```

- [ ] **Step 2: Apply Migration Recipe to `existsAllowsMatch` (lines 36–59)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            // No facts → EXISTS unsatisfied, rule does not fire.
            assertThat(kieSession.fireAllRules()).isZero();
            assertThat(listener.getAfterMatchFired()).isEmpty();

            // Insert a child → still no adult, EXISTS unsatisfied.
            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isZero();

            // Insert an adult → EXISTS satisfied, rule fires once.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("HasAdult");
        });
```

- [ ] **Step 3: Apply Migration Recipe to `existsWithOuterBinding` (lines 88–109)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");
            final EntryPoint orders = kieSession.getEntryPoint("orders");

            // Person but no order → EXISTS unsatisfied, no firing.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isZero();
            assertThat(listener.getAfterMatchFired()).isEmpty();

            // Add a matching order (customerId == Alice.age = 30) → fires.
            orders.insert(new Order("O1", 30, 100));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("ConfirmedPerson");
        });
```

- [ ] **Step 4: Apply Migration Recipe to `existsMultiElement_crossProduct` (lines 133–157)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");
            final EntryPoint orders = kieSession.getEntryPoint("orders");

            // Neither side → EXISTS unsatisfied, no firing.
            assertThat(kieSession.fireAllRules()).isZero();

            // Only adult → EXISTS still unsatisfied (missing order side).
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isZero();

            // Add a high-value order → both sides match → EXISTS satisfied,
            // rule fires once.
            orders.insert(new Order("O1", 99, 5000));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("HasAdultWithHighValueOrder");
        });
```

- [ ] **Step 5: Leave the two `_failsParse` methods alone**

Their lambda headers were already updated in Task 2.

- [ ] **Step 6: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=ExistsTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 5 passing.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate ExistsTest to TrackingAgendaEventListener

Refs #18
EOF
)"
```

---

## Task 9: Migrate `PassivePatternTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PassivePatternTest.java`

Three test methods, all use the listener. Passive-pattern semantics: rule must NOT fire when only the passive side is inserted; MUST fire when the reactive side is inserted.

| Method | Final `listener.getAfterMatchFired()` |
|--------|----------------------------------------|
| `passiveSideInsertionDoesNotWakeRule` | `isEmpty()` (rule never fires) |
| `reactiveSideInsertionWakesRuleIncludingPendingPassiveMatches` | `containsExactly("R1")` (one fire) |
| `barePassivePatternInsideAndGroup` | `isEmpty()` (rule never fires) |

- [ ] **Step 1: Remove obsolete imports**

Delete:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```

- [ ] **Step 2: Apply Migration Recipe to `passiveSideInsertionDoesNotWakeRule` (lines 49–71)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint locations = kieSession.getEntryPoint("locations");
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            // 1. Reactive-side data first. No person yet — no match.
            locations.insert(new Location("paris", "centre"));
            assertThat(kieSession.fireAllRules()).isEqualTo(0);

            // 2. Passive-side insertion alone. A match now exists in the
            //    right-side memory (location × person), but because the
            //    pattern is passive, this insertion MUST NOT wake the rule.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(0);
            assertThat(listener.getAfterMatchFired()).isEmpty();
        });
```

- [ ] **Step 3: Apply Migration Recipe to `reactiveSideInsertionWakesRuleIncludingPendingPassiveMatches` (lines 95–116)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint locations = kieSession.getEntryPoint("locations");
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            // Insert passive-side first — match is pending in memory
            // but rule does not wake.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(0);

            // Reactive-side insertion wakes the rule — one match fires.
            locations.insert(new Location("paris", "centre"));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
```

- [ ] **Step 4: Apply Migration Recipe to `barePassivePatternInsideAndGroup` (lines 138–157)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint locations = kieSession.getEntryPoint("locations");
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            locations.insert(new Location("paris", "centre"));
            assertThat(kieSession.fireAllRules()).isEqualTo(0);

            // Passive-side insertion — must not wake.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(0);
            assertThat(listener.getAfterMatchFired()).isEmpty();
        });
```

- [ ] **Step 5: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=PassivePatternTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 3 passing.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PassivePatternTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate PassivePatternTest to TrackingAgendaEventListener

Refs #18
EOF
)"
```

---

## Task 10: Migrate `AndTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AndTest.java`

Four test methods. Two use `assertThatThrownBy` — no changes. Two use the listener, both already use `containsExactly("RuleName")` on the local `fired` list — migration is mostly subject rename.

| Method | Final `listener.getAfterMatchFired()` |
|--------|----------------------------------------|
| `andAllowsMatch` | `containsExactly("AdultAndBob")` |
| `andSingleChild` | `containsExactly("SingleChildAnd")` |

Both are multi-fire-boundary tests (isZero → insert → isZero → insert → isEqualTo(1)). Listener accumulates; only the final cumulative state matters.

- [ ] **Step 1: Remove obsolete imports**

Delete:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```

- [ ] **Step 2: Apply Migration Recipe to `andAllowsMatch` (lines 36–58)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            assertThat(kieSession.fireAllRules()).isZero();

            // Adult only → AND unsatisfied (no Bob yet).
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isZero();

            // Add Bob (child) → Alice is adult and Bob matches name=="Bob".
            // AND satisfied; rule fires.
            persons.insert(new Person("Bob", 10));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("AdultAndBob");
        });
```

- [ ] **Step 3: Apply Migration Recipe to `andSingleChild` (lines 80–96)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            assertThat(kieSession.fireAllRules()).isZero();

            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("SingleChildAnd");
        });
```

- [ ] **Step 4: Leave the two `_failsParse` methods alone**

Their lambda headers were already updated in Task 2.

- [ ] **Step 5: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=AndTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 4 passing.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AndTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate AndTest to TrackingAgendaEventListener

Refs #18
EOF
)"
```

---

## Task 11: Migrate `NotTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

Six test methods. Two are not `withSession`-with-listener:
- `protoRoundTrip_withNot` — no listener, stays untouched.
- `notWithInnerBinding_failsParse` — `assertThatThrownBy`, already updated in Task 2.

Four use the listener with a multi-fire pattern that calls `fired.clear()`. Per the spec: drop `fired.clear()` calls; assert cumulative state at the end.

| Method | Final `listener.getAfterMatchFired()` |
|--------|----------------------------------------|
| `notSuppressesMatch` | `containsExactly("OnlyAdults")` (fires once on cycle 1, cycle 2 fires nothing) |
| `notWithOuterBinding` | `containsExactly("OrphanedPerson")` (fires once on cycle 1, cycle 2 fires nothing) |
| `notParens_singleElement` | `containsExactly("OnlyAdultsParen")` (fires once; cycle 2 fires nothing) |
| `notMultiElement_crossProduct` | `containsExactly("NoUnderageHighValuePair")` (fires once; 3 subsequent cycles fire nothing) |

**Special care — the `fired.clear()` pattern:** Current tests use:
```java
assertThat(fired).containsExactly("X");
fired.clear();
// ... more inserts ...
assertThat(fired).isEmpty();
```

After migration, the listener accumulates. Drop `fired.clear()` and use one final `containsExactly` at the end covering the CUMULATIVE fires. Intermediate `fireAllRules()` calls keep their `.isZero()` / `.isEqualTo(N)` assertions unchanged.

This means the intermediate `assertThat(fired).isEmpty()` after a clear + zero-fire is DROPPED entirely — `fireAllRules() == 0` already confirms nothing new fired.

- [ ] **Step 1: Remove only the anon-listener imports**

Delete:
```java
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```
**Keep** `import java.util.ArrayList;` and `import java.util.List;` — the `protoRoundTrip_withNot` test still uses `List.of(...)`.

- [ ] **Step 2: Apply Migration Recipe to `notSuppressesMatch` (lines 43–64)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            // No under-18 → rule fires once.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("OnlyAdults");

            // Insert an under-18 → the NOT becomes unsatisfied; no further firings.
            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isZero();
            assertThat(listener.getAfterMatchFired()).containsExactly("OnlyAdults");
        });
```

Note: dropped `fired.clear()`; the cumulative assertion is identical before and after the second cycle because nothing new fired.

- [ ] **Step 3: Apply Migration Recipe to `notWithOuterBinding` (lines 92–111)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");
            final EntryPoint orders = kieSession.getEntryPoint("orders");

            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("OrphanedPerson");

            orders.insert(new Order("O1", 30, 100));
            assertThat(kieSession.fireAllRules()).isZero();
        });
```

- [ ] **Step 4: Apply Migration Recipe to `notParens_singleElement` (lines 184–203)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("OnlyAdultsParen");

            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isZero();
        });
```

- [ ] **Step 5: Apply Migration Recipe to `notMultiElement_crossProduct` (lines 225–253)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");
            final EntryPoint orders = kieSession.getEntryPoint("orders");

            // Neither side present → NOT satisfied, rule fires once.
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("NoUnderageHighValuePair");

            // Only under-18 person → NOT still satisfied (no order match), no new firing.
            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isZero();

            // Add high-value order → both sides match, NOT unsatisfied, still no firing.
            orders.insert(new Order("O1", 99, 5000));
            assertThat(kieSession.fireAllRules()).isZero();

            // Add orphan adult person — doesn't affect NOT state.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isZero();
        });
```

- [ ] **Step 6: Leave `protoRoundTrip_withNot` and `notWithInnerBinding_failsParse` alone**

- [ ] **Step 7: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=NotTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 6 passing.

- [ ] **Step 8: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate NotTest to TrackingAgendaEventListener

Multi-fire tests drop fired.clear() and assert cumulative state
on listener.getAfterMatchFired() at each cycle boundary.

Refs #18
EOF
)"
```

---

## Task 12: Migrate `NestedGroupTest`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NestedGroupTest.java`

Four test methods, all use the listener. Mix of single-fire and multi-fire.

| Method | Final `listener.getAfterMatchFired()` |
|--------|----------------------------------------|
| `orOfAnds` | `containsExactly("OrOfAnds")` (fires once) |
| `andContainingNot` | `containsExactly("AdultWithoutBob")` (fires once in cycle 1; cycle 2 fires nothing) |
| `notContainingOr` | `containsExactly("NoAliceNoBob")` (fires once in cycle 1; cycles 2-3 fire nothing) |
| `andWithNestedNotCarryingBinding` | `containsExactly("NoSameNameInOther")` (fires once) |

- [ ] **Step 1: Remove obsolete imports**

Delete:
```java
import java.util.ArrayList;
import java.util.List;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
```

- [ ] **Step 2: Apply Migration Recipe to `orOfAnds` (lines 37–57)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            // First branch satisfied: adult in persons1, young adult in persons2.
            persons1.insert(new Person("Alice", 25));
            persons2.insert(new Person("Bob", 22));

            // Second branch unsatisfied (no senior in persons1, no kid in persons3).
            // First branch fires once.
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("OrOfAnds");
        });
```

- [ ] **Step 3: Apply Migration Recipe to `andContainingNot` (lines 79–100)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            // Adult with no Bob → fires.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);

            // Insert Bob → NOT now unsatisfied, rule no longer activates.
            persons.insert(new Person("Bob", 40));
            assertThat(kieSession.fireAllRules()).isZero();

            // Still one total firing.
            assertThat(listener.getAfterMatchFired()).containsExactly("AdultWithoutBob");
        });
```

- [ ] **Step 4: Apply Migration Recipe to `notContainingOr` (lines 122–146)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");

            // Empty — NOT satisfied (vacuously), rule fires once.
            assertThat(kieSession.fireAllRules()).isEqualTo(1);

            // Insert Charlie — still no Alice, no Bob — NOT still satisfied.
            // Match already fired; no new activation.
            persons.insert(new Person("Charlie", 50));
            assertThat(kieSession.fireAllRules()).isZero();

            // Insert Alice — OR becomes satisfied, so NOT is not. No fire.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isZero();

            assertThat(listener.getAfterMatchFired()).containsExactly("NoAliceNoBob");
        });
```

- [ ] **Step 5: Apply Migration Recipe to `andWithNestedNotCarryingBinding` (lines 170–191)**

Replace `withSession` body with:
```java
        withSession(rule, (kieSession, listener) -> {
            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            // Alice and Bob in persons1; only Alice in persons2.
            // → For Bob, no match in persons2 → fires.
            // → For Alice, match in persons2 → does not fire.
            persons1.insert(new Person("Alice", 30));
            persons1.insert(new Person("Bob", 40));
            persons2.insert(new Person("Alice", 25));

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("NoSameNameInOther");
        });
```

- [ ] **Step 6: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=NestedGroupTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 4 passing.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NestedGroupTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate NestedGroupTest to TrackingAgendaEventListener

Refs #18
EOF
)"
```

---

## Task 13: Migrate `DrlxRuleBuilderTest` (inline pattern)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleBuilderTest.java`

This file does NOT extend `DrlxBuilderTestSupport` and does NOT use `withSession`. It constructs `KieSession` directly. Each relevant test gets the inline listener pattern.

Tests to migrate (have multi-rule behavior worth asserting by name):
| Method | Expected rule names |
|--------|-----------------------|
| `testBuildBasicRule` | one rule `CheckAge`; single-rule — **but** per spec's "multi-rule requires listener" policy, this is a single-rule test. We choose to migrate it anyway for consistency with the multi-rule tests in the same file. Final: `containsExactly("CheckAge")` |
| `testLambdaSharing` | two rules `CheckAge1`, `CheckAge2`; multi-rule → MUST migrate. Both fire once. Final: `containsExactlyInAnyOrder("CheckAge1", "CheckAge2")` (order depends on agenda — use `containsExactlyInAnyOrder` to avoid flakiness) |
| `testRuntimeBuildWithPreCompiledClasses` | two rules `CheckAge1`, `CheckAge2`; multi-rule → MUST migrate. Final: `containsExactlyInAnyOrder("CheckAge1", "CheckAge2")` |
| `testFallbackWhenMetadataMissing` | single rule `CheckAge`; single-rule — migrate for consistency. Final: `containsExactly("CheckAge")` |

Tests NOT migrated:
- `testPreBuild` — no `fireAllRules` call; just metadata assertions.

- [ ] **Step 1: Add the import**

Add to the import block:
```java
import org.drools.core.event.TrackingAgendaEventListener;
```

Maintain alphabetical import order (insert after `org.drools.core.reteoo.ReteDumper`).

- [ ] **Step 2: Migrate `testBuildBasicRule` (lines 32–60)**

Replace the test body with:

```java
    @Test
    void testBuildBasicRule() {
        String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule CheckAge {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        DrlxRuleBuilder builder = new DrlxRuleBuilder();
        KieBase kieBase = builder.build(rule);

        KieSession kieSession = kieBase.newKieSession();
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);

        EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        Person person = new Person("John", 25);
        entryPoint.insert(person);
        int fired = kieSession.fireAllRules();

        assertThat(fired).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("CheckAge");

        kieSession.dispose();
    }
```

- [ ] **Step 3: Migrate `testLambdaSharing` (lines 62–123)**

After line `KieSession kieSession = kieBase.newKieSession();` (around line 109), insert:
```java
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);
```

Replace:
```java
        assertThat(fired).isEqualTo(2);
```
with:
```java
        assertThat(fired).isEqualTo(2);
        assertThat(listener.getAfterMatchFired()).containsExactlyInAnyOrder("CheckAge1", "CheckAge2");
```

- [ ] **Step 4: Migrate `testRuntimeBuildWithPreCompiledClasses` (lines 194–238)**

After line `KieSession kieSession = kieBase.newKieSession();` (around line 227), insert:
```java
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);
```

Replace:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(2);
```
with:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(2);
        assertThat(listener.getAfterMatchFired()).containsExactlyInAnyOrder("CheckAge1", "CheckAge2");
```

- [ ] **Step 5: Migrate `testFallbackWhenMetadataMissing` (lines 240–279)**

Inside the `try` block, after line `KieSession kieSession = kieBase.newKieSession();`, insert:
```java
            TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
            kieSession.addEventListener(listener);
```

Replace:
```java
            assertThat(fired).isEqualTo(1);
```
with:
```java
            assertThat(fired).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("CheckAge");
```

- [ ] **Step 6: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=DrlxRuleBuilderTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleBuilderTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate DrlxRuleBuilderTest to TrackingAgendaEventListener

DrlxRuleBuilderTest does not extend DrlxBuilderTestSupport — use the
inline attach pattern: construct, addEventListener, assert.

Refs #18
EOF
)"
```

---

## Task 14: Migrate `DrlxCompilerTest` (inline pattern)

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/tools/DrlxCompilerTest.java`

Six test methods. All use multi-rule or multi-fire patterns. Apply the inline pattern to each.

| Method | Expected rule names |
|--------|-----------------------|
| `testTwoStepBuildWithRuleAstParseResult` | `"JoinRule"` — fires 1× |
| `testTwoStepBuild` | `"CheckAge1", "CheckAge2"` — fires 2× total (order depends on agenda) |
| `testTwoStepBuildWithJoin` | `"JoinRule"` — fires 1× |
| `testTwoStepBuildWithMultiLevelJoin` | `"MultiLevelJoinRule"` — fires 1× initially; then 0 after the second insert |
| `testLambdaSharingInPreBuild` | `"Rule_0", "Rule_1", "Rule_2"` — fires 3× (order depends on agenda) |
| `testBuildWithoutPreBuild` | `"CheckAge"` — fires 1× |

- [ ] **Step 1: Add the import**

Add to the import block:
```java
import org.drools.core.event.TrackingAgendaEventListener;
```

Maintain alphabetical order — insert after `org.drools.drlx.domain.Person`.

- [ ] **Step 2: Migrate `testTwoStepBuildWithRuleAstParseResult` (lines 23–72)**

After line `KieSession kieSession = kieBase.newKieSession();` (around line 57), insert:
```java
            TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
            kieSession.addEventListener(listener);
```

Replace:
```java
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(1);
```
with:
```java
            int fired = kieSession.fireAllRules();
            assertThat(fired).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("JoinRule");
```

- [ ] **Step 3: Migrate `testTwoStepBuild` (lines 74–119)**

After line `KieSession kieSession = kieBase.newKieSession();` (around line 111), insert:
```java
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);
```

Replace:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(2);
```
with:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(2);
        assertThat(listener.getAfterMatchFired()).containsExactlyInAnyOrder("CheckAge1", "CheckAge2");
```

- [ ] **Step 4: Migrate `testTwoStepBuildWithJoin` (lines 121–156)**

After `KieSession kieSession = kieBase.newKieSession();` (around line 148), insert:
```java
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);
```

Replace:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(1);
```
with:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("JoinRule");
```

- [ ] **Step 5: Migrate `testTwoStepBuildWithMultiLevelJoin` (lines 158–202)**

After `KieSession kieSession = kieBase.newKieSession();` (around line 186), insert:
```java
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);
```

Replace:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(1);

        // Insert a person that should NOT match: age 10 > 15 => false
        kieSession.getEntryPoint("persons3").insert(new Person("Dave", 10));
        int fired2 = kieSession.fireAllRules();
        assertThat(fired2).isEqualTo(0);
```
with:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("MultiLevelJoinRule");

        // Insert a person that should NOT match: age 10 > 15 => false
        kieSession.getEntryPoint("persons3").insert(new Person("Dave", 10));
        int fired2 = kieSession.fireAllRules();
        assertThat(fired2).isEqualTo(0);
        assertThat(listener.getAfterMatchFired()).containsExactly("MultiLevelJoinRule");
```

- [ ] **Step 6: Migrate `testLambdaSharingInPreBuild` (lines 204–267)**

After `KieSession kieSession = kieBase.newKieSession();` (around line 258), insert:
```java
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);
```

Replace:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(3);
```
with:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(3);
        assertThat(listener.getAfterMatchFired()).containsExactlyInAnyOrder("Rule_0", "Rule_1", "Rule_2");
```

- [ ] **Step 7: Migrate `testBuildWithoutPreBuild` (lines 269–298)**

After `KieSession kieSession = kieBase.newKieSession();` (around line 291), insert:
```java
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        kieSession.addEventListener(listener);
```

Replace:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(1);
```
with:
```java
        int fired = kieSession.fireAllRules();
        assertThat(fired).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("CheckAge");
```

- [ ] **Step 8: Run this file's tests**

```bash
mvn -pl drlx-parser-core test -Dtest=DrlxCompilerTest -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: all tests pass.

- [ ] **Step 9: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/tools/DrlxCompilerTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: migrate DrlxCompilerTest to TrackingAgendaEventListener

Inline attach pattern. Multi-rule tests use containsExactlyInAnyOrder
(agenda order depends on rule-compilation order, which is an
implementation detail).

Refs #18
EOF
)"
```

---

## Task 15: Final verification

No files modified — this is a full-matrix verification gate.

- [ ] **Step 1: Run default test suite (persistence on)**

```bash
mvn -pl drlx-parser-core test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 96 tests run, 0 failures, 0 errors.

- [ ] **Step 2: Run no-persist test subset**

```bash
mvn -pl drlx-parser-core test -Dmvel3.compiler.lambda.persistence=false -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml
```
Expected: 5 tests run (the ones NOT annotated `@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")`), 0 failures, 0 errors.

- [ ] **Step 3: Grep for leftover anon-listener patterns (should be empty)**

```bash
grep -rn "new DefaultAgendaEventListener" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test/java/
```
Expected: empty output.

```bash
grep -rn "import org.kie.api.event.rule.DefaultAgendaEventListener\|import org.kie.api.event.rule.AfterMatchFiredEvent" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test/java/
```
Expected: empty output.

- [ ] **Step 4: Confirm the 4 stay-as-is files are untouched**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser diff --name-only HEAD~14 HEAD | grep -E "(InlineCastTest|TypeInferenceTest|PositionalTest|DrlxCompilerNoPersistTest)\.java"
```
- `InlineCastTest`, `TypeInferenceTest`, `PositionalTest` should appear (they had lambda-header changes in Task 2, nothing else).
- `DrlxCompilerNoPersistTest` should NOT appear (fully untouched).

- [ ] **Step 5: Confirm test count regression boundary**

```bash
mvn -pl drlx-parser-core test -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml 2>&1 | grep "Tests run"
```
Expected: `Tests run: 96, Failures: 0, Errors: 0, Skipped: 0` (or equivalent — test count must match the baseline of 96, not lower). A reduced count indicates an accidental `@Disabled` or missing `@Test`.

- [ ] **Step 6: No commit**

This task is a verification gate only.

---

## Risks recap (from spec)

- `BiConsumer` signature change will break compilation of any overlooked `withSession` caller — caught by the compiler at Task 2 Step 3, not silent.
- Upgraded `containsExactly("Name")` assertions may surface latent bugs where a test was silently firing the wrong rule. If a migrated test fails with a rule-name mismatch (not a count mismatch), the old `hasSize(N)` was masking the bug — investigate the production code or test fixture, do not weaken the assertion.
- `containsExactlyInAnyOrder` is used in multi-rule tests where agenda order is an implementation detail (salience ties, insertion order). `containsExactly` is used where order is deterministic (salience difference, single rule fired N times).
