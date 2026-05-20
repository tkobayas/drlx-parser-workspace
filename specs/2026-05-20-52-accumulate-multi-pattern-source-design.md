# #52 — Accumulate: multi-pattern source via and(...)

**Issue:** tkobayas/drlx-parser#52
**Epic:** #26 (DrlxCompiler enhancement round 2)
**Depends on:** #51 (acc() keyword forms — shipped)
**Date:** 2026-05-20

## Overview

Extend the `acc()` keyword to accept `and(...)` as the source expression,
enabling accumulates over joined tuples from multiple patterns. Currently
`accSource` only accepts a single `boundOopath`. After this change:

```
acc(and(p : /persons, t : /transactions[personId == p.id]),
    var total = sum(t.amount))
```

Applies to all acc forms: 2-param (built-in functions), 3-param, 4-param,
and 5-param (custom inline logic).

## Approach

Widen the IR `source` field from `PatternIR` to `LhsItemIR`. A single
pattern remains `PatternIR`; a multi-pattern source becomes
`GroupElementIR(AND, [PatternIR, ...])`. This reuses the existing sealed
hierarchy with no new types.

## Design

### 1. Grammar

Extend `accSource` to accept `andElement` as an alternative:

```antlr
accSource
    : boundOopath
    | andElement
    ;
```

`andElement` already exists as `DRLX_AND '(' groupChild (',' groupChild)* ')'`,
and `groupChild` includes `boundOopath`, so patterns like
`and(p : /persons, a : /addresses[city == p.city])` parse with no new tokens
or rules.

### 2. IR Model

Both `AccumulatePatternIR` and `CustomAccumulateIR` widen `source` from
`PatternIR` to `LhsItemIR`:

```java
public record AccumulatePatternIR(LhsItemIR source,
                                  List<AccumulatorIR> accumulators) implements LhsItemIR {}

public record CustomAccumulateIR(LhsItemIR source,
                                 ...) implements LhsItemIR {}
```

Single-source → `PatternIR` (unchanged behavior).
Multi-source → `GroupElementIR(AND, [PatternIR, PatternIR, ...])`.

Semantic validation is enforced by the grammar (only `boundOopath | andElement`
is parseable as `accSource`), so invalid IR shapes like `EvalIR` or nested
`AccumulatePatternIR` cannot be constructed from parsed DRLX.

### 3. Visitor (Parser → IR)

In `DrlxToRuleAstVisitor.buildAccKeywordItem()`, branch on which alternative
`accSource` matched:

```java
LhsItemIR source;
if (ctx.accSource().boundOopath() != null) {
    source = buildPatternFromBoundOopath(ctx.accSource().boundOopath());
} else {
    source = buildGroupElement(ctx.accSource().andElement());
}
```

Reuses the existing `buildGroupElement` / `buildGroupChild` logic that already
handles `andElement` for the top-level LHS.

### 4. Runtime Builder (IR → Drools objects)

In `buildAccumulatePattern()` and `buildCustomAccumulatePattern()`, the source
construction branches on the IR type:

```java
LhsItemIR srcIr = accPat.source();
RuleConditionElement srcElement;
Map<String, BoundVariable> innerScope = new LinkedHashMap<>(outerScope);

if (srcIr instanceof PatternIR patIr) {
    Pattern srcPattern = buildPattern(patIr, ...);
    srcElement = srcPattern;
    Declaration srcDecl = srcPattern.getDeclaration();
    if (srcDecl != null) {
        innerScope.put(srcDecl.getIdentifier(), new BoundVariable(...));
    }
} else if (srcIr instanceof GroupElementIR groupIr) {
    GroupElement andGroup = GroupElementFactory.newAndInstance();
    buildLhs(groupIr.children(), andGroup, typeResolver, entryPointTypes,
             unitClass, innerScope);
    srcElement = andGroup;
}
```

The runtime `Accumulate.source` is typed as `RuleConditionElement`, so it
already accepts a `GroupElement` — no Drools runtime changes needed.

`innerScope` collects bindings from all child patterns via `buildLhs`, so
accumulator extractors can reference bindings from any source pattern.

### 5. Runtime Accumulator Plumbing (multi-pattern)

The current accumulator classes (`DrlxLambdaAccumulator`, `DrlxValueExtractor`,
`DrlxCustomAccumulator`) assume a single source pattern and extract the fact
via `handle.getObject()`. With a multi-pattern AND source, the Drools
AccumulateNode splits joined tuples: `handle` holds the **rightmost** fact,
and `tuple` (BaseTuple) holds earlier facts, accessible via declarations.

