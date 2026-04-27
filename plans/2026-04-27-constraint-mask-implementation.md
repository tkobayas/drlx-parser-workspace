# Constraint Property-Reactive Mask Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `DrlxLambdaConstraint.getListenedPropertyMask` return a property-specific BitMask so that the alpha-node listened mask reflects actual constraint reads ∪ watch list, instead of the AllSetButLastBitMask default that defeats the watch list from #13.

**Architecture:** Two-repo change.

1. **MVEL3** (`mvel3-development/mvel`, branch `mvel3-read-props`): expose `String[] getReadProperties()` on the compiled `Evaluator`. Populated during the existing transpiler AST walk in `VariableAnalyser`, surfaced via a generated method on the evaluator class.
2. **DRLX** (`mvel3-development/drlx-parser`, new branch `19-constraint-mask`): consume MVEL3's read-properties to override `getListenedPropertyMask`, mirroring `drools-modelcompiler.LambdaConstraint:170-188`.

**Tech Stack:** Java 17, MVEL3 transpiler, JavaParser AST, Drools 10.1.0 base API, JUnit 5, AssertJ.

**Spec:** `specs/2026-04-27-constraint-mask-design.md`

**Branch convention:**
- MVEL3: `mvel3-read-props` (already set)
- DRLX: `19-constraint-mask` (create at start of Phase 2)

**Issue tracking:**
- DRLX commits: `Refs #19` (Phase 2 final commit: `Closes #19`)
- MVEL3 commits: `Refs mvel/mvel#423` (use the fully-qualified form because `tkobayas/mvel` (origin) has issues disabled — the bare `#423` would only resolve on `mvel/mvel`)

---

## File Structure

### MVEL3 changes

| File | Change |
|------|--------|
| `mvel/src/main/java/org/mvel3/Evaluator.java` | Add `default String[] getReadProperties()` |
| `mvel/src/main/java/org/mvel3/transpiler/VariableAnalyser.java` | Extend to collect free `NameExpr` and no-arg getter `MethodCallExpr`. Add `getReadProperties()` accessor. Add JavaBeans helper `getter2property`. |
| `mvel/src/main/java/org/mvel3/transpiler/MVELTranspiler.java` | After running the analyser, emit a `getReadProperties()` override on the generated class. |
| `mvel/src/test/java/org/mvel3/transpiler/ReadPropertiesTest.java` (NEW) | End-to-end tests: compile expression, assert `evaluator.getReadProperties()` content. |
| `mvel/src/test/java/org/mvel3/transpiler/ReadPropsFixture.java` (NEW) | Minimal POJO with `salary`, `basePay`, `active` properties. |

### DRLX changes

| File | Change |
|------|--------|
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaConstraint.java` | Override `getListenedPropertyMask` reading `evaluator.getReadProperties()`. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java` | Add 3 condition+watch-list tests. Remove `// MVP scope:` comment block. |

No changes to grammar, IR, proto, or visitor.

---

# Phase 1 — MVEL3

Working directory: `/home/tkobayas/usr/work/mvel3-development/mvel`
Branch: `mvel3-read-props` (already set)
Build/install: `mvn -pl mvel -am install` from repo root (or `mvn install` from `mvel/` directory) — DRLX consumes via `3.0.0-SNAPSHOT`.

---

### Task 1: Add `getReadProperties()` default to `Evaluator` interface

**Files:**
- Modify: `mvel/src/main/java/org/mvel3/Evaluator.java`

- [ ] **Step 1: Add the default method**

Open `Evaluator.java`. Add this default method inside the interface, after `evalWith` (line 79):

```java
    /**
     * Returns the names of properties that the compiled expression reads from
     * its context. Populated only for POJO-context evaluators; empty otherwise.
     * <p>
     * The returned names are <em>candidates</em> derived from AST analysis — bare
     * identifiers and no-arg getter calls. Consumers are expected to filter
     * against the actual settable-property set of the context type.
     *
     * @return property names referenced by the expression, or an empty array
     */
    default String[] getReadProperties() {
        return new String[0];
    }
```

- [ ] **Step 2: Compile and verify**

Run from `mvel/`: `mvn -pl . compile -q`
Expected: build succeeds (no behavioral change yet).

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add src/main/java/org/mvel3/Evaluator.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
add Evaluator.getReadProperties() default returning empty array

