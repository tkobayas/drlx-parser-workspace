# #41 Queries v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add query support to the DRLX parser — parameterized rule definitions and positional-argument invocation from other rules.

**Architecture:** Grammar gets optional parameter list on rules. IR gets `RuleParameterIR`. Runtime builder splits into two passes: queries first (producing `QueryImpl` with drools prefix pattern), then regular rules (with `QueryElement` for invocations). A small custom `ReadAccessor` bridges `DroolsQuery.getElements()` since `drools-mvel`'s `ClassFieldReader` is not on the classpath.

**Tech Stack:** ANTLR4 grammar, Java 17 records, drools-base (`QueryImpl`, `QueryElement`, `QueryArgument`, `QueryNameConstraint`, `ArrayElementReader`), JUnit 5 + AssertJ.

---

### Task 1: Grammar — add rule parameter list

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:39-42`

- [ ] **Step 1: Add grammar rules**

Add the optional parameter list to `ruleDeclaration` and two new rules after it:

```antlr
// Rule declaration — annotations may prefix the RULE keyword (e.g. @Salience(10))
ruleDeclaration
    : annotation* RULE identifier ruleParameterList? '{' ruleBody '}'
    ;

ruleParameterList
    : '(' ruleParameter (',' ruleParameter)* ')'
    ;

ruleParameter
    : typeType identifier
    ;
```

Only the `ruleDeclaration` rule changes (adds `ruleParameterList?`). The two new rules follow it.

- [ ] **Step 2: Verify existing tests still pass**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -am
```
Expected: All existing tests pass. The grammar change is additive — `ruleParameterList?` is optional.

- [ ] **Step 3: Commit**

```bash
git add drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4
git commit -m "feat(grammar): add optional parameter list to rule declaration

Refs #41"
```

---

### Task 2: IR model — add RuleParameterIR and extend RuleIR

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:21-25`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:182` (RuleIR constructor call)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java:90,165` (proto serialization)
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto:16-22`

- [ ] **Step 1: Add RuleParameterIR record and extend RuleIR**

In `DrlxRuleAstModel.java`, add the new record before `RuleIR` and add the `parameters` field:

```java
public record RuleParameterIR(String typeName, String paramName) { }

public record RuleIR(String name,
                     List<RuleAnnotationIR> annotations,
                     List<RuleParameterIR> parameters,
                     List<LhsItemIR> lhs,
                     ConsequenceIR rhs) {
}
```

- [ ] **Step 2: Fix visitor — pass empty parameters for existing rules**

In `DrlxToRuleAstVisitor.java`, update the `buildRule()` method. At the end (~line 182), the `RuleIR` constructor call currently passes 4 args. Add `List.of()` for parameters:

```java
return new RuleIR(name, annotations, List.of(), List.copyOf(lhs), rhs);
```

- [ ] **Step 3: Fix proto serialization — add parameters to RuleParseResult**

In `drlx_rule_ast.proto`, add a `RuleParameterParseResult` message and a `parameters` field to `RuleParseResult`:

```protobuf
message RuleParseResult {
  string name = 1;
  reserved 2;
  repeated RuleAnnotationParseResult annotations = 3;
  repeated LhsItemParseResult lhs = 4;
  ConsequenceParseResult rhs = 5;
  repeated RuleParameterParseResult parameters = 6;
}

message RuleParameterParseResult {
  string type_name = 1;
  string param_name = 2;
}
```

In `DrlxRuleAstParseResult.java`, update `toProtoRule()` to serialize parameters:

```java
private static DrlxRuleAstProto.RuleParseResult toProtoRule(RuleIR rule) {
    DrlxRuleAstProto.RuleParseResult.Builder builder = DrlxRuleAstProto.RuleParseResult.newBuilder()
            .setName(rule.name());
    rule.lhs().forEach(item -> builder.addLhs(toProtoLhs(item)));
    if (rule.rhs() != null) {
        builder.setRhs(DrlxRuleAstProto.ConsequenceParseResult.newBuilder()
                .setBlock(rule.rhs().block()));
    }
    for (RuleAnnotationIR ann : rule.annotations()) {
        builder.addAnnotations(DrlxRuleAstProto.RuleAnnotationParseResult.newBuilder()
                .setKind(toProtoKind(ann.kind()))
                .setRawValue(ann.rawValue())
                .build());
    }
    for (RuleParameterIR param : rule.parameters()) {
        builder.addParameters(DrlxRuleAstProto.RuleParameterParseResult.newBuilder()
                .setTypeName(param.typeName())
                .setParamName(param.paramName())
                .build());
    }
    return builder.build();
}
```

Update the deserialization in the `load()` method to read parameters:

```java
List<RuleParameterIR> parameters = new ArrayList<>();
for (DrlxRuleAstProto.RuleParameterParseResult paramPR : ruleParseResult.getParametersList()) {
    parameters.add(new RuleParameterIR(paramPR.getTypeName(), paramPR.getParamName()));
}

rules.add(new RuleIR(
        ruleParseResult.getName(),
        List.copyOf(ruleAnnotations),
        List.copyOf(parameters),
        List.copyOf(lhs),
        rhs));
```

- [ ] **Step 4: Verify existing tests still pass**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -am
```
Expected: All existing tests pass. Regular rules use `List.of()` for parameters.

- [ ] **Step 5: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
       drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
       drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java \
       drlx-parser-core/src/main/proto/drlx_rule_ast.proto
git commit -m "feat(ir): add RuleParameterIR and extend RuleIR with parameters

Refs #41"
```

---

### Task 3: Visitor — parse rule parameters from grammar

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:90-93,182`

- [ ] **Step 1: Write the failing test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryParseTest.java`:

```java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.builder.DrlxRuleAstModel.CompilationUnitIR;
import org.drools.drlx.builder.DrlxRuleAstModel.RuleIR;
import org.drools.drlx.builder.DrlxRuleAstModel.RuleParameterIR;
import org.drools.drlx.builder.DrlxRuleBuilder;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class QueryParseTest {

    @Test
    void queryParametersParsed() {
        String source = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule FindPerson(String name, Person result) {
                    result : /persons[name == name]
                }
                """;

        CompilationUnitIR ast = DrlxRuleBuilder.parseToAst(source);
        RuleIR rule = ast.rules().get(0);
        assertThat(rule.name()).isEqualTo("FindPerson");
        assertThat(rule.parameters()).hasSize(2);
        assertThat(rule.parameters().get(0)).isEqualTo(new RuleParameterIR("String", "name"));
        assertThat(rule.parameters().get(1)).isEqualTo(new RuleParameterIR("Person", "result"));
    }

    @Test
    void regularRuleHasEmptyParameters() {
        String source = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R1 {
                    Person p : /persons[age > 10],
                    do { System.out.println(p); }
                }
                """;

        CompilationUnitIR ast = DrlxRuleBuilder.parseToAst(source);
        RuleIR rule = ast.rules().get(0);
        assertThat(rule.parameters()).isEmpty();
    }
}
```

Note: `DrlxRuleBuilder.parseToAst()` is a new public method that exposes the existing `parseToRuleAst()` (currently private). Add it:

In `DrlxRuleBuilder.java`, add a public static method:
```java
public static CompilationUnitIR parseToAst(String drlxSource) {
    return parseToRuleAst(drlxSource);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -Dtest=QueryParseTest -pl . -am
```
Expected: FAIL — `queryParametersParsed` fails because visitor doesn't extract parameters yet (still passes `List.of()`).

- [ ] **Step 3: Implement parameter extraction in visitor**

In `DrlxToRuleAstVisitor.java`, update `buildRule()`. After line 92 (`String name = ctx.identifier().getText();`), add parameter extraction:

```java
List<RuleParameterIR> parameters = List.of();
if (ctx.ruleParameterList() != null) {
    parameters = ctx.ruleParameterList().ruleParameter().stream()
            .map(p -> new RuleParameterIR(p.typeType().getText(), p.identifier().getText()))
            .toList();
}
```

Update the `RuleIR` constructor call at the end of `buildRule()`:

```java
return new RuleIR(name, annotations, parameters, List.copyOf(lhs), rhs);
```

Add the import at the top:
```java
import org.drools.drlx.builder.DrlxRuleAstModel.RuleParameterIR;
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -Dtest=QueryParseTest -pl . -am
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryParseTest.java \
       drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
       drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleBuilder.java
git commit -m "feat(visitor): parse rule parameters into RuleParameterIR

Refs #41"
```

---

### Task 4: DroolsQueryElementsReader — custom ReadAccessor for DroolsQuery.getElements()

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DroolsQueryElementsReader.java`

- [ ] **Step 1: Create the ReadAccessor implementation**

`drools-mvel`'s `ClassFieldReader` is not on the DRLX classpath. Create a lightweight `ReadAccessor` that reads `DroolsQuery.getElements()`:

```java
package org.drools.drlx.builder;

import org.drools.base.base.DroolsQuery;
import org.drools.base.base.ValueResolver;
import org.drools.base.base.ValueType;
import org.drools.base.rule.accessor.ReadAccessor;
import org.kie.api.runtime.rule.FactHandle;

/**
 * Reads the {@code elements} array from a {@link DroolsQuery} fact.
 * Used as the delegate for {@link org.drools.base.base.extractors.ArrayElementReader}
 * when building query parameter declarations in the prefix pattern.
 */
public final class DroolsQueryElementsReader implements ReadAccessor {

    public static final DroolsQueryElementsReader INSTANCE = new DroolsQueryElementsReader();

    private DroolsQueryElementsReader() {}

    @Override
    public Object getValue(ValueResolver valueResolver, Object object) {
        return ((DroolsQuery) object).getElements();
    }

    @Override
    public Object getValue(ValueResolver valueResolver, FactHandle fh) {
        return getValue(valueResolver, fh.getObject());
    }

    @Override public int getIndex() { return -1; }
    @Override public Class<?> getExtractToClass() { return Object[].class; }
    @Override public String getExtractToClassName() { return "java.lang.Object[]"; }
    @Override public ValueType getValueType() { return ValueType.OBJECT_TYPE; }
    @Override public boolean isNullValue(ValueResolver vr, Object o) { return getValue(vr, o) == null; }

    @Override public int getHashCode(ValueResolver vr, Object o) {
        Object v = getValue(vr, o);
        return v != null ? v.hashCode() : 0;
    }

    @Override public boolean getBooleanValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public byte getByteValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public char getCharValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public double getDoubleValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public float getFloatValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public int getIntValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public long getLongValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
    @Override public short getShortValue(ValueResolver vr, Object o) { throw new UnsupportedOperationException(); }
}
```

- [ ] **Step 2: Verify it compiles**

Run:
```bash
mvn -f drlx-parser-core/pom.xml compile -pl . -am
```
Expected: Compiles successfully.

- [ ] **Step 3: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DroolsQueryElementsReader.java
git commit -m "feat: add DroolsQueryElementsReader for query prefix pattern

Refs #41"
```

---

### Task 5: Runtime builder — query definition (QueryImpl + prefix pattern)

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:72-109,324-356`

- [ ] **Step 1: Write the failing test**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`:

```java
package org.drools.drlx.builder.syntax;

import java.util.ArrayList;
import java.util.List;

import org.drools.drlx.domain.Person;
import org.drools.drlx.ruleunit.DrlxRuleUnitInstance;
import org.drools.drlx.ruleunit.MyUnit;
import org.junit.jupiter.api.Test;
import org.kie.api.KieBase;
import org.kie.api.runtime.rule.QueryResults;
import org.kie.api.runtime.rule.QueryResultsRow;
import org.kie.api.runtime.rule.Variable;

import static org.assertj.core.api.Assertions.assertThat;

class QueryTest extends DrlxBuilderTestSupport {

    @Test
    void queryDefinitionViaApi() {
        String source = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule PersonsByAge(int minAge, Person result) {
                    result : /persons[age >= minAge]
                }
                """;

        KieBase kieBase = newBuilder().build(source);
        MyUnit unit = new MyUnit();
        unit.persons.add(new Person("Alice", 30));
        unit.persons.add(new Person("Bob", 20));
        unit.persons.add(new Person("Charlie", 40));

        try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
            QueryResults results = instance.executeQuery("PersonsByAge", 25, Variable.v);

            List<String> names = new ArrayList<>();
            for (QueryResultsRow row : results) {
                Person p = (Person) row.get("result");
                names.add(p.getName());
            }
            assertThat(names).containsExactlyInAnyOrder("Alice", "Charlie");
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -Dtest=QueryTest#queryDefinitionViaApi -pl . -am
```
Expected: FAIL — runtime builder creates `RuleImpl` instead of `QueryImpl`.

- [ ] **Step 3: Implement query definition in runtime builder**

In `DrlxRuleAstRuntimeBuilder.java`:

**3a. Add imports:**

```java
import org.drools.base.base.DroolsQuery;
import org.drools.base.definitions.rule.impl.QueryImpl;
import org.drools.base.rule.QueryElement;
import org.drools.base.rule.QueryArgument;
import org.drools.base.rule.constraint.QueryNameConstraint;
import org.drools.drlx.builder.DrlxRuleAstModel.RuleParameterIR;
```

**3b. Restructure `build()` for two-pass compilation:**

Replace the single `parseResult.rules().forEach(...)` block (~line 102-104) with:

```java
Map<String, QueryImpl> queryRegistry = new LinkedHashMap<>();

// Pass 1: compile queries (rules with parameters)
for (RuleIR rule : parseResult.rules()) {
    if (!rule.parameters().isEmpty()) {
        QueryImpl query = buildQuery(rule, pkg.getTypeResolver(), entryPointTypes, unitClass);
        String entryPointName = Character.toLowerCase(rule.name().charAt(0)) + rule.name().substring(1);
        queryRegistry.put(entryPointName, query);
        pkg.addRule(query);
    }
}

// Pass 2: compile regular rules (can reference queries)
for (RuleIR rule : parseResult.rules()) {
    if (rule.parameters().isEmpty()) {
        pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass,
                             globalTypes, dataStoreGlobalNames, updateRewriter, queryRegistry));
    }
}
```

**3c. Add `buildQuery()` method** (after `buildRule()`):

```java
private QueryImpl buildQuery(RuleIR parseResult,
                             TypeResolver typeResolver,
                             Map<String, Class<?>> entryPointTypes,
                             Class<?> unitClass) {
    lambdaCompiler.beginRule(parseResult.name());

    QueryImpl query = new QueryImpl(parseResult.name());

    // Build prefix pattern on DroolsQuery
    Pattern prefixPattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0,
            ClassObjectType.DroolsQuery_ObjectType, null);

    // QueryNameConstraint matches query name at runtime
    QueryNameConstraint nameConstraint = new QueryNameConstraint(null, query.getName());
    prefixPattern.addConstraint(nameConstraint);

    // ArrayElementReader for each parameter
    List<RuleParameterIR> params = parseResult.parameters();
    Declaration[] paramDecls = new Declaration[params.size()];
    for (int i = 0; i < params.size(); i++) {
        RuleParameterIR param = params.get(i);
        Class<?> paramType = resolveOrThrow(param.typeName(), typeResolver);
        Declaration decl = prefixPattern.addDeclaration(param.paramName());
        ArrayElementReader reader = new ArrayElementReader(
                DroolsQueryElementsReader.INSTANCE, i, paramType);
        decl.setReadAccessor(reader);
        paramDecls[i] = decl;
    }
    query.setParameters(paramDecls);

    // Build LHS with parameter bindings pre-populated
    GroupElement root = GroupElementFactory.newAndInstance();
    root.addChild(prefixPattern);

    Map<String, BoundVariable> boundVariables = new LinkedHashMap<>();
    for (int i = 0; i < params.size(); i++) {
        RuleParameterIR param = params.get(i);
        Class<?> paramType = resolveOrThrow(param.typeName(), typeResolver);
        boundVariables.put(param.paramName(),
                new BoundVariable(param.paramName(), paramType, prefixPattern, paramDecls[i]));
    }

    buildLhs(parseResult.lhs(), root, typeResolver, entryPointTypes, unitClass, boundVariables);

    // No consequence for queries
    query.setLhs(root);
    return query;
}
```

