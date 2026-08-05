# Multi-Segment OOPath Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Support multi-segment OOPath expressions like `/persons/addresses[city == "London"]` end-to-end: parsing → IR → protobuf → runtime execution.

**Architecture:** Add `OopathSegmentIR` to the IR model, populate it in `DrlxToRuleAstVisitor`, serialize it in protobuf, and consume it in `DrlxRuleAstRuntimeBuilder` by chaining drools `Pattern`/`From` objects. A new `OopathFieldDataProvider` bridges the `From` source to a getter invocation.

**Tech Stack:** Java 17, ANTLR4, protobuf, drools-core (`Pattern`, `From`, `DataProvider`), JUnit 5, AssertJ.

## Global Constraints

- Ruleunit-only — first OOPath segment is always a `DataStore` entry point
- Single-segment OOPath must remain unchanged (backward compatible)
- All existing tests must continue to pass
- Use `mvn -f <pom> test` to run tests, not `cd`

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `DrlxRuleAstModel.java` | Modify | Add `OopathSegmentIR` record, extend `PatternIR` |
| `DrlxToRuleAstVisitor.java` | Modify | Extract segments from ANTLR chunks, fix `extractConditions` |
| `drlx_rule_ast.proto` | Modify | Add `OopathSegmentParseResult` message, extend `PatternParseResult` |
| `DrlxRuleAstParseResult.java` | Modify | Serialize/deserialize segments |
| `OopathFieldDataProvider.java` | Create | `DataProvider` impl for `From` source |
| `DrlxRuleAstRuntimeBuilder.java` | Modify | Chain patterns for multi-segment OOPath |
| `Person.java` (test domain) | Modify | Add `List<Address> addresses` field |
| `DrlxRuleAstParseResultTest.java` | Modify | Add protobuf round-trip test for segments |
| `MultiSegmentOOPathTest.java` | Create | Execution tests |

All source files are under:
`/home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/`

---

### Task 1: IR Model — `OopathSegmentIR` and `PatternIR` Extension

**Files:**
- Modify: `main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`

**Interfaces:**
- Produces: `OopathSegmentIR(String field, List<String> conditions, String inlineCast)` record
- Produces: `PatternIR` with new 12th parameter `List<OopathSegmentIR> oopathSegments`

- [ ] **Step 1: Add `OopathSegmentIR` record**

In `DrlxRuleAstModel.java`, add after the `TemporalConditionIR` record (after line 79):

```java
public record OopathSegmentIR(String field,
                               List<String> conditions,
                               String inlineCast) {
    public OopathSegmentIR {
        conditions = List.copyOf(conditions);
    }
}
```

- [ ] **Step 2: Extend `PatternIR` with `oopathSegments`**

Replace the `PatternIR` record (lines 57-68) with:

```java
public record PatternIR(String typeName,
                        String bindName,
                        String entryPoint,
                        List<String> conditions,
                        List<TemporalConditionIR> temporalConditions,
                        String castTypeName,
                        List<String> positionalArgs,
                        boolean passive,
                        List<String> watchedProperties,
                        String windowType,
                        String windowParameter,
                        List<OopathSegmentIR> oopathSegments) implements LhsItemIR {
}
```

- [ ] **Step 3: Fix all existing `PatternIR` constructor calls**

Every existing `new PatternIR(...)` call now needs a 12th argument: `List.of()`.

**In `DrlxToRuleAstVisitor.java`** — 4 call sites at lines 1213, 1240, 1252, 1263. Add `List.of()` as last argument to each.

**In `DrlxRuleAstParseResult.java`** — 1 call site at line 183 (`patternFromProto`). Add `List.of()` as last argument.

**In `DrlxRuleAstParseResultTest.java`** — 4 call sites at lines 49, 89, 127, 158. Add `List.of()` as last argument to each.

**In `DrlxRuleAstModelTest.java`** — search for `new PatternIR(` and add `List.of()` if present.

