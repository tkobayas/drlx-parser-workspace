# `if` / `else` branching (Form A) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Form A of DRLXXXX `if` / `else if` / `else` in DRLX rule body — `if (cond) { ruleItems… } …` with a single trailing `do` consequence — by desugaring in the visitor to `or(and(test(guard), body…), …)` over the existing `or` / `and` infrastructure plus a new internal `EvalIR` (the DRLXXXX `test` construct, internal-only).

**Architecture:** Visitor-level desugar. Grammar parses `if/else if/else`; `DrlxToRuleAstVisitor.buildConditionalBranch` immediately rewrites to `GroupElementIR(OR, [ GroupElementIR(AND, [ EvalIR(guard_or_negation), …branch_body_IRs ]) ])`. Cumulative guards are split into multiple sequential `EvalIR` nodes (one per condition or negation) rather than one compound expression — simpler to build, equivalent at runtime. New `EvalIR` carries (expression text, referenced binding names) and is mapped at runtime-build time to a drools-core `EvalCondition`. **No drools-core changes; no drools-model `expr(...)` use** (the runtime builder works with drools-core directly).

**Tech Stack:** Java 21 (records, sealed interfaces, switch expressions), ANTLR 4, drools-core 10.1.0, drools-base 10.1.0, MVEL3 3.0.0-SNAPSHOT (lambda compiler), Protocol Buffers 3 (proto3 oneof), JUnit 5, AssertJ.

---

## File map

**Modify:**
- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4` — add `IF`, `ELSE` tokens
- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` — add `conditionalBranch` + `branchBody` productions; add to `ruleItem` alternatives
- `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` — add `EvalParseResult` message; add `eval = 3` to `LhsItemParseResult.kind` oneof
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` — add `EvalIR` record; extend `LhsItemIR permits` clause
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` — extend `fromProtoLhs` and `toProtoLhs` for `EvalIR`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` — add `buildConditionalBranch` method; add dispatch in `buildRule`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` — add `EvalIR` branch in `buildLhs` switch; emit drools-core `EvalCondition`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java` — add `createEvalExpression(String expression, List<BoundVariable> referencedBindings)` returning an `EvalExpression`

**Create:**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java` — pure parse-tree assertions
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java` — runtime semantics (extends `DrlxBuilderTestSupport`)
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java` — programmatic `EvalIR` built without grammar (verifies runtime mapping in isolation)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Customer.java` — test fact (credit rating)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Product.java` — test fact (rate)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Rating.java` — enum (LOW, MEDIUM, HIGH)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Rates.java` — enum (LOW, MEDIUM, HIGH)
- `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/CreditUnit.java` — `DataStore<Customer> customers; DataStore<Product> products;`

**Reuse (read-only):**
- `Person`, `MyUnit` from existing tests (some early tasks may use them instead of new domain types if parity is simpler)

---

## Build & test commands

- Generate ANTLR + compile + run tests for drlx-parser-core only:
  - `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml clean install`
- Run a single test class:
  - `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest`
- Run full DRLX test suite:
  - `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`
- Force ANTLR re-generation only:
  - `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml clean generate-sources`

All `git` commands use `git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser <cmd>` — never `cd`.

---

## Task 1: Grammar — add `IF` / `ELSE` tokens and `conditionalBranch` production

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4`
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java` (new)

- [ ] **Step 1: Write failing parse-only test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java
package org.drools.drlx.builder.syntax;

import org.antlr.v4.runtime.CharStreams;
import org.antlr.v4.runtime.CommonTokenStream;
import org.drools.drlx.parser.DrlxLexer;
import org.drools.drlx.parser.DrlxParser;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class IfElseParseTest {

    @Test
    void parsesBinaryIfElse() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    if (p.age > 60) {
                        var s : /seniors[ age == p.age ]
                    } else {
                        var j : /juniors[ age == p.age ]
                    },
                    do { System.out.println(p); }
                }
                """;
        DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(rule));
        DrlxParser parser = new DrlxParser(new CommonTokenStream(lexer));
        DrlxParser.CompilationUnitContext ctx = parser.compilationUnit();
        assertThat(parser.getNumberOfSyntaxErrors()).isZero();
        // Walk to the conditionalBranch node and assert it exists
        DrlxParser.RuleDeclarationContext rule0 = ctx.ruleDeclaration(0);
        assertThat(rule0.ruleBody().ruleItem()).anyMatch(
                ri -> ri.conditionalBranch() != null);
    }

    @Test
    void parsesElseIfChain() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    if (p.age > 60) { var s : /seniors[ age == p.age ] }
                    else if (p.age > 30) { var a : /adults[ age == p.age ] }
                    else { var j : /juniors[ age == p.age ] },
                    do { System.out.println(p); }
                }
                """;
        DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(rule));
        DrlxParser parser = new DrlxParser(new CommonTokenStream(lexer));
        parser.compilationUnit();
        assertThat(parser.getNumberOfSyntaxErrors()).isZero();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest`

Expected: FAIL (compilation error — `RuleItemContext.conditionalBranch()` does not exist; `IF`/`ELSE` tokens unknown).

- [ ] **Step 3: Add lexer tokens**

In `DrlxLexer.g4`, add (alongside `NOT`, `EXISTS`, `DRLX_AND`, `DRLX_OR`):

```antlr
IF       : 'if';
ELSE     : 'else';
```

- [ ] **Step 4: Add parser productions**

In `DrlxParser.g4`, modify `ruleItem`:

```antlr
ruleItem
    : rulePattern
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | conditionalBranch ','
    | ruleConsequence
    ;
```

Add new productions (place after `orElement`):

```antlr
conditionalBranch
    : IF '(' expression ')' branchBody
      ( ELSE IF '(' expression ')' branchBody )*
      ( ELSE branchBody )?
    ;

branchBody
    : '{' ruleItem* '}'
    ;
```

(Use the same single-quoted brace/paren tokens that `notElement` uses — do not introduce new lexer rules for `{`/`}` if `notElement` already inlines them.)

- [ ] **Step 5: Run ANTLR generation + test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest`

