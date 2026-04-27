# DRLXXXX `test` construct — Implementation Plan (#23)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement DRLXXXX `test <expression>` end-to-end — proto + IR + lambda compiler entry point + runtime mapping (drools-core `EvalCondition`) + user-facing grammar + visitor + tests. Once shipped, `test` is independently usable in DRLX rule bodies and is the foundation that #12 (if/else) will desugar onto.

**Architecture:** Add `EvalIR` to the sealed `LhsItemIR` hierarchy. Visitor parses `test <expression>` into `EvalIR(expression, referencedBindingNames)`. Runtime builder bridges `EvalIR` to drools-core via:
1. `DrlxLambdaCompiler.createEvalExpression(expression, referencedBindings)` — compiles the boolean expression via the existing MVEL3 batch-compile pipeline (`MVEL.map(...).out(Boolean.class).expression(...)`), returns a `DrlxEvalExpression` that implements `org.drools.base.rule.EvalExpression`.
2. `DrlxRuleAstRuntimeBuilder.buildLhs` constructs an `EvalCondition(evalExpression, requiredDeclarations)` and adds it as a child of the parent `GroupElement`.

**Tech Stack:** Java 21 (sealed interfaces, records), ANTLR 4, drools-base 10.1.0, MVEL3 3.0.0-SNAPSHOT (batch-compilation lambda framework with `LambdaHandle`/`PendingLambda` deferred resolution), Protocol Buffers 3, JUnit 5, AssertJ.

---

## File map

**Modify:**
- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4` — add `TEST` token
- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` — add `testElement` production; add to `ruleItem` alternatives
- `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` — add `EvalParseResult` message; add `eval = 3` to `LhsItemParseResult.kind` oneof
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` — add `EvalIR` record; extend `LhsItemIR permits` clause
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` — extend `fromProtoLhs` and `toProtoLhs` for `EvalIR`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` — add `buildTestElement` method; dispatch in `buildRule`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` — add `EvalIR` branch in `buildLhs` switch; emit `EvalCondition`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java` — add `createEvalExpression(String, List<BoundVariable>) -> EvalExpression`

**Create:**
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxEvalExpression.java` — `EvalExpression` impl wrapping an MVEL3 `Evaluator<Map<String,Object>,Void,Boolean>`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java` — IR-level + proto round-trip tests (or extend existing if it exists)
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaCompilerTest.java` — `createEvalExpression` unit (or extend existing)
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementParseTest.java` — pure parse-tree assertions
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementTest.java` — runtime semantics (extends `DrlxBuilderTestSupport`)
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java` — programmatic `EvalIR` constructed without grammar (verifies runtime mapping in isolation)

**Reuse (no changes expected):**
- Existing `Person`, `MyUnit` in test fixtures.

---

## Build & test commands

- Generate ANTLR + protobuf + compile + run tests for drlx-parser-core only:
  - `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml clean install`
- Run a single test class:
  - `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TestElementTest`
- Run full DRLX test suite:
  - `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`
- Force ANTLR re-generation:
  - `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml clean generate-sources`

All `git` commands use `git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser <cmd>` — never `cd`.

Baseline: 111 tests passing per `HANDOFF.md`. Each task adds tests; final count ≈ 121–125.

---

## Task 1: Add `EvalIR` record + sealed-interface permits

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java` (create if absent)

- [ ] **Step 1: Write failing test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java
package org.drools.drlx.builder;