- [ ] **Step 4: Verify compilation**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile test-compile
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Run existing tests to confirm no regressions**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```
Expected: All tests pass. No test should break because every call site passes `List.of()`.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add -A
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: add OopathSegmentIR and extend PatternIR (#103)

Add OopathSegmentIR record for multi-segment OOPath support.
Extend PatternIR with oopathSegments list (empty for all existing code).

Refs #103"
```

---

### Task 2: Visitor — Extract OOPath Segments

**Files:**
- Modify: `main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Test: `test/java/org/drools/drlx/parser/DrlxToJavaParserVisitorTest.java` (parser test already exists from earlier work)

**Interfaces:**
- Consumes: `OopathSegmentIR(String field, List<String> conditions, String inlineCast)` from Task 1
- Produces: `PatternIR` instances with populated `oopathSegments` when multi-segment OOPath is parsed

- [ ] **Step 1: Add `extractOopathSegments` method**

In `DrlxToRuleAstVisitor.java`, add after `extractConditions` (around line 1336):

```java
private List<DrlxRuleAstModel.OopathSegmentIR> extractOopathSegments(
        DrlxParser.OopathExpressionContext ctx) {
    List<DrlxParser.OopathChunkContext> chunks = ctx.oopathChunk();
    if (chunks.isEmpty()) {
        return List.of();
    }
    return chunks.stream().map(c -> {
        String field = c.identifier(0).getText();
        String cast = c.identifier().size() > 1
                ? c.identifier(1).getText() : null;
        List<String> conds = c.drlxExpression().stream()
                .filter(de -> de.customConstraint() == null)
                .map(this::getText)
                .toList();
        return new DrlxRuleAstModel.OopathSegmentIR(field, conds, cast);
    }).toList();
}
```

- [ ] **Step 2: Fix `extractConditions` to return root conditions when segments are present**

Replace the existing `extractConditions` method (lines 1331-1336) with:

```java
private List<String> extractConditions(DrlxParser.OopathExpressionContext ctx) {
    return collectRootDrlxExpressions(ctx).stream()
            .filter(de -> de.customConstraint() == null)
            .map(this::getText)
            .toList();
}
```

Replace `collectDrlxExpressions` (lines 1348-1359) with a new version that always returns the **root's** expressions (not the last chunk's). When segments are present, their conditions go into `OopathSegmentIR` via `extractOopathSegments` instead:

```java
private List<DrlxParser.DrlxExpressionContext> collectRootDrlxExpressions(
        DrlxParser.OopathExpressionContext ctx) {
    DrlxParser.OopathRootContext root = ctx.oopathRoot();
    if (root == null) {
        return List.of();
    }
    return root.drlxExpression();
}
```

Also update `extractTemporalConditions` (lines 1338-1346) to use `collectRootDrlxExpressions`:

```java
private List<DrlxRuleAstModel.TemporalConditionIR> extractTemporalConditions(
        DrlxParser.OopathExpressionContext ctx) {
    List<DrlxRuleAstModel.TemporalConditionIR> result = new ArrayList<>();
    for (var de : collectRootDrlxExpressions(ctx)) {
        if (de.customConstraint() != null) {
            result.add(buildTemporalCondition(de.customConstraint()));
        }
    }
    return List.copyOf(result);
}
```

- [ ] **Step 3: Pass segments to all `buildPatternFrom*` methods**

Update `buildPatternFromBoundOopath` (lines 1219-1243) — add segment extraction and pass to `PatternIR`:

```java
private PatternIR buildPatternFromBoundOopath(DrlxParser.BoundOopathContext ctx) {
    String typeName = ctx.identifier(0).getText();
    String bindName = ctx.identifier(1).getText();
    DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
    String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
    String castTypeName = extractCastType(oopathCtx);
    List<String> conditions = extractConditions(oopathCtx);
    List<DrlxRuleAstModel.TemporalConditionIR> temporalConditions = extractTemporalConditions(oopathCtx);
    List<String> positionalArgs = extractPositionalArgs(oopathCtx);
    boolean passive = ctx.oopathExpression().QUESTION() != null;
    List<String> watchedProperties = extractWatchedProperties(ctx.oopathExpression());
    List<DrlxRuleAstModel.OopathSegmentIR> segments = extractOopathSegments(oopathCtx);
    String windowType = null;
    String windowParameter = null;
    if (ctx.windowFilter() != null) {
        windowType = ctx.windowFilter().identifier().getText();
        if (!"length".equals(windowType) && !"time".equals(windowType)) {
            throw new IllegalArgumentException("Unknown window type: " + windowType
                    + ". Expected 'length' or 'time'.");
        }
        windowParameter = ctx.windowFilter().windowParam().getText();
    }
    return new PatternIR(typeName, bindName, entryPoint, conditions, temporalConditions,
                          castTypeName, positionalArgs, passive, watchedProperties,
                          windowType, windowParameter, segments);
}
```

Update both `buildPatternFromOopath` overloads (lines 1245-1253 and 1255-1265) similarly — add `extractOopathSegments(oopathCtx)` as the last argument.

Update `buildWindowDeclaration` (lines 1197-1217) — add `extractOopathSegments(oopathCtx)` as the last argument to the `PatternIR` constructor at line 1213.

- [ ] **Step 4: Run the existing multi-segment parser test**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxToJavaParserVisitorTest#testParseMultiSegmentOOPath
```
Expected: PASS (this test exercises `DrlxToJavaParserVisitor`, not `DrlxToRuleAstVisitor`, so it was already working).

