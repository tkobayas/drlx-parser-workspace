# DRLX DataStore.update(T) Coercion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `alerts.update(t)` work in DRLX consequences by rewriting it to `alerts.update(java.util.Objects.requireNonNull(alerts.lookup(t), "..."), t)` at runtime-build time, before MVEL3 compiles the consequence body. Adds the third CRUD op (after `add` and `remove(T)`) without porting upstream's `ConsequenceDataStoreImpl` wrapper.

**Architecture:** New stateless helper `DataStoreUpdateRewriter` that takes consequence text + DataStore-global names, parses with JavaParser, walks for `<global>.update(<arg>)` matches, rewrites in place, and re-emits text. Wired into `DrlxRuleAstRuntimeBuilder.buildRule()` between consequence-text capture and `lambdaCompiler.createLambdaConsequence`. Cheap String guards (empty globals, no `.update(` substring) gate the JavaParser parse so the cost scales with rules-with-DataStore-update, not total rule count.

**Tech Stack:** Java 17, JUnit 5, AssertJ, JavaParser (`org.mvel.javaparser:javaparser-core`, version `3.25.5-mvel3-1`, packages `com.github.javaparser.*`), MVEL3, Drools 10.1.

**Spec:** `specs/2026-05-12-drlx-datastore-update-coercion-design.md`
**Issue:** https://github.com/tkobayas/drlx-parser/issues/45
**Epic:** #26
**Builds on:** cab2862 (unit-field globals)

---

## File Structure

**Create — production (1 file):**
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreUpdateRewriter.java` — stateless rewriter holding one reusable `JavaParser`

**Create — test (1 file):**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/DataStoreUpdateRewriterTest.java` — text-in / text-out unit tests, no MVEL3, no Drools

**Modify — production (1 file):**
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` — compute `dataStoreGlobalNames` once in `build()`; instantiate `DataStoreUpdateRewriter`; thread both through to `buildRule()`; rewrite consequence body before passing to `createLambdaConsequence`

**Modify — test (1 file):**
- `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java` — add `updateByObjectViaDataStore` and `updateOfMissingFactThrows`. Update Javadoc class-comment to mention `update(T)` is now covered.

**Out of scope:** any other file. No changes to `DrlxLambdaConsequence`, `DrlxLambdaCompiler`, `DrlxToRuleAstVisitor`, `DataStore` API, drools-core, drools-ruleunits.

---

## Test command reference

```bash
# Single test class
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -Dtest=ClassName

# Full module suite
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test

# Install (run after src/main changes complete; required by project CLAUDE.md)
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml install -DskipTests
```

---

## Task 1: `DataStoreUpdateRewriter` (TDD in isolation)

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreUpdateRewriter.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DataStoreUpdateRewriterTest.java` (new)

- [ ] **Step 1: Write the failing test file**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/DataStoreUpdateRewriterTest.java`:

```java
package org.drools.drlx.builder;

import java.util.Set;