Expected: PASS. ANTLR regenerates `DrlxParser.java` with `RuleItemContext.conditionalBranch()` and `ConditionalBranchContext` / `BranchBodyContext` classes.

- [ ] **Step 6: Verify no regression in existing parse tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`

Expected: All previously-passing tests still pass (111 tests, 0 failures per HANDOFF.md baseline). Two new tests in `IfElseParseTest` are passing.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4 \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
grammar: add IF / ELSE tokens and conditionalBranch production

Form A of DRLXXXX if/else. Branch body reuses ruleItem* so nested
if/else, group elements, and patterns inside branches all work.
Visitor desugaring is a separate task.

Refs #12
EOF
)"
```

---

## Task 2: Add `EvalIR` record + sealed interface permits

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`

- [ ] **Step 1: Write failing test**

Add to `IfElseParseTest.java` (or create a new model-level test class — choose `DrlxRuleAstModelTest.java` if neither exists):

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
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstModelTest`

Expected: FAIL — `EvalIR` class does not exist.

- [ ] **Step 3: Add `EvalIR` record + extend permits**

In `DrlxRuleAstModel.java`:

```java
// Replace existing line:  public sealed interface LhsItemIR permits PatternIR, GroupElementIR {}
public sealed interface LhsItemIR permits PatternIR, GroupElementIR, EvalIR {
}

// Append to the file (after GroupElementIR):
public record EvalIR(String expression, List<String> referencedBindings) implements LhsItemIR {
    public EvalIR {
        // Defensive copy / canonicalization
        referencedBindings = List.copyOf(referencedBindings);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstModelTest`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
ir: add EvalIR record (DRLXXXX 'test' construct, internal)

Used by the if/else desugar to carry guard expressions. Not yet
exposed in user-facing grammar (#23 will).

Refs #12
EOF
)"
```

---

## Task 3: Proto — add `EvalParseResult` message + extend round-trip

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`

- [ ] **Step 1: Write failing round-trip test**

```java
// Append to DrlxRuleAstModelTest.java
@Test
void evalIrRoundTripsThroughProto() {
    EvalIR original = new EvalIR("p.age > 30", List.of("p"));
    GroupElementIR andGroup = new GroupElementIR(
            GroupElementIR.Kind.AND,
            List.of(original));

    // Round-trip: IR → proto → IR
    var proto = DrlxRuleAstParseResult.toProtoLhs(andGroup);
    LhsItemIR roundTripped = DrlxRuleAstParseResult.fromProtoLhs(
            proto, java.nio.file.Path.of("test.drlx"));

    assertThat(roundTripped).isInstanceOf(GroupElementIR.class);
    GroupElementIR g = (GroupElementIR) roundTripped;
    assertThat(g.children()).hasSize(1);
    assertThat(g.children().get(0)).isInstanceOf(EvalIR.class);
    EvalIR e = (EvalIR) g.children().get(0);
    assertThat(e.expression()).isEqualTo("p.age > 30");
    assertThat(e.referencedBindings()).containsExactly("p");
}
```

(If `toProtoLhs` / `fromProtoLhs` are package-private in `DrlxRuleAstParseResult`, place this test in the same `org.drools.drlx.builder` package — which it already is.)

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstModelTest#evalIrRoundTripsThroughProto`

Expected: FAIL — `toProtoLhs` throws "Unsupported LHS item: EvalIR…".

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

- [ ] **Step 4: Extend `fromProtoLhs` and `toProtoLhs`**

In `DrlxRuleAstParseResult.java`, in the `fromProtoLhs` switch (currently has `PATTERN`, `GROUP`, `KIND_NOT_SET`), add:

```java
case EVAL -> {
    DrlxRuleAstProto.EvalParseResult eval = item.getEval();
    yield new EvalIR(
            eval.getExpression(),
            List.copyOf(eval.getReferencedBindingsList()));
}
```

In `toProtoLhs`, after the `GroupElementIR` branch and before the `else` throw:

```java
} else if (item instanceof EvalIR e) {
    DrlxRuleAstProto.EvalParseResult.Builder eb =
            DrlxRuleAstProto.EvalParseResult.newBuilder()
                    .setExpression(e.expression());
    e.referencedBindings().forEach(eb::addReferencedBindings);
    builder.setEval(eb);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstModelTest#evalIrRoundTripsThroughProto`

Expected: PASS. Maven regenerates the proto Java sources via `protoc` (run during `generate-sources`).

- [ ] **Step 6: Verify no regression**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`

Expected: 113 tests pass (111 baseline + 2 from Task 1 + 2 from Tasks 2–3 — adjust upward as we add tests).

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstModelTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
proto: serialize EvalIR (eval = 3 in LhsItemParseResult oneof)

Refs #12
EOF
)"
```

---

## Task 4: `DrlxLambdaCompiler.createEvalExpression` — guard-lambda generation

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaCompilerTest.java` (new — or append if it exists)

**Goal:** Add a method that compiles a boolean expression text + referenced bindings into an `org.drools.base.rule.EvalExpression` ready to plug into `EvalCondition`.

**Approach:** Reuse the existing beta-constraint pipeline (which already produces a boolean evaluator over a `Map<String,Object>` of bindings) and wrap it in an adapter that implements `EvalExpression.evaluate(Tuple, Declaration[], ReteEvaluator) -> boolean`.

- [ ] **Step 1: Write failing test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaCompilerTest.java
package org.drools.drlx.builder;

import java.util.List;
import org.drools.base.rule.EvalCondition;
import org.drools.base.rule.EvalExpression;
import org.drools.drlx.domain.Person;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class DrlxLambdaCompilerTest {

    @Test
    void createEvalExpressionForBoundVariable() {
        DrlxLambdaCompiler compiler = new DrlxLambdaCompiler();
        // Simulate "p" already bound to a Person
        BoundVariable pBinding = BoundVariableTestFactory.bind("p", Person.class);

        EvalExpression eval = compiler.createEvalExpression(
                "p.age > 30",
                List.of(pBinding));

        assertThat(eval).isNotNull();
        // Build a single-fact tuple via the existing test helper, evaluate
        boolean result = EvalExpressionTestHelper.evaluate(
                eval, java.util.Map.of("p", new Person("Alice", 40)));
        assertThat(result).isTrue();

        boolean result2 = EvalExpressionTestHelper.evaluate(
                eval, java.util.Map.of("p", new Person("Bob", 25)));
        assertThat(result2).isFalse();
    }
}
```

