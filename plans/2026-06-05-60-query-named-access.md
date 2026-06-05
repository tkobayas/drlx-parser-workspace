# Query Named Access (#60) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Support named (non-positional) query invocation: `/trusts[a == subject, var object : b]` where arguments are mapped by query parameter name instead of position.

**Architecture:** The grammar gains one new alternative in `drlxExpression` for `var bindName : paramName`. The parser collects it as a condition string via `getText()`. The compiler detects named access when a query entry point has conditions (not positional args), reorders them by parameter index into the same `List<String>` positional access produces, and feeds them to the existing `buildQueryElement()` unchanged.

**Tech Stack:** ANTLR4 grammar, Java 17, drools runtime API

---

## File Structure

| File | Responsibility |
|------|---------------|
| `DrlxParser.g4` | Add `VAR` output-binding alternative to `drlxExpression`; label identifiers to avoid accessor collision |
| `DrlxToJavaParserVisitor.java` | Update `visitDrlxExpression` to use labeled accessor and handle `VAR` alternative |
| `DrlxRuleAstRuntimeBuilder.java` | Extend query detection; add `buildNamedQueryArgs()` method |
| `QueryTest.java` | 10 new test methods for named access |

All files are under `/home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/`.

---

### Task 1: Grammar — add `VAR identifier ':' identifier` to `drlxExpression`

**Files:**
- Modify: `main/antlr4/org/drools/drlx/parser/DrlxParser.g4:224-228`

- [ ] **Step 1: Add the new alternative with element labels**

The new `VAR identifier ':' identifier` alternative introduces two `identifier` references in one alternative, which would cause ANTLR to generate a list accessor for `identifier()`, breaking the existing `visitDrlxExpression` code. Fix: use element labels on all identifier references.

```antlr
drlxExpression
    : VAR varBind=identifier ':' varParam=identifier
    | bind=identifier ':' expression
    | customConstraint
    | expression
    ;
```

- [ ] **Step 2: Regenerate the ANTLR parser**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources`
Expected: BUILD SUCCESS, generated `DrlxParser.java` / `DrlxParserBaseVisitor.java` updated with `varBind()`, `varParam()`, `bind()` accessors on `DrlxExpressionContext`.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(grammar): add VAR output-binding alternative to drlxExpression (#60)

Refs #60"
```

---

### Task 2: Visitor — update `visitDrlxExpression` for labeled accessors

**Files:**
- Modify: `main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java:552-572`

- [ ] **Step 1: Write a parser test that parses `var object : b` as a drlxExpression**

Add a test in a suitable parser test class (or QueryTest) that parses a source containing `var object : b` inside brackets and verifies the parse tree text. This verifies the grammar change works.

```java
@Test
void parseNamedQueryOutputBinding() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                /personsByAge[minAge == 25, var p : result],
                do { results.add(p); }
            }
            """;

    // Just verify it parses without error — full behavior tested in later tasks
    assertThatCode(() -> newBuilder().build(source)).doesNotThrowAnyException();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#parseNamedQueryOutputBinding -pl .`
Expected: Compilation failure or runtime error because `visitDrlxExpression` tries to access `ctx.identifier()` which no longer works with labeled grammar.

- [ ] **Step 3: Update visitDrlxExpression**

Change `ctx.identifier()` to `ctx.bind()` and add early return for the `VAR` alternative. The `VAR` case creates a `DrlxExpression` with the full text as a NameExpr (since it flows through as a condition string and the JavaParser AST representation is secondary to the IR path).

```java
@Override
public Node visitDrlxExpression(DrlxParser.DrlxExpressionContext ctx) {
    // VAR bindName : paramName — query output binding; store as opaque text node
    if (ctx.VAR() != null) {
        String text = ctx.VAR().getText() + " " + ctx.varBind().getText() + " : " + ctx.varParam().getText();
        NameExpr nameExpr = new NameExpr(text);
        nameExpr.setTokenRange(createTokenRange(ctx));
        DrlxExpression drlxExpression = new DrlxExpression(null, nameExpr);
        drlxExpression.setTokenRange(createTokenRange(ctx));
        nameExpr.setParentNode(drlxExpression);
        return drlxExpression;
    }

    SimpleName bind = ctx.bind() != null ? new SimpleName(ctx.bind().getText()) : null;
    if (bind != null) {
        bind.setTokenRange(createTokenRange(ctx.bind()));
    }

    Expression expr = (Expression) visit(ctx.expression());
    if (!expr.getTokenRange().isPresent()) {
        expr.setTokenRange(createTokenRange(ctx.expression()));
    }

    DrlxExpression drlxExpression = new DrlxExpression(bind, expr);
    drlxExpression.setTokenRange(createTokenRange(ctx));
    if (bind != null) {
        bind.setParentNode(drlxExpression);
    }
    expr.setParentNode(drlxExpression);

    return drlxExpression;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#parseNamedQueryOutputBinding -pl .`