import com.github.javaparser.JavaParser;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DataStoreUpdateRewriterTest {

    private DataStoreUpdateRewriter rewriter;

    @BeforeEach
    void setUp() {
        rewriter = new DataStoreUpdateRewriter(new JavaParser());
    }

    @Test
    void emptyGlobalsReturnsInputUnchanged() {
        String body = "alerts.update(t);";
        assertThat(rewriter.rewrite(body, Set.of())).isEqualTo(body);
    }

    @Test
    void noUpdateCallReturnsInputUnchanged() {
        String body = "alerts.add(t); alerts.remove(t);";
        assertThat(rewriter.rewrite(body, Set.of("alerts"))).isEqualTo(body);
    }

    @Test
    void simpleUpdateWithNameExprArgIsRewritten() {
        String body = "alerts.update(t);";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        assertThat(result).contains("alerts.update(");
        assertThat(result).contains("java.util.Objects.requireNonNull(alerts.lookup(t)");
        assertThat(result).contains("DataStore 'alerts' has no DataHandle");
        // The trailing fact arg must still be present
        assertThat(result.replaceAll("\\s+", "")).contains(",t);");
    }

    @Test
    void updateOnFieldAccessExprArgIsRewritten() {
        String body = "alerts.update(this.t);";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        assertThat(result).contains("alerts.lookup(this.t)");
        assertThat(result.replaceAll("\\s+", "")).contains(",this.t);");
    }

    @Test
    void updateWithComplexArgIsLeftUntouched() {
        String body = "alerts.update(getThing(t));";
        // The pre-check sees `.update(` and triggers the parse, but the AST
        // walk rejects the non-NameExpr/non-FieldAccessExpr arg. Output is
        // semantically equivalent to the input (modulo serialization
        // formatting, which is why we compare with whitespace stripped).
        String result = rewriter.rewrite(body, Set.of("alerts"));

        assertThat(result.replaceAll("\\s+", ""))
                .isEqualTo("alerts.update(getThing(t));");
        assertThat(result).doesNotContain("requireNonNull");
    }

    @Test
    void updateOnUnrelatedScopeIsLeftUntouched() {
        // `other` is not in dataStoreGlobalNames — leave alone.
        String body = "other.update(t);";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        // String pre-check should short-circuit; result is exactly the input.
        assertThat(result).isEqualTo(body);
    }

    @Test
    void multipleMatchesAreAllRewritten() {
        String body = "alerts.update(t); alerts.update(u);";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        long count = result.split("requireNonNull", -1).length - 1;
        assertThat(count).isEqualTo(2);
    }

    @Test
    void mixedRewritableAndNonRewritableUpdatesAreHandled() {
        String body = "alerts.update(t); alerts.update(getThing(u));";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        // First call rewritten, second left alone.
        long count = result.split("requireNonNull", -1).length - 1;
        assertThat(count).isEqualTo(1);
        assertThat(result).contains("alerts.update(getThing(u))");
    }

    @Test
    void chainedScopeIsLeftUntouched() {
        // getStore().update(t) — scope is a MethodCallExpr, not a NameExpr,
        // so even if the cheap pre-check matched (it does because of
        // ".update("), the AST walk rejects it. Result equals input
        // (modulo formatting).
        String body = "alerts.add(t); getStore().update(t);";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        assertThat(result).doesNotContain("requireNonNull");
    }

    @Test
    void malformedJavaReturnsInputUnchanged() {
        String body = "alerts.update(t // missing semicolon and paren";
        // String pre-check fires, JavaParser fails — we return the input
        // unchanged so MVEL3 can produce the syntax-error diagnostic.
        assertThat(rewriter.rewrite(body, Set.of("alerts"))).isEqualTo(body);
    }

    @Test
    void shadowedGlobalIsRewrittenAnyway() {
        // Documents the known limitation: the rewriter does no scope analysis.
        // If a local variable shadows a DataStore global, the rewrite still
        // fires. See Risks section in the spec.
        String body = "DataStore<Person> alerts = other; alerts.update(t);";
        String result = rewriter.rewrite(body, Set.of("alerts"));

        assertThat(result).contains("requireNonNull");
    }
}
```

- [ ] **Step 2: Run the test, verify it fails**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -Dtest=DataStoreUpdateRewriterTest
```

Expected: compilation failure — `DataStoreUpdateRewriter` does not exist.

- [ ] **Step 3: Create the rewriter**

Create `drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreUpdateRewriter.java`:

```java
package org.drools.drlx.builder;

import java.util.Optional;
import java.util.Set;

import com.github.javaparser.JavaParser;
import com.github.javaparser.ParseResult;
import com.github.javaparser.StaticJavaParser;
import com.github.javaparser.ast.NodeList;
import com.github.javaparser.ast.expr.Expression;
import com.github.javaparser.ast.expr.FieldAccessExpr;
import com.github.javaparser.ast.expr.MethodCallExpr;
import com.github.javaparser.ast.expr.NameExpr;
import com.github.javaparser.ast.stmt.BlockStmt;

/**
 * Rewrites {@code <global>.update(<arg>)} in DRLX consequence bodies to the
 * handle-aware two-arg form
 * {@code <global>.update(java.util.Objects.requireNonNull(<global>.lookup(<arg>), "..."), <arg>)}.
 *
 * <p>Stateless apart from a reusable {@link JavaParser}. Pure: same input
 * yields same output. Cheap string guards skip the parse for bodies where
 * no rewrite could apply.
 */
public final class DataStoreUpdateRewriter {

    private final JavaParser javaParser;

    public DataStoreUpdateRewriter(JavaParser javaParser) {
        this.javaParser = javaParser;
    }

    public String rewrite(String consequenceBody, Set<String> dataStoreGlobalNames) {
        if (dataStoreGlobalNames.isEmpty()) {
            return consequenceBody;
        }
        boolean anyCandidateSubstring = false;
        for (String name : dataStoreGlobalNames) {
            if (consequenceBody.contains(name + ".update(")) {
                anyCandidateSubstring = true;
                break;
            }
        }
        if (!anyCandidateSubstring) {
            return consequenceBody;
        }

        String wrapped = "{\n" + consequenceBody + "\n}";
        ParseResult<BlockStmt> parseResult;
        try {
            parseResult = javaParser.parseBlock(wrapped);
        } catch (RuntimeException e) {
            return consequenceBody;
        }
        if (!parseResult.isSuccessful() || parseResult.getResult().isEmpty()) {
            return consequenceBody;
        }
        BlockStmt block = parseResult.getResult().get();

        boolean modified = false;
        for (MethodCallExpr call : block.findAll(MethodCallExpr.class)) {
            if (rewriteCallIfMatch(call, dataStoreGlobalNames)) {
                modified = true;
            }
        }

        if (!modified) {
            return consequenceBody;
        }

        String emitted = block.toString();
        int firstBrace = emitted.indexOf('{');
        int lastBrace = emitted.lastIndexOf('}');
        if (firstBrace < 0 || lastBrace < 0 || lastBrace <= firstBrace) {
            return consequenceBody;
        }
        return emitted.substring(firstBrace + 1, lastBrace);
    }

    private boolean rewriteCallIfMatch(MethodCallExpr call, Set<String> dataStoreGlobalNames) {
        if (!"update".equals(call.getNameAsString())) {
            return false;
        }
        if (call.getArguments().size() != 1) {
            return false;
        }
        Optional<Expression> scope = call.getScope();
        if (scope.isEmpty() || !(scope.get() instanceof NameExpr scopeName)) {
            return false;
        }
        if (!dataStoreGlobalNames.contains(scopeName.getNameAsString())) {
            return false;
        }
        Expression arg = call.getArgument(0);
        if (!(arg instanceof NameExpr) && !(arg instanceof FieldAccessExpr)) {
            return false;
        }

        String globalName = scopeName.getNameAsString();
        String argText = arg.toString();
        String message = "\"DataStore '" + globalName + "' has no DataHandle for the given fact\"";
        Expression requireNonNullCall = StaticJavaParser.parseExpression(
                "java.util.Objects.requireNonNull("
                        + globalName + ".lookup(" + argText + "), "
                        + message + ")");

        NodeList<Expression> newArgs = new NodeList<>();
        newArgs.add(requireNonNullCall);
        newArgs.add(arg.clone());
        call.setArguments(newArgs);
        return true;
    }
}
```