`BoundVariableTestFactory` and `EvalExpressionTestHelper` are package-private helpers — you may need to add them in the same task. Keep them minimal: `BoundVariableTestFactory.bind(name, type)` constructs a `BoundVariable` with a synthetic `Pattern`; `EvalExpressionTestHelper.evaluate(EvalExpression, Map<String,Object>)` builds a `Tuple` containing each map entry as a fact handle and invokes `eval.evaluate(...)`. If creating these helpers turns out to require >30 lines, simplify the test to call `compiler.createEvalExpression(...)` and just assert non-null + check the produced lambda directly via reflection on its internal `Predicate`.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxLambdaCompilerTest`

Expected: FAIL — `createEvalExpression` does not exist.

- [ ] **Step 3: Implement `createEvalExpression`**

In `DrlxLambdaCompiler.java`, add (placement: near `createBetaLambdaConstraint`):

```java
public EvalExpression createEvalExpression(String expression,
                                           List<BoundVariable> referencedBindings) {
    // Reuse the same MVEL3 batch-compilation infra used by createBetaLambdaConstraint.
    // 1. Build a Map<String, Type<?>> declaration map keyed by binding name.
    Map<String, org.mvel3.transpiler.context.Declaration<?>> declarations =
            new java.util.LinkedHashMap<>();
    for (BoundVariable bv : referencedBindings) {
        declarations.put(bv.name(),
                org.mvel3.transpiler.context.Declaration.of(bv.name(), bv.type()));
    }

    // 2. Compile the boolean expression to an MVEL3 Evaluator<Map<String,Object>, Void, Boolean>.
    org.mvel3.transpiler.api.Evaluator<Map<String, Object>, Void, Boolean> evaluator =
            internalCompileBooleanLambda(expression, declarations);

    // 3. Wrap into EvalExpression.
    String[] declarationNames = referencedBindings.stream()
            .map(BoundVariable::name)
            .toArray(String[]::new);
    return new DrlxEvalExpression(evaluator, declarationNames);
}
```

Add the wrapper class as a package-private nested or sibling class:

```java
final class DrlxEvalExpression implements EvalExpression {
    private final org.mvel3.transpiler.api.Evaluator<Map<String, Object>, Void, Boolean> evaluator;
    private final String[] declarationNames;

    DrlxEvalExpression(org.mvel3.transpiler.api.Evaluator<Map<String, Object>, Void, Boolean> evaluator,
                       String[] declarationNames) {
        this.evaluator = evaluator;
        this.declarationNames = declarationNames;
    }

    @Override
    public boolean evaluate(Tuple tuple,
                            Declaration[] requiredDeclarations,
                            ReteEvaluator reteEvaluator) {
        Map<String, Object> input = new java.util.HashMap<>(requiredDeclarations.length);
        for (Declaration d : requiredDeclarations) {
            Object value = d.getValue(reteEvaluator, tuple.getObject(d));
            input.put(d.getIdentifier(), value);
        }
        return Boolean.TRUE.equals(evaluator.eval(input));
    }

    @Override
    public Object createContext() { return null; }

    @Override
    public EvalExpression clone() { return this; /* stateless */ }

    @Override
    public void replaceDeclaration(Declaration declaration, Declaration resolved) {
        // No-op — DrlxEvalExpression carries names, not Declaration refs.
    }
}
```

`internalCompileBooleanLambda` is the existing private MVEL3-based compile method (the same one used by `createBetaLambdaConstraint`). If its signature is private/different, factor it out so both alpha/beta constraint paths and the new eval path share it. Check existing code; if a refactor is needed, do it as a minimal extraction (no behavior change for existing callers).

If `Tuple.getObject(Declaration)` is not the correct API for resolving a declaration's fact in current drools, adjust to whichever method the existing pattern-constraint evaluation uses (look at how `DrlxLambdaConstraint.isAllowedCachedRight` or similar resolves declaration values).

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxLambdaCompilerTest`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxEvalExpression.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaCompilerTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
lambda: createEvalExpression for guard expressions

Compiles a boolean MVEL3 expression over named declarations into a
drools-base EvalExpression suitable for plugging into EvalCondition.
Used by EvalIR runtime mapping.

Refs #12
EOF
)"
```

---

## Task 5: Runtime builder — `EvalIR` → `EvalCondition`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java` (new)

**Goal:** When `buildLhs` encounters an `EvalIR`, build an `EvalCondition` with the compiled `EvalExpression` and the referenced declarations, and add it to the parent `GroupElement`.

- [ ] **Step 1: Write failing end-to-end test (programmatic IR — no grammar)**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java
package org.drools.drlx.builder.syntax;

import java.util.List;
import org.drools.drlx.builder.EvalIR;
import org.drools.drlx.builder.GroupElementIR;
import org.drools.drlx.builder.PatternIR;
import org.drools.drlx.builder.RuleIR;
import org.drools.drlx.builder.ConsequenceIR;
import org.drools.drlx.builder.LhsItemIR;
import org.drools.drlx.builder.DrlxRuleAstRuntimeBuilder;
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
        // Build IR programmatically: rule R { var p : /persons, eval(p.age > 30), do { ... } }
        PatternIR pattern = new PatternIR(
                "Person", "p", "persons",
                List.of(),       // no constraints on p
                null, List.of(), false, List.of());
        EvalIR eval = new EvalIR("p.age > 30", List.of("p"));
        ConsequenceIR rhs = new ConsequenceIR("System.out.println(p);");

        RuleIR rule = new RuleIR(
                "R", List.of(),
                List.of((LhsItemIR) pattern, (LhsItemIR) eval),
                rhs);

        // Wrap in CompilationUnitIR (or the equivalent top-level) and build.
        // Use the existing DrlxRuleBuilder helper if it accepts an IR directly,
        // otherwise call the runtime builder directly.
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