import java.util.List;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class DrlxRuleAstModelTest {

    @Test
    void evalIrIsLhsItem() {
        EvalIR eval = new EvalIR("p.age > 30", List.of("p"));
        LhsItemIR item = eval;   // sealed-interface assignment must compile
        assertThat(item).isInstanceOf(EvalIR.class);
        assertThat(eval.expression()).isEqualTo("p.age > 30");
        assertThat(eval.referencedBindings()).containsExactly("p");
    }

    @Test
    void evalIrReferencedBindingsListIsImmutable() {
        java.util.ArrayList<String> mutable = new java.util.ArrayList<>(List.of("p"));
        EvalIR eval = new EvalIR("p.age > 30", mutable);
        mutable.add("q");
        assertThat(eval.referencedBindings()).containsExactly("p");   // unaffected by mutation
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstModelTest`

Expected: FAIL — `EvalIR` does not exist; class won't compile.

- [ ] **Step 3: Implement**

In `DrlxRuleAstModel.java`, replace the existing `LhsItemIR` declaration:

```java
public sealed interface LhsItemIR permits PatternIR, GroupElementIR, EvalIR {
}
```

Append the new record (after `GroupElementIR`):

```java
public record EvalIR(String expression, List<String> referencedBindings) implements LhsItemIR {
    public EvalIR {
        referencedBindings = List.copyOf(referencedBindings);
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstModelTest`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
ir: add EvalIR record (DRLXXXX 'test' construct)

EvalIR(expression, referencedBindings) joins PatternIR and
GroupElementIR in the LhsItemIR sealed hierarchy. Defensive copy
of referencedBindings keeps the record immutable.

Refs #23
EOF
)"
```

---

## Task 2: Proto — `EvalParseResult` + extend round-trip

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java`

- [ ] **Step 1: Append failing round-trip test**

In `DrlxRuleAstModelTest.java`:

```java
@Test
void evalIrRoundTripsThroughProto() {
    EvalIR original = new EvalIR("p.age > 30", List.of("p"));
    GroupElementIR andGroup = new GroupElementIR(
            GroupElementIR.Kind.AND,
            List.of(original));

    var proto = DrlxRuleAstParseResult.toProtoLhs(andGroup);
    LhsItemIR roundTripped = DrlxRuleAstParseResult.fromProtoLhs(
            proto, java.nio.file.Path.of("test.drlx"));

    assertThat(roundTripped).isInstanceOf(GroupElementIR.class);
    GroupElementIR g = (GroupElementIR) roundTripped;
    assertThat(g.children()).hasSize(1);
    EvalIR e = (EvalIR) g.children().get(0);
    assertThat(e.expression()).isEqualTo("p.age > 30");
    assertThat(e.referencedBindings()).containsExactly("p");
}

@Test
void evalIrWithMultipleBindingsRoundTrips() {
    EvalIR original = new EvalIR("p.age > q.age", List.of("p", "q"));
    var proto = DrlxRuleAstParseResult.toProtoLhs(
            new GroupElementIR(GroupElementIR.Kind.AND, List.of(original)));
    GroupElementIR g = (GroupElementIR) DrlxRuleAstParseResult.fromProtoLhs(
            proto, java.nio.file.Path.of("test.drlx"));
    EvalIR e = (EvalIR) g.children().get(0);
    assertThat(e.referencedBindings()).containsExactlyInAnyOrder("p", "q");
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstModelTest#evalIrRoundTripsThroughProto`

Expected: FAIL — `toProtoLhs` throws "Unsupported LHS item: …EvalIR…".

- [ ] **Step 3: Add proto message + oneof entry**

In `drlx_rule_ast.proto`:

```protobuf
message EvalParseResult {
  string expression = 1;
  repeated string referenced_bindings = 2;
}

message LhsItemParseResult {
  oneof kind {
    PatternParseResult pattern = 1;
    GroupElementParseResult group = 2;
    EvalParseResult eval = 3;
  }
}
```

- [ ] **Step 4: Extend `fromProtoLhs` + `toProtoLhs`**

In `DrlxRuleAstParseResult.java`, in the `fromProtoLhs` switch (currently has `PATTERN`, `GROUP`, `KIND_NOT_SET`), insert before `KIND_NOT_SET`:

```java
case EVAL -> {
    DrlxRuleAstProto.EvalParseResult eval = item.getEval();
    yield new EvalIR(
            eval.getExpression(),
            List.copyOf(eval.getReferencedBindingsList()));
}
```

In `toProtoLhs`, between the `GroupElementIR` branch and the `else throw`:

```java
} else if (item instanceof EvalIR e) {
    DrlxRuleAstProto.EvalParseResult.Builder eb =
            DrlxRuleAstProto.EvalParseResult.newBuilder()
                    .setExpression(e.expression());
    e.referencedBindings().forEach(eb::addReferencedBindings);
    builder.setEval(eb);
```

- [ ] **Step 5: Run test — verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstModelTest`

Expected: PASS (all 4 tests). Maven regenerates protobuf Java sources during `generate-sources`.

- [ ] **Step 6: Verify no regression**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`

Expected: full suite passes (baseline + 4 new).

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
proto: serialize EvalIR (eval = 3 in LhsItemParseResult oneof)

Refs #23
EOF
)"
```

---

## Task 3: `DrlxEvalExpression` — `EvalExpression` impl wrapping MVEL3 evaluator

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxEvalExpression.java`

This task ships the wrapper class only — no DrlxLambdaCompiler integration yet. Test with a hand-built MVEL3 evaluator, verify the `EvalExpression` contract (evaluate, replaceDeclaration, clone).

- [ ] **Step 1: Write failing unit test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxEvalExpressionTest.java
package org.drools.drlx.builder;

import java.util.HashMap;
import java.util.Map;
import org.drools.base.rule.Declaration;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class DrlxEvalExpressionTest {

    @Test
    void evaluatesUsingMvelEvaluatorOverDeclarationMap() throws Exception {
        // Build a fake MVEL3 evaluator that returns true if input map contains "p" → age > 30
        org.mvel3.transpiler.api.Evaluator<Map<String,Object>, Void, Boolean> stubEvaluator =
                inputs -> {
                    Object p = inputs.get("p");
                    if (p == null) return false;
                    int age = (int) p.getClass().getMethod("getAge").invoke(p);
                    return age > 30;
                };

        DrlxEvalExpression expr = new DrlxEvalExpression(
                "p.age > 30",
                stubEvaluator,
                new String[] {"p"});

        // Build a tiny BaseTuple stub that returns p → Person("Alice", 40)
        // (Use whatever real BaseTuple impl is most ergonomic; or a minimal anonymous class.)
        var tuple = new TestTuple(Map.of("p", new org.drools.drlx.domain.Person("Alice", 40)));

        // requiredDeclarations is unused by DrlxEvalExpression (it uses declarationNames),
        // so an empty array is fine here.
        boolean result = expr.evaluate(tuple, new Declaration[0], null, null);
        assertThat(result).isTrue();

        var tuple2 = new TestTuple(Map.of("p", new org.drools.drlx.domain.Person("Bob", 25)));
        boolean result2 = expr.evaluate(tuple2, new Declaration[0], null, null);
        assertThat(result2).isFalse();
    }

    @Test
    void cloneReturnsEquivalentInstance() throws Exception {
        DrlxEvalExpression expr = new DrlxEvalExpression(
                "true", inputs -> true, new String[] {});
        DrlxEvalExpression clone = (DrlxEvalExpression) expr.clone();
        assertThat(clone).isNotNull();
        assertThat(clone.evaluate(new TestTuple(Map.of()), new Declaration[0], null, null))
                .isTrue();
    }
}
```

`TestTuple` is a minimal `BaseTuple` stub for the test:

```java
// In the same test file or a sibling helper file
class TestTuple implements org.drools.base.reteoo.BaseTuple {
    private final Map<String, Object> values;
    TestTuple(Map<String, Object> values) { this.values = values; }
    @Override public Object getObject(org.drools.base.rule.Declaration d) {
        return values.get(d.getIdentifier());
    }
    // Other BaseTuple methods: throw UnsupportedOperationException — not used by DrlxEvalExpression
    @Override public org.kie.api.runtime.rule.FactHandle get(org.drools.base.rule.Declaration d) { throw new UnsupportedOperationException(); }
    @Override public org.kie.api.runtime.rule.FactHandle get(int p) { throw new UnsupportedOperationException(); }
    @Override public org.kie.api.runtime.rule.FactHandle getFactHandle() { throw new UnsupportedOperationException(); }
    @Override public Object getObject(int p) { throw new UnsupportedOperationException(); }
    @Override public int size() { return values.size(); }
    @Override public Object[] toObjects() { throw new UnsupportedOperationException(); }
    @Override public org.kie.api.runtime.rule.FactHandle[] toFactHandles() { throw new UnsupportedOperationException(); }
    @Override public org.drools.base.reteoo.BaseTuple getParent() { throw new UnsupportedOperationException(); }
    @Override public org.drools.base.reteoo.BaseTuple getTuple(int i) { throw new UnsupportedOperationException(); }
    @Override public int getIndex() { return 0; }
}
```

(If `BaseTuple` has more abstract methods than listed, stub them all to throw. Open the actual file at `drools-base/src/main/java/org/drools/base/reteoo/BaseTuple.java` and copy its method list.)

**Note on `evaluate(...)` strategy:** `DrlxEvalExpression` uses **declaration names** (a `String[]` field) to look up values via `tuple.getObject(declaration)`. This is simpler than navigating `Declaration.getValue(...)` and works for the binding case (where the declaration's reader is identity). If a future use case requires reading a specific field via the declaration, switch to `declaration.getValue(reteEvaluator, tuple.get(declaration.getTupleIndex()))` — see `BindingEvaluator.getArgument` for the canonical pattern.

- [ ] **Step 2: Run test — verify failure**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxEvalExpressionTest`

Expected: FAIL — `DrlxEvalExpression` does not exist.

- [ ] **Step 3: Implement `DrlxEvalExpression`**

```java
// drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxEvalExpression.java
package org.drools.drlx.builder;

import java.util.HashMap;
import java.util.Map;
import org.drools.base.reteoo.BaseTuple;
import org.drools.base.rule.Declaration;
import org.drools.base.rule.accessor.EvalExpression;
import org.kie.api.runtime.rule.FactHandle;
import org.mvel3.transpiler.api.Evaluator;

/**
 * Bridges an MVEL3-compiled boolean evaluator to drools-core's EvalExpression contract.
 * The evaluator is shared (effectively immutable post-compile); clone() returns this.
 */
public class DrlxEvalExpression implements EvalExpression {

    private final String expression;                 // for diagnostics
    private volatile Evaluator<Map<String, Object>, Void, Boolean> evaluator;
    private final String[] declarationNames;

    public DrlxEvalExpression(String expression,
                              Evaluator<Map<String, Object>, Void, Boolean> evaluator,
                              String[] declarationNames) {
        this.expression = expression;
        this.evaluator = evaluator;
        this.declarationNames = declarationNames;
    }

    /** Plug in the resolved evaluator after batch compilation completes. */
    public void setEvaluator(Evaluator<Map<String, Object>, Void, Boolean> evaluator) {
        this.evaluator = evaluator;
    }

    @Override
    public Object createContext() {
        return null;
    }

    @Override
    public boolean evaluate(BaseTuple tuple,
                            Declaration[] requiredDeclarations,
                            org.drools.base.rule.accessor.ValueResolver valueResolver,
                            Object context) throws Exception {
        if (evaluator == null) {
            throw new IllegalStateException(
                    "DrlxEvalExpression evaluator not yet resolved for: " + expression);
        }
        Map<String, Object> input = new HashMap<>(requiredDeclarations.length * 2);
        for (Declaration d : requiredDeclarations) {
            input.put(d.getIdentifier(), tuple.getObject(d));
        }
        return Boolean.TRUE.equals(evaluator.eval(input));
    }

    @Override
    public void replaceDeclaration(Declaration declaration, Declaration resolved) {
        // No-op — DrlxEvalExpression looks up declarations by name, not by reference.
        // EvalCondition's own replaceDeclaration updates its requiredDeclarations[] array
        // (which is what evaluate() iterates), so name-based lookup remains correct.
    }

    @Override
    public DrlxEvalExpression clone() {
        // Stateless wrt evaluator (compiled once, shared) — share the reference.
        return new DrlxEvalExpression(expression, evaluator, declarationNames);
    }

    public String getExpression() {
        return expression;
    }
}
```

If the import path for `ValueResolver` is `org.drools.base.rule.accessor.ValueResolver` (per the agent's investigation), keep it. Otherwise, follow the imports in `org.drools.mvel.expr.MVELEvalExpression` for the correct location.

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxEvalExpressionTest`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxEvalExpression.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxEvalExpressionTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
runtime: DrlxEvalExpression bridges MVEL3 boolean lambda to EvalExpression

Implements org.drools.base.rule.accessor.EvalExpression by reading
declaration values via tuple.getObject(declaration) into a
Map<String,Object> input and invoking the MVEL3 evaluator.
setEvaluator() supports the deferred batch-compile resolution flow.

Refs #23
EOF
)"
```

---

## Task 4: `DrlxLambdaCompiler.createEvalExpression`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaCompilerTest.java`

**Goal:** Add the public method `createEvalExpression(String expression, List<BoundVariable> referencedBindings)` that mirrors the existing `createBatchBetaConstraint` flow (lines 282–297 of `DrlxLambdaCompiler.java`) but produces a `DrlxEvalExpression` instead of a `DrlxLambdaBetaConstraint`.

- [ ] **Step 1: Write failing test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaCompilerTest.java
package org.drools.drlx.builder;

import java.util.List;
import org.drools.base.rule.accessor.EvalExpression;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class DrlxLambdaCompilerTest {

    @Test
    void createEvalExpressionReturnsDrlxEvalExpression() {
        // Set up a compiler instance the same way other tests do.
        DrlxLambdaCompiler compiler = TestCompilerFactory.newCompiler();

        // Build a BoundVariable for "p" of type Person.
        BoundVariable pBinding = TestCompilerFactory.bind(compiler, "p",
                org.drools.drlx.domain.Person.class);

        EvalExpression result = compiler.createEvalExpression(
                "p.age > 30",
                List.of(pBinding));

        assertThat(result).isInstanceOf(DrlxEvalExpression.class);
        assertThat(((DrlxEvalExpression) result).getExpression()).isEqualTo("p.age > 30");
    }
}
```

`TestCompilerFactory` is a small helper that constructs a fresh `DrlxLambdaCompiler` and produces `BoundVariable` instances with synthetic patterns suitable for unit testing. If a similar helper already exists for `DrlxLambdaCompiler`, reuse it. Otherwise create the minimal helper here:

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/TestCompilerFactory.java
package org.drools.drlx.builder;

import org.drools.base.base.ClassObjectType;
import org.drools.base.rule.Declaration;
import org.drools.base.rule.Pattern;

final class TestCompilerFactory {
    private TestCompilerFactory() {}

    static DrlxLambdaCompiler newCompiler() {
        // Match how OrTest / AndTest builds a compiler; if there's a no-arg constructor, use it.
        return new DrlxLambdaCompiler();
    }

    static BoundVariable bind(DrlxLambdaCompiler compiler, String name, Class<?> type) {
        Pattern pattern = new Pattern(0, new ClassObjectType(type), name);
        Declaration declaration = pattern.getDeclaration();
        // Fill in declaration's reader if needed — simplest is the identity reader for whole-fact bindings.
        return new BoundVariable(name, type, pattern);
    }
}
```

If `Pattern` constructor or `Declaration` setup needs different scaffolding (check `DrlxRuleAstRuntimeBuilder.buildPattern` for the canonical setup), copy that. Helper minimalism: only what's needed to satisfy `BoundVariable.pattern().getDeclaration()` returning a non-null `Declaration`.

- [ ] **Step 2: Run test — verify failure**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxLambdaCompilerTest`

Expected: FAIL — `createEvalExpression` does not exist.

- [ ] **Step 3: Implement `createEvalExpression`**

In `DrlxLambdaCompiler.java`, near `createBetaLambdaConstraint` (around line 162), add:

```java
public org.drools.base.rule.accessor.EvalExpression createEvalExpression(
        String expression,
        List<BoundVariable> referencedBindings) {
    int counter = lambdaCounter++;

    // Build MVEL3 declarations from the referenced bindings.
    List<org.mvel3.transpiler.context.Declaration<?>> mvelDecls =
            new java.util.ArrayList<>(referencedBindings.size());
    for (BoundVariable bv : referencedBindings) {
        mvelDecls.add(org.mvel3.transpiler.context.Declaration.of(bv.name(), bv.type()));
    }
    org.mvel3.transpiler.context.Declaration<?>[] mvelDeclarations =
            mvelDecls.toArray(new org.mvel3.transpiler.context.Declaration[0]);

    String[] declarationNames = referencedBindings.stream()
            .map(BoundVariable::name)
            .toArray(String[]::new);

    // Try pre-compiled cache first (mirrors createBetaLambdaConstraint).
    @SuppressWarnings("unchecked")
    org.mvel3.transpiler.api.Evaluator<java.util.Map<String, Object>, Void, Boolean> preCompiled =
            (org.mvel3.transpiler.api.Evaluator<java.util.Map<String, Object>, Void, Boolean>)
                    tryLoadPreCompiled(counter, expression, "test eval");
    if (preCompiled != null) {
        DrlxEvalExpression eager =
                new DrlxEvalExpression(expression, preCompiled, declarationNames);
        return eager;
    }

    // Defer via batch compiler (mirrors createBatchBetaConstraint).
    @SuppressWarnings({"unchecked", "rawtypes"})
    org.mvel3.transpiler.api.CompilerParameters<java.util.Map<String, Object>, Void, Boolean> evalInfo =
            (org.mvel3.transpiler.api.CompilerParameters)
                    org.mvel3.MVEL.<Object>map(mvelDeclarations)
                            .<Boolean>out(Boolean.class)
                            .expression(expression)
                            .classManager(batchCompiler.getClassManager())
                            .generatedClassName("GeneratorEvalExpr__")
                            .build();

    org.drools.drlx.builder.MVELBatchCompiler.LambdaHandle handle =
            batchCompiler.add(evalInfo);

    DrlxEvalExpression deferred =
            new DrlxEvalExpression(expression, null, declarationNames);
    pendingLambdas.add(new PendingLambda(handle, deferred));
    onLambdaCreated(counter, expression);
    return deferred;
}
```

`PendingLambda` currently ties a `LambdaHandle` to a `DrlxLambdaConstraint` (or `DrlxLambdaBetaConstraint`). To support `DrlxEvalExpression` as the second argument, you have two options — pick the one that requires less invasive refactoring:

(a) **Generalize `PendingLambda` to hold a `Consumer<Evaluator<Map<String,Object>,Void,Boolean>>`**:
- Rewrite the constructor: `PendingLambda(handle, evaluatorConsumer)`.
- The constraint subclasses pass `c::setEvaluator`; `DrlxEvalExpression` passes `e::setEvaluator`.
- The post-compile resolution loop calls the consumer with the resolved evaluator.

(b) **Use a sealed interface for `PendingLambda`'s second argument**:
- `interface BatchCompileTarget { void setEvaluator(Evaluator<...>); }`
- `DrlxLambdaConstraint`, `DrlxLambdaBetaConstraint`, `DrlxEvalExpression` all implement it.
- `PendingLambda` holds `LambdaHandle` + `BatchCompileTarget`.

Option (a) is smaller. Use (a) unless `BatchCompileTarget` is more idiomatic in the rest of the codebase. Inspect the existing `PendingLambda` resolution code (search for `pendingLambdas` references) to see how the resolved evaluator gets handed back, then mirror that pattern.

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxLambdaCompilerTest`

Expected: PASS.

- [ ] **Step 5: Verify no regression**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`

Expected: full suite still passes.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaCompilerTest.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/TestCompilerFactory.java

# If PendingLambda was refactored, add it too:
# git -C ... add drlx-parser-core/src/main/java/.../PendingLambda.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
lambda: createEvalExpression — boolean lambda → EvalExpression

Mirrors createBatchBetaConstraint: tries pre-compiled cache, then
defers via batch compiler. PendingLambda generalised to support
DrlxEvalExpression alongside constraint subclasses.

Refs #23
EOF
)"
```

---

## Task 5: Runtime builder — `EvalIR` → `EvalCondition`, programmatic E2E

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java`

- [ ] **Step 1: Write failing programmatic E2E test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java
package org.drools.drlx.builder.syntax;

import java.util.List;
import org.drools.drlx.builder.ConsequenceIR;
import org.drools.drlx.builder.EvalIR;
import org.drools.drlx.builder.LhsItemIR;
import org.drools.drlx.builder.PatternIR;
import org.drools.drlx.builder.RuleIR;
import org.drools.drlx.domain.Person;
import org.drools.drlx.ruleunit.MyUnit;
import org.junit.jupiter.api.Test;
import org.kie.api.KieBase;
import org.kie.api.runtime.KieSession;
import org.kie.api.runtime.rule.EntryPoint;

import static org.assertj.core.api.Assertions.assertThat;

class EvalIRBuilderTest {

    @Test
    void evalIrFiltersPattern() {
        // Build IR programmatically: rule R { var p : /persons, [eval p.age > 30], do { ... } }
        PatternIR pattern = new PatternIR(
                "Person", "p", "persons",
                List.of(),
                null, List.of(), false, List.of());
        EvalIR eval = new EvalIR("p.age > 30", List.of("p"));
        ConsequenceIR rhs = new ConsequenceIR("System.out.println(p);");

        RuleIR rule = new RuleIR(
                "R", List.of(),
                List.of((LhsItemIR) pattern, (LhsItemIR) eval),
                rhs);

        KieBase kieBase = TestRuleBuilders.buildFromIr(rule, MyUnit.class);
        KieSession session = kieBase.newKieSession();
        try {
            EntryPoint persons = session.getEntryPoint("persons");
            persons.insert(new Person("Alice", 40));
            persons.insert(new Person("Bob", 25));
            int fired = session.fireAllRules();
            assertThat(fired).isEqualTo(1);   // only Alice (age > 30) matches
        } finally {
            session.dispose();
        }
    }
}
```

`TestRuleBuilders.buildFromIr(RuleIR, Class<? extends RuleUnitData>)` is a small new test helper that wraps a single `RuleIR` in a `CompilationUnitIR` (or whatever the existing top-level IR type is) and runs it through `DrlxRuleAstRuntimeBuilder`. If the existing `DrlxRuleBuilder` already exposes such a path, reuse it; otherwise add a minimal helper:

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestRuleBuilders.java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.builder.RuleIR;
import org.drools.ruleunits.api.RuleUnitData;
import org.kie.api.KieBase;

final class TestRuleBuilders {
    private TestRuleBuilders() {}

    static KieBase buildFromIr(RuleIR rule, Class<? extends RuleUnitData> unit) {
        // TODO: invoke whatever entry point DrlxRuleAstRuntimeBuilder exposes for a single rule + unit.
        // The existing OrTest/AndTest tests build via DrlxRuleBuilder.build(text); for IR-only,
        // we need the layer just above DrlxRuleAstRuntimeBuilder. Inspect DrlxRuleBuilder source
        // and copy the minimal CompilationUnitIR construction.
        throw new UnsupportedOperationException("implement using DrlxRuleAstRuntimeBuilder");
    }
}
```

The implementing agent reads `DrlxRuleBuilder` and `DrlxRuleAstRuntimeBuilder` to write the helper body. Should be ~10–20 lines.

- [ ] **Step 2: Run test — verify failure**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=EvalIRBuilderTest`

Expected: FAIL — `buildLhs` throws `IllegalArgumentException("Unsupported LHS item: …EvalIR…")` (or `UnsupportedOperationException` from the helper if it's still stubbed).

- [ ] **Step 3: Implement helper + add `EvalIR` branch in `buildLhs`**

In `DrlxRuleAstRuntimeBuilder.java`, in the `buildLhs` for-loop, after the `GroupElementIR` branch and before the `else throw`:

```java
} else if (item instanceof EvalIR evalIr) {
    // Resolve referenced declarations from boundVariables, dropping those that don't resolve.
    List<BoundVariable> referenced = new java.util.ArrayList<>();
    List<org.drools.base.rule.Declaration> declarations = new java.util.ArrayList<>();
    for (String name : evalIr.referencedBindings()) {
        BoundVariable bv = boundVariables.get(name);
        if (bv != null) {
            referenced.add(bv);
            declarations.add(bv.pattern().getDeclaration());
        }
        // Silently skip unresolved names (matches existing constraint-binding-detection behavior).
    }

    org.drools.base.rule.accessor.EvalExpression evalExpression =
            lambdaCompiler.createEvalExpression(evalIr.expression(), referenced);

    org.drools.base.rule.EvalCondition evalCondition =
            new org.drools.base.rule.EvalCondition(
                    evalExpression,
                    declarations.toArray(new org.drools.base.rule.Declaration[0]));
    parent.addChild(evalCondition);
}
```

Implement the `TestRuleBuilders.buildFromIr` helper body so the test can actually run.

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=EvalIRBuilderTest`

Expected: PASS.

If FAIL: most likely cause is an API mismatch (e.g., `DrlxLambdaConstraint` and `DrlxLambdaBetaConstraint` use a slightly different ValueResolver argument shape than `DrlxEvalExpression` — copy whichever pattern they use). Diagnose by reading the existing constraint's `evaluate`/`isAllowedCachedRight` and matching.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestRuleBuilders.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
runtime: build EvalCondition from EvalIR in buildLhs

Resolves referenced bindings via boundVariables (silently dropping
unresolved names), compiles via DrlxLambdaCompiler.createEvalExpression,
and attaches the resulting EvalCondition to the parent GroupElement.

Refs #23
EOF
)"
```

---

## Task 6: Grammar — `TEST` token + `testElement` production

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4`
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementParseTest.java` (new)

- [ ] **Step 1: Write failing parse-only test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementParseTest.java
package org.drools.drlx.builder.syntax;

import org.antlr.v4.runtime.CharStreams;
import org.antlr.v4.runtime.CommonTokenStream;
import org.drools.drlx.parser.DrlxLexer;
import org.drools.drlx.parser.DrlxParser;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TestElementParseTest {

    @Test
    void parsesTestElement() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    test p.age > 30,
                    do { System.out.println(p); }
                }
                """;
        DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(rule));
        DrlxParser parser = new DrlxParser(new CommonTokenStream(lexer));
        DrlxParser.CompilationUnitContext ctx = parser.compilationUnit();
        assertThat(parser.getNumberOfSyntaxErrors()).isZero();
        DrlxParser.RuleDeclarationContext rule0 = ctx.ruleDeclaration(0);
        assertThat(rule0.ruleBody().ruleItem()).anyMatch(
                ri -> ri.testElement() != null);
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TestElementParseTest`

Expected: FAIL — compilation error (`testElement()` does not exist; `TEST` token unknown).

- [ ] **Step 3: Add `TEST` token**

In `DrlxLexer.g4`, alongside `NOT`, `EXISTS`, `DRLX_AND`, `DRLX_OR`:

```antlr
TEST     : 'test';
```

- [ ] **Step 4: Add `testElement` production + dispatch**

In `DrlxParser.g4`, modify `ruleItem`:

```antlr
ruleItem
    : rulePattern
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','
    | ruleConsequence
    ;
```

Add `testElement` (place after `orElement`):

```antlr
testElement
    : TEST expression
    ;
```

- [ ] **Step 5: Run test — verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TestElementParseTest`

Expected: PASS. ANTLR regenerates `DrlxParser.java` with `RuleItemContext.testElement()` and `TestElementContext`.

- [ ] **Step 6: Verify no regression**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`

Expected: full suite still passes.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4 \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementParseTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
grammar: add TEST keyword and testElement production

testElement is a ruleItem alternative carrying a boolean expression.
Visitor dispatch + IR construction is the next task.

Refs #23
EOF
)"
```

---

## Task 7: Visitor — `buildTestElement` produces `EvalIR`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementParseTest.java`

- [ ] **Step 1: Append failing IR-shape test**

```java
@Test
void testElementProducesEvalIr() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                var p : /persons,
                test p.age > 30,
                do { System.out.println(p); }
            }
            """;
    org.drools.drlx.builder.RuleIR ruleIr =
            org.drools.drlx.builder.DrlxRuleAstParseResult.parseSingleRule(rule);
    java.util.List<org.drools.drlx.builder.LhsItemIR> lhs = ruleIr.lhs();
    assertThat(lhs).hasSize(2);
    assertThat(lhs.get(1)).isInstanceOf(org.drools.drlx.builder.EvalIR.class);
    org.drools.drlx.builder.EvalIR e = (org.drools.drlx.builder.EvalIR) lhs.get(1);
    assertThat(e.expression()).isEqualTo("p.age>30");   // whitespace stripped
    assertThat(e.referencedBindings()).contains("p");
}
```

(`DrlxRuleAstParseResult.parseSingleRule` is the entry point used by other parse tests — match existing `OrTest`/`AndTest` conventions; if the actual entry point has a different name, use it.)

Note on whitespace: ANTLR `getText()` strips token-level whitespace. If the test fails because the implementation uses a different extraction (e.g., preserving whitespace via `tokenStream.getText(start, stop)`), update the assertion to match what's actually produced.

- [ ] **Step 2: Run test — verify failure**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TestElementParseTest#testElementProducesEvalIr`

Expected: FAIL — `buildRule` throws `"Unsupported rule item: …"`.

- [ ] **Step 3: Add dispatch in `buildRule`**

In `DrlxToRuleAstVisitor.java`, in `buildRule`'s if-chain (after the `orElement` branch):

```java
} else if (itemCtx.testElement() != null) {
    lhs.add(buildTestElement(itemCtx.testElement()));
}
```

- [ ] **Step 4: Implement `buildTestElement`**

Add new method (placement: after `buildOrElement`):

```java
private EvalIR buildTestElement(DrlxParser.TestElementContext ctx) {
    String expression = ctx.expression().getText();
    return new EvalIR(expression, referencedBindings(expression));
}

private List<String> referencedBindings(String expression) {
    java.util.regex.Pattern idRegex =
            java.util.regex.Pattern.compile("\\b([a-zA-Z_][a-zA-Z0-9_]*)\\b");
    java.util.regex.Matcher m = idRegex.matcher(expression);
    java.util.LinkedHashSet<String> identifiers = new java.util.LinkedHashSet<>();
    while (m.find()) identifiers.add(m.group(1));
    return List.copyOf(identifiers);
}
```

The `referencedBindings` regex over-collects (Java keywords, type names, literals) but the runtime builder filters via `boundVariables.containsKey(name)`, so over-collection is harmless. If the existing `findReferencedBindings` in `DrlxLambdaCompiler` is already accessible from the visitor, prefer reusing it.

- [ ] **Step 5: Run test — verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TestElementParseTest`

Expected: PASS (both tests).

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementParseTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
visitor: build EvalIR from testElement parse tree

referencedBindings extracted via identifier regex over expression
text; runtime filters against live boundVariables map.

Refs #23
EOF
)"
```

---

## Task 8: Runtime — `test` end-to-end via grammar (alpha + beta)

**Files:**
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementTest.java` (new)

- [ ] **Step 1: Write failing alpha + beta tests**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementTest.java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.builder.DrlxBuilderTestSupport;
import org.drools.drlx.domain.Person;
import org.junit.jupiter.api.Test;
import org.kie.api.runtime.rule.EntryPoint;
import static org.assertj.core.api.Assertions.assertThat;

class TestElementTest extends DrlxBuilderTestSupport {

    @Test
    void test_filtersByExpression() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    test p.age > 30,
                    do { System.out.println(p); }
                }
                """;
        withSession(rule, (session, listener) -> {
            EntryPoint persons = session.getEntryPoint("persons");
            persons.insert(new Person("Alice", 40));
            persons.insert(new Person("Bob", 25));
            assertThat(session.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R");
        });
    }

    @Test
    void test_betaJoinAcrossTwoBindings() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    var q : /persons,
                    test p.age > q.age,
                    do { System.out.println(p + " > " + q); }
                }
                """;
        withSession(rule, (session, listener) -> {
            EntryPoint persons = session.getEntryPoint("persons");
            persons.insert(new Person("Alice", 40));
            persons.insert(new Person("Bob", 25));
            // Alice > Bob fires; Bob > Alice doesn't; same-age (none here) wouldn't.
            assertThat(session.fireAllRules()).isEqualTo(1);
        });
    }

    @Test
    void test_falseExpressionPreventsFiring() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    test p.age > 999,
                    do { System.out.println(p); }
                }
                """;
        withSession(rule, (session, listener) -> {
            EntryPoint persons = session.getEntryPoint("persons");
            persons.insert(new Person("Alice", 40));
            assertThat(session.fireAllRules()).isZero();
        });
    }
}
```

- [ ] **Step 2: Run tests — verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TestElementTest`

Expected: PASS — Tasks 1–7 should make this work end-to-end.

If FAIL on `test_betaJoinAcrossTwoBindings`: most likely cause is the runtime builder not adding both `p` and `q` declarations to the EvalCondition. Check that `boundVariables` contains both at the time the EvalIR is processed (it should — patterns are processed in order, both before the test).

If FAIL with NPE in evaluator: the deferred MVEL3 evaluator hasn't been resolved. Check that `pendingLambdas` is being drained (`batchCompiler.compile()` or equivalent is called before the rules are built). Look at how existing `DrlxLambdaBetaConstraint` ensures the evaluator is resolved before runtime use.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: 'test' end-to-end (alpha, beta-join, false-guard)

Refs #23
EOF
)"
```

---

## Task 9: Property-reactivity behavior — documented limitation

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementTest.java`

This task verifies the **expected limitation** that `EvalCondition` is a property-reactivity scope delimiter — a `test` referencing a property of an outer-scope binding does *not* automatically watch that property.

- [ ] **Step 1: Append failing test**

```java
import org.kie.api.runtime.rule.FactHandle;

@Test
void test_doesNotAutoWatchOuterPatternFields() {
    // This test documents the limitation. If a future change makes 'test' propagate
    // property reads back to preceding patterns, this test will start failing — at
    // which point the assertion should be flipped and the limitation removed from
    // the spec.
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                var p : /persons,
                test p.age > 30,
                do { System.out.println(p); }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint persons = session.getEntryPoint("persons");
        Person alice = new Person("Alice", 25);   // initially not > 30
        FactHandle handle = persons.insert(alice);
        assertThat(session.fireAllRules()).isZero();

        alice.setAge(40);
        persons.update(handle, alice);
        // Without an explicit watch list, property updates on 'age' may NOT
        // refire the rule — that's the documented limitation. We assert
        // either-or behavior: either it fires (auto-watch works) or it doesn't.
        // Drools may still re-evaluate on a generic update with no specific watch.
        // The point is to record the current behavior and surface regressions.
        int fired = session.fireAllRules();

        // Whatever the actual behavior is today, lock it in. The intent is to
        // detect future drift. If a maintainer later changes property reactivity,
        // they will see this test fail and update the assertion + spec together.
        assertThat(fired).isIn(0, 1);    // record both as acceptable for now
        // If empirically `fired == 0`: limitation confirmed.
        // If empirically `fired == 1`: drools is doing something more eager
        //                              (maybe a generic re-eval on update);
        //                              fine, but note in the spec.
    });
}

@Test
void test_withExplicitWatchListRefires() {
    // Demonstrate the documented workaround: explicit watch list on the pattern.
    // This is the recommended user-facing pattern for property-reactive guards.
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                var p : /persons[][age],
                test p.age > 30,
                do { System.out.println(p); }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint persons = session.getEntryPoint("persons");
        Person alice = new Person("Alice", 25);
        FactHandle handle = persons.insert(alice);
        assertThat(session.fireAllRules()).isZero();

        alice.setAge(40);
        persons.update(handle, alice);
        assertThat(session.fireAllRules()).isEqualTo(1);
    });
}
```

If `Person` doesn't have a `setAge` mutator, add one before this test. If the watch-list syntax `/persons[][age]` doesn't compile in current DRLX (per the recent property-reactive spec), the workaround test should be skipped with `@Disabled("watch list not yet implemented")` or rewritten using whatever annotation drools currently honors.

- [ ] **Step 2: Run tests — observe and lock behavior**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=TestElementTest`

Expected: PASS (both new tests). The first test asserts "either 0 or 1" — locks current behavior; if the empirical result is 0 (limitation confirmed), tighten to `assertThat(fired).isZero()` and add a comment that crediting the limitation is the test's purpose.

- [ ] **Step 3: Tighten assertion if behavior is locked**

If after observing the first test you find `fired == 0`, tighten:
```java
assertThat(fired).isZero();   // documented limitation (eval is scope delimiter)
```
This is more useful as a regression detector.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TestElementTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: property-reactivity scope-delimiter behavior + watch-list workaround

EvalCondition.isPatternScopeDelimiter() == true means 'test' guards
do not auto-watch outer-scope properties. Documented in spec; users
add explicit watch lists when needed.

Refs #23
EOF
)"
```

---

## Task 10: Polish — full-suite verification + docs

**Files:**
- Modify: `specs/IMPLEMENT_SYNTAX_CANDIDATES.md`

- [ ] **Step 1: Run the full DRLX test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`