Surface a metadata accessor on the compiled Evaluator. Default returns []
so existing consumers see no behavior change. Will be overridden in
generated POJO evaluator classes once VariableAnalyser collects reads
and MVELTranspiler emits the override.

Refs mvel/mvel#423
EOF
)"
```

---

### Task 2: Add fixture POJO for tests

**Files:**
- Create: `mvel/src/test/java/org/mvel3/transpiler/ReadPropsFixture.java`

- [ ] **Step 1: Create the fixture**

```java
package org.mvel3.transpiler;

public class ReadPropsFixture {
    private int salary;
    private int basePay;
    private boolean active;

    public int getSalary() { return salary; }
    public void setSalary(int salary) { this.salary = salary; }

    public int getBasePay() { return basePay; }
    public void setBasePay(int basePay) { this.basePay = basePay; }

    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
}
```

- [ ] **Step 2: Compile (fixture only — no test references yet)**

Run: `mvn -pl . test-compile -q`
Expected: build succeeds.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add src/test/java/org/mvel3/transpiler/ReadPropsFixture.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
add ReadPropsFixture POJO for read-properties tests

Refs mvel/mvel#423
EOF
)"
```

---

### Task 3: Write the first failing read-properties test (bare NameExpr)

**Files:**
- Create: `mvel/src/test/java/org/mvel3/transpiler/ReadPropertiesTest.java`

- [ ] **Step 1: Write the test class with one test**

```java
package org.mvel3.transpiler;

import org.junit.jupiter.api.Test;
import org.mvel3.ClassManager;
import org.mvel3.CompilerParameters;
import org.mvel3.Evaluator;
import org.mvel3.MVEL;

import static org.assertj.core.api.Assertions.assertThat;

class ReadPropertiesTest {

    private Evaluator<Object, Void, Boolean> compile(String expression) {
        CompilerParameters<Object, Void, Boolean> info = MVEL.pojo(ReadPropsFixture.class)
                .<Boolean>out(Boolean.class)
                .expression(expression)
                .classManager(new ClassManager())
                .build();
        return new MVEL().compilePojoEvaluator(info);
    }

    @Test
    void bareNameExpr_collectsProperty() {
        Evaluator<Object, Void, Boolean> ev = compile("salary > 0");
        assertThat(ev.getReadProperties()).containsExactlyInAnyOrder("salary");
    }
}
```

- [ ] **Step 2: Run the test and confirm it fails**

Run: `mvn -pl . test -Dtest=ReadPropertiesTest#bareNameExpr_collectsProperty -q`
Expected: FAIL — assertion fails because `getReadProperties()` returns empty array (the default). Confirms the test is exercising the right surface.

- [ ] **Step 3: Do not commit yet** — no production code change. Move to Task 4.

---

### Task 4: Extend `VariableAnalyser` to collect free `NameExpr`

**Files:**
- Modify: `mvel/src/main/java/org/mvel3/transpiler/VariableAnalyser.java`

- [ ] **Step 1: Add `readProperties` collection and visit logic**

Replace the entire file with:

```java
package org.mvel3.transpiler;
import com.github.javaparser.ast.expr.MethodCallExpr;
import com.github.javaparser.ast.expr.NameExpr;
import org.mvel3.parser.ast.expr.DrlNameExpr;
import org.mvel3.parser.ast.visitor.DrlVoidVisitorAdapter;

import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.Set;

public class VariableAnalyser extends DrlVoidVisitorAdapter<Void> {

    private Set<String> available;
    private Set<String> used = new HashSet<>();

    private Set<String> found = new HashSet<>();

    // Iteration order preserved so that generated arrays are deterministic across builds.
    private Set<String> readProperties = new LinkedHashSet<>();

    public VariableAnalyser(Set<String> available) {
        this.available = available;
    }

    public void visit(NameExpr n, Void arg) {
        String name = n.getNameAsString();
        if (available.contains(name)) {
            used.add(name);
        } else {
            readProperties.add(name);
        }
    }

    public void visit(DrlNameExpr n, Void arg) {
        String name = n.getNameAsString();
        if (available.contains(name)) {
            used.add(name);
        } else {
            found.add(name);
            readProperties.add(name);
        }
    }

    public void visit(MethodCallExpr n, Void arg) {
        if (n.getScope().isEmpty() && n.getArguments().isEmpty()) {
            String prop = getter2property(n.getNameAsString());
            if (prop != null) {
                readProperties.add(prop);
            }
        }
        // Always recurse into children so nested expressions are still visited.
        super.visit(n, arg);
    }

    public Set<String> getUsed() {
        return used;
    }

    public Set<String> getFound() {
        return found;
    }

    public Set<String> getReadProperties() {
        return readProperties;
    }

    /**
     * JavaBeans getter name → property name. Returns null if the name does not
     * follow the {@code getXxx} / {@code isXxx} convention.
     */
    static String getter2property(String methodName) {
        if (methodName.startsWith("get") && methodName.length() > 3) {
            return Character.toLowerCase(methodName.charAt(3)) + methodName.substring(4);
        }
        if (methodName.startsWith("is") && methodName.length() > 2) {
            return Character.toLowerCase(methodName.charAt(2)) + methodName.substring(3);
        }
        return null;
    }
}
```