`TestRuleBuilders.buildFromIr` is a small new test helper. If the existing `DrlxRuleBuilder` does not accept an IR directly, expose a small package-private path that calls `DrlxRuleAstRuntimeBuilder` with a `CompilationUnitIR` containing the single rule. Keep the helper minimal.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=EvalIRBuilderTest`

Expected: FAIL — `buildLhs` throws `IllegalArgumentException("Unsupported LHS item: org.drools.drlx.builder.EvalIR")`.

- [ ] **Step 3: Add `EvalIR` branch in `buildLhs`**

In `DrlxRuleAstRuntimeBuilder.java`, in the `buildLhs` for-loop, after the `GroupElementIR` branch and before the `else throw`:

```java
} else if (item instanceof EvalIR evalIr) {
    // Resolve referenced declarations from boundVariables map.
    List<BoundVariable> referenced = new java.util.ArrayList<>();
    Declaration[] declarations = new Declaration[evalIr.referencedBindings().size()];
    int i = 0;
    for (String name : evalIr.referencedBindings()) {
        BoundVariable bv = boundVariables.get(name);
        if (bv == null) {
            throw new RuntimeException(
                    "Eval references unknown binding '" + name + "'");
        }
        referenced.add(bv);
        declarations[i++] = bv.pattern().getDeclaration();
    }

    // Compile guard expression to EvalExpression.
    org.drools.base.rule.EvalExpression evalExpression =
            lambdaCompiler.createEvalExpression(evalIr.expression(), referenced);

    // Build EvalCondition and attach to parent.
    org.drools.base.rule.EvalCondition evalCondition =
            new org.drools.base.rule.EvalCondition(evalExpression, declarations);
    parent.addChild(evalCondition);
}
```

(`Declaration` here is `org.drools.base.rule.Declaration` — same import already present in this file.)

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=EvalIRBuilderTest`

Expected: PASS.

- [ ] **Step 5: Verify full suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`

Expected: All tests pass; new test count includes `EvalIRBuilderTest`.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/EvalIRBuilderTest.java

# Also add TestRuleBuilders helper if created:
# git -C ... add drlx-parser-core/src/test/java/.../TestRuleBuilders.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
runtime: build EvalCondition from EvalIR

Resolves referenced bindings via boundVariables map, compiles guard
expression with DrlxLambdaCompiler.createEvalExpression, and attaches
the resulting EvalCondition to the parent GroupElement.

Refs #12
EOF
)"
```

---

## Task 6: Visitor — `buildConditionalBranch` desugar

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java` (extend with IR-shape assertion)

**Goal:** Translate `if (c1) { B1 } else if (c2) { B2 } else { B3 }` into:

```
GroupElementIR(OR, [
  GroupElementIR(AND, [ EvalIR(c1),                B1... ]),
  GroupElementIR(AND, [ EvalIR(!(c1)), EvalIR(c2), B2... ]),
  GroupElementIR(AND, [ EvalIR(!(c1)), EvalIR(!(c2)), B3... ]),
])
```

Cumulative guards split into multiple sequential `EvalIR` nodes (per the spec's plan-phase decision).

- [ ] **Step 1: Write failing IR-shape test**

Append to `IfElseParseTest.java`:

```java
import java.util.List;
import org.drools.drlx.builder.DrlxRuleAstParseResult;
import org.drools.drlx.builder.EvalIR;
import org.drools.drlx.builder.GroupElementIR;
import org.drools.drlx.builder.LhsItemIR;
import org.drools.drlx.builder.PatternIR;
import org.drools.drlx.builder.RuleIR;

