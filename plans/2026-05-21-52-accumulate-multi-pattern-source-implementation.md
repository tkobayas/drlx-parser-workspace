# #52 Accumulate Multi-Pattern Source Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable `acc(and(p : /persons, o : /orders[...]), ...)` — accumulate over joined tuples from multiple source patterns.

**Architecture:** Widen the IR `source` field from `PatternIR` to `LhsItemIR`. A single pattern remains `PatternIR`; a multi-pattern source becomes `GroupElementIR(AND, [...])`. The runtime accumulator classes gain a `multiSource` flag that switches from `handle.getObject()` extraction (single-source) to tuple-based extraction via `innerDecls` (multi-source). The Drools rete builder automatically creates a subnetwork + RightInputAdapterNode when it sees a `GroupElement` source — no Drools engine changes needed.

**Tech Stack:** ANTLR4, Java 21 records/sealed interfaces, Protocol Buffers 3, MVEL3 batch compiler, Drools 10.x rete/phreak engine

**Spec:** `specs/2026-05-20-52-accumulate-multi-pattern-source-design.md`

**Test data for multi-pattern tests:**
```
Person("Alice", 1), Person("Bob", 2)
Order("O1", 1, 100), Order("O2", 2, 200)
Join: p.age == o.customerId
→ 2 joined tuples: (Alice, O1), (Bob, O2)
→ count() = 2
→ sum(o.amount) = 300
→ sum(p.age * o.amount) = 1*100 + 2*200 = 500
```

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `drlx-parser-core/src/main/antlr4/.../DrlxParser.g4` | Modify | Add `andElement` alternative to `accSource` rule |
| `drlx-parser-core/src/main/java/.../DrlxRuleAstModel.java` | Modify | Widen `source` from `PatternIR` to `LhsItemIR` in `AccumulatePatternIR` and `CustomAccumulateIR` |
| `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` | Modify | Change `source` field type to `LhsItemParseResult` |
| `drlx-parser-core/src/main/java/.../DrlxRuleAstParseResult.java` | Modify | Use `toProtoLhs`/`fromProtoLhs` for source serialization |
| `drlx-parser-core/src/main/java/.../DrlxToRuleAstVisitor.java` | Modify | Branch on `accSource` alternative (boundOopath vs andElement) |
| `drlx-parser-core/src/main/java/.../DrlxValueExtractor.java` | Modify | Add `applyMulti(Map)` method |
| `drlx-parser-core/src/main/java/.../DrlxLambdaAccumulator.java` | Modify | Add `multiSource` flag, tuple-aware `accumulate()` |
| `drlx-parser-core/src/main/java/.../DrlxCustomAccumulator.java` | Modify | Add `multiSource` flag, multi-binding `accumulate()`/`tryReverse()` |
| `drlx-parser-core/src/main/java/.../DrlxLambdaCompiler.java` | Modify | Multi-declaration `createValueExtractor`/`createCustomAccumulator` overloads |
| `drlx-parser-core/src/main/java/.../DrlxRuleAstRuntimeBuilder.java` | Modify | Handle `GroupElementIR` source in `buildAccumulatePattern`/`buildCustomAccumulatePattern` |
| `drlx-parser-core/src/test/java/.../DrlxParserTest.java` | Modify | Parser test for `andElement` in `accSource` |
| `drlx-parser-core/src/test/java/.../AccumulateVisitorTest.java` | Modify | Visitor/IR tests for multi-pattern source |
| `drlx-parser-core/src/test/java/.../AccumulateTest.java` | Modify | End-to-end runtime tests |

All paths relative to `/home/tkobayas/usr/work/mvel3-development/drlx-parser/`.

---

### Task 1: Grammar — extend accSource with andElement

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:223-225`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java`

- [ ] **Step 1: Write the failing parser test**

Add to `DrlxParserTest.java`:

```java
@Test
void parsesAccSourceWithAndElement() {
    final String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    var total = sum(o.amount)),
                do { results.add(total); }
            }
            """;
    final var tree = parseDrlxCompilationUnitAsAntlrAST(rule);
    final var accKeywordItem = findFirst(tree, DrlxParser.AccKeywordItemContext.class);
    assertThat(accKeywordItem).isNotNull();
    final var accSource = accKeywordItem.accSource();
    assertThat(accSource.boundOopath()).isNull();
    assertThat(accSource.andElement()).isNotNull();
    assertThat(accSource.andElement().groupChild()).hasSize(2);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=DrlxParserTest#parsesAccSourceWithAndElement -pl . -Dmaven.compiler.showWarnings=false -q`

Expected: FAIL — `andElement` is not a valid alternative in `accSource`, so parsing fails or `accSource.andElement()` method doesn't exist.

- [ ] **Step 3: Modify the grammar**

In `DrlxParser.g4`, change `accSource` from:

```antlr
accSource
    : boundOopath
    ;
```

to:

```antlr
accSource
    : boundOopath
    | andElement
    ;
```

- [ ] **Step 4: Regenerate the ANTLR parser and run the test**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=DrlxParserTest#parsesAccSourceWithAndElement -pl . -Dmaven.compiler.showWarnings=false -q`

Expected: PASS

- [ ] **Step 5: Run all parser tests for regression**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=DrlxParserTest -pl . -Dmaven.compiler.showWarnings=false -q`

Expected: All pass.

- [ ] **Step 6: Commit**

```
feat(grammar): extend accSource to accept andElement