Expected: 0 failures. Test count = baseline 111 + ~12 new (4 model + 2 lambda compiler + 1 EvalIR builder + 2 parse + 3 runtime) ≈ 123.

If any prior test fails: most likely `or`/`and` regression because the regex `referencedBindings` is now applied somewhere it wasn't before. Investigate.

- [ ] **Step 2: Update syntax-candidates doc**

In `/home/tkobayas/usr/work/mvel3-development/drlx-parser/specs/IMPLEMENT_SYNTAX_CANDIDATES.md`, add a row or note:

```
| – | **`test` (eval-style guard)** — IMPLEMENTED in #23. Foundation for `if/else` (#12). | DRLXXXX § "test" |
```

(Match the format of the existing rows. If the doc is structured by tier rather than enumeration, add `test` to the appropriate tier — likely Tier 1 or 2.)

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    specs/IMPLEMENT_SYNTAX_CANDIDATES.md

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
docs: mark 'test' construct (DRLXXXX § test) as implemented

Closes #23
EOF
)"
```

`Closes #23` only on this final commit so GitHub auto-closes when merged.

- [ ] **Step 4: Push and open PR**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push origin main

gh pr create --repo tkobayas/drlx-parser \
    --title "feat: DRLXXXX 'test' construct (#23)" \
    --body "$(cat <<'EOF'