Three changes are needed:

#### 5a. DrlxLambdaAccumulator — tuple-aware accumulate

`DrlxLambdaAccumulator.accumulate()` currently does:
```java
Object value = extractor.apply(handle.getObject());  // single fact only
```

For multi-pattern sources, it must use `innerDecls` (populated by
`Accumulate.initInnerDeclarationCache()` → `source.getInnerDeclarations()`,
which correctly aggregates declarations from all AND-group children) to
extract all bound facts from the tuple:

```java
if (multiSource) {
    Map<String, Object> bindings = new HashMap<>();
    for (Declaration d : innerDecls) {
        bindings.put(d.getIdentifier(), d.getValue(vr, tuple));
    }
    value = extractor.applyMulti(bindings);
} else {
    value = extractor.apply(handle.getObject());  // unchanged
}
```

The `multiSource` flag is set at build time (constructor) based on whether
the source is a GroupElement. The single-source path stays untouched for
regression safety.

Same applies to `tryReverse()`.

#### 5b. DrlxValueExtractor — multi-binding variant

`DrlxValueExtractor.apply(Object fact)` currently builds a one-entry map:
```java
map.put(sourceBindingName, fact);
```

Add an `applyMulti(Map<String, Object> bindings)` method that passes the
pre-built binding map directly to the MVEL evaluator:
```java
public Object applyMulti(Map<String, Object> bindings) {
    return evaluator.eval(bindings);
}
```

The single-arg `apply(Object fact)` remains unchanged. `DrlxValueExtractor`
needs a constructor variant that accepts multiple binding names (for
compile-time type resolution) instead of a single `sourceBindingName`.

#### 5c. DrlxCustomAccumulator — multi-binding context

`DrlxCustomAccumulator.accumulate()` currently does:
```java
Object srcFact = handle.getObject();
map.put(srcBindingName, srcFact);
```

For multi-pattern sources, replace single `srcBindingName` with a list of
source binding names. In `accumulate()` and `tryReverse()`, use `innerDecls`
to extract all bound facts and inject them all into the context map:

```java
if (multiSource) {
    for (Declaration d : innerDecls) {
        map.put(d.getIdentifier(), d.getValue(vr, tuple));
    }
} else {
    map.put(srcBindingName, handle.getObject());  // unchanged
}
```

The action/reverse/result MVEL evaluators then see all source bindings in
the map and can reference any of them.

### 6. Accumulator Build-Time Compilation

`buildSingleAccumulator()` currently takes a single `srcClass` and
`srcBindingName`. For multi-pattern sources, an expression like
`p.age * t.amount` references bindings of different types.

Modify `buildSingleAccumulator` to accept `Map<String, BoundVariable> innerScope`
instead of (or in addition to) the single class/name pair. For single-source,
the scope has one entry — behavior unchanged. For multi-source, all source
bindings are available to the MVEL3 compiler.

Pass `multiSource = true` to `DrlxLambdaAccumulator` and
`DrlxCustomAccumulator` constructors so they know which extraction path
to use at runtime.

The `requiredDeclarations` passed to `SingleAccumulate`/`MultiAccumulate`
must include declarations from all source patterns whose bindings are
referenced by the accumulator's extractor expression. For single-source
this is at most one declaration (unchanged). For multi-source, it is the
subset of source-pattern declarations that appear in extractor expressions.

Same applies to `DrlxCustomAccumulator` compilation in
`buildCustomAccumulatePattern()`.

### 7. Testing

**Parser test:** `accSource` with `andElement` produces correct parse tree.

**Visitor/IR test:** visitor produces `AccumulatePatternIR(GroupElementIR(AND, [...]), ...)`
and `CustomAccumulateIR(GroupElementIR(AND, [...]), ...)`.

**End-to-end runtime tests:**

- 2-param acc with and(), count: `acc(and(p : /persons, a : /addresses[city == p.city]), var count = count())` — count joined tuples
- 2-param acc, single-binding extractor: `acc(and(p : /persons, t : /transactions[personId == p.id]), var total = sum(t.amount))` — sum referencing rightmost binding only
- 2-param acc, **cross-binding extractor**: `acc(and(p : /persons, t : /transactions[personId == p.id]), var weighted = sum(p.age * t.amount))` — extractor references bindings from both source patterns (exercises tuple-aware extraction, would fail with handle-only approach)
- 3-param custom acc with and(): inline init/action/result over joined patterns
- 5-param custom acc with and(): inline with reverse block over joined patterns
- Single-source regression: all existing accumulate tests pass unchanged