**3d. Update `buildRule()` signature** to accept `queryRegistry`:

```java
private RuleImpl buildRule(RuleIR parseResult,
                           TypeResolver typeResolver,
                           Map<String, Class<?>> entryPointTypes,
                           Class<?> unitClass,
                           Map<String, java.lang.reflect.Type> globalTypes,
                           Set<String> dataStoreGlobalNames,
                           DataStoreUpdateRewriter updateRewriter,
                           Map<String, QueryImpl> queryRegistry) {
```

The body of `buildRule()` remains unchanged for now (query invocation is Task 6).

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -Dtest=QueryTest#queryDefinitionViaApi -pl . -am
```
Expected: PASS

- [ ] **Step 5: Run all existing tests to verify no regressions**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -am
```
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
       drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git commit -m "feat(builder): compile query definitions into QueryImpl with prefix pattern

Refs #41"
```

---

### Task 6: Runtime builder — query invocation (QueryElement)

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` (buildLhs, buildPattern area)

- [ ] **Step 1: Write the failing test**

Add to `QueryTest.java`:

```java
@Test
void queryInvocationFromRule() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                result : /persons[age >= minAge]
            }

            rule R1 {
                /personsByAge(25, var p),
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

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -Dtest=QueryTest#queryInvocationFromRule -pl . -am
```
Expected: FAIL — builder treats `/personsByAge(25, var p)` as a DataSource pattern.

- [ ] **Step 3: Implement query invocation detection and QueryElement building**

In `DrlxRuleAstRuntimeBuilder.java`:

**3a. Thread `queryRegistry` through `buildLhs()`:**

Update `buildLhs()` signature to accept `queryRegistry`:

```java
private void buildLhs(List<LhsItemIR> items,
                      GroupElement parent,
                      TypeResolver typeResolver,
                      Map<String, Class<?>> entryPointTypes,
                      Class<?> unitClass,
                      Map<String, BoundVariable> boundVariables,
                      Map<String, QueryImpl> queryRegistry) {
```

Update all call sites of `buildLhs()` (in `buildRule()`, `buildQuery()`, and inside `buildLhs()` itself for nested groups, and in `buildAccumulatePattern()` / `buildCustomAccumulatePattern()`) to pass `queryRegistry`. In `buildQuery()`, pass `Map.of()` since queries don't invoke other queries in v1.

**3b. Add query invocation detection in `buildLhs()`:**

In the `PatternIR` branch of `buildLhs()`, before building a normal pattern, check if the entry point maps to a query:

```java
if (item instanceof PatternIR patternIr) {
    QueryImpl targetQuery = queryRegistry.get(patternIr.entryPoint());
    if (targetQuery != null && !patternIr.positionalArgs().isEmpty()) {
        QueryElement queryElement = buildQueryElement(
                patternIr, targetQuery, typeResolver, boundVariables);
        parent.addChild(queryElement);
        // Register output variable bindings
        Declaration[] queryParams = targetQuery.getParameters();
        List<String> args = patternIr.positionalArgs();
        for (int i = 0; i < args.size(); i++) {
            String arg = args.get(i);
            if (arg.startsWith("var ") || arg.startsWith("var\t")) {
                String varName = arg.substring(4).trim();
                Class<?> paramType = queryParams[i].getDeclarationClass();
                Pattern resultPattern = queryElement.getResultPattern();
                Declaration decl = resultPattern.getDeclarations().get(varName);
                if (decl != null) {
                    boundVariables.put(varName,
                            new BoundVariable(varName, paramType, resultPattern, decl));
                }
            }
        }
        continue; // skip normal pattern building
    }
    Pattern pattern = buildPattern(patternIr, typeResolver, entryPointTypes, unitClass, boundVariables);
    // ... rest unchanged
```

Note: restructure the if-else so the existing Pattern logic stays in the else branch. Use a loop with continue or restructure as needed.

**3c. Add `buildQueryElement()` method:**

```java
private QueryElement buildQueryElement(PatternIR patternIr,
                                       QueryImpl targetQuery,
                                       TypeResolver typeResolver,
                                       Map<String, BoundVariable> boundVariables) {
    Declaration[] queryParams = targetQuery.getParameters();
    List<String> args = patternIr.positionalArgs();

    if (args.size() != queryParams.length) {
        throw new RuntimeException(
                "query '" + targetQuery.getName() + "' expects " + queryParams.length
                + " arguments but got " + args.size());
    }

    QueryArgument[] arguments = new QueryArgument[args.size()];
    List<Integer> varIndexes = new ArrayList<>();
    List<Declaration> requiredDeclarations = new ArrayList<>();

    for (int i = 0; i < args.size(); i++) {
        String arg = args.get(i);
        if (arg.startsWith("var ") || arg.startsWith("var\t")) {
            // Output variable
            arguments[i] = QueryArgument.VAR;
            varIndexes.add(i);
        } else {
            BoundVariable bv = boundVariables.get(arg);
            if (bv != null) {
                // Input from a prior binding
                arguments[i] = new QueryArgument.Declr(bv.declaration());
                requiredDeclarations.add(bv.declaration());
            } else {
                // Literal value
                arguments[i] = new QueryArgument.Literal(parseLiteral(arg));
            }
        }
    }

    // Build result pattern for output variables
    ReadAccessor selfReader = new SelfReferenceClassFieldReader(Object[].class);
    Pattern resultPattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0,
            new ClassObjectType(Object[].class), null);

    for (int idx : varIndexes) {
        String varName = args.get(idx).substring(4).trim();
        Class<?> paramType = queryParams[idx].getDeclarationClass();
        Declaration decl = resultPattern.addDeclaration(varName);
        decl.setReadAccessor(new ArrayElementReader(selfReader, idx, paramType));
    }

    int[] varIndexArray = varIndexes.stream().mapToInt(Integer::intValue).toArray();

    return new QueryElement(
            resultPattern,
            targetQuery.getName(),
            arguments,
            varIndexArray,
            requiredDeclarations.toArray(new Declaration[0]),
            true,   // openQuery (reactive)
            false); // not abductive
}

private static Object parseLiteral(String literal) {
    if (literal.startsWith("\"") && literal.endsWith("\"")) {
        return literal.substring(1, literal.length() - 1);
    }
    try {
        return Integer.parseInt(literal);
    } catch (NumberFormatException e) {
        try {
            return Long.parseLong(literal);
        } catch (NumberFormatException e2) {
            try {
                return Double.parseDouble(literal);
            } catch (NumberFormatException e3) {
                return literal;
            }
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -Dtest=QueryTest#queryInvocationFromRule -pl . -am
```
Expected: PASS

- [ ] **Step 5: Run all tests to verify no regressions**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -am
```
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
       drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git commit -m "feat(builder): compile query invocations into QueryElement

Refs #41"
```