- [ ] **Step 4: Run the test, verify it passes**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -Dtest=DataStoreUpdateRewriterTest
```

Expected: 11 tests pass.

Common failure modes:

1. **`com.github.javaparser.*` import resolution fails** — confirm `javaparser-core` is on the test classpath (it should be, transitively from drlx-parser-core's main deps).
2. **`StaticJavaParser.parseExpression` throws** — the message-quoting in the synthesized expression is malformed. Re-check that the embedded message string uses doubled-up quotes correctly.
3. **`updateWithComplexArgIsLeftUntouched` fails because output contains `requireNonNull`** — the `arg instanceof NameExpr || FieldAccessExpr` guard is missing or inverted. Re-check `rewriteCallIfMatch`.
4. **Test compares whitespace-sensitive output and JavaParser reformats** — assertions already strip whitespace where needed. If a new failure appears, prefer asserting on substring-presence over exact equality.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DataStoreUpdateRewriter.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DataStoreUpdateRewriterTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat: add DataStoreUpdateRewriter for single-arg update coercion

Pure helper: parses a consequence body with JavaParser, finds
`<global>.update(<arg>)` calls where the scope is a known DataStore
global and the arg is a NameExpr or FieldAccessExpr, and rewrites them
to `<global>.update(Objects.requireNonNull(<global>.lookup(<arg>), ...), <arg>)`.

Cheap string guards (empty globals, no `.update(` substring) skip the
parse for bodies that can't possibly match. Not yet wired into the
runtime builder.

Refs #45
EOF
)"
```

---

## Task 2: Wire rewriter into `DrlxRuleAstRuntimeBuilder` + happy-path integration test

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java`

- [ ] **Step 1: Add the failing happy-path test**

Append the following test method inside the `DataStoreCrudTest` class in `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java`, after `consequenceCanReferenceMultipleUnitFields`:

```java
    @Test
    void updateByObjectViaDataStore() {
        String rule =
                """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule RenameAdults {
                    Person p : /persons[ age > 30 ],
                    do { p.setName("renamed"); persons.update(p); }
                }
                """;
        KieBase kieBase = new DrlxRuleBuilder().build(rule);

        MyUnit unit = new MyUnit();
        Person alice = new Person("Alice", 40);
        unit.persons.add(alice);

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            TestDataObserver<Person> obs = TestDataObserver.subscribeTo(unit.persons);

            assertThat(instance.fire()).isEqualTo(1);
            assertThat(obs.updated()).hasSize(1);
            assertThat(alice.getName()).isEqualTo("renamed");
        }
    }
```

Also update the class-level Javadoc to mention `update(T)` coverage. Replace:

```java
/**
 * End-to-end tests for issue #37 (DataStore CRUD): each rule consequence
 * calls a {@code DataStore} method on a unit-field reference (e.g.
 * {@code persons1.add(p)}), and {@link DrlxRuleUnitInstance} provides the
 * runtime surface. The class is intentionally named after the broader
 * #37 scope; it will grow as additional sub-pieces (update, with-block) land.
 */
```

with:

```java
/**
 * End-to-end tests for issues #37 and #45 (DataStore CRUD): each rule
 * consequence calls a {@code DataStore} method on a unit-field reference
 * (e.g. {@code persons1.add(p)}, {@code persons.update(p)}), and
 * {@link DrlxRuleUnitInstance} provides the runtime surface. The class
 * covers add / remove(T) / update(T); the with-block compact update
 * (#34) is not yet covered.
 */
```

- [ ] **Step 2: Run the test, verify it fails with `cannot resolve method update(Person)`**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -Dtest=DataStoreCrudTest#updateByObjectViaDataStore
```