@Test
void desugarsBinaryIfElseToOrAnd() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                var p : /persons,
                if (p.age > 30) { var s : /seniors[ age == p.age ] }
                else { var j : /juniors[ age == p.age ] },
                do { System.out.println(p); }
            }
            """;
    RuleIR ruleIr = DrlxRuleAstParseResult.parseSingleRule(rule);
    // LHS: [ Pattern(p), GroupElementIR(OR, [...]) ]
    List<LhsItemIR> lhs = ruleIr.lhs();
    assertThat(lhs).hasSize(2);
    assertThat(lhs.get(1)).isInstanceOf(GroupElementIR.class);
    GroupElementIR or = (GroupElementIR) lhs.get(1);
    assertThat(or.kind()).isEqualTo(GroupElementIR.Kind.OR);
    assertThat(or.children()).hasSize(2);

    // Branch 1: AND of [EvalIR("p.age > 30"), Pattern(s)]
    GroupElementIR branch1 = (GroupElementIR) or.children().get(0);
    assertThat(branch1.kind()).isEqualTo(GroupElementIR.Kind.AND);
    assertThat(branch1.children().get(0)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).isEqualTo("p.age > 30"));
    assertThat(branch1.children().get(1)).isInstanceOf(PatternIR.class);

    // Branch 2: AND of [EvalIR("!(p.age > 30)"), Pattern(j)]
    GroupElementIR branch2 = (GroupElementIR) or.children().get(1);
    assertThat(branch2.kind()).isEqualTo(GroupElementIR.Kind.AND);
    assertThat(branch2.children().get(0)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).isEqualTo("!(p.age > 30)"));
    assertThat(branch2.children().get(1)).isInstanceOf(PatternIR.class);
}
```

If `DrlxRuleAstParseResult.parseSingleRule` does not exist, use whichever public/package-private entry point already drives the visitor (likely something like `new DrlxRuleParseRunner().parse(text)` — match what existing tests use; consult `OrTest`'s setup if needed).

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest#desugarsBinaryIfElseToOrAnd`

Expected: FAIL — `buildRule` throws `"Unsupported rule item:"` because `buildConditionalBranch` does not exist.

- [ ] **Step 3: Add dispatch in `buildRule`**

In `DrlxToRuleAstVisitor.java`, in `buildRule`'s if-chain (after the `orElement` branch, before the `else throw`):

```java
} else if (itemCtx.conditionalBranch() != null) {
    lhs.add(buildConditionalBranch(itemCtx.conditionalBranch()));
}
```

- [ ] **Step 4: Implement `buildConditionalBranch`**

Add new method (placement: after `buildOrElement`):

```java
private GroupElementIR buildConditionalBranch(DrlxParser.ConditionalBranchContext ctx) {
    // Collect (condition, body) pairs in document order.
    record BranchInput(String condition, DrlxParser.BranchBodyContext body) {}
    List<BranchInput> branches = new java.util.ArrayList<>();

    // Primary 'if'
    branches.add(new BranchInput(extractExpression(ctx.expression(0)), ctx.branchBody(0)));

    // 'else if' chain — ANTLR exposes them as repeated expression(i) and branchBody(i)
    int branchCount = ctx.branchBody().size();
    int conditionCount = ctx.expression().size();
    for (int i = 1; i < conditionCount; i++) {
        branches.add(new BranchInput(extractExpression(ctx.expression(i)), ctx.branchBody(i)));
    }

    // Trailing 'else' (no condition)
    boolean hasFinalElse = branchCount > conditionCount;
    DrlxParser.BranchBodyContext finalElseBody = hasFinalElse
            ? ctx.branchBody(branchCount - 1)
            : null;

    // Reject empty branches.
    for (BranchInput b : branches) {
        if (b.body().ruleItem().isEmpty()) {
            throw new RuntimeException("empty branch body");
        }
    }
    if (hasFinalElse && finalElseBody.ruleItem().isEmpty()) {
        throw new RuntimeException("empty branch body");
    }

    // Build OR group.
    List<LhsItemIR> orChildren = new java.util.ArrayList<>();
    List<String> priorConditions = new java.util.ArrayList<>();
    for (BranchInput b : branches) {
        List<LhsItemIR> andChildren = new java.util.ArrayList<>();
        // Negations of all prior conditions, in order.
        for (String prior : priorConditions) {
            andChildren.add(new EvalIR("!(" + prior + ")", referencedBindings(prior)));
        }
        // Own condition.
        andChildren.add(new EvalIR(b.condition(), referencedBindings(b.condition())));
        // Branch body items, recursively.
        for (DrlxParser.RuleItemContext ri : b.body().ruleItem()) {
            andChildren.add(visitBranchRuleItem(ri));
        }
        orChildren.add(new GroupElementIR(GroupElementIR.Kind.AND, List.copyOf(andChildren)));
        priorConditions.add(b.condition());
    }
    if (hasFinalElse) {
        List<LhsItemIR> andChildren = new java.util.ArrayList<>();
        for (String prior : priorConditions) {
            andChildren.add(new EvalIR("!(" + prior + ")", referencedBindings(prior)));
        }
        for (DrlxParser.RuleItemContext ri : finalElseBody.ruleItem()) {
            andChildren.add(visitBranchRuleItem(ri));
        }
        orChildren.add(new GroupElementIR(GroupElementIR.Kind.AND, List.copyOf(andChildren)));
    }
    return new GroupElementIR(GroupElementIR.Kind.OR, List.copyOf(orChildren));
}

private LhsItemIR visitBranchRuleItem(DrlxParser.RuleItemContext itemCtx) {
    if (itemCtx.rulePattern() != null)        return buildPattern(itemCtx.rulePattern());
    if (itemCtx.notElement() != null)         return buildNotElement(itemCtx.notElement());
    if (itemCtx.existsElement() != null)      return buildExistsElement(itemCtx.existsElement());
    if (itemCtx.andElement() != null)         return buildAndElement(itemCtx.andElement());
    if (itemCtx.orElement() != null)          return buildOrElement(itemCtx.orElement());
    if (itemCtx.conditionalBranch() != null)  return buildConditionalBranch(itemCtx.conditionalBranch());
    if (itemCtx.ruleConsequence() != null) {
        throw new RuntimeException("'do' is not allowed inside an if/else branch (Form A)");
    }
    throw new IllegalArgumentException("Unsupported branch rule item: " + itemCtx.getText());
}

private String extractExpression(DrlxParser.ExpressionContext ctx) {
    return ctx.getText();   // raw text — same approach used by existing constraint parsing
}

private List<String> referencedBindings(String expression) {
    // Extract identifier candidates and intersect with currently-known bindings.
    // For Task 6, simplest correct answer: regex-scan for identifiers, return all.
    // The runtime builder later filters via boundVariables map.
    java.util.regex.Pattern idRegex = java.util.regex.Pattern.compile(
            "\\b([a-zA-Z_][a-zA-Z0-9_]*)\\b");
    java.util.regex.Matcher m = idRegex.matcher(expression);
    java.util.LinkedHashSet<String> identifiers = new java.util.LinkedHashSet<>();
    while (m.find()) identifiers.add(m.group(1));
    // Filter out Java keywords / known type names if needed; for now return all.
    return List.copyOf(identifiers);
}
```

If the existing visitor already has a `buildGroupChild`-style dispatcher that covers the same set of `ruleItem` alternatives, reuse it instead of reimplementing as `visitBranchRuleItem`. Check `buildGroupElementFromChildren` first.

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest`

Expected: PASS (all 3 tests in `IfElseParseTest`).

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
visitor: desugar conditionalBranch to OR(AND(eval, body...))

Cumulative guards split into multiple sequential EvalIRs (one per
prior-condition negation + own condition). Empty branch bodies are
rejected. Nested if/else handled by recursion through visitBranchRuleItem.

Refs #12
EOF
)"
```

---

## Task 7: Runtime — binary `if/else` end-to-end

**Files:**
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java` (new)
- Test fixtures: `Customer`, `Product`, `Rating`, `Rates`, `CreditUnit` (new)

- [ ] **Step 1: Add domain test fixtures**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/domain/Rating.java
package org.drools.drlx.domain;
public enum Rating { LOW, MEDIUM, HIGH }
```

```java
// drlx-parser-core/src/test/java/org/drools/drlx/domain/Rates.java
package org.drools.drlx.domain;
public enum Rates { LOW, MEDIUM, HIGH }
```

```java
// drlx-parser-core/src/test/java/org/drools/drlx/domain/Customer.java
package org.drools.drlx.domain;
public class Customer {
    private final String name;
    private Rating creditRating;
    public Customer(String name, Rating creditRating) {
        this.name = name; this.creditRating = creditRating;
    }
    public String getName() { return name; }
    public Rating getCreditRating() { return creditRating; }
    public void setCreditRating(Rating r) { this.creditRating = r; }
    @Override public String toString() { return name; }
}
```

```java
// drlx-parser-core/src/test/java/org/drools/drlx/domain/Product.java
package org.drools.drlx.domain;
public class Product {
    private final String name;
    private final Rates rate;
    public Product(String name, Rates rate) { this.name = name; this.rate = rate; }
    public String getName() { return name; }
    public Rates getRate() { return rate; }
    @Override public String toString() { return name; }
}
```

```java
// drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/CreditUnit.java
package org.drools.drlx.ruleunit;

import org.drools.ruleunits.api.DataStore;
import org.drools.ruleunits.api.RuleUnitData;
import org.drools.ruleunits.impl.datasources.DataStoreImpl;
import org.drools.drlx.domain.Customer;
import org.drools.drlx.domain.Product;

public class CreditUnit implements RuleUnitData {
    private final DataStore<Customer> customers = new DataStoreImpl<>();
    private final DataStore<Product> products = new DataStoreImpl<>();
    public DataStore<Customer> getCustomers() { return customers; }
    public DataStore<Product> getProducts() { return products; }
}
```

(Match the exact `RuleUnitData` / `DataStore` / `DataStoreImpl` import paths used by the existing `MyUnit` — read that file before writing this and copy its package conventions.)

- [ ] **Step 2: Write failing runtime test (binary if/else)**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.builder.DrlxBuilderTestSupport;
import org.drools.drlx.domain.Customer;
import org.drools.drlx.domain.Product;
import org.drools.drlx.domain.Rates;
import org.drools.drlx.domain.Rating;
import org.junit.jupiter.api.Test;
import org.kie.api.runtime.rule.EntryPoint;
import static org.assertj.core.api.Assertions.assertThat;

class IfElseTest extends DrlxBuilderTestSupport {

    @Test
    void binaryIfElse_lowRating_picksHighRateProduct() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ]
                    } else {
                        var p : /products[ rate == Rates.LOW ]
                    },
                    do { System.out.println(c + " " + p); }
                }
                """;

        withSession(rule, (session, listener) -> {
            EntryPoint customers = session.getEntryPoint("customers");
            EntryPoint products = session.getEntryPoint("products");
            customers.insert(new Customer("Alice", Rating.LOW));
            products.insert(new Product("luxury", Rates.HIGH));
            products.insert(new Product("budget", Rates.LOW));
            assertThat(session.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }

    @Test
    void binaryIfElse_highRating_picksLowRateProduct() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ]
                    } else {
                        var p : /products[ rate == Rates.LOW ]
                    },
                    do { System.out.println(c + " " + p); }
                }
                """;

        withSession(rule, (session, listener) -> {
            EntryPoint customers = session.getEntryPoint("customers");
            EntryPoint products = session.getEntryPoint("products");
            customers.insert(new Customer("Bob", Rating.HIGH));
            products.insert(new Product("luxury", Rates.HIGH));
            products.insert(new Product("budget", Rates.LOW));
            assertThat(session.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }
}
```

- [ ] **Step 3: Run test to verify it fails / passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest`

Expected: PASS (the desugar from Task 6 + runtime mapping from Task 5 should make this work end-to-end).

If FAIL: diagnose. Common failure modes:
- Guard expression `c.creditRating == Rating.LOW` not resolving `c` — referencedBindings logic missed `c` (Task 6 regex). Fix the regex.
- `EvalCondition` declarations not aligned with `boundVariables` — check the runtime builder (Task 5) declaration array order matches what `EvalExpression.evaluate` expects.
- Same-name `p` across branches conflicts — check that `or` is decomposing into subrules; if not, this might surface a deeper drools-core configuration question.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/domain/Customer.java \
    drlx-parser-core/src/test/java/org/drools/drlx/domain/Product.java \
    drlx-parser-core/src/test/java/org/drools/drlx/domain/Rating.java \
    drlx-parser-core/src/test/java/org/drools/drlx/domain/Rates.java \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/CreditUnit.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: binary if/else end-to-end (DRLXXXX A example)

Refs #12
EOF
)"
```

---

## Task 8: Runtime — `else if` chain

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

- [ ] **Step 1: Append failing test**

```java
@Test
void elseIfChain_threeRatings() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R1 {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ]
                } else if (c.creditRating == Rating.MEDIUM) {
                    var p : /products[ rate == Rates.MEDIUM ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { System.out.println(c + " " + p); }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.MEDIUM));
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("standard", Rates.MEDIUM));
        products.insert(new Product("budget", Rates.LOW));
        assertThat(session.fireAllRules()).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("R1");
    });
}
```

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest#elseIfChain_threeRatings`

Expected: PASS (no implementation changes needed — chain handling already in Task 6).

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: else-if chain (3 ratings)

Refs #12
EOF
)"
```

---

## Task 9: Runtime — no-final-else, rule does not fire

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

- [ ] **Step 1: Append failing test**

```java
@Test
void noFinalElse_doesNotFireWhenNoBranchMatches() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R1 {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ]
                } else if (c.creditRating == Rating.MEDIUM) {
                    var p : /products[ rate == Rates.MEDIUM ]
                },
                do { System.out.println(c + " " + p); }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Carol", Rating.HIGH));   // matches no branch
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("standard", Rates.MEDIUM));
        assertThat(session.fireAllRules()).isZero();
        assertThat(listener.getAfterMatchFired()).isEmpty();
    });
}
```

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest#noFinalElse_doesNotFireWhenNoBranchMatches`

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: no-final-else rule does not fire when no branch matches

Refs #12
EOF
)"
```

---

## Task 10: Runtime — same-name binding consumed by consequence

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

- [ ] **Step 1: Append test that consumes `p` in the consequence**

```java
import java.util.ArrayList;
import java.util.List;

@Test
void sameNameBindingFlowsToConsequence() {
    List<String> emitted = new ArrayList<>();
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R1 {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { /* consequence captured by listener; product name not assertable from here directly */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.LOW));
        Product hi = new Product("luxury", Rates.HIGH);
        Product lo = new Product("budget", Rates.LOW);
        products.insert(hi);
        products.insert(lo);
        assertThat(session.fireAllRules()).isEqualTo(1);
        // Verify the *correct* p flowed: only the HIGH-rated luxury matched (Alice is LOW credit)
        // by checking the agenda match's tuple contents.
        assertThat(listener.getAfterMatchFiredTuples())
                .anyMatch(t -> t.contains(hi))
                .noneMatch(t -> t.contains(lo));
    });
}
```

If `TrackingAgendaEventListener` does not expose tuple contents (`getAfterMatchFiredTuples`), extend it now with a small accessor that captures `event.getMatch().getObjects()` — minimal addition, useful for many future tests. Otherwise rewrite the assertion to use `kie-api`'s ad-hoc agenda inspection.

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest#sameNameBindingFlowsToConsequence`

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
# also add TrackingAgendaEventListener changes if applicable

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: same-name binding flows to consequence per branch

Refs #12
EOF
)"
```

---

## Task 11: Runtime — multi-element branch, group element inside branch, nested if/else

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

These three live together — they all exercise `visitBranchRuleItem`'s recursion.

- [ ] **Step 1: Append `multi_elementBranch` test**

```java
@Test
void multiElementBranch_implicitAnd() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R1 {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p1 : /products[ rate == Rates.HIGH ],
                    var p2 : /products[ rate == Rates.MEDIUM ]
                } else {
                    var p1 : /products[ rate == Rates.LOW ],
                    var p2 : /products[ rate == Rates.LOW ]
                },
                do { /* uses p1, p2 */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.LOW));
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("standard", Rates.MEDIUM));
        assertThat(session.fireAllRules()).isEqualTo(1);  // both p1 and p2 must match
    });
}
```

- [ ] **Step 2: Append `groupElementInsideBranch` test**

```java
@Test
void groupElementInsideBranch_not() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R1 {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    not /products[ rate == Rates.HIGH ],
                    var p : /products[ rate == Rates.MEDIUM ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { /* uses p */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.LOW));
        products.insert(new Product("standard", Rates.MEDIUM));
        // Without HIGH, branch matches via NOT
        assertThat(session.fireAllRules()).isEqualTo(1);
    });
}
```

- [ ] **Step 3: Append `nestedIfElse` test**

```java
@Test
void nestedIfElseInsideBranch() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R1 {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    if (c.name == "Alice") {
                        var p : /products[ rate == Rates.HIGH ]
                    } else {
                        var p : /products[ rate == Rates.MEDIUM ]
                    }
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { /* uses p */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.LOW));
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("standard", Rates.MEDIUM));
        assertThat(session.fireAllRules()).isEqualTo(1);
    });
}
```

- [ ] **Step 4: Run all three**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest`

Expected: All pass. If `nestedIfElse` fails, look at the inner `branchBody`'s parsing — `branchBody : '{' ruleItem* '}'` should accept nested `conditionalBranch` because `ruleItem` includes it. If not, diagnose ANTLR ambiguity around nested braces.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: multi-element / nested-CE / nested-if branches

Refs #12
EOF
)"
```

---

## Task 12: Empty-branch rejection — parse-error test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java`

- [ ] **Step 1: Append failing test**

```java
@Test
void emptyBranchBodyIsRejected() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                var p : /persons,
                if (p.age > 30) { } else { var j : /juniors[ age == p.age ] },
                do { System.out.println(p); }
            }
            """;
    org.assertj.core.api.Assertions.assertThatThrownBy(
            () -> DrlxRuleAstParseResult.parseSingleRule(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("empty branch body");
}
```

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest#emptyBranchBodyIsRejected`

Expected: PASS (Task 6 already added the empty-branch check).

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: empty branch body is rejected

Refs #12
EOF
)"
```

---

## Task 13: Property-reactive interaction — sanity test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`
- May modify domain `Customer` to have a mutable `creditRating` setter (already has one per Task 7's fixture).

- [ ] **Step 1: Append failing test**

```java
@Test
void propertyReactivity_branchPatternFiresOnPropertyChange() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R1 {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { /* uses p */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        Customer alice = new Customer("Alice", Rating.HIGH);   // initial: not LOW → else branch
        org.kie.api.runtime.rule.FactHandle handle = customers.insert(alice);
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("budget", Rates.LOW));
        assertThat(session.fireAllRules()).isEqualTo(1);
        listener.getAfterMatchFired().clear();

        alice.setCreditRating(Rating.LOW);
        customers.update(handle, alice);     // notify engine of property change
        assertThat(session.fireAllRules()).isEqualTo(1);  // re-fires on the LOW branch
    });
}
```

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest#propertyReactivity_branchPatternFiresOnPropertyChange`

Expected: PASS.

If FAIL: review whether the desugared eval guard registers `creditRating` as a watched property of `Customer`. May need to add `@watch(creditRating)` annotation in the rule, or rely on default property reactivity behavior. Investigate `PatternIR.watchedProperties` semantics from the recent #13 work.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: property reactivity through if/else branches

Refs #12
EOF
)"
```

---

## Task 14: Final polish — full-suite verification + documentation

**Files:**
- Modify: `specs/IMPLEMENT_SYNTAX_CANDIDATES.md` (mark item #9 done)
- Modify: workspace `HANDOFF.md` will be created at session end via the `handover` skill — not in this plan.

- [ ] **Step 1: Run the full DRLX test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`

