# #56 Passive Query Invocation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `?/queryName(...)` set `openQuery=false` on the resulting `QueryElement`, and validate that passive invocation of queries with agenda `do` blocks is rejected at compile time.

**Architecture:** The grammar, visitor, and IR already support the `?` prefix and thread it into `PatternIR.passive`. The only gap is in `DrlxRuleAstRuntimeBuilder.buildQueryElement()` which hardcodes `openQuery=true`. Fix that one value, add a forward guard for queries with consequences, and add tests.

**Tech Stack:** Java 17, ANTLR4, JUnit 5, AssertJ

---

### Task 1: Write failing test for passive query invocation

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Add test method `passiveQueryInvocationDoesNotWakeRule`**

This test mirrors `PassivePatternTest.passiveSideInsertionDoesNotWakeRule` — insert reactive-side data first, then insert passive-side data. A complete match now exists in memory, but because the query invocation is passive, the passive-side insertion must NOT wake the rule.

```java
@Test
void passiveQueryInvocationDoesNotWakeRule() {
    final String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Location;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;

            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                var l : /locations[city == "paris"],
                ?/personsByAge(25, var p),
                do { results.add(p); }
            }
            """;

    withSession(source, (kieSession, listener) -> {
        final EntryPoint locations = kieSession.getEntryPoint("locations");
        final EntryPoint persons = kieSession.getEntryPoint("persons");

        // 1. Reactive side first — no query match yet, no fire
        locations.insert(new Location("paris", "centre"));
        assertThat(kieSession.fireAllRules()).isEqualTo(0);

        // 2. Passive query side — a complete match now exists
        //    (location × query result), but because the query invocation
        //    is passive, this insertion MUST NOT wake R1
        persons.insert(new Person("Alice", 30));
        assertThat(kieSession.fireAllRules()).isEqualTo(0);
        assertThat(listener.getAfterMatchFired()).isEmpty();
    });
}
```

Add the required import at the top of the file:

```java
import org.kie.api.runtime.rule.EntryPoint;
import org.drools.drlx.domain.Location;
```

- [ ] **Step 2: Add test method `passiveQueryInvocationWakesWhenReactiveSideFires`**

Contrast test: inserting on the reactive side DOES wake the rule, picking up pending passive query results.

```java
@Test
void passiveQueryInvocationWakesWhenReactiveSideFires() {
    final String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Location;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;

            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                var l : /locations[city == "paris"],
                ?/personsByAge(25, var p),
                do { results.add(p); }
            }
            """;

    withSession(source, (kieSession, listener) -> {
        final EntryPoint locations = kieSession.getEntryPoint("locations");
        final EntryPoint persons = kieSession.getEntryPoint("persons");

        // Passive side first — no fire
        persons.insert(new Person("Alice", 30));
        assertThat(kieSession.fireAllRules()).isEqualTo(0);

        // Reactive side triggers — picks up pending passive query results
        locations.insert(new Location("paris", "centre"));
        assertThat(kieSession.fireAllRules()).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("R1");
    });
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="QueryTest#passiveQueryInvocationDoesNotWakeRule+passiveQueryInvocationWakesWhenReactiveSideFires" -pl .
```

Expected: FAIL — the `openQuery` is hardcoded to `true`, so the passive query invocation currently behaves as reactive.

### Task 2: Fix `openQuery` and add validation guard

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:955-1014`

- [ ] **Step 1: Add the validation guard before `QueryElement` construction**

In method `buildQueryElement`, insert the validation check before the `return new QueryElement(...)` statement at line 1006. This goes right after the `int[] varIndexArray` line (line 1004):

```java
        int[] varIndexArray = varIndexes.stream().mapToInt(Integer::intValue).toArray();

        if (patternIr.passive() && targetQuery.getConsequence() != null) {
            throw new RuntimeException(
                    "Cannot passively invoke query '" + targetQuery.getName()
                    + "': the query has an agenda-based 'do' block which is incompatible with passive invocation");
        }

        return new QueryElement(
```

- [ ] **Step 2: Change hardcoded `true` to `!patternIr.passive()`**

In the `QueryElement` constructor call, change the 6th argument from `true` to `!patternIr.passive()`:

Old (line 1012):
```java
                true,
```

New:
```java
                !patternIr.passive(),
```

- [ ] **Step 3: Install the modified module and run the tests**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -DskipTests
```

Then run the new tests:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="QueryTest#passiveQueryInvocationDoesNotWakeRule+passiveQueryInvocationWakesWhenReactiveSideFires" -pl .
```

Expected: PASS

### Task 3: Run full test suite and commit

- [ ] **Step 1: Run the full drlx-parser test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test
```

Expected: All tests pass (no regressions).

- [ ] **Step 2: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(query): support passive query invocation (?/queryName)

Thread PatternIR.passive flag into QueryElement construction so that
?/ prefix sets openQuery=false. Add compile-time guard rejecting passive
invocation of queries with agenda-based do blocks.

Closes #56"
```