- [ ] **Step 5: Run all tests to confirm no regressions**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```
Expected: All tests pass. The `extractConditions` change matters: single-segment OOPath has no chunks, so `collectRootDrlxExpressions` returns the root's conditions, which is the same behavior as before. Multi-segment OOPath now correctly splits root conditions into `PatternIR.conditions` and segment conditions into `oopathSegments`.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add -A
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: extract OOPath segments in DrlxToRuleAstVisitor (#103)

Add extractOopathSegments method. Fix extractConditions to return root
conditions only (segment conditions go into OopathSegmentIR).
Update all buildPatternFrom* methods to pass segments to PatternIR.

Refs #103"
```

---

### Task 3: Protobuf Serialization

**Files:**
- Modify: `main/proto/drlx_rule_ast.proto`
- Modify: `main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`
- Modify: `test/java/org/drools/drlx/builder/DrlxRuleAstParseResultTest.java`

**Interfaces:**
- Consumes: `OopathSegmentIR` record from Task 1
- Produces: Protobuf round-trip support for `PatternIR` with segments

- [ ] **Step 1: Write the failing round-trip test**

In `DrlxRuleAstParseResultTest.java`, add:

```java
@Test
void oopathSegmentsRoundTripThroughProto() {
    PatternIR ir = new PatternIR(
            "var", "addr", "persons",
            List.of("age > 18"),
            List.of(),
            null,
            List.of(),
            false,
            List.of(),
            null, null,
            List.of(
                new DrlxRuleAstModel.OopathSegmentIR("addresses",
                        List.of("city == \"London\""), null),
                new DrlxRuleAstModel.OopathSegmentIR("streets",
                        List.of(), "MainStreet")
            ));

    DrlxRuleAstProto.LhsItemParseResult lhsItem = DrlxRuleAstParseResult.toProtoLhs(ir);
    PatternIR back = (PatternIR) DrlxRuleAstParseResult.fromProtoLhs(
            lhsItem, Path.of("test.drlx"));

    assertThat(back.entryPoint()).isEqualTo("persons");
    assertThat(back.conditions()).containsExactly("age > 18");
    assertThat(back.oopathSegments()).hasSize(2);
    assertThat(back.oopathSegments().get(0).field()).isEqualTo("addresses");
    assertThat(back.oopathSegments().get(0).conditions())
            .containsExactly("city == \"London\"");
    assertThat(back.oopathSegments().get(0).inlineCast()).isNull();
    assertThat(back.oopathSegments().get(1).field()).isEqualTo("streets");
    assertThat(back.oopathSegments().get(1).conditions()).isEmpty();
    assertThat(back.oopathSegments().get(1).inlineCast()).isEqualTo("MainStreet");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstParseResultTest#oopathSegmentsRoundTripThroughProto
```
Expected: FAIL — segments are not serialized yet.