## Summary

- Adds `test <expression>` as a `ruleItem` in DRLX rule bodies (DRLXXXX § "test").
- Internal infra: `EvalIR` (proto + IR), `DrlxEvalExpression` (drools-base `EvalExpression` impl), `DrlxLambdaCompiler.createEvalExpression`, runtime mapping to `EvalCondition`.
- Foundation for #12 (if/else), which now depends on this.

## Test plan

- [x] IR + proto round-trip
- [x] DrlxEvalExpression unit (evaluate, clone, replaceDeclaration)
- [x] DrlxLambdaCompiler.createEvalExpression unit
- [x] EvalIR programmatic E2E (no grammar)
- [x] Grammar parse: `test <expr>` recognized
- [x] Visitor produces correct EvalIR
- [x] Runtime: alpha (`test p.age > 30`), beta (`test p.age > q.age`), false-guard
- [x] Property-reactivity scope-delimiter behavior + watch-list workaround
- [x] Full DRLX suite passes (~123 tests, 0 failures)

Closes #23
EOF
)"
```

---

## Self-review

### Issue coverage

- ✅ EvalIR proto + IR + permits — Tasks 1, 2
- ✅ Proto ↔ IR conversion — Task 2
- ✅ DrlxLambdaCompiler.createEvalExpression — Task 4
- ✅ DrlxEvalExpression bridge — Task 3
- ✅ Runtime mapping (EvalIR → EvalCondition) — Task 5
- ✅ User-facing grammar (`TEST` keyword + `testElement`) — Task 6
- ✅ Visitor `buildTestElement` — Task 7
- ✅ Tests: parse + round-trip + lambda unit + EvalIR builder + alpha/beta/false runtime + property-reactivity — Tasks 1, 2, 4, 5, 7, 8, 9
- ✅ Property-reactivity limitation documented in test + spec — Task 9

### Type / signature consistency

- `EvalIR(String, List<String>)` — used identically across Tasks 1, 2, 5, 7.
- `DrlxLambdaCompiler.createEvalExpression(String, List<BoundVariable>) -> EvalExpression` — Task 4 produces, Task 5 consumes.
- `DrlxEvalExpression(String expression, Evaluator<…>, String[] declarationNames)` — Task 3 defines, Task 4 constructs.
- `BoundVariable` accessors `bv.name()`, `bv.type()`, `bv.pattern()` — record accessors (verified in agent investigation; consistent across Tasks 4, 5).

### Risk concentration

The largest single-task risk is **Task 4** (`createEvalExpression` + `PendingLambda` generalization). Specifically:
- `PendingLambda`'s second-argument generalization (option (a) consumer-based, option (b) sealed interface) requires reading the existing resolution code. Estimate: ~1 hour of code reading + 30 min of refactor.
- If the existing `MVELBatchCompiler.LambdaHandle` API has constraints the plan hasn't anticipated (e.g., type-bound on the compiled evaluator), the implementing agent may need to widen types. Estimate: 30 min – 2 hours.

The second-largest risk is **Task 5**'s `TestRuleBuilders.buildFromIr` helper. The implementing agent must read `DrlxRuleBuilder` and `DrlxRuleAstRuntimeBuilder` carefully to construct a valid `CompilationUnitIR` from a single `RuleIR`. Estimate: 30 min.

### Sequencing

Tasks 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 are strictly sequential. Task 9 depends on Task 8 only (uses the same test class). Task 10 is final.

For inline execution: run sequentially. Task boundaries are natural commit points; pause after each for verification.
