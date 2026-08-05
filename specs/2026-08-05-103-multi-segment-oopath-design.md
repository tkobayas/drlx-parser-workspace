# Multi-Segment OOPath Support (#103)

## Problem

The DRLX grammar and `DrlxToJavaParserVisitor` correctly parse multi-segment
OOPath expressions like `/persons/addresses[city == "London"]`, but
`DrlxToRuleAstVisitor` discards intermediate segments. `PatternIR` has no
OOPath-specific fields, and `DrlxRuleAstRuntimeBuilder` produces a single
flat pattern per `PatternIR` sourced from an `EntryPointId`. Multi-segment
OOPath requires a chain of patterns where subsequent segments use a `From`
source navigating the object graph.

## Context

- **Ruleunit-only scope.** The first segment always resolves to a `DataStore`
  on the `RuleUnitData`. Subsequent segments navigate properties on the matched
  type.
- **drools runtime support confirmed.** A multi-segment OOPath test
  (`OOPathMultilevelTest`) in `drools-ruleunits-impl` passes — the runtime
  can execute chained `From` patterns.
- **drools classic reference.** `OOPathExprGenerator` iterates `OOPathChunk`
  nodes and chains `reactiveFrom(previousBind, accessorLambda)` calls. We
  follow the same conceptual pattern but build drools-core `Pattern`/`From`
  objects directly (no DSL code generation).

## Design

### 1. IR model (`DrlxRuleAstModel`)

New record:

```java
public record OopathSegmentIR(String field,
                               List<String> conditions,
                               String inlineCast) {}
```

Add to `PatternIR`:

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
                        List<OopathSegmentIR> oopathSegments)
    implements LhsItemIR {}
```

For single-segment OOPath (the common case), `oopathSegments` is empty.
`entryPoint` and `conditions` work exactly as before. For multi-segment,
`conditions` holds the root segment's conditions, and `oopathSegments` holds
each subsequent segment.

Example — `var addr : /persons[age > 18]/addresses[city == "London"]`:
- `entryPoint = "persons"`, `conditions = ["age > 18"]`
- `oopathSegments = [OopathSegmentIR("addresses", ["city == \"London\""], null)]`

### 2. Visitor (`DrlxToRuleAstVisitor`)

Add a new method `extractOopathSegments`:

```java
private List<OopathSegmentIR> extractOopathSegments(
        DrlxParser.OopathExpressionContext ctx) {
    List<DrlxParser.OopathChunkContext> chunks = ctx.oopathChunk();
    if (chunks.isEmpty()) return List.of();
    return chunks.stream().map(c -> {
        String field = c.identifier(0).getText();
        String cast = c.identifier().size() > 1
                ? c.identifier(1).getText() : null;
        List<String> conds = c.drlxExpression().stream()
                .filter(de -> de.customConstraint() == null)
                .map(this::getText).toList();
        return new OopathSegmentIR(field, conds, cast);
    }).toList();
}
```

Update `extractConditions` so that when chunks are present it returns the
**root's** conditions (not the last chunk's). Currently it returns the last
chunk's conditions — this must change to return the root's, since the chunks'
conditions now go into `oopathSegments`.

All `buildPatternFrom*` methods pass the new segments list to `PatternIR`.

### 3. Runtime builder (`DrlxRuleAstRuntimeBuilder`)

When `PatternIR.oopathSegments()` is non-empty, `buildPattern` produces a
chain of patterns:

1. **Root pattern** — built as today: type from `entryPointTypes.get(entryPoint)`,
   source is `EntryPointId`. Gets an auto-generated bind name (e.g.,
   `"_oopath$0"`) if it doesn't already have one.
2. **Per-segment pattern** — for each `OopathSegmentIR`:
   - Resolve the getter for `segment.field()` on the previous pattern's class
     via reflection.
   - Determine the element type: if the getter returns `Iterable`/`List`,
     extract the generic type parameter; otherwise use the return type directly.
     If `segment.inlineCast()` is set, use that type instead.
   - Create a new `Pattern` with that element type.
   - Set its source to `new From(new OopathFieldDataProvider(previousDecl, getter))`.
   - Apply `segment.conditions()` as constraints.
3. **Bind name** — only the last pattern in the chain carries the user's
   explicit `bindName`. Intermediate patterns get auto-generated names.
4. **All patterns** are added to the parent `GroupElement`.

The call site at line 687 changes from adding a single pattern to adding
multiple.

#### `OopathFieldDataProvider`

New class in `org.drools.drlx.builder` implementing `DataProvider`:

```java
public class OopathFieldDataProvider implements DataProvider {
    private final Declaration sourceDeclaration;
    private final Method getter;

    // getResults(): invoke getter on the source object from the tuple,
    // if result is Iterable return its iterator,
    // if single value return singleton iterator,
    // if null return empty iterator.
}
```

This is analogous to drools' `XpathDataProvider` but simpler — no reactive
evaluation, just getter invocation.

### 4. Protobuf (`drlx_rule_ast.proto`)

New message:

```proto
message OopathSegmentParseResult {
  string field = 1;
  repeated string conditions = 2;
  string inline_cast = 3;
}
```

Add to `PatternParseResult`:

```proto
repeated OopathSegmentParseResult oopath_segments = 12;
```

Field number 12 is next available. Serialization/deserialization in
`DrlxRuleAstParseResult` maps between `OopathSegmentIR` and the proto
message. Existing cached results deserialize with an empty segment list —
backward compatible.

### 5. Domain model changes (test only)

Add `List<Address> addresses` with getter/setter and `addAddress()` to
`org.drools.drlx.domain.Person`. Purely additive — existing constructors and
tests are unaffected.

### 6. Test plan

- **Parser test** (already exists): `DrlxToJavaParserVisitorTest.testParseMultiSegmentOOPath`
  — verifies 2 chunks are produced.
- **Execution test** (new): `withMyUnitInstance` style test that runs
  `var addr : /persons/addresses[city == "London"]` and asserts correct
  firings.
- **Conditions on root + segment**: test `/persons[age > 18]/addresses[city == "London"]`
  to verify conditions on both segments work.
- **Protobuf round-trip**: serialize and deserialize a `PatternIR` with
  segments, assert equality.

## Scope

**In scope:**
- Multi-segment OOPath with conditions on any segment
- Inline cast on segments
- Bind name on the last segment
- Protobuf round-trip

**Out of scope:**
- Reactivity for intermediate segments (use plain `From`, not reactive)
- Multi-segment inside `not`/`exists`/`accumulate` — may work naturally but
  not explicitly tested or designed for
- Passive (`?`) on intermediate segments