- [ ] **Step 2: Compile**

Run: `mvn -pl . test-compile -q`
Expected: build succeeds.

---

### Task 5: Wire the analyser output into the generated `Evaluator` class

**Files:**
- Modify: `mvel/src/main/java/org/mvel3/transpiler/MVELTranspiler.java`

- [ ] **Step 1: Add codegen for `getReadProperties()` override**

Find the block in `transpileContent` that runs the analyser (lines ~120-128):

```java
        VariableAnalyser analyser = new VariableAnalyser(context.getEvaluatorInfo().allVars().keySet());
        blockStmt.accept(analyser, null);

        if (!context.getEvaluatorInfo().withDeclaration().type().isVoid() &&
            !context.getEvaluatorInfo().withDeclaration().equals(context.getEvaluatorInfo().contextDeclaration())) {
            analyser.getUsed().add(context.getEvaluatorInfo().withDeclaration().name());
        }

        analyser.getUsed().stream().forEach(v -> context.addInput(v));
```

After `classDeclaration.addImplementedType(implementedType);` (around line 150) and before the `MethodDeclaration method = classDeclaration.addMethod(...)` for the eval method (around line 152), add the read-properties override emission. Place it inline so it runs once per generated class:

```java
        // Emit a getReadProperties() override on the generated class so that
        // the compiled evaluator returns names of properties referenced in the
        // expression (used downstream for property-reactive listening masks).
        emitGetReadPropertiesOverride(classDeclaration, analyser.getReadProperties());
```

Then add the helper method at the end of the class (before the closing brace):

```java
    private static void emitGetReadPropertiesOverride(
            com.github.javaparser.ast.body.ClassOrInterfaceDeclaration classDeclaration,
            java.util.Set<String> readProperties) {
        com.github.javaparser.ast.body.MethodDeclaration m = classDeclaration.addMethod("getReadProperties");
        m.setPublic(true);
        m.setType("String[]");
        m.addAnnotation("Override");
        StringBuilder body = new StringBuilder("return new String[]{");
        boolean first = true;
        for (String p : readProperties) {
            if (!first) body.append(", ");
            body.append('"').append(p.replace("\"", "\\\"")).append('"');
            first = false;
        }
        body.append("};");
        m.setBody(new com.github.javaparser.ast.stmt.BlockStmt()
                .addStatement(body.toString()));
    }
```

(Inline FQNs to avoid adding new imports for a single helper. If the file already imports these classes, swap to short names.)

- [ ] **Step 2: Compile**

Run: `mvn -pl . test-compile -q`
Expected: build succeeds.

- [ ] **Step 3: Run the failing test from Task 3**

Run: `mvn -pl . test -Dtest=ReadPropertiesTest#bareNameExpr_collectsProperty -q`
Expected: PASS.

- [ ] **Step 4: Commit Tasks 3+4+5 together** (test-driven slice)

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add \
    src/main/java/org/mvel3/transpiler/VariableAnalyser.java \
    src/main/java/org/mvel3/transpiler/MVELTranspiler.java \
    src/test/java/org/mvel3/transpiler/ReadPropertiesTest.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
collect read properties via VariableAnalyser, emit on Evaluator

VariableAnalyser now records free NameExpr and no-arg getter
MethodCallExpr names alongside its existing used/found sets.
MVELTranspiler emits a getReadProperties() override on the
generated class so the compiled Evaluator surfaces the names.