Expected: 0 failures. Test count = baseline 111 + ~12 new (2 parse + 2 round-trip/IR + 1 lambda + 1 EvalIR builder + 6 IfElseTest) = ~123.

If any failure: diagnose individually. Common issues:
- An older test that exercised `or` may now fire differently because the visitor's regex `referencedBindings` over-eagerly matched a Java keyword as a binding. Fix the regex to filter keywords.

- [ ] **Step 2: Update syntax-candidates doc**

In `/home/tkobayas/usr/work/mvel3-development/drlx-parser/specs/IMPLEMENT_SYNTAX_CANDIDATES.md`, change row 9 to indicate Form A is implemented:

```
| 9 | ~~**`if`/`else` branching in rule body** (Form A — single trailing consequence)~~ — **IMPLEMENTED in #12**. Form B (per-branch consequences) tracked in #22. | Conditional elements within rule body |
```

- [ ] **Step 3: Verify lint / no warnings introduced**

If the project has a checkstyle / spotless config (look in pom.xml), run it:

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml verify -DskipTests
```

Expected: no new warnings.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    specs/IMPLEMENT_SYNTAX_CANDIDATES.md

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
docs: mark candidate #9 (if/else Form A) as implemented in #12

Closes #12
EOF
)"
```

(`Closes #12` only on this final commit so GitHub auto-closes when merged.)