---

### Task 7: Additional tests — multiple parameters and error cases

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Add multi-parameter test**

Add to `QueryTest.java`:

```java
@Test
void queryMultipleParameters() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAgeRange(int minAge, int maxAge, Person result) {
                result : /persons[age >= minAge, age <= maxAge]
            }

            rule R1 {
                /personsByAgeRange(25, 35, var p),
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
        assertThat(names).containsExactlyInAnyOrder("Alice");
    }
}
```

- [ ] **Step 2: Add wrong argument count error test**

```java
@Test
void queryWrongArgCountFails() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAge(int minAge, Person result) {
                result : /persons[age >= minAge]
            }

            rule R1 {
                /personsByAge(25),
                do { System.out.println("wrong"); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(source))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("expects 2 arguments but got 1");
}
```

Add import for `assertThatThrownBy`:
```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
```

- [ ] **Step 3: Add query definition with API and multiple parameters test**

```java
@Test
void queryDefinitionViaApiMultipleParams() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule PersonsByAgeRange(int minAge, int maxAge, Person result) {
                result : /persons[age >= minAge, age <= maxAge]
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 30));
    unit.persons.add(new Person("Bob", 20));
    unit.persons.add(new Person("Charlie", 40));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        QueryResults results = instance.executeQuery("PersonsByAgeRange", 25, 35, Variable.v);

        List<String> names = new ArrayList<>();
        for (QueryResultsRow row : results) {
            Person p = (Person) row.get("result");
            names.add(p.getName());
        }
        assertThat(names).containsExactlyInAnyOrder("Alice");
    }
}
```

- [ ] **Step 4: Run all query tests**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -Dtest=QueryTest -pl . -am
```
Expected: All 4 tests pass.

- [ ] **Step 5: Run full test suite**

Run:
```bash
mvn -f drlx-parser-core/pom.xml test -pl . -am
```
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git commit -m "test: add multi-parameter and error case tests for queries

Refs #41"
```

---

### Task 8: MyUnit — add DataStore fields for query entry points

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`

This task may need to be done before Task 5 depending on whether the unit class requires DataSource fields for query entry points. The query `PersonsByAge` maps to entry point `personsByAge`, which needs a `DataStore<Person> personsByAge` on the unit class.

- [ ] **Step 1: Add query-related DataStore fields to MyUnit**

```java
public DataStore<Person> personsByAge = DataSource.createStore();
public DataStore<Person> personsByAgeRange = DataSource.createStore();
```

- [ ] **Step 2: Commit**

```bash
git add drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java
git commit -m "test: add DataStore fields for query entry points in MyUnit

Refs #41"
```

**Important:** This task should be executed before Task 5, as the tests in Task 5 depend on these fields.

---

## Execution Order

The tasks should be executed in this order:
1. Task 1 (Grammar)
2. Task 2 (IR model)
3. Task 3 (Visitor)
4. Task 4 (DroolsQueryElementsReader)
5. **Task 8 (MyUnit DataStore fields)** — before Task 5
6. Task 5 (Runtime builder — query definition)
7. Task 6 (Runtime builder — query invocation)
8. Task 7 (Additional tests)

## Notes

- The `var` keyword in positional args (`var p`) is detected as a string prefix in the positional arg text. The ANTLR grammar already parses `var p` as a single expression (`var` is a keyword in the lexer producing a `VAR` token, which gets merged with the identifier `p` in the expression rule). Verify during implementation that `patternIr.positionalArgs()` produces `"var p"` as a single string for this input — if not, the visitor's `extractPositionalArgs()` may need adjustment.
- The `QueryNameConstraint` constructor receives `null` for the `ReadAccessor` (first parameter). This is safe because `QueryNameConstraint.isAllowed()` does not use the read accessor — it directly casts the fact handle's object to `DroolsQuery` and calls `getName()`. The read accessor field exists for index-building, which is not needed for the query name match.