Expected: FAIL — the test should still fail because the compiler doesn't handle named access yet (it will try to process conditions as regular constraints). The parse step itself should succeed though. If the test is checking "no exception", it may fail at compile time. Adjust the test to check only parsing if needed, or accept the failure for now.

- [ ] **Step 5: Run full test suite to check for regressions**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl .`
Expected: All existing tests pass (the label change from `ctx.identifier()` to `ctx.bind()` should be transparent).

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(visitor): handle VAR output-binding in visitDrlxExpression (#60)

Refs #60"
```

---

### Task 3: Compiler — extend query detection for named access

**Files:**
- Modify: `main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:446-511`

- [ ] **Step 1: Write a basic named access test**

```java
@Test
void namedQueryAccessBasic() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                /personsByAge[minAge == 25, var p : result],
                do { results.add(p); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));
    unit.persons.add(new Person("Bob", 20));
    unit.persons.add(new Person("Charlie", 40));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        List<String> names = unit.results.stream()
                .map(o -> ((Person) o).getName())
                .toList();
        assertThat(names).containsExactlyInAnyOrder("Alice", "Charlie");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessBasic -pl .`
Expected: FAIL — the compiler treats conditions as regular constraints, not named query args.

- [ ] **Step 3: Add `buildNamedQueryArgs()` method**

Add this private method to `DrlxRuleAstRuntimeBuilder.java` (after `buildQueryElement`, around line 1036):

```java
private static List<String> buildNamedQueryArgs(List<String> conditions, QueryImpl targetQuery) {
    Declaration[] queryParams = targetQuery.getParameters();
    Map<String, Integer> nameToIndex = new LinkedHashMap<>();
    for (int i = 0; i < queryParams.length; i++) {
        nameToIndex.put(queryParams[i].getIdentifier(), i);
    }

    String[] args = new String[queryParams.length];
    for (String condition : conditions) {
        condition = condition.trim();
        if (condition.startsWith("var ")) {
            // "var bindName : paramName"
            int colonIdx = condition.indexOf(':');
            if (colonIdx < 0) {
                throw new RuntimeException("invalid named query argument: '" + condition + "'");
            }
            String bindName = condition.substring(4, colonIdx).trim();
            String paramName = condition.substring(colonIdx + 1).trim();
            Integer index = nameToIndex.get(paramName);
            if (index == null) {
                throw new RuntimeException(
                        "unknown query parameter '" + paramName + "' in named access for query '"
                        + targetQuery.getName() + "'. Known parameters: " + nameToIndex.keySet());
            }
            if (args[index] != null) {
                throw new RuntimeException(
                        "duplicate assignment to query parameter '" + paramName + "' in named access for query '"
                        + targetQuery.getName() + "'");
            }
            args[index] = "var " + bindName;
        } else {
            // "paramName == expr"
            int eqIdx = condition.indexOf("==");
            if (eqIdx < 0) {
                throw new RuntimeException("invalid named query argument: '" + condition
                        + "'. Input arguments must use '==' (e.g., paramName == value)");
            }
            String paramName = condition.substring(0, eqIdx).trim();
            String expr = condition.substring(eqIdx + 2).trim();
            Integer index = nameToIndex.get(paramName);
            if (index == null) {
                throw new RuntimeException(
                        "unknown query parameter '" + paramName + "' in named access for query '"
                        + targetQuery.getName() + "'. Known parameters: " + nameToIndex.keySet());
            }
            if (args[index] != null) {
                throw new RuntimeException(
                        "duplicate assignment to query parameter '" + paramName + "' in named access for query '"
                        + targetQuery.getName() + "'");
            }
            args[index] = expr;
        }
    }

    // Validate all parameters assigned
    for (int i = 0; i < args.length; i++) {
        if (args[i] == null) {
            throw new RuntimeException(
                    "named access for query '" + targetQuery.getName()
                    + "' is missing parameter '" + queryParams[i].getIdentifier() + "'");
        }
    }

    return List.of(args);
}
```

- [ ] **Step 4: Extend query detection in `buildLhsPatterns`**

Modify the existing query detection block at line 447. Currently:

```java
if (targetQuery != null && !patternIr.positionalArgs().isEmpty()) {
```

Replace with extended logic:

```java
if (targetQuery != null && !patternIr.positionalArgs().isEmpty()) {
    // --- existing positional path (unchanged) ---
    if (!patternIr.conditions().isEmpty()) {
        throw new RuntimeException(
                "query '" + targetQuery.getName()
                + "' cannot mix positional arguments (...) with named access [...]");
    }
    // ... rest of existing positional code unchanged ...
}
if (targetQuery != null && !patternIr.conditions().isEmpty()) {
    // Named access path
    if (targetQuery == currentQuery) {
        throw new RuntimeException(
                "self-referencing query '" + targetQuery.getName()
                + "' cannot use named access; use positional syntax instead");
    }
    // Convert named conditions to ordered positional args
    List<String> orderedArgs = buildNamedQueryArgs(patternIr.conditions(), targetQuery);
    // Create a synthetic PatternIR with positional args for buildQueryElement
    PatternIR positionalIr = new PatternIR(
            patternIr.typeName(), patternIr.bindName(), patternIr.entryPoint(),
            List.of(), patternIr.temporalConditions(), patternIr.castTypeName(),
            orderedArgs, patternIr.passive(), patternIr.watchedProperties(),
            patternIr.windowType(), patternIr.windowParameter());
    QueryElement queryElement = buildQueryElement(positionalIr, targetQuery, boundVariables);
    parent.addChild(queryElement);
    // Post-element variable registration (same as positional path)
    Declaration[] queryParams = targetQuery.getParameters();
    List<String> args = orderedArgs;
    for (int i = 0; i < args.size(); i++) {
        String arg = args.get(i);
        String varName = null;
        if (arg.startsWith("var ")) {
            varName = arg.substring(4).trim();
        } else if (!boundVariables.containsKey(arg) && isSimpleIdentifier(arg)) {
            varName = arg;
        }
        if (varName != null) {
            Class<?> paramType = queryParams[i].getDeclarationClass();
            Pattern resultPattern = queryElement.getResultPattern();
            Declaration decl = resultPattern.getDeclarations().get(varName);
            if (decl != null) {
                boundVariables.put(varName,
                        new BoundVariable(varName, paramType, resultPattern, decl));
            }
        }
    }
    if (patternIr.bindName() != null) {
        Map<String, Integer> nameToIndex = new LinkedHashMap<>();
        Declaration[] qParams = targetQuery.getParameters();
        for (int i = 0; i < qParams.length; i++) {
            nameToIndex.put(qParams[i].getIdentifier(), i);
        }
        QueryResultRowReader rowReader = new QueryResultRowReader(nameToIndex);
        Pattern resultPattern = queryElement.getResultPattern();
        Declaration rowDecl = new Declaration(patternIr.bindName(), rowReader, resultPattern);
        resultPattern.addDeclaration(rowDecl);
        boundVariables.put(patternIr.bindName(),
                new BoundVariable(patternIr.bindName(), QueryResultRow.class, resultPattern, rowDecl));
    }
    continue;
}
```

Note: The post-element registration code (lines 477-509 in the existing positional path) is duplicated here. Consider extracting a helper method to avoid duplication if warranted, but keep it inline for now since the spec doesn't call for refactoring.

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessBasic -pl .`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl .`
Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(compiler): named query access detection and buildNamedQueryArgs (#60)

Refs #60"
```

---

### Task 4: Tests — all inputs named access

**Files:**
- Modify: `test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write test**

```java
@Test
void namedQueryAccessAllInputs() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAgeRange(int minAge, int maxAge, Person result) {
                Person result : /persons[age >= minAge, age <= maxAge],
            }

            rule R1 {
                /personsByAgeRange[minAge == 25, maxAge == 35, var p : result],
                do { results.add(p); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));
    unit.persons.add(new Person("Bob", 20));
    unit.persons.add(new Person("Charlie", 40));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        List<String> names = unit.results.stream()
                .map(o -> ((Person) o).getName())
                .toList();
        assertThat(names).containsExactly("Alice");
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessAllInputs -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: named access with all-input params (#60)

Refs #60"
```

---

### Task 5: Tests — parameter order independence

**Files:**
- Modify: `test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write test**

```java
@Test
void namedQueryAccessOrderIndependence() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                /personsByAge[var p : result, minAge == 25],
                do { results.add(p); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));
    unit.persons.add(new Person("Bob", 20));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        List<String> names = unit.results.stream()
                .map(o -> ((Person) o).getName())
                .toList();
        assertThat(names).containsExactly("Alice");
    }
}
```

- [ ] **Step 2: Run and verify pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessOrderIndependence -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: named access parameter order independence (#60)

Refs #60"
```

---

### Task 6: Tests — error: missing parameter

**Files:**
- Modify: `test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write test**

```java
@Test
void namedQueryAccessErrorMissingParameter() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                /personsByAge[minAge == 25],
                do { results.add("wrong"); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(source))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("missing parameter")
            .hasMessageContaining("result");
}
```

- [ ] **Step 2: Run and verify pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessErrorMissingParameter -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: named access error on missing parameter (#60)

Refs #60"
```