- [ ] **Step 5: Push and open PR**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push -u origin if-else-form-a

gh pr create --repo tkobayas/drlx-parser \
    --title "feat: if/else branching (Form A) (#12)" \
    --body "$(cat <<'EOF'
## Summary

- Adds `if (cond) { … } else if (cond) { … } else { … }` Form A to DRLX rule bodies — single trailing `do` consequence.
- Visitor desugars to `or(and(test(guard), body…), …)` over existing `or` / `and` infra.
- New internal `EvalIR` (DRLXXXX `test` construct) — used by the desugar; user-facing `test` grammar is split to #23.
- Form B (per-branch consequences) tracked separately in #22.

## Test plan

- [x] Parse-only: binary, else-if chain, IR shape (3 tests)
- [x] Runtime: binary if/else (2 ratings × 2 cases), else-if chain, no-final-else, same-name binding consumption, multi-element branch, nested CE, nested if/else, property-reactivity (~10 tests)
- [x] EvalIR: programmatic construction round-trip + runtime
- [x] Lambda compiler: createEvalExpression unit
- [x] Empty-branch rejection
- [x] Full DRLX suite passes (~123 tests, 0 failures)

Closes #12
EOF
)"
```

(Branch name `if-else-form-a` is suggested; adjust if the repo uses a different convention.)

---

## Self-review

### Spec coverage

- ✅ Grammar additions — Task 1
- ✅ Parse-tree → IR desugar (visitor `buildConditionalBranch`) — Task 6
- ✅ `test` (eval) IR — proto + IR class + runtime — Tasks 2, 3, 4, 5
- ✅ Bindings & scope (inherits from `or`) — exercised by Tasks 7, 8, 10, 11
- ✅ Empty-branch rejection — Task 12
- ✅ Branch with only group element — Task 11
- ✅ Nested if/else — Task 11
- ✅ No-final-else — Task 9
- ✅ Property-reactive interaction — Task 13
- ✅ Plan-phase open items: cumulative-guard split into multiple `EvalIR`s — encoded in Task 6 logic.
- ✅ `EvalIR` standalone unit test (`EvalIRBuilderTest`) — Task 5

### Type / signature consistency

- `EvalIR(String expression, List<String> referencedBindings)` — used identically in Tasks 2, 3, 5, 6, 7+.
- `DrlxLambdaCompiler.createEvalExpression(String, List<BoundVariable>) -> EvalExpression` — referenced in Task 4 (creation) and Task 5 (consumer).
- `BoundVariable` accessor `bv.pattern().getDeclaration()` — used in Task 5; matches the agent-explored shape.
- `referencedBindings(String expression)` returns `List<String>` (binding *names*) — matches `EvalIR.referencedBindings()`. The runtime builder (Task 5) is responsible for mapping name → `BoundVariable` from the `boundVariables` map.

### No-placeholder check

The plan contains task-local "if X is not the right name, look at Y" instructions in places where the implementing agent will need to verify against codebase conventions (e.g., `parseSingleRule`, `Tuple.getObject`, `DataStoreImpl` import path). These are not placeholders — they're explicit verification points with fallback guidance. Each task still ships with concrete code and a passing test as proof.

### Risk concentration

The largest single-task risk is **Task 4** (`createEvalExpression`). Specifically:
- The `EvalExpression.evaluate(Tuple, Declaration[], ReteEvaluator)` API may extract values differently than assumed. The implementing agent should read one existing `EvalExpression` implementation in drools-core (e.g., the one used by DRL `eval(...)`) before wiring `DrlxEvalExpression`.
- `Tuple.getObject(Declaration)` may not be the correct method name in the current drools API. Adjust to whatever the existing `DrlxLambdaConstraint` uses.

If Task 4 is harder than scoped (>1 day), open a sub-issue for "boolean lambda generation in DrlxLambdaCompiler" and have the if/else work depend on it. The plan otherwise stays the same.

### Sequencing

Tasks 1 → 2 → 3 → 4 → 5 → 6 are strictly sequential (each depends on the previous). Tasks 7–13 can run in any order once 6 is done. Task 14 is final.

For **subagent-driven execution**, dispatch sequentially through Task 6, then parallelize 7–13 if desired (each is an independent end-to-end test on top of the same implementation).