Refs #52
```

---

### Task 2: IR Model + Protobuf — widen source to LhsItemIR

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:63-68,81-95`
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto:78-81,91-100`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java:120-146,196-227`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`

- [ ] **Step 1: Widen AccumulatePatternIR.source**

In `DrlxRuleAstModel.java`, change:

```java
public record AccumulatePatternIR(PatternIR source,
                                  List<AccumulatorIR> accumulators) implements LhsItemIR {
```

to:

```java
public record AccumulatePatternIR(LhsItemIR source,
                                  List<AccumulatorIR> accumulators) implements LhsItemIR {
```

- [ ] **Step 2: Widen CustomAccumulateIR.source**

In `DrlxRuleAstModel.java`, change:

```java
public record CustomAccumulateIR(
    PatternIR source,
    List<InitVarIR> initVars,
```

to:

```java
public record CustomAccumulateIR(
    LhsItemIR source,
    List<InitVarIR> initVars,
```

- [ ] **Step 3: Fix compilation errors**

The widening changes `source` from `PatternIR` to `LhsItemIR`. All callers that pass a `PatternIR` still compile (it implements `LhsItemIR`). But callers that call methods specific to `PatternIR` on `source()` will break — these are the sites that need multi-pattern support. Fix the obvious compilation errors introduced by the type change:

- `DrlxToRuleAstVisitor.buildAccKeyword2Param()` passes `PatternIR` → still compiles (upcast)
- `DrlxToRuleAstVisitor.buildAccKeywordItem()` passes `PatternIR` → still compiles (upcast)
- `DrlxRuleAstRuntimeBuilder.buildAccumulatePattern()` calls `accPat.source()` and assigns to `PatternIR srcIr` → needs cast for now: `PatternIR srcIr = (PatternIR) accPat.source();` (will be replaced in Task 7)
- `DrlxRuleAstRuntimeBuilder.buildCustomAccumulatePattern()` same → cast: `PatternIR srcIr = (PatternIR) customAcc.source();`
- `DrlxRuleAstParseResult.toProtoLhs()` calls `patternToProto(accPat.source())` → needs cast: `patternToProto((PatternIR) accPat.source())` (will be replaced in Step 6)
- `DrlxRuleAstParseResult.fromProtoLhs()` — source deserialization unchanged for now

- [ ] **Step 4: Update protobuf schema**

In `drlx_rule_ast.proto`, change both messages:

```protobuf
message AccumulatePatternParseResult {
  LhsItemParseResult source = 1;
  repeated AccumulatorParseResult accumulators = 2;
}
```

```protobuf
message CustomAccumulateParseResult {
  LhsItemParseResult source = 1;
  repeated InitVarParseResult init_vars = 2;
  string action_block = 3;
  string reverse_block = 4;
  string result_type_name = 5;
  string result_bind_name = 6;
  string result_expression = 7;
  repeated string referenced_bindings = 8;
}
```

- [ ] **Step 5: Regenerate protobuf Java classes**

Run: `mvn -f drlx-parser-core/pom.xml generate-sources -pl . -q`

- [ ] **Step 6: Update serialization code**

In `DrlxRuleAstParseResult.java`, update `toProtoLhs()`:

Replace the accumulate-pattern source serialization (line 199):
```java
.setSource(patternToProto(accPat.source()));
```
with:
```java
.setSource(toProtoLhs(accPat.source()));
```

Replace the custom-accumulate source serialization (line 214):
```java
.setSource(patternToProto(customAcc.source()))
```
with:
```java
.setSource(toProtoLhs(customAcc.source()))
```

Update `fromProtoLhs()`:

Replace the accumulate-pattern source deserialization (line 122):
```java
PatternIR srcIr = patternFromProto(accPat.getSource());
```
with:
```java
LhsItemIR srcIr = fromProtoLhs(accPat.getSource(), file);
```

Replace the custom-accumulate source deserialization (line 136):
```java
PatternIR srcIr = patternFromProto(cap.getSource());
```
with:
```java
LhsItemIR srcIr = fromProtoLhs(cap.getSource(), file);
```

- [ ] **Step 7: Verify existing protobuf round-trip tests pass**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateVisitorTest#customAccumulateIRProtobufRoundTrip+customAccumulateIRProtobufRoundTripNullReverse -pl . -q`

Expected: PASS (single-source round-trips should still work)

- [ ] **Step 8: Run all AccumulateVisitorTest tests**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateVisitorTest -pl . -q`

Expected: All pass.

- [ ] **Step 9: Commit**

```
refactor(ir): widen accumulate source from PatternIR to LhsItemIR

Refs #52
```

---

### Task 3: Visitor — multi-pattern source in buildAccKeywordItem

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:362-416`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/AccumulateVisitorTest.java`

- [ ] **Step 1: Write the failing visitor test — 2-param with and()**

Add to `AccumulateVisitorTest.java`:

```java
@Test
void accKeyword2ParamWithAndSourceProducesGroupElementSource() {
    RuleIR rule = parseRule("""
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    var total = sum(o.amount)),
                do { results.add(total); }
            }
            """);
    assertThat(rule.lhs()).hasSize(1);
    AccumulatePatternIR accPat = (AccumulatePatternIR) rule.lhs().get(0);
    assertThat(accPat.source()).isInstanceOf(GroupElementIR.class);
    GroupElementIR group = (GroupElementIR) accPat.source();
    assertThat(group.kind()).isEqualTo(GroupElementIR.Kind.AND);
    assertThat(group.children()).hasSize(2);
    assertThat(group.children().get(0)).isInstanceOf(PatternIR.class);
    assertThat(group.children().get(1)).isInstanceOf(PatternIR.class);
    PatternIR p0 = (PatternIR) group.children().get(0);
    PatternIR p1 = (PatternIR) group.children().get(1);
    assertThat(p0.bindName()).isEqualTo("p");
    assertThat(p1.bindName()).isEqualTo("o");
    assertThat(accPat.accumulators()).hasSize(1);
    assertThat(accPat.accumulators().get(0).functionName()).isEqualTo("sum");
}
```

- [ ] **Step 2: Write the failing visitor test — custom acc with and()**

```java
@Test
void accKeyword3ParamWithAndSourceProducesGroupElementSource() {
    RuleIR rule = parseRule("""
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    int s = 0;,
                    s = s + o.amount,
                    int total = s),
                do { results.add(total); }
            }
            """);
    assertThat(rule.lhs()).hasSize(1);
    CustomAccumulateIR customAcc = (CustomAccumulateIR) rule.lhs().get(0);
    assertThat(customAcc.source()).isInstanceOf(GroupElementIR.class);
    GroupElementIR group = (GroupElementIR) customAcc.source();
    assertThat(group.kind()).isEqualTo(GroupElementIR.Kind.AND);
    assertThat(group.children()).hasSize(2);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest="AccumulateVisitorTest#accKeyword2ParamWithAndSourceProducesGroupElementSource+accKeyword3ParamWithAndSourceProducesGroupElementSource" -pl . -q`

Expected: FAIL — visitor tries `ctx.accSource().boundOopath()` which is null for `andElement`.

- [ ] **Step 4: Update the visitor**

In `DrlxToRuleAstVisitor.buildAccKeywordItem()`, replace line 370:

```java
PatternIR source = buildPatternFromBoundOopath(ctx.accSource().boundOopath());
```

with:

```java
LhsItemIR source;
if (ctx.accSource().boundOopath() != null) {
    source = buildPatternFromBoundOopath(ctx.accSource().boundOopath());
} else {
    source = buildGroupElementFromChildren(
            ctx.accSource().andElement().groupChild(),
            GroupElementIR.Kind.AND);
}
```

Then update `buildAccKeyword2Param` signature and body. Currently:

```java
private LhsItemIR buildAccKeyword2Param(PatternIR source,
                                         DrlxParser.AccFunctionListContext funcListCtx) {
```

Change to:

```java
private LhsItemIR buildAccKeyword2Param(LhsItemIR source,
                                         DrlxParser.AccFunctionListContext funcListCtx) {
```

In the custom-accumulate path (bottom of `buildAccKeywordItem`), the `CustomAccumulateIR` constructor already accepts `LhsItemIR source` after Task 2. No additional change needed — just pass `source` (which is now `LhsItemIR`).

The `srcBindName` extraction (used for `validateResultExpression` and `buildInitVars`) needs adjustment. For single-source, `srcBindName` comes from `((PatternIR) source).bindName()`. For multi-source, there is no single source bind name — init var name collisions are checked against all source bindings. Update:

```java
String srcBindName;
if (source instanceof PatternIR pat) {
    srcBindName = pat.bindName();
} else {
    srcBindName = null;
}
```

The `validateResultExpression(resultExpression, srcBindName)` call should only apply when `srcBindName != null`. For multi-source, the result expression is validated at build time when the MVEL3 compiler checks that all referenced bindings are declared.

The `buildInitVars(body.accInitVars(), srcBindName)` call validates that no init var collides with `srcBindName`. For multi-source, extend the collision check to all source binding names.

Change the `buildInitVars` method signature from:

```java
private List<InitVarIR> buildInitVars(DrlxParser.AccInitVarsContext ctx,
                                       String srcBindName) {
```

to:

```java
private List<InitVarIR> buildInitVars(DrlxParser.AccInitVarsContext ctx,
                                       java.util.Set<String> sourceBindNames) {
```

And change the collision check inside (line 446) from:

```java
if (varName.equals(srcBindName)) {
```

to:

```java
if (sourceBindNames.contains(varName)) {
```

At the call site, build the set:

```java
java.util.Set<String> sourceBindNames;
if (source instanceof PatternIR pat) {
    sourceBindNames = java.util.Set.of(pat.bindName());
} else if (source instanceof GroupElementIR group) {
    sourceBindNames = group.children().stream()
            .filter(PatternIR.class::isInstance)
            .map(c -> ((PatternIR) c).bindName())
            .collect(java.util.stream.Collectors.toSet());
} else {
    sourceBindNames = java.util.Set.of();
}
List<InitVarIR> initVars = buildInitVars(body.accInitVars(), sourceBindNames);
```

- [ ] **Step 5: Run visitor tests**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateVisitorTest -pl . -q`

Expected: All pass including the two new tests.

- [ ] **Step 6: Write protobuf round-trip test for multi-pattern source**

```java
@Test
void accumulatePatternWithAndSourceProtobufRoundTrip() throws Exception {
    RuleIR rule = parseRule("""
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    var total = sum(o.amount)),
                do { results.add(total); }
            }
            """);
    Path dir = Files.createTempDirectory("proto-rt");
    try {
        CompilationUnitIR original = parseUnit("""
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.Order;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                        var total = sum(o.amount)),
                    do { results.add(total); }
                }
                """);
        String src = original.toString();
        DrlxRuleAstParseResult.save(src, original, dir);
        CompilationUnitIR loaded = DrlxRuleAstParseResult.load(src, DrlxRuleAstParseResult.parseResultFilePath(dir));
        assertThat(loaded).isNotNull();
        AccumulatePatternIR accPat = (AccumulatePatternIR) loaded.rules().get(0).lhs().get(0);
        assertThat(accPat.source()).isInstanceOf(GroupElementIR.class);
        GroupElementIR group = (GroupElementIR) accPat.source();
        assertThat(group.kind()).isEqualTo(GroupElementIR.Kind.AND);
        assertThat(group.children()).hasSize(2);
    } finally {
        deleteDir(dir);
    }
}
```

Note: Adapt this test to match the existing protobuf round-trip test patterns (see `customAccumulateIRProtobufRoundTrip` at line 207 for the exact helper methods and cleanup pattern used).

- [ ] **Step 7: Run round-trip test**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest="AccumulateVisitorTest#accumulatePatternWithAndSourceProtobufRoundTrip" -pl . -q`

Expected: PASS

- [ ] **Step 8: Commit**

```
feat(visitor): handle andElement in accSource for multi-pattern accumulate

Refs #52
```

---

### Task 4: DrlxValueExtractor — add applyMulti

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxValueExtractor.java`

- [ ] **Step 1: Add applyMulti method**

Add to `DrlxValueExtractor.java`:

```java
public Object applyMulti(Map<String, Object> bindings) {
    try {
        return evaluator.eval(bindings);
    } catch (Exception e) {
        throw new RuntimeException(
                "value extractor '" + expression + "' failed at runtime (multi-source)", e);
    }
}
```

- [ ] **Step 2: Verify compilation**

Run: `mvn -f drlx-parser-core/pom.xml compile -pl . -q`

Expected: Compiles.

- [ ] **Step 3: Commit**

```
feat(extractor): add applyMulti for multi-pattern source extraction

Refs #52
```

---

### Task 5: DrlxLambdaAccumulator — multi-source support

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaAccumulator.java`

- [ ] **Step 1: Add multiSource flag and constructor overload**

Add a `multiSource` field and a second constructor. Keep the original constructor for single-source backward compatibility:

```java
public final class DrlxLambdaAccumulator implements Accumulator {

    private final AccumulateFunction<Serializable> accFunction;
    private final Function<Object, Object> extractor;
    private final DrlxValueExtractor multiExtractor;
    private final boolean multiSource;

    public DrlxLambdaAccumulator(AccumulateFunction<Serializable> accFunction,
                                 Function<Object, Object> extractor) {
        this.accFunction = accFunction;
        this.extractor = extractor;
        this.multiExtractor = null;
        this.multiSource = false;
    }

    public DrlxLambdaAccumulator(AccumulateFunction<Serializable> accFunction,
                                 DrlxValueExtractor multiExtractor,
                                 boolean multiSource) {
        this.accFunction = accFunction;
        this.extractor = null;
        this.multiExtractor = multiExtractor;
        this.multiSource = multiSource;
    }
```

The `multiExtractor` field is typed as `DrlxValueExtractor` (not `Function<Object, Object>`) so we can call `applyMulti(Map)`.

- [ ] **Step 2: Update accumulate() for multi-source**

Replace the current `accumulate()` body:

```java
@Override
public Object accumulate(Object wmContext, Object context, BaseTuple tuple,
                         FactHandle handle, Declaration[] decls,
                         Declaration[] innerDecls, ValueResolver vr) {
    Object value;
    if (multiSource) {
        Map<String, Object> bindings = new HashMap<>(innerDecls.length);
        for (Declaration d : innerDecls) {
            bindings.put(d.getIdentifier(), d.getValue(vr, tuple));
        }
        value = (multiExtractor == null) ? bindings : multiExtractor.applyMulti(bindings);
    } else {
        value = (extractor == null) ? handle.getObject() : extractor.apply(handle.getObject());
    }
    try {
        accFunction.accumulate((Serializable) context, value);
    } catch (Exception e) {
        throw new RuntimeException("accumulate failed for " + accFunction.getClass().getSimpleName(), e);
    }
    return value;
}
```

Add the required imports at the top of the file:

```java
import java.util.HashMap;
import java.util.Map;
import org.drools.base.rule.Declaration;
import org.kie.api.runtime.rule.FactHandle;
```

Note: `Declaration` and `ValueResolver` are already available via existing `Accumulator` method signatures but may need explicit import. `BaseTuple` is already imported. Check existing imports and add only what's missing.

- [ ] **Step 3: tryReverse() — no changes needed**

`tryReverse()` uses the pre-computed `value` from `accumulate()` and passes it to `accFunction.reverse(ctx, value)`. Built-in functions reverse by the cached value, not by re-extraction. No changes needed for multi-source.

- [ ] **Step 4: Verify compilation**

Run: `mvn -f drlx-parser-core/pom.xml compile -pl . -q`

Expected: Compiles.

- [ ] **Step 5: Commit**

```
feat(accumulator): add multi-source tuple extraction to DrlxLambdaAccumulator

Refs #52
```

---

### Task 6: DrlxCustomAccumulator — multi-source support

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxCustomAccumulator.java`

- [ ] **Step 1: Add multiSource flag and source binding name list**

Change fields and constructor:

```java
public final class DrlxCustomAccumulator implements Accumulator {

    private final List<InitVarIR> initVars;
    private final String srcBindingName;
    private final List<String> srcBindingNames;
    private final boolean multiSource;
    private final Map<String, Object> initDefaults;

    // Single-source constructor (unchanged behavior)
    public DrlxCustomAccumulator(List<InitVarIR> initVars, String srcBindingName) {
        this.initVars = initVars;
        this.srcBindingName = srcBindingName;
        this.srcBindingNames = null;
        this.multiSource = false;
        this.initDefaults = buildDefaults(initVars);
    }

    // Multi-source constructor
    public DrlxCustomAccumulator(List<InitVarIR> initVars, List<String> srcBindingNames) {
        this.initVars = initVars;
        this.srcBindingName = null;
        this.srcBindingNames = List.copyOf(srcBindingNames);
        this.multiSource = true;
        this.initDefaults = buildDefaults(initVars);
    }

    // Update createContext() to account for multiple source bindings:
    @Override public Object createContext() {
        int extra = multiSource ? srcBindingNames.size() : 1;
        return new HashMap<String, Object>(initDefaults.size() + extra);
    }
```

- [ ] **Step 2: Update accumulate() for multi-source**

```java
@Override
public Object accumulate(Object wmContext, Object context, BaseTuple tuple,
                         FactHandle handle, Declaration[] decls,
                         Declaration[] innerDecls, ValueResolver vr) {
    @SuppressWarnings("unchecked")
    Map<String, Object> map = (Map<String, Object>) context;
    if (multiSource) {
        for (Declaration d : innerDecls) {
            map.put(d.getIdentifier(), d.getValue(vr, tuple));
        }
        try {
            actionEval.eval(map);
        } finally {
            for (Declaration d : innerDecls) {
                map.remove(d.getIdentifier());
            }
        }
        return null;
    } else {
        Object srcFact = handle.getObject();
        map.put(srcBindingName, srcFact);
        try {
            actionEval.eval(map);
        } finally {
            map.remove(srcBindingName);
        }
        return srcFact;
    }
}
```

- [ ] **Step 3: Update tryReverse() for multi-source**

```java
@Override
public boolean tryReverse(Object wmContext, Object context, BaseTuple tuple,
                          FactHandle handle, Object value,
                          Declaration[] decls, Declaration[] innerDecls,
                          ValueResolver vr) {
    if (reverseEval == null) return false;
    @SuppressWarnings("unchecked")
    Map<String, Object> map = (Map<String, Object>) context;
    if (multiSource) {
        for (Declaration d : innerDecls) {
            map.put(d.getIdentifier(), d.getValue(vr, tuple));
        }
        try {
            reverseEval.eval(map);
        } finally {
            for (Declaration d : innerDecls) {
                map.remove(d.getIdentifier());
            }
        }
    } else {
        map.put(srcBindingName, value);
        try {
            reverseEval.eval(map);
        } finally {
            map.remove(srcBindingName);
        }
    }
    return true;
}
```

- [ ] **Step 4: Add required imports**

```java
import org.drools.base.rule.Declaration;
import org.kie.api.runtime.rule.FactHandle;
```

Check existing imports; add only what's missing.

- [ ] **Step 5: Verify compilation**

Run: `mvn -f drlx-parser-core/pom.xml compile -pl . -q`

Expected: Compiles.

- [ ] **Step 6: Commit**

```
feat(accumulator): add multi-source tuple extraction to DrlxCustomAccumulator

Refs #52
```

---

### Task 7: DrlxLambdaCompiler — multi-declaration overloads

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java`

- [ ] **Step 1: Add multi-source createValueExtractor overload**

Add a new overload that accepts multiple declarations via `Map<String, BoundVariable>`:

```java
public DrlxValueExtractor createValueExtractor(String argExpr,
                                               Map<String, BoundVariable> sourceScope) {
    int counter = lambdaCounter++;

    @SuppressWarnings("unchecked")
    Evaluator<Map<String, Object>, Void, Object> preCompiled =
            (Evaluator<Map<String, Object>, Void, Object>) tryLoadPreCompiled(counter, argExpr, "value extractor");
    if (preCompiled != null) {
        return new DrlxValueExtractor(argExpr, null, preCompiled);
    }

    DrlxValueExtractor deferred = createBatchValueExtractorMulti(argExpr, sourceScope);
    onLambdaCreated(counter, argExpr);
    return deferred;
}

@SuppressWarnings({"unchecked", "rawtypes"})
private DrlxValueExtractor createBatchValueExtractorMulti(String argExpr,
                                                          Map<String, BoundVariable> sourceScope) {
    org.mvel3.transpiler.context.Declaration<?>[] decls = sourceScope.entrySet().stream()
            .map(e -> org.mvel3.transpiler.context.Declaration.of(e.getKey(), e.getValue().type()))
            .toArray(org.mvel3.transpiler.context.Declaration[]::new);

    CompilerParameters<Map<String, Object>, Void, Object> evalInfo =
            (CompilerParameters) MVEL.<Object>map(decls)
                    .<Object>out(Object.class)
                    .expression(argExpr)
                    .imports(new HashSet<>(imports))
                    .classManager(batchCompiler.getClassManager())
                    .generatedClassName("GeneratorEvaluator__")
                    .build();
    MVELBatchCompiler.LambdaHandle handle = batchCompiler.add(evalInfo);
    DrlxValueExtractor extractor = new DrlxValueExtractor(argExpr, null, null);
    pendingLambdas.add(new PendingLambda(handle, extractor));
    return extractor;
}
```

- [ ] **Step 2: Add multi-source createCustomAccumulator overload**

Add a new overload that accepts `Map<String, BoundVariable> sourceScope`:

```java
@SuppressWarnings({"unchecked", "rawtypes"})
public DrlxCustomAccumulator createCustomAccumulator(
        DrlxRuleAstModel.CustomAccumulateIR ir,
        Map<String, BoundVariable> sourceScope) {

    List<String> srcBindingNames = new ArrayList<>(sourceScope.keySet());
    DrlxCustomAccumulator acc = new DrlxCustomAccumulator(ir.initVars(), srcBindingNames);

    List<org.mvel3.transpiler.context.Declaration<?>> holderDecls = new ArrayList<>();
    for (DrlxRuleAstModel.InitVarIR iv : ir.initVars()) {
        holderDecls.add(org.mvel3.transpiler.context.Declaration.of(iv.name(), resolveInitVarType(iv.typeName())));
    }

    List<org.mvel3.transpiler.context.Declaration<?>> actionDecls = new ArrayList<>(holderDecls);
    for (Map.Entry<String, BoundVariable> e : sourceScope.entrySet()) {
        actionDecls.add(org.mvel3.transpiler.context.Declaration.of(e.getKey(), e.getValue().type()));
    }
    org.mvel3.transpiler.context.Declaration<?>[] actionDeclArray =
            actionDecls.toArray(new org.mvel3.transpiler.context.Declaration[0]);

    // Action block — same pattern as existing single-source method
    {
        int counter = lambdaCounter++;
        String normalizedAction = normalizeBlockText(ir.actionBlock());
        Evaluator<Map<String, Object>, Void, ?> preCompiled =
                (Evaluator) tryLoadPreCompiled(counter, normalizedAction, "custom acc action");
        if (preCompiled != null) {
            acc.setActionEval(preCompiled);
        } else {
            CompilerParameters<Map<String, Object>, Void, String> evalInfo =
                    (CompilerParameters) MVEL.<Object>map(actionDeclArray)
                            .<String>out(String.class)
                            .block(normalizedAction + RETURN_NULL)
                            .imports(new HashSet<>(imports))
                            .classManager(batchCompiler.getClassManager())
                            .generatedClassName("GeneratorEvaluator__")
                            .build();
            MVELBatchCompiler.LambdaHandle handle = batchCompiler.add(evalInfo);
            pendingLambdas.add(new PendingLambda(handle, new DrlxCustomAccumulator.ActionSink(acc)));
            onLambdaCreated(counter, normalizedAction);
        }
    }

    // Reverse block (optional) — same pattern
    if (ir.reverseBlock() != null) {
        int counter = lambdaCounter++;
        String normalizedReverse = normalizeBlockText(ir.reverseBlock());
        Evaluator<Map<String, Object>, Void, ?> preCompiled =
                (Evaluator) tryLoadPreCompiled(counter, normalizedReverse, "custom acc reverse");
        if (preCompiled != null) {
            acc.setReverseEval(preCompiled);
        } else {
            CompilerParameters<Map<String, Object>, Void, String> evalInfo =
                    (CompilerParameters) MVEL.<Object>map(actionDeclArray)
                            .<String>out(String.class)
                            .block(normalizedReverse + RETURN_NULL)
                            .imports(new HashSet<>(imports))
                            .classManager(batchCompiler.getClassManager())
                            .generatedClassName("GeneratorEvaluator__")
                            .build();
            MVELBatchCompiler.LambdaHandle handle = batchCompiler.add(evalInfo);
            pendingLambdas.add(new PendingLambda(handle, new DrlxCustomAccumulator.ReverseSink(acc)));
            onLambdaCreated(counter, normalizedReverse);
        }
    }

    // Result expression — same as existing (only holder decls, no source bindings)
    {
        int counter = lambdaCounter++;
        org.mvel3.transpiler.context.Declaration<?>[] holderDeclArray =
                holderDecls.toArray(new org.mvel3.transpiler.context.Declaration[0]);
        Class<?> resultClass = resolveInitVarType(ir.resultTypeName());

        Evaluator<Map<String, Object>, Void, Object> preCompiled =
                (Evaluator) tryLoadPreCompiled(counter, ir.resultExpression(), "custom acc result");
        if (preCompiled != null) {
            acc.setResultEval(preCompiled);
        } else {
            CompilerParameters<Map<String, Object>, Void, Object> evalInfo =
                    (CompilerParameters) MVEL.<Object>map(holderDeclArray)
                            .<Object>out(resultClass)
                            .expression(ir.resultExpression())
                            .imports(new HashSet<>(imports))
                            .classManager(batchCompiler.getClassManager())
                            .generatedClassName("GeneratorEvaluator__")
                            .build();
            MVELBatchCompiler.LambdaHandle handle = batchCompiler.add(evalInfo);
            pendingLambdas.add(new PendingLambda(handle, new DrlxCustomAccumulator.ResultSink(acc)));
            onLambdaCreated(counter, ir.resultExpression());
        }
    }

    return acc;
}
```

- [ ] **Step 3: Verify compilation**

Run: `mvn -f drlx-parser-core/pom.xml compile -pl . -q`

Expected: Compiles.

- [ ] **Step 4: Commit**

```
feat(compiler): add multi-declaration overloads for multi-pattern accumulate

Refs #52
```

---

### Task 8: DrlxRuleAstRuntimeBuilder — multi-pattern source

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:412-505`

- [ ] **Step 1: Update buildAccumulatePattern for GroupElementIR source**

Replace the current method with one that handles both `PatternIR` and `GroupElementIR`:

```java
private void buildAccumulatePattern(AccumulatePatternIR accPat,
                                    GroupElement parent,
                                    TypeResolver typeResolver,
                                    Map<String, Class<?>> entryPointTypes,
                                    Class<?> unitClass,
                                    Map<String, BoundVariable> outerScope) {

    LhsItemIR srcIr = accPat.source();

    Map<String, BoundVariable> innerScope = new java.util.LinkedHashMap<>(outerScope);
    RuleConditionElement srcElement;
    boolean multiSource;

    if (srcIr instanceof PatternIR patIr) {
        // --- single-source (existing path) ---
        Pattern srcPattern = buildPattern(patIr, typeResolver, entryPointTypes, unitClass, outerScope);
        srcElement = srcPattern;
        multiSource = false;
        Declaration srcDecl = srcPattern.getDeclaration();
        if (srcDecl != null) {
            Class<?> srcClass = ((ClassObjectType) srcPattern.getObjectType()).getClassType();
            innerScope.put(srcDecl.getIdentifier(),
                    new BoundVariable(srcDecl.getIdentifier(), srcClass, srcPattern, srcDecl));
        }
    } else if (srcIr instanceof GroupElementIR groupIr) {
        // --- multi-source ---
        GroupElement andGroup = GroupElementFactory.newAndInstance();
        buildLhs(groupIr.children(), andGroup, typeResolver, entryPointTypes,
                 unitClass, innerScope);
        srcElement = andGroup;
        multiSource = true;
    } else {
        throw new IllegalArgumentException("Unsupported accumulate source: " + srcIr.getClass());
    }

    List<AccumulatorIR> accumulators = accPat.accumulators();
    int n = accumulators.size();

    // For multi-source: extract only source bindings (innerScope minus outerScope).
    // The MVEL3 compiler must only see source declarations; outer bindings are not
    // available in the runtime extraction map built from innerDecls.
    Map<String, BoundVariable> sourceScope = null;
    if (multiSource) {
        sourceScope = new java.util.LinkedHashMap<>();
        for (Map.Entry<String, BoundVariable> e : innerScope.entrySet()) {
            if (!outerScope.containsKey(e.getKey())) {
                sourceScope.put(e.getKey(), e.getValue());
            }
        }
    }

    org.drools.base.rule.accessor.Accumulator[] accs =
            new org.drools.base.rule.accessor.Accumulator[n];
    for (int i = 0; i < n; i++) {
        if (multiSource) {
            accs[i] = buildSingleAccumulatorMulti(accumulators.get(i), sourceScope);
        } else {
            Declaration srcDecl = ((Pattern) srcElement).getDeclaration();
            Class<?> srcClass = ((ClassObjectType) ((Pattern) srcElement).getObjectType()).getClassType();
            String srcBindingName = srcDecl != null ? srcDecl.getIdentifier() : null;
            accs[i] = buildSingleAccumulator(accumulators.get(i), srcClass, srcBindingName);
        }
    }

    // requiredDeclarations = OUTER scope declarations only (not source declarations).
    // For multi-source, source declarations live in innerDeclarationCache (populated
    // automatically by Accumulate.initInnerDeclarationCache() from GroupElement.
    // getInnerDeclarations()). Putting source declarations in requiredDeclarations
    // would confuse LogicTransformer.replaceDeclarations() and
    // AccumulateNode.addAccFunctionDeclarationsToLeftMask(). Use empty required
    // for multi-source, matching what buildCustomAccumulatePattern already does.
    Pattern wrap;
    if (n == 1) {
        Declaration[] required = multiSource
                ? new Declaration[0]
                : requiredFor(accumulators.get(0), innerScope);
        SingleAccumulate single = new SingleAccumulate(srcElement, required, accs[0]);
        wrap = wrapResultPattern(accumulators.get(0), single);
    } else {
        MultiAccumulate multi = new MultiAccumulate(srcElement, new Declaration[0], accs, n);
        wrap = wrapMultiResultPattern(accumulators, multi);
    }

    parent.addChild(wrap);

    for (int i = 0; i < n; i++) {
        AccumulatorIR acc = accumulators.get(i);
        Class<?> resultClass = resultClassFor(acc);
        Declaration decl = wrap.getDeclarations().get(acc.resultBindName());
        outerScope.put(acc.resultBindName(),
                new BoundVariable(acc.resultBindName(), resultClass, wrap, decl));
    }
}
```

- [ ] **Step 2: Add buildSingleAccumulatorMulti helper**

```java
private org.drools.base.rule.accessor.Accumulator buildSingleAccumulatorMulti(
        AccumulatorIR acc,
        Map<String, BoundVariable> sourceScope) {
    AccumulateFunctionRegistry.Resolution resolved =
            AccumulateFunctionRegistry.resolve(acc.functionName());

    int argCount = acc.argExpressions().size();
    if (resolved.acceptsZeroArgs()) {
        if (argCount > 1) {
            throw new RuntimeException("function '" + acc.functionName()
                    + "' accepts 0 or 1 argument, got " + argCount);
        }
    } else if (argCount != 1) {
        throw new RuntimeException("function '" + acc.functionName()
                + "' requires exactly 1 argument, got " + argCount);
    }

    DrlxValueExtractor multiExtractor = null;
    if (argCount == 1 && !resolved.acceptsZeroArgs()) {
        multiExtractor = lambdaCompiler.createValueExtractor(
                acc.argExpressions().get(0), sourceScope);
    }

    @SuppressWarnings("unchecked")
    AccumulateFunction<Serializable> fn;
    try {
        fn = (AccumulateFunction<Serializable>) resolved.functionClass()
                .getDeclaredConstructor().newInstance();
    } catch (ReflectiveOperationException e) {
        throw new RuntimeException("cannot instantiate " + resolved.functionClass(), e);
    }

    return new DrlxLambdaAccumulator(fn, multiExtractor, true);
}
```

- [ ] **Step 3: Update buildCustomAccumulatePattern for GroupElementIR source**

```java
private void buildCustomAccumulatePattern(CustomAccumulateIR customAcc,
                                           GroupElement parent,
                                           TypeResolver typeResolver,
                                           Map<String, Class<?>> entryPointTypes,
                                           Class<?> unitClass,
                                           Map<String, BoundVariable> outerScope) {

    LhsItemIR srcIr = customAcc.source();

    Map<String, BoundVariable> innerScope = new java.util.LinkedHashMap<>(outerScope);
    RuleConditionElement srcElement;
    boolean multiSource;

    if (srcIr instanceof PatternIR patIr) {
        Pattern srcPattern = buildPattern(patIr, typeResolver, entryPointTypes, unitClass, outerScope);
        srcElement = srcPattern;
        multiSource = false;
        Declaration srcDecl = srcPattern.getDeclaration();
        if (srcDecl != null) {
            Class<?> srcClass = ((ClassObjectType) srcPattern.getObjectType()).getClassType();
            innerScope.put(srcDecl.getIdentifier(),
                    new BoundVariable(srcDecl.getIdentifier(), srcClass, srcPattern, srcDecl));
        }
    } else if (srcIr instanceof GroupElementIR groupIr) {
        GroupElement andGroup = GroupElementFactory.newAndInstance();
        buildLhs(groupIr.children(), andGroup, typeResolver, entryPointTypes,
                 unitClass, innerScope);
        srcElement = andGroup;
        multiSource = true;
    } else {
        throw new IllegalArgumentException("Unsupported accumulate source: " + srcIr.getClass());
    }

    // Reject outer-binding references in action/reverse/result blocks (#54)
    java.util.Set<String> allowedNames = new java.util.LinkedHashSet<>();
    if (multiSource) {
        // All source pattern binding names are allowed
        for (Map.Entry<String, BoundVariable> e : innerScope.entrySet()) {
            if (!outerScope.containsKey(e.getKey())) {
                allowedNames.add(e.getKey());
            }
        }
    } else {
        Declaration srcDecl = ((Pattern) srcElement).getDeclaration();
        if (srcDecl != null) allowedNames.add(srcDecl.getIdentifier());
    }
    for (InitVarIR iv : customAcc.initVars()) {
        allowedNames.add(iv.name());
    }
    for (String ref : customAcc.referencedBindings()) {
        if (!allowedNames.contains(ref) && outerScope.containsKey(ref)) {
            throw new RuntimeException(
                    "outer-binding reference '" + ref + "' in custom accumulate is not yet supported (see #54)");
        }
    }

    DrlxCustomAccumulator accumulator;
    if (multiSource) {
        // Build sourceScope: only source bindings (innerScope minus outerScope)
        Map<String, BoundVariable> sourceScope = new java.util.LinkedHashMap<>();
        for (Map.Entry<String, BoundVariable> e : innerScope.entrySet()) {
            if (!outerScope.containsKey(e.getKey())) {
                sourceScope.put(e.getKey(), e.getValue());
            }
        }
        accumulator = lambdaCompiler.createCustomAccumulator(customAcc, sourceScope);
    } else {
        Class<?> srcClass = ((ClassObjectType) ((Pattern) srcElement).getObjectType()).getClassType();
        String srcBindingName = ((Pattern) srcElement).getDeclaration() != null
                ? ((Pattern) srcElement).getDeclaration().getIdentifier() : null;
        accumulator = lambdaCompiler.createCustomAccumulator(customAcc, srcClass, srcBindingName);
    }

    Declaration[] required = new Declaration[0];
    SingleAccumulate single = new SingleAccumulate(srcElement, required, accumulator);

    Class<?> resultClass = resolveCustomResultType(customAcc.resultTypeName(), typeResolver);
    Pattern wrap = new Pattern(lambdaCompiler.nextPatternId(), new ClassObjectType(resultClass),
                               customAcc.resultBindName());
    wrap.addDeclaration(new Declaration(customAcc.resultBindName(),
            new SelfReferenceClassFieldReader(resultClass), wrap, true));
    wrap.setSource(single);

    parent.addChild(wrap);

    Declaration decl = wrap.getDeclarations().get(customAcc.resultBindName());
    outerScope.put(customAcc.resultBindName(),
            new BoundVariable(customAcc.resultBindName(), resultClass, wrap, decl));
}
```

- [ ] **Step 4: Add required imports**

Add to `DrlxRuleAstRuntimeBuilder.java`:

```java
import org.drools.base.rule.RuleConditionElement;
import org.drools.drlx.builder.DrlxRuleAstModel.GroupElementIR;
import org.drools.drlx.builder.DrlxRuleAstModel.LhsItemIR;
```

Check existing imports; add only what's missing.

- [ ] **Step 5: Remove temporary casts from Task 2**

Remove the `(PatternIR)` casts added in Task 2 Step 3 — the methods now handle `LhsItemIR` natively.

- [ ] **Step 6: Verify compilation**

Run: `mvn -f drlx-parser-core/pom.xml compile -pl . -q`

Expected: Compiles.

- [ ] **Step 7: Run existing accumulate tests for regression**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateTest -pl . -q`

Expected: All existing tests pass (single-source paths unchanged).

- [ ] **Step 8: Commit**

```
feat(builder): handle GroupElementIR source in accumulate builder

Refs #52
```

---

### Task 9: End-to-end tests — 2-param accumulate with and()

**Files:**
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write test — count() with and()**

```java
@Test
void accMultiPatternCount() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule CountJoined {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    var count = count()),
                do { results.add(count); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        kieSession.getEntryPoint("persons").insert(new Person("Alice", 1));
        kieSession.getEntryPoint("persons").insert(new Person("Bob", 2));
        kieSession.getEntryPoint("orders").insert(new Order("O1", 1, 100));
        kieSession.getEntryPoint("orders").insert(new Order("O2", 2, 200));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(2L);
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateTest#accMultiPatternCount -pl . -q`

Expected: PASS. If FAIL, debug — this exercises the no-extractor multi-source path.

- [ ] **Step 3: Write test — sum with single-binding extractor**

```java
@Test
void accMultiPatternSumSingleBinding() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule SumAmount {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    var total = sum(o.amount)),
                do { results.add(total); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        kieSession.getEntryPoint("persons").insert(new Person("Alice", 1));
        kieSession.getEntryPoint("persons").insert(new Person("Bob", 2));
        kieSession.getEntryPoint("orders").insert(new Order("O1", 1, 100));
        kieSession.getEntryPoint("orders").insert(new Order("O2", 2, 200));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(300);
}
```

- [ ] **Step 4: Run the test**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateTest#accMultiPatternSumSingleBinding -pl . -q`

Expected: PASS

- [ ] **Step 5: Write test — sum with cross-binding extractor**

```java
@Test
void accMultiPatternSumCrossBinding() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule WeightedSum {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    var weighted = sum(p.age * o.amount)),
                do { results.add(weighted); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        kieSession.getEntryPoint("persons").insert(new Person("Alice", 1));
        kieSession.getEntryPoint("persons").insert(new Person("Bob", 2));
        kieSession.getEntryPoint("orders").insert(new Order("O1", 1, 100));
        kieSession.getEntryPoint("orders").insert(new Order("O2", 2, 200));
        kieSession.fireAllRules();
    });
    // Alice(age=1) * O1(amount=100) = 100, Bob(age=2) * O2(amount=200) = 400
    assertThat(observed).containsExactly(500);
}
```

- [ ] **Step 6: Run the test**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateTest#accMultiPatternSumCrossBinding -pl . -q`

Expected: PASS. This is the critical test — it exercises multi-declaration MVEL3 compilation and tuple-based extraction with bindings from both source patterns.

- [ ] **Step 7: Commit**

```
test: add end-to-end tests for 2-param acc with multi-pattern source

Refs #52
```

---

### Task 10: End-to-end tests — custom accumulate with and()

**Files:**
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write test — 3-param custom acc with and()**

```java
@Test
void accMultiPatternCustom3Param() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule CustomSum {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    int s = 0;,
                    s = s + o.amount,
                    int total = s),
                do { results.add(total); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        kieSession.getEntryPoint("persons").insert(new Person("Alice", 1));
        kieSession.getEntryPoint("persons").insert(new Person("Bob", 2));
        kieSession.getEntryPoint("orders").insert(new Order("O1", 1, 100));
        kieSession.getEntryPoint("orders").insert(new Order("O2", 2, 200));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(300);
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateTest#accMultiPatternCustom3Param -pl . -q`

Expected: PASS

- [ ] **Step 3: Write test — 5-param custom acc with reverse**

```java
@Test
void accMultiPatternCustom5ParamWithReverse() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule CustomSumReverse {
                acc(and(var p : /persons, var o : /orders[customerId == p.age]),
                    int s = 0;,
                    { s = s + o.amount; },
                    { s = s - o.amount; },
                    int total = s),
                do { results.add(total); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        kieSession.getEntryPoint("persons").insert(new Person("Alice", 1));
        kieSession.getEntryPoint("persons").insert(new Person("Bob", 2));
        kieSession.getEntryPoint("orders").insert(new Order("O1", 1, 100));
        kieSession.getEntryPoint("orders").insert(new Order("O2", 2, 200));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(300);
}
```

- [ ] **Step 4: Run the test**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateTest#accMultiPatternCustom5ParamWithReverse -pl . -q`

Expected: PASS

- [ ] **Step 5: Commit**

```
test: add end-to-end tests for custom acc with multi-pattern source

Refs #52
```

---

### Task 11: Edge case — single-child and() + full regression

**Files:**
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`

- [ ] **Step 1: Write test — and() with single child pattern**

```java
@Test
void accSingleChildAndBehavesLikeSingleSource() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule SingleChildAnd {
                acc(and(var p : /persons),
                    var avgAge = avg(p.age)),
                do { results.add(avgAge); }
            }
            """;

    final List<Object> observed = new ArrayList<>();
    withSession(rule, (kieSession, listener) -> {
        kieSession.setGlobal("results", observed);
        kieSession.getEntryPoint("persons").insert(new Person("A", 20));
        kieSession.getEntryPoint("persons").insert(new Person("B", 40));
        kieSession.getEntryPoint("persons").insert(new Person("C", 60));
        kieSession.fireAllRules();
    });
    assertThat(observed).containsExactly(40.0);
}
```

Note: The Drools rete builder unwraps single-child AND groups (`ge.isAnd() && ge.getChildren().size() == 1`), so this should behave identically to a single-source accumulate even though our builder creates a multi-source path.

- [ ] **Step 2: Run the test**

Run: `mvn -f drlx-parser-core/pom.xml test -Dtest=AccumulateTest#accSingleChildAndBehavesLikeSingleSource -pl . -q`

Expected: PASS

- [ ] **Step 3: Run full regression**

Run: `mvn -f drlx-parser-core/pom.xml test -pl . -q`

Expected: All tests pass.

- [ ] **Step 4: Install module**

Run: `mvn -f drlx-parser-core/pom.xml install -pl . -DskipTests -q`

- [ ] **Step 5: Commit**

```
test: add single-child-and edge case, verify full regression

Closes #52
```