Expected: 1 test errors with an MVEL3 method-resolution failure on `update(Person)` (the one-arg overload doesn't exist on `DataStore<T>`).

- [ ] **Step 3: Wire the rewriter into `DrlxRuleAstRuntimeBuilder.build()`**

In `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`:

**3a.** Add imports near the existing imports (after the line `import org.drools.ruleunits.api.DataSource;`):

```java
import org.drools.ruleunits.api.DataStore;

import com.github.javaparser.JavaParser;

import java.util.stream.Collectors;
```

(The `Collectors` import goes with the other `java.util.*` imports near the top; the others next to existing same-package imports.)

**3b.** In the `build(CompilationUnitIR parseResult)` method, locate the existing block where `globalTypes` is registered and `buildRule` is called (currently around line 65-80, exact line depends on revision — search for `globalTypes.forEach(pkg::addGlobal)` to find it).

The current block looks like:

```java
        Map<String, Class<?>> entryPointTypes = buildEntryPointTypeMap(unitClass);
        Map<String, java.lang.reflect.Type> globalTypes = buildGlobalTypeMap(unitClass);
        globalTypes.forEach(pkg::addGlobal);

        Map<String, KnowledgePackageImpl> typeDeclPackages = new LinkedHashMap<>();
        registerTypeDeclarations(typeDeclPackages, parseResult, pkg.getTypeResolver(), entryPointTypes, unitClass);

        parseResult.rules().forEach(rule ->
                pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass, globalTypes)));
```

Replace it with:

```java
        Map<String, Class<?>> entryPointTypes = buildEntryPointTypeMap(unitClass);
        Map<String, java.lang.reflect.Type> globalTypes = buildGlobalTypeMap(unitClass);
        globalTypes.forEach(pkg::addGlobal);

        Set<String> dataStoreGlobalNames = globalTypes.entrySet().stream()
                .filter(e -> {
                    Class<?> raw = erasure(e.getValue());
                    return raw != null && DataStore.class.isAssignableFrom(raw);
                })
                .map(Map.Entry::getKey)
                .collect(Collectors.toSet());
        DataStoreUpdateRewriter updateRewriter = new DataStoreUpdateRewriter(new JavaParser());

        Map<String, KnowledgePackageImpl> typeDeclPackages = new LinkedHashMap<>();
        registerTypeDeclarations(typeDeclPackages, parseResult, pkg.getTypeResolver(), entryPointTypes, unitClass);

        parseResult.rules().forEach(rule ->
                pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass, globalTypes, dataStoreGlobalNames, updateRewriter)));
```

**3c.** Update the `buildRule` signature and body (currently around line 286-315) to accept and use the rewriter. Replace the existing method:

```java
    private RuleImpl buildRule(RuleIR parseResult,
                               TypeResolver typeResolver,
                               Map<String, Class<?>> entryPointTypes,
                               Class<?> unitClass,
                               Map<String, java.lang.reflect.Type> globalTypes) {
        lambdaCompiler.beginRule(parseResult.name());

        RuleImpl rule = new RuleImpl(parseResult.name());
        rule.setResource(rule.getResource());
        applyAnnotations(rule, parseResult.annotations());

        GroupElement root = GroupElementFactory.newAndInstance();
        Map<String, BoundVariable> boundVariables = new LinkedHashMap<>();

        buildLhs(parseResult.lhs(), root, typeResolver, entryPointTypes, unitClass, boundVariables);

        if (parseResult.rhs() != null) {
            Map<String, Type<?>> types = lambdaCompiler.getTypeMap(root);
            for (Map.Entry<String, java.lang.reflect.Type> e : globalTypes.entrySet()) {
                Class<?> raw = erasure(e.getValue());
                if (raw != null) {
                    types.put(e.getKey(), Type.type(raw));
                }
            }
            rule.setConsequence(lambdaCompiler.createLambdaConsequence(parseResult.rhs().block(), types, globalTypes.keySet()));
        }

        rule.setLhs(root);
        return rule;
    }
```

with:

```java
    private RuleImpl buildRule(RuleIR parseResult,
                               TypeResolver typeResolver,
                               Map<String, Class<?>> entryPointTypes,
                               Class<?> unitClass,
                               Map<String, java.lang.reflect.Type> globalTypes,
                               Set<String> dataStoreGlobalNames,
                               DataStoreUpdateRewriter updateRewriter) {
        lambdaCompiler.beginRule(parseResult.name());

        RuleImpl rule = new RuleImpl(parseResult.name());
        rule.setResource(rule.getResource());
        applyAnnotations(rule, parseResult.annotations());

        GroupElement root = GroupElementFactory.newAndInstance();
        Map<String, BoundVariable> boundVariables = new LinkedHashMap<>();

        buildLhs(parseResult.lhs(), root, typeResolver, entryPointTypes, unitClass, boundVariables);

        if (parseResult.rhs() != null) {
            Map<String, Type<?>> types = lambdaCompiler.getTypeMap(root);
            for (Map.Entry<String, java.lang.reflect.Type> e : globalTypes.entrySet()) {
                Class<?> raw = erasure(e.getValue());
                if (raw != null) {
                    types.put(e.getKey(), Type.type(raw));
                }
            }
            String body = updateRewriter.rewrite(parseResult.rhs().block(), dataStoreGlobalNames);
            rule.setConsequence(lambdaCompiler.createLambdaConsequence(body, types, globalTypes.keySet()));
        }

        rule.setLhs(root);
        return rule;
    }
```

The only changes from the previous version: two extra parameters (`dataStoreGlobalNames`, `updateRewriter`), and the consequence body is now passed through `updateRewriter.rewrite(...)` before `createLambdaConsequence`.

- [ ] **Step 4: Run the failing test, verify it now passes**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -Dtest=DataStoreCrudTest#updateByObjectViaDataStore
```

Expected: 1 test passes. The observer sees one `update` event, and `alice.getName()` is `"renamed"`.

Common failure modes if it does not:

1. **MVEL3 still complains about `update(Person)`** — the rewriter ran but produced something MVEL3 can't parse. Add `System.out.println(body)` immediately before `createLambdaConsequence` and inspect the output. The expected shape is `p.setName("renamed"); persons.update(java.util.Objects.requireNonNull(persons.lookup(p), "DataStore 'persons' has no DataHandle for the given fact"), p);`.
2. **`obs.updated()` is empty even though the rule fires** — likely `lookup(p)` returned null because the fact wasn't actually inserted into the store before `instance.fire()`. Re-verify the fixture: `unit.persons.add(alice)` must happen before the `try` block.
3. **`NoClassDefFoundError com.github.javaparser.JavaParser`** — the `javaparser-core` dep is `provided` or test-scope. Check `pom.xml` — it should be in main `<dependencies>`. (Verified at plan-write time: `org.mvel.javaparser:javaparser-core` is a normal compile-scope dep on `drlx-parser-core`.)

- [ ] **Step 5: Run the full module suite, verify nothing regressed**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test
```

Expected: all tests pass, **0 failures, 0 errors**. Test count goes up by 12 vs the pre-Task-1 baseline (11 from Task 1's `DataStoreUpdateRewriterTest` + 1 from this step's `updateByObjectViaDataStore`).

- [ ] **Step 6: Install the module so downstream consumers resolve the changes**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml install -DskipTests
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat: rewrite DataStore.update(T) to handle-aware form in DRLX consequences

DrlxRuleAstRuntimeBuilder now derives DataStore-typed global names from
the existing globalTypes map, instantiates a DataStoreUpdateRewriter
once per build, and threads it through buildRule. The consequence body
is rewritten before MVEL3 compiles it: `persons.update(p)` becomes
`persons.update(Objects.requireNonNull(persons.lookup(p), "..."), p)`.

End-to-end coverage in DataStoreCrudTest.updateByObjectViaDataStore.

Refs #45
EOF
)"
```

---

## Task 3: Negative-path test for missing fact

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java`