- [ ] **Step 3: Add proto message**

In `drlx_rule_ast.proto`, add before the `ConsequenceParseResult` message (before line 77):

```proto
message OopathSegmentParseResult {
  string field = 1;
  repeated string conditions = 2;
  string inline_cast = 3;
}
```

Add to `PatternParseResult` (after line 67):

```proto
  repeated OopathSegmentParseResult oopath_segments = 12;
```

- [ ] **Step 4: Regenerate protobuf Java classes**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources
```

- [ ] **Step 5: Update `patternToProto` in `DrlxRuleAstParseResult.java`**

In `patternToProto` (around line 272), add before the `return pb.build()` at line 296:

```java
for (DrlxRuleAstModel.OopathSegmentIR seg : p.oopathSegments()) {
    DrlxRuleAstProto.OopathSegmentParseResult.Builder sb =
            DrlxRuleAstProto.OopathSegmentParseResult.newBuilder()
                    .setField(seg.field());
    seg.conditions().forEach(sb::addConditions);
    if (seg.inlineCast() != null) {
        sb.setInlineCast(seg.inlineCast());
    }
    pb.addOopathSegments(sb);
}
```

- [ ] **Step 6: Update `patternFromProto` in `DrlxRuleAstParseResult.java`**

In `patternFromProto` (around line 172), add before the `return new PatternIR(...)` and include segments as the 12th constructor argument:

```java
List<DrlxRuleAstModel.OopathSegmentIR> oopathSegments =
        pattern.getOopathSegmentsList().stream()
                .map(seg -> new DrlxRuleAstModel.OopathSegmentIR(
                        seg.getField(),
                        List.copyOf(seg.getConditionsList()),
                        seg.getInlineCast().isEmpty() ? null : seg.getInlineCast()))
                .toList();
```

Then replace `List.of()` (the 12th argument added in Task 1) with `oopathSegments` in the `return new PatternIR(...)` call.

- [ ] **Step 7: Run test to verify it passes**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=DrlxRuleAstParseResultTest
```
Expected: All round-trip tests pass, including the new one.