Refs mvel/mvel#423
EOF
)"
```

---

### Task 6: Add coverage tests for getter, boolean getter, multi-prop, no-prop

**Files:**
- Modify: `mvel/src/test/java/org/mvel3/transpiler/ReadPropertiesTest.java`

- [ ] **Step 1: Add four more tests**

Add these methods to `ReadPropertiesTest`:

```java
    @Test
    void getterCall_collectsProperty() {
        Evaluator<Object, Void, Boolean> ev = compile("getBasePay() == 5000");
        assertThat(ev.getReadProperties()).containsExactlyInAnyOrder("basePay");
    }

    @Test
    void booleanGetter_collectsProperty() {
        Evaluator<Object, Void, Boolean> ev = compile("isActive()");
        assertThat(ev.getReadProperties()).containsExactlyInAnyOrder("active");
    }

    @Test
    void multipleProperties_collectsAll() {
        Evaluator<Object, Void, Boolean> ev = compile("salary > 0 && basePay == 1000");
        assertThat(ev.getReadProperties()).containsExactlyInAnyOrder("salary", "basePay");
    }

    @Test
    void noProperties_returnsEmpty() {
        Evaluator<Object, Void, Boolean> ev = compile("1 == 1");
        assertThat(ev.getReadProperties()).isEmpty();
    }
```

- [ ] **Step 2: Run the full test class**

Run: `mvn -pl . test -Dtest=ReadPropertiesTest -q`
Expected: all 5 tests PASS.

- [ ] **Step 3: Run full MVEL3 test suite (regression check)**

Run: `mvn -pl . test -q`
Expected: BUILD SUCCESS — no regressions in pre-existing tests. If any pre-existing test fails because it now sees non-empty `getReadProperties()`, investigate; the analyser change should not affect any other code path.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add src/test/java/org/mvel3/transpiler/ReadPropertiesTest.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
test: cover getter, boolean getter, multi-prop, no-prop cases

Refs mvel/mvel#423
EOF
)"
```

---

### Task 7: Install MVEL3 SNAPSHOT locally so DRLX can consume it

- [ ] **Step 1: Run install**

Run: `mvn -pl . install -q -DskipTests` from `/home/tkobayas/usr/work/mvel3-development/mvel`
Expected: BUILD SUCCESS. Artifact `mvel3:3.0.0-SNAPSHOT` installed to local Maven repo.

- [ ] **Step 2: Verify installation**

Run: `ls ~/.m2/repository/org/mvel3/mvel3/3.0.0-SNAPSHOT/ | head -5`
Expected: jar(s) listed with today's date.

- [ ] **Step 3: No commit** — install is a build action, not a code change.

---

# Phase 2 — DRLX (#19)

Working directory: `/home/tkobayas/usr/work/mvel3-development/drlx-parser`
Pre-condition: Phase 1 complete, MVEL3 SNAPSHOT installed locally.

---

### Task 8: Create the DRLX feature branch

- [ ] **Step 1: Branch from main**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser checkout -b 19-constraint-mask main
```

Expected: switched to new branch `19-constraint-mask`.

---

### Task 9: Write the regression test (failing) — `conditionPlusWatchList_doesNotFireOnUnrelated`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`

- [ ] **Step 1: Add the failing test**

Add this new test method to `PropertyReactiveWatchListTest`:

```java
    @Test
    void conditionPlusWatchList_doesNotFireOnUnrelated() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[salary > 0][basePay],
                    do { }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(6000, 4000, 1000);
            FactHandle fh = ep.insert(emp);

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            listener.getAfterMatchFired().clear();

            // bonusPay neither in watch list nor read by constraint → must NOT re-fire.
            emp.setBonusPay(2000);
            ep.update(fh, emp, "bonusPay");
            assertThat(kieSession.fireAllRules()).isEqualTo(0);
            assertThat(listener.getAfterMatchFired()).isEmpty();
        });
    }
```

- [ ] **Step 2: Run the test, confirm failure**

Run: `mvn -pl drlx-parser-core test -Dtest=PropertyReactiveWatchListTest#conditionPlusWatchList_doesNotFireOnUnrelated -q`
Expected: FAIL — `fireAllRules()` returns 1 (alpha mask is `AllSetButLastBitMask` so `bonusPay` triggers re-eval). This confirms #19 is reproducible and the test exercises the right surface.

- [ ] **Step 3: Do not commit yet** — production change comes next.

---

### Task 10: Implement `getListenedPropertyMask` override

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaConstraint.java`

- [ ] **Step 1: Add imports**

At the top of `DrlxLambdaConstraint.java`, add these imports (alphabetical placement):

```java
import java.util.List;
import java.util.Optional;