- [ ] **Step 1: Add the negative-path test**

Append the following test method inside the `DataStoreCrudTest` class, after `updateByObjectViaDataStore`:

```java
    @Test
    void updateOfMissingFactThrows() {
        String rule =
                """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule UpdateMissing {
                    Person p : /persons[ age > 30 ],
                    do { Person stranger = new Person("Stranger", 99); persons.update(stranger); }
                }
                """;
        KieBase kieBase = new DrlxRuleBuilder().build(rule);

        MyUnit unit = new MyUnit();
        Person alice = new Person("Alice", 40);
        unit.persons.add(alice);

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            assertThatThrownBy(instance::fire)
                    .hasMessageContaining("DataStore 'persons' has no DataHandle for the given fact");
        }
    }
```

Add the AssertJ static import at the top of the file (after the existing `import static org.assertj.core.api.Assertions.assertThat;`):

```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
```

- [ ] **Step 2: Run the test, verify it passes**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -Dtest=DataStoreCrudTest#updateOfMissingFactThrows
```

Expected: 1 test passes. The `Objects.requireNonNull` call throws an NPE with the expected message; the rule firing surfaces it (likely wrapped, hence `hasMessageContaining` rather than exact-message assertion).

Common failure modes:

1. **The test fails with `expected throwable but nothing was thrown`** — `instance.fire()` swallowed the exception. Inspect `DrlxRuleUnitInstance.fire()` to confirm exception propagation; if it logs and returns, change the test to capture the logged exception or to assert via the rule-engine's exception listener.
2. **The thrown exception's message does not contain the expected substring** — `Objects.requireNonNull` produced a generic NPE without the message. Re-verify the rewriter built the requireNonNull call with the literal message string. Also possible: the message survived but is wrapped in a `RuntimeException` whose `getMessage()` doesn't include the cause's message — switch the assertion to `.hasRootCauseMessage(...)` or `.getRootCause().hasMessageContaining(...)`.
3. **Compile-time error on `new Person("Stranger", 99)` inside the consequence** — MVEL3 needs to know about `Person`. The import in the rule string covers this; if it still fails, MVEL3 may need a more explicit type binding. (Person is already used in many tests via consequence; this should not surface.)

- [ ] **Step 3: Run the full module suite, verify nothing regressed**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test
```

