# Assignment Hint in Constraints — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a constraint compilation fails with a boolean type mismatch, append a hint suggesting `=` may have been used where `==` was intended.

**Architecture:** Catch `KieMemoryCompilerException` in `DrlxLambdaCompiler.compileBatch()`, check the error message for boolean type mismatch indicators, and re-throw with an appended hint.

**Tech Stack:** Java 17, JUnit 5, AssertJ

## Global Constraints

- Refs #94
- Change is in drlx-parser-core module only
- Follow existing error handling pattern: throw `RuntimeException` wrapping the original exception

---

### Task 1: Add hint to compileBatch error path and test

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java:310-319`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ConstraintAssignmentHintTest.java`

- [ ] **Step 1: Write the failing test**

Create `ConstraintAssignmentHintTest.java`:

```java
package org.drools.drlx.builder.syntax;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ConstraintAssignmentHintTest extends DrlxBuilderTestSupport {

    @Test
    void assignmentInConstraintShowsHint() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule AssignmentInConstraint {
                    var p : /persons1[ age = 25 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> withMyUnitInstance(rule, (instance, unit, listener) -> { /* never runs */ }))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("did you mean '=='");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=ConstraintAssignmentHintTest -pl . -am`

Expected: FAIL — exception is thrown but message does not contain "did you mean '=='"

- [ ] **Step 3: Implement the hint in compileBatch()**

In `DrlxLambdaCompiler.java`, add import and modify `compileBatch()`:

```java
import org.mvel3.javacompiler.KieMemoryCompilerException;
```

Replace the `compileBatch()` method body:

```java
    public void compileBatch(ClassLoader classLoader) {
        if (pendingLambdas.isEmpty()) {
            return;
        }
        try {
            batchCompiler.compile(classLoader);
        } catch (KieMemoryCompilerException e) {
            String msg = e.getMessage();
            if (msg != null && msg.contains("incompatible types") && msg.contains("boolean")) {
                throw new KieMemoryCompilerException(
                        msg + "\nHint: a constraint expression contains '=' — did you mean '=='?", e);
            }
            throw e;
        }
        for (PendingLambda pl : pendingLambdas) {
            pl.target().bindEvaluator(batchCompiler.resolve(pl.handle()));
        }
        pendingLambdas.clear();
    }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=ConstraintAssignmentHintTest -pl . -am`

Expected: PASS

- [ ] **Step 5: Run the full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`

Expected: All tests pass (485+)

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ConstraintAssignmentHintTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: hint message when = used in constraint where == intended

Refs #94"
```