import org.drools.base.base.ObjectType;
import org.drools.base.reteoo.PropertySpecificUtil;
import org.drools.base.rule.Pattern;
import org.drools.util.bitmask.BitMask;

import static org.drools.base.reteoo.PropertySpecificUtil.getEmptyPropertyReactiveMask;
```

- [ ] **Step 2: Add the override**

Insert the override after `replaceDeclaration` (around line 83) and before `clone()`:

```java
    @Override
    public BitMask getListenedPropertyMask(Optional<Pattern> pattern,
                                           ObjectType objectType,
                                           List<String> settableProperties) {
        String[] reads = evaluator.getReadProperties();
        if (reads.length == 0) {
            return super.getListenedPropertyMask(pattern, objectType, settableProperties);
        }
        BitMask mask = getEmptyPropertyReactiveMask(settableProperties.size());
        for (String prop : reads) {
            int pos = settableProperties.indexOf(prop);
            if (pos >= 0) { // ignore properties that aren't settable
                mask = mask.set(pos + PropertySpecificUtil.CUSTOM_BITS_OFFSET);
            }
        }
        return mask;
    }
```

- [ ] **Step 3: Build the dependent module**

Run: `mvn -pl drlx-parser-core -am install -DskipTests -q` from drlx-parser repo root.
Expected: BUILD SUCCESS.

- [ ] **Step 4: Run the failing test from Task 9**

Run: `mvn -pl drlx-parser-core test -Dtest=PropertyReactiveWatchListTest#conditionPlusWatchList_doesNotFireOnUnrelated -q`
Expected: PASS.

- [ ] **Step 5: Commit Tasks 9+10**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaConstraint.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
override getListenedPropertyMask via Evaluator.getReadProperties()

Mirror drools-modelcompiler.LambdaConstraint:170-188. Constraints with
expression reads now contribute a property-specific BitMask to the
alpha-node listened mask instead of the AllSetButLastBitMask default.
This finishes the watch-list behavior added in #13: modifications to
properties neither in the constraint nor the watch list no longer
re-fire alpha-bearing patterns.

Refs #19
EOF
)"
```

---

### Task 11: Add the two remaining condition+watch tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`

- [ ] **Step 1: Add `conditionPlusWatchList_firesOnConstraintProp`**

```java
    @Test
    void conditionPlusWatchList_firesOnConstraintProp() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[salary > 0][basePay],
                    do { }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(6000, 4000, 1000);
            FactHandle fh = ep.insert(emp);

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            listener.getAfterMatchFired().clear();

            // salary read by constraint → re-fire required.
            emp.setSalary(7000);
            ep.update(fh, emp, "salary");
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }
```

- [ ] **Step 2: Add `conditionPlusWatchList_firesOnWatchProp`**

```java
    @Test
    void conditionPlusWatchList_firesOnWatchProp() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[salary > 0][basePay],
                    do { }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(6000, 4000, 1000);
            FactHandle fh = ep.insert(emp);

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            listener.getAfterMatchFired().clear();

            // basePay in watch list → re-fire required.
            emp.setBasePay(5000);
            ep.update(fh, emp, "basePay");
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }
```

- [ ] **Step 3: Run the new tests**

Run: `mvn -pl drlx-parser-core test -Dtest=PropertyReactiveWatchListTest -q`
Expected: all tests PASS (existing 7 + 3 new = 10).

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: condition+watch-list cases for re-fire on read and watched props

Refs #19
EOF
)"
```

---

### Task 12: Remove the `// MVP scope:` comment block

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`

- [ ] **Step 1: Delete lines 12-15 of the file**

Remove this block:

```java
// MVP scope: watch list fully restricts modification re-evaluation only when
// the pattern has no alpha constraints. With conditions present, modifications
// still propagate through the alpha because DrlxLambdaConstraint does not yet
// override getListenedPropertyMask. See follow-up issue on DrlxLambdaConstraint
// property-mask reporting.
```

- [ ] **Step 2: Compile**

Run: `mvn -pl drlx-parser-core test-compile -q`
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: drop MVP-scope limitation comment now that #19 is fixed

Refs #19
EOF
)"
```

---

### Task 13: Full regression run

- [ ] **Step 1: Run full DRLX test suite**

Run: `mvn -pl drlx-parser-core test -q` from drlx-parser repo root.
Expected: BUILD SUCCESS. 111 tests passing (was 108 — added 3).