- [ ] **Step 8: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add -A
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: protobuf serialization for OOPath segments (#103)

Add OopathSegmentParseResult proto message. Serialize/deserialize
OopathSegmentIR in patternToProto/patternFromProto.

Refs #103"
```

---

### Task 4: Domain Model, DataProvider, and Runtime Builder

**Files:**
- Modify: `test/java/org/drools/drlx/domain/Person.java`
- Create: `main/java/org/drools/drlx/builder/OopathFieldDataProvider.java`
- Modify: `main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Create: `test/java/org/drools/drlx/builder/syntax/MultiSegmentOOPathTest.java`

**Interfaces:**
- Consumes: `PatternIR.oopathSegments()` from Task 1
- Consumes: `OopathSegmentIR(field, conditions, inlineCast)` from Task 1
- Produces: Chained `Pattern`/`From` objects in drools runtime

- [ ] **Step 1: Add `List<Address> addresses` to `Person.java`**

In `test/java/org/drools/drlx/domain/Person.java`, add after line 10 (`private String value3;`):

```java
private final List<Address> addresses = new ArrayList<>();
```

Add import at top:
```java
import java.util.ArrayList;
import java.util.List;
```

Add methods after the `setValue3` method:

```java
public List<Address> getAddresses() {
    return addresses;
}

public void addAddress(Address address) {
    addresses.add(address);
}
```

- [ ] **Step 2: Write the failing execution test**

Create `test/java/org/drools/drlx/builder/syntax/MultiSegmentOOPathTest.java`:

```java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.domain.Address;
import org.drools.drlx.domain.Person;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MultiSegmentOOPathTest extends DrlxBuilderTestSupport {

    @Test
    void twoSegmentOOPathCollectsMatchingAddresses() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.Address;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var addr : /persons/addresses[ city == "London" ],
                    do { results.add(addr.city); }
                }
                """;
        withMyUnitInstance(rule, (instance, unit, listener) -> {
            Person alice = new Person("Alice", 30);
            alice.addAddress(new Address("London"));
            alice.addAddress(new Address("Paris"));

            Person bob = new Person("Bob", 25);
            bob.addAddress(new Address("London"));

            unit.persons.add(alice);
            unit.persons.add(bob);

            assertThat(instance.fire()).isEqualTo(2);
            assertThat(unit.results).containsExactlyInAnyOrder("London", "London");
        });
    }

    @Test
    void twoSegmentWithRootConstraint() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.Address;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var addr : /persons[ age > 25 ]/addresses[ city == "London" ],
                    do { results.add(addr.city); }
                }
                """;
        withMyUnitInstance(rule, (instance, unit, listener) -> {
            Person alice = new Person("Alice", 30);
            alice.addAddress(new Address("London"));

            Person bob = new Person("Bob", 20);
            bob.addAddress(new Address("London"));

            unit.persons.add(alice);
            unit.persons.add(bob);

            // Only Alice's London matches (Bob is age 20, fails root constraint)
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(unit.results).containsExactly("London");
        });
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=MultiSegmentOOPathTest
```
Expected: FAIL — runtime builder doesn't handle segments yet.

- [ ] **Step 4: Create `OopathFieldDataProvider`**

Create `main/java/org/drools/drlx/builder/OopathFieldDataProvider.java`:

```java
package org.drools.drlx.builder;

import java.lang.reflect.Method;
import java.util.Collections;
import java.util.Iterator;

import org.drools.base.base.ValueResolver;
import org.drools.base.reteoo.BaseTuple;
import org.drools.base.rule.Declaration;
import org.drools.base.rule.accessor.DataProvider;

public class OopathFieldDataProvider implements DataProvider {

    private final Declaration sourceDeclaration;
    private final Method getter;

    public OopathFieldDataProvider(Declaration sourceDeclaration, Method getter) {
        this.sourceDeclaration = sourceDeclaration;
        this.getter = getter;
    }

    @Override
    public Declaration[] getRequiredDeclarations() {
        return new Declaration[]{ sourceDeclaration };
    }

    @Override
    public Object createContext() {
        return null;
    }

    @Override
    public Iterator getResults(BaseTuple tuple, ValueResolver valueResolver,
                               Object providerContext) {
        Object sourceObj = sourceDeclaration.getValue(valueResolver,
                tuple.getFactHandle().getObject());
        Object result;
        try {
            result = getter.invoke(sourceObj);
        } catch (Exception e) {
            throw new RuntimeException("Failed to invoke getter " + getter.getName()
                    + " on " + sourceObj, e);
        }
        if (result == null) {
            return Collections.emptyIterator();
        }
        if (result instanceof Iterable<?> iterable) {
            return iterable.iterator();
        }
        return Collections.singletonList(result).iterator();
    }

    @Override
    public DataProvider clone() {
        return this;
    }

    @Override
    public void replaceDeclaration(Declaration declaration, Declaration resolved) {
        // no-op — declarations are fixed at build time
    }

    @Override
    public boolean isReactive() {
        return false;
    }
}
```

- [ ] **Step 5: Implement multi-segment pattern chaining in `DrlxRuleAstRuntimeBuilder`**

In `DrlxRuleAstRuntimeBuilder.java`, modify the `buildPattern` method (starting at line 1728). Currently it returns a single `Pattern`. Change it to accept the `GroupElement parent` and `Map<String, BoundVariable> boundVariables` and add all patterns directly.

Replace the call site at lines 687-694:

```java
// BEFORE (single pattern):
Pattern pattern = buildPattern(patternIr, typeResolver, entryPointTypes, unitClass, boundVariables);
parent.addChild(pattern);
Declaration declaration = pattern.getDeclaration();
if (declaration != null) {
    Class<?> patternClass = ((ClassObjectType) pattern.getObjectType()).getClassType();
    boundVariables.put(declaration.getIdentifier(),
            new BoundVariable(declaration.getIdentifier(), patternClass, pattern, declaration));
}
```

```java
// AFTER (delegates to method that handles single and multi):
buildPatternChain(patternIr, typeResolver, entryPointTypes, unitClass, boundVariables, parent);
```

Add the new `buildPatternChain` method:

```java
private void buildPatternChain(PatternIR parseResult,
                               TypeResolver typeResolver,
                               Map<String, Class<?>> entryPointTypes,
                               Class<?> unitClass,
                               Map<String, BoundVariable> boundVariables,
                               GroupElement parent) {
    if (parseResult.oopathSegments().isEmpty()) {
        // Single-segment — existing behavior
        Pattern pattern = buildPattern(parseResult, typeResolver, entryPointTypes,
                                       unitClass, boundVariables);
        parent.addChild(pattern);
        Declaration declaration = pattern.getDeclaration();
        if (declaration != null) {
            Class<?> patternClass = ((ClassObjectType) pattern.getObjectType()).getClassType();
            boundVariables.put(declaration.getIdentifier(),
                    new BoundVariable(declaration.getIdentifier(), patternClass,
                                      pattern, declaration));
        }
        return;
    }

    // Multi-segment — build root pattern with auto-generated bind
    String originalBind = parseResult.bindName();
    String rootBind = "_oopath$" + lambdaCompiler.nextPatternId();
    PatternIR rootIr = new PatternIR(
            parseResult.typeName(), rootBind, parseResult.entryPoint(),
            parseResult.conditions(), parseResult.temporalConditions(),
            parseResult.castTypeName(), parseResult.positionalArgs(),
            parseResult.passive(), parseResult.watchedProperties(),
            parseResult.windowType(), parseResult.windowParameter(),
            List.of());
    Pattern rootPattern = buildPattern(rootIr, typeResolver, entryPointTypes,
                                       unitClass, boundVariables);
    parent.addChild(rootPattern);

    Declaration rootDecl = rootPattern.addDeclaration(rootBind);
    Class<?> rootClass = ((ClassObjectType) rootPattern.getObjectType()).getClassType();
    rootDecl.setReadAccessor(new SelfReferenceClassFieldReader(rootClass));
    boundVariables.put(rootBind, new BoundVariable(rootBind, rootClass,
                                                    rootPattern, rootDecl));

    Pattern previousPattern = rootPattern;
    Class<?> previousClass = rootClass;
    Declaration previousDecl = rootDecl;

    for (int i = 0; i < parseResult.oopathSegments().size(); i++) {
        DrlxRuleAstModel.OopathSegmentIR segment = parseResult.oopathSegments().get(i);
        boolean isLast = (i == parseResult.oopathSegments().size() - 1);

        // Resolve getter
        Method getter = findGetterForField(previousClass, segment.field());
        Class<?> fieldType = getter.getReturnType();

        // If Iterable/List, extract generic type parameter
        if (Iterable.class.isAssignableFrom(fieldType)) {
            java.lang.reflect.Type genericType = getter.getGenericReturnType();
            if (genericType instanceof java.lang.reflect.ParameterizedType pt) {
                java.lang.reflect.Type[] typeArgs = pt.getActualTypeArguments();
                if (typeArgs.length > 0 && typeArgs[0] instanceof Class<?> c) {
                    fieldType = c;
                }
            }
        }

        // Inline cast override
        if (segment.inlineCast() != null) {
            fieldType = resolveOrThrow(segment.inlineCast(), typeResolver);
        }

        // Build the From-sourced pattern
        ObjectType objectType = new ClassObjectType(fieldType, false);
        String segBind = isLast ? originalBind : "_oopath$" + lambdaCompiler.nextPatternId();
        Pattern segPattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0,
                                         objectType, segBind, false);
        segPattern.setSource(new From(new OopathFieldDataProvider(previousDecl, getter)));

        // Apply constraints
        org.mvel3.transpiler.context.Declaration<?>[] declarations =
                DrlxLambdaCompiler.extractDeclarations(fieldType);
        for (String expression : segment.conditions()) {
            List<BoundVariable> referencedBindings =
                    lambdaCompiler.findReferencedBindings(expression, boundVariables);
            Constraint constraint = referencedBindings.isEmpty()
                    ? lambdaCompiler.createLambdaConstraint(expression, fieldType, declarations)
                    : lambdaCompiler.createBetaLambdaConstraint(expression, fieldType,
                                                                 declarations, referencedBindings);
            segPattern.addConstraint(constraint);
        }

        parent.addChild(segPattern);

        // Set up for next segment
        Declaration segDecl = segPattern.addDeclaration(segBind);
        segDecl.setReadAccessor(new SelfReferenceClassFieldReader(fieldType));
        boundVariables.put(segBind, new BoundVariable(segBind, fieldType,
                                                       segPattern, segDecl));
        previousPattern = segPattern;
        previousClass = fieldType;
        previousDecl = segDecl;
    }
}
```

Add necessary imports at the top of `DrlxRuleAstRuntimeBuilder.java`:

```java
import org.drools.base.rule.From;
import org.drools.base.base.ClassObjectType;
import org.drools.core.base.SelfReferenceClassFieldReader;
```

(`ClassObjectType` and `SelfReferenceClassFieldReader` may already be imported — check first.)

- [ ] **Step 6: Verify `findGetterForField` exists**

Search for `findGetterForField` in `DrlxRuleAstRuntimeBuilder.java`. It should already exist (used for positional args). If not, add:

```java
private static Method findGetterForField(Class<?> clazz, String fieldName) {
    String getterName = "get" + Character.toUpperCase(fieldName.charAt(0)) + fieldName.substring(1);
    try {
        return clazz.getMethod(getterName);
    } catch (NoSuchMethodException e) {
        String isName = "is" + Character.toUpperCase(fieldName.charAt(0)) + fieldName.substring(1);
        try {
            return clazz.getMethod(isName);
        } catch (NoSuchMethodException e2) {
            throw new RuntimeException("No getter found for field '" + fieldName
                    + "' on class " + clazz.getName(), e2);
        }
    }
}
```

- [ ] **Step 7: Update `collectPatternClasses` for multi-segment type pre-scanning**

In `collectPatternClasses` (line 256), after `classes.add(resolvePatternType(p, typeResolver, entryPointTypes, unitClass))`, add:

```java
// Also collect classes from OOPath segments
if (!p.oopathSegments().isEmpty()) {
    Class<?> prevClass = resolvePatternType(p, typeResolver, entryPointTypes, unitClass);
    for (DrlxRuleAstModel.OopathSegmentIR seg : p.oopathSegments()) {
        Method getter = findGetterForField(prevClass, seg.field());
        Class<?> segClass = getter.getReturnType();
        if (Iterable.class.isAssignableFrom(segClass)) {
            java.lang.reflect.Type genericType = getter.getGenericReturnType();
            if (genericType instanceof java.lang.reflect.ParameterizedType pt) {
                java.lang.reflect.Type[] typeArgs = pt.getActualTypeArguments();
                if (typeArgs.length > 0 && typeArgs[0] instanceof Class<?> c) {
                    segClass = c;
                }
            }
        }
        if (seg.inlineCast() != null) {
            segClass = resolveOrThrow(seg.inlineCast(), typeResolver);
        }
        classes.add(segClass);
        prevClass = segClass;
    }
}
```

- [ ] **Step 8: Run the execution tests**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=MultiSegmentOOPathTest
```
Expected: Both tests pass.

- [ ] **Step 9: Run full test suite**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test
```
Expected: All tests pass, including all existing single-segment OOPath tests.

- [ ] **Step 10: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add -A
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: runtime execution of multi-segment OOPath (#103)

Add OopathFieldDataProvider implementing DataProvider for From source.
Add buildPatternChain to DrlxRuleAstRuntimeBuilder that chains
Pattern/From objects for multi-segment OOPath expressions.
Add List<Address> to Person domain class.
Add execution tests for /persons/addresses and /persons[constraint]/addresses.

Closes #103"
```