---

### Task 7: Tests — error: unknown parameter name

**Files:**
- Modify: `test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write test**

```java
@Test
void namedQueryAccessErrorUnknownParameter() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                /personsByAge[badName == 25, var p : result],
                do { results.add(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(source))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("unknown query parameter")
            .hasMessageContaining("badName");
}
```

- [ ] **Step 2: Run and verify pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessErrorUnknownParameter -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: named access error on unknown parameter name (#60)

Refs #60"
```

---

### Task 8: Tests — error: mixing positional and named

**Files:**
- Modify: `test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write test**

```java
@Test
void namedQueryAccessErrorMixingPositionalAndNamed() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                /personsByAge(25)[var p : result],
                do { results.add(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(source))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("cannot mix positional arguments");
}
```

- [ ] **Step 2: Run and verify pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessErrorMixingPositionalAndNamed -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: named access error on mixing positional and named (#60)

Refs #60"
```

---

### Task 9: Tests — error: self-referencing named access

**Files:**
- Modify: `test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write test**

```java
@Test
void namedQueryAccessErrorSelfReferencing() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Trust;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule Trusts(String a, String b) {
                or(
                    /trusts[a == a, b == b],
                    and(/trusts(a, var z), /trusts(z, b))
                ),
            }

            rule R1 {
                /trusts("A", var t),
                do { results.add(t); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(source))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("self-referencing query")
            .hasMessageContaining("cannot use named access");
}
```

- [ ] **Step 2: Run and verify pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessErrorSelfReferencing -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: named access error on self-referencing query (#60)

Refs #60"
```

---

### Task 10: Tests — error: non-`==` operator for input

**Files:**
- Modify: `test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write test**

```java
@Test
void namedQueryAccessErrorNonEqOperator() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                /personsByAge[minAge >= 25, var p : result],
                do { results.add(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(source))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("must use '=='");
}
```

- [ ] **Step 2: Run and verify pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessErrorNonEqOperator -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: named access error on non-== operator (#60)

Refs #60"
```

---

### Task 11: Tests — named access with result binding

**Files:**
- Modify: `test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Write test**

```java
@Test
void namedQueryAccessWithResultBinding() {
    String source = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                Person result : /persons[age >= minAge],
            }

            rule R1 {
                var t : /personsByAge[minAge == 25, var p : result],
                do { results.add(t.result); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));
    unit.persons.add(new Person("Bob", 20));
    unit.persons.add(new Person("Charlie", 40));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        List<String> names = unit.results.stream()
                .map(o -> ((Person) o).getName())
                .toList();
        assertThat(names).containsExactlyInAnyOrder("Alice", "Charlie");
    }
}
```

- [ ] **Step 2: Run and verify pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=QueryTest#namedQueryAccessWithResultBinding -pl .`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: named access with result row binding (#60)

Refs #60"
```

---

### Task 12: Final verification — full test suite

- [ ] **Step 1: Run the complete drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`
Expected: All tests pass, no regressions.

- [ ] **Step 2: Remove the preliminary `parseNamedQueryOutputBinding` test if redundant**

The `namedQueryAccessBasic` test in Task 3 covers the same scenario more thoroughly. If `parseNamedQueryOutputBinding` from Task 2 is redundant, remove it.

---

## Design notes

**ANTLR element labels:** The spec proposes `VAR identifier ':' identifier` without labels. This would cause ANTLR to generate `identifier()` as a list accessor (two `identifier` refs in one alternative), breaking `DrlxToJavaParserVisitor.visitDrlxExpression()` which uses `ctx.identifier()` expecting a single context. The plan uses element labels (`varBind`, `varParam`, `bind`) to give each identifier its own accessor.

**DrlxToJavaParserVisitor vs DrlxToRuleAstVisitor:** The main compilation path uses `DrlxToRuleAstVisitor` (via `DrlxRuleBuilder.parseToRuleAst()`), which extracts conditions as raw text via `getText()`. `DrlxToJavaParserVisitor` is used by `DrlxHelper` utility methods for AST construction (used by tests, IDE tooling). Both must handle the new grammar alternative, but the IR path needs no condition-extraction changes.

**Synthetic PatternIR:** The named access path creates a synthetic `PatternIR` with `positionalArgs` set to the reordered arguments and `conditions` cleared. This lets `buildQueryElement()` process it identically to positional access — no changes needed to that method.

## Verification

1. All 10 new tests pass in `QueryTest.java`
2. All existing tests pass (no regressions from grammar labels or compiler changes)
3. Full `mvn test` across the drlx-parser project succeeds