- [ ] **Step 2 (optional): Run no-persist mode if previously enabled**

If the codebase is normally tested in both modes (per the handover note), inspect `DrlxBuilderTestSupport` for the `@DisabledIfSystemProperty` annotation to find the exact system property and run the suite with it set. Skip if uncertain — Step 1 is the gating run for #19.

- [ ] **Step 3: No commit** — verification only.

---

### Task 14: Push and close #19

- [ ] **Step 1: Push DRLX branch**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push -u origin 19-constraint-mask
```

- [ ] **Step 2: Push MVEL3 branch (if not already pushed)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel push -u origin mvel3-read-props
```

- [ ] **Step 3: Close #19 with a summary comment**

```bash
gh issue close 19 --repo tkobayas/drlx-parser --comment "$(cat <<'EOF'
Fixed by:
- MVEL3 branch `mvel3-read-props`: surface `Evaluator.getReadProperties()` populated by `VariableAnalyser`.
- DRLX branch `19-constraint-mask`: override `DrlxLambdaConstraint.getListenedPropertyMask` to consume read properties.

Closes the gap from #13 — `[salary > 0][basePay]` now correctly suppresses re-fire on modifications to constraint-unrelated, non-watched properties.

Out of scope (deferred):
- Nested property access (`address.city` would yield only `address`).
- Method calls with arguments.
- Explicit `@Watch`-style override / `getReactivityBitMask()`.
EOF
)"
```

- [ ] **Step 4: Decide on PR or merge**

If user wants to merge directly to main: `git checkout main && git merge --ff-only 19-constraint-mask && git push`. Otherwise open a PR via `gh pr create`.

Same for MVEL3 — merge or PR per user preference.

---

## Watch for during execution

Spec open questions were resolved before execution (see spec §"Open questions (resolved 2026-04-27)"). The two below remain relevant only as regression-investigation hints:

1. **`FieldAccessExpr` regression.** If a test fails at Task 13 with a "rule doesn't fire on a property modify" symptom, check whether the test uses `e.salary` (current-pattern-aliased) form. The v1 analyser's empty-result branch falls back to AllSetButLastBitMask, which over-listens (safe). Investigate only if a test actually regresses.

2. **Proto round-trip.** `DrlxRuleAstParseResultTest` round-trip tests should pass unchanged. If they fail, investigate `DrlxLambdaCompiler.tryLoadPreCompiled` — the pre-built evaluator class on disk must include the new `getReadProperties()` override, which means the MVEL3 install (Task 7) must complete before any pre-build artifacts are regenerated.

---

## Execution outcome (added 2026-04-27)

All 14 tasks completed. Three deviations from the plan-as-written — see the spec's "Deviations during implementation" section for the full rationale:

1. **Task 4 — `VariableAnalyser`:** The `if (available.contains(name))` gate in the proposed code was dropped. `readProperties` collects every `NameExpr` / `DrlNameExpr`. Consumer-side filtering via `settableProperties` handles non-property identifiers.

2. **Task 4 — `MethodCallExpr` branch:** Removed entirely. MVEL3 rejects bare `getX()` calls in user expressions, so the branch was unreachable. The `getter2property` helper was removed.

3. **Task 5 — Codegen ordering:** `emitGetReadPropertiesOverride` is called *after* the rewrite pass (after `method.setBody(...)` and `MVELToJavaRewriter`), not before the eval method is built. Required because `MVELCompiler.registerAndRename` uses `findFirst(MethodDeclaration.class)`.

4. **Task 6 — Test count:** 3 cases (`bareNameExpr_collectsProperty`, `multipleProperties_collectsAll`, `noProperties_returnsEmpty`) instead of the planned 5. The two getter-form tests were removed alongside the analyser branch.

5. **Task 14 — Branch fates:**
   - MVEL3 branch `mvel3-read-props` merged upstream via [mvel/mvel#423](https://github.com/mvel/mvel/issues/423).
   - DRLX branch `19-constraint-mask` pushed to `origin`; PR open at `https://github.com/tkobayas/drlx-parser/pull/new/19-constraint-mask`.
   - Issue [tkobayas/drlx-parser#19](https://github.com/tkobayas/drlx-parser/issues/19) closed with summary.

Final test counts: full MVEL3 suite 720 passing (0 failures); full DRLX suite 111 passing (was 108, +3).