Expected: all tests pass, **0 failures, 0 errors**. Test count goes up by 1 vs the post-Task-2 count.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreCrudTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: cover update(T) of a fact not in the DataStore

When alerts.update(t) is called with a fact that was never added to the
store, ListDataStore.lookup returns null and the rewriter-injected
Objects.requireNonNull surfaces a clear "no DataHandle for the given
fact" message instead of an opaque NPE.

Refs #45
EOF
)"
```

---

## Verification

After all three tasks land:

- [ ] **Full module test suite**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test
```

Expected: all tests pass, **0 failures, 0 errors**. Net delta vs the baseline at the start of this plan: +13 tests (11 unit tests in `DataStoreUpdateRewriterTest` + 2 integration tests in `DataStoreCrudTest`). The pre-existing `DataStoreCrudTest` tests (`consequenceCanCallDataStoreAdd`, `removeByObjectViaDataStore`, `consequenceCanReferenceMultipleUnitFields`) still pass unchanged.

- [ ] **Full module install**

```bash
mvn -pl drlx-parser-core -am -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml install
```

Expected: BUILD SUCCESS across the reactor.

- [ ] **Issue close**

When closing #45, paste a short summary referencing the new tests and noting that `with`-block compact update (#34) remains separately tracked.

- [ ] **Confirm spec → tasks alignment**

| Spec section | Plan task |
|--------------|-----------|
| `DataStoreUpdateRewriter` class with cheap String guards + JavaParser parse + AST walk + serialise | Task 1, Step 3 |
| Argument-shape restriction (NameExpr / FieldAccessExpr only) | Task 1, Step 3 (`rewriteCallIfMatch`) |
| Inline `Objects.requireNonNull` rewrite shape | Task 1, Step 3 |
| Parse-failure → return input unchanged | Task 1, Step 3 (`if (!parseResult.isSuccessful()...)` branch) |
| `DrlxRuleAstRuntimeBuilder` derives `dataStoreGlobalNames` from `globalTypes` | Task 2, Step 3b |
| Single `JavaParser` instance per build | Task 2, Step 3b (instantiated in `build()`) |
| Rewriter invoked before `createLambdaConsequence` | Task 2, Step 3c |
| `DrlxLambdaConsequence` unchanged | Verified by file list above (not in modified list) |
| `updateByObjectViaDataStore` integration test | Task 2, Step 1 |
| `updateOfMissingFactThrows` integration test | Task 3, Step 1 |
| `updateWithComplexArgIsLeftAlone` (deferred to unit-level) | Task 1, Step 1 (`updateWithComplexArgIsLeftUntouched`, `chainedScopeIsLeftUntouched`) |
| Unit tests (no globals / no `update` / multiple matches / shadowing / malformed body) | Task 1, Step 1 (full table) |
| `with`-block compact update deferred (#34) | Out of scope — no task |
| Property-reactive bitmask deferred | Out of scope — no task |
| No upstream Drools change | Verified by file list (only files under drlx-parser-core/src) |

If any spec requirement lacks a task, stop and add it before starting Task 1.
