# #52 — Accumulate: multi-pattern source via and(...)

**Issue:** tkobayas/drlx-parser#52
**Epic:** #26 (DrlxCompiler enhancement round 2)
**Depends on:** #51 (acc() keyword forms — shipped)
**Date:** 2026-05-20
**Reviewed:** 2026-05-21 — corrections from Drools DRL runtime analysis

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
via `handle.getObject()`. With a multi-pattern AND source, Drools creates a
subnetwork with a `RightInputAdapterNode` (RIA). The RIA wraps the entire
subnetwork tuple into a synthetic fact handle:

- `handle.getObject()` returns the **SubnetworkTuple itself** (a `TupleImpl`
  instance), **not** a domain fact object
- `tuple` (set to the RIA's right tuple by `PhreakAccumulateNode`) contains
  the full pattern chain, with individual facts accessible via declarations

This means **all** fact access in multi-pattern context must go through the
tuple, never through `handle.getObject()`. Even single-binding extractors
like `sum(t.amount)` cannot use `handle.getObject()` — it would receive a
SubnetworkTuple, not the `t` fact.

The `Accumulator.accumulate()` method receives `Declaration[] innerDecls`
populated from `Accumulate.getInnerDeclarationCache()`, which aggregates
declarations from all source patterns via `GroupElement.getInnerDeclarations()`.
Each declaration has a `tupleIndex` assigned by the rete builder, so
`declaration.getValue(valueResolver, tuple)` correctly navigates the tuple
chain.

`Declaration.getValue(ValueResolver, BaseTuple)` exists in Drools
(`Declaration.java:228`) and calls `tuple.get(this)` → walks the tuple
chain by `tupleIndex` → returns the fact via `readAccessor`. For pattern-level
DRLX bindings (`p : /persons`), the readAccessor is `SelfReferenceClassFieldReader`
which returns the fact object itself — exactly what the MVEL3 evaluator map needs.

Three changes are needed:

#### 5a. DrlxLambdaAccumulator — tuple-aware accumulate

`DrlxLambdaAccumulator.accumulate()` currently does:
```java
Object value = (extractor == null) ? handle.getObject() : extractor.apply(handle.getObject());
```

For multi-pattern sources, `handle.getObject()` returns a `SubnetworkTuple`
(not a fact), so this path fails for any extractor and gives meaningless
results even for `count()` (which accidentally works because `CountFunction`
ignores its argument).

For multi-pattern, extract all bound facts from the tuple via `innerDecls`:

```java
if (multiSource) {
    Map<String, Object> bindings = new HashMap<>();
    for (Declaration d : innerDecls) {
        bindings.put(d.getIdentifier(), d.getValue(vr, tuple));
    }
    value = (extractor == null) ? bindings : extractor.applyMulti(bindings);
} else {
    value = (extractor == null) ? handle.getObject() : extractor.apply(handle.getObject());
}
```

The `multiSource` flag is set at build time (constructor) based on whether
the source is a GroupElement. The single-source path stays untouched for
regression safety.

`tryReverse()` for `DrlxLambdaAccumulator` does **not** need tuple access.
It receives the pre-computed `value` returned by `accumulate()` and passes
it to `accFunction.reverse(ctx, value)`. Built-in functions (sum, avg, etc.)
reverse by subtracting/undoing the previously accumulated value, so the
cached `value` is sufficient regardless of single-source or multi-source.

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

The single-arg `apply(Object fact)` remains unchanged. The constructor
does not need to change — `sourceBindingName` is only used by `apply()`.
Multi-source compilation is handled entirely in `DrlxLambdaCompiler`
(see section 6).

#### 5c. DrlxCustomAccumulator — multi-binding context

`DrlxCustomAccumulator.accumulate()` currently does:
```java
Object srcFact = handle.getObject();
map.put(srcBindingName, srcFact);
```

For multi-pattern sources, replace single `srcBindingName` with a list of
source binding names. In `accumulate()`, use `innerDecls` to extract all
bound facts and inject them all into the context map:

```java
if (multiSource) {
    for (Declaration d : innerDecls) {
        map.put(d.getIdentifier(), d.getValue(vr, tuple));
    }
} else {
    map.put(srcBindingName, handle.getObject());  // unchanged
}
```

The action MVEL evaluator then sees all source bindings in the map.
In the `finally` block, remove all injected source bindings (loop over the
same declarations or keep a list of injected keys).

`accumulate()` return value: for single-source, currently returns `srcFact`
(used by `tryReverse()` as `value` parameter). For multi-source, return
`null` — the reverse block must re-extract from the tuple, not rely on
the cached value.

`tryReverse()`: for multi-source, must also extract all source facts from
the tuple via `innerDecls` (same as `accumulate()`), ignoring the `value`
parameter. This mirrors how Drools MVEL handles reversal — the MVEL
compilation unit re-extracts via `readLocalsFromTuple` rather than using
the cached value.

```java
// tryReverse() multi-source path
if (multiSource) {
    for (Declaration d : innerDecls) {
        map.put(d.getIdentifier(), d.getValue(vr, tuple));
    }
    try { reverseEval.eval(map); }
    finally { /* remove injected keys */ }
} else {
    map.put(srcBindingName, value);  // unchanged
    try { reverseEval.eval(map); }
    finally { map.remove(srcBindingName); }
}
```

The result evaluator is unchanged — it only reads init vars, not source
bindings.

### 6. Accumulator Build-Time Compilation

#### 6a. buildSingleAccumulator

`buildSingleAccumulator()` currently takes a single `srcClass` and
`srcBindingName`. For multi-pattern sources, an expression like
`p.age * t.amount` references bindings of different types.

Modify `buildSingleAccumulator` to accept `Map<String, BoundVariable> innerScope`
instead of (or in addition to) the single class/name pair. For single-source,
the scope has one entry — behavior unchanged. For multi-source, all source
bindings are available to the MVEL3 compiler.

Pass `multiSource = true` to `DrlxLambdaAccumulator` constructor so it
knows which extraction path to use at runtime.

#### 6b. DrlxLambdaCompiler.createValueExtractor — multi-declaration

`createBatchValueExtractor()` currently compiles with a single MVEL3
declaration:

```java
MVEL.map(Declaration.of(sourceBindingName, srcClass))
    .out(Object.class).expression(argExpr).build();
```

For multi-source, compile with declarations for all source bindings:

```java
Declaration<?>[] decls = innerScope.entrySet().stream()
    .map(e -> Declaration.of(e.getKey(), e.getValue().type()))
    .toArray(Declaration[]::new);
MVEL.map(decls).out(Object.class).expression(argExpr).build();
```

Add a `createValueExtractor` overload (or replace the existing one) that
accepts `Map<String, BoundVariable> innerScope` instead of
`(Class<?> srcClass, String sourceBindingName)`.

#### 6c. DrlxLambdaCompiler.createCustomAccumulator — multi-declaration

`createCustomAccumulator()` currently builds action/reverse declarations
with a single source binding:

```java
actionDecls.add(Declaration.of(srcBindingName, srcClass));
```

For multi-source, add declarations for all source bindings:

```java
for (Map.Entry<String, BoundVariable> e : sourceScope.entrySet()) {
    actionDecls.add(Declaration.of(e.getKey(), e.getValue().type()));
}
```

Pass `multiSource = true` and the list of source binding names to
`DrlxCustomAccumulator` constructor.

#### 6d. Custom accumulate allowedNames validation (#54)

`buildCustomAccumulatePattern()` currently rejects outer-binding references
with `allowedNames` containing only `srcBindingName` + init var names.
For multi-source, `allowedNames` must include **all source pattern binding
names** from `innerScope`, not just one. Otherwise a reference to a
non-rightmost source binding would incorrectly trigger the #54 error.

#### 6e. requiredDeclarations

The `requiredDeclarations` passed to `SingleAccumulate`/`MultiAccumulate`
must include declarations from all source patterns whose bindings are
referenced by the accumulator's extractor expression. For single-source
this is at most one declaration (unchanged). For multi-source, it is the
subset of source-pattern declarations that appear in extractor expressions.
The existing `requiredFor()` method should work unchanged — `innerScope`
will contain all source bindings, and `requiredFor()` filters to those
referenced by the accumulator.

### 7. Testing

**Parser test:** `accSource` with `andElement` produces correct parse tree.

**Visitor/IR test:** visitor produces `AccumulatePatternIR(GroupElementIR(AND, [...]), ...)`
and `CustomAccumulateIR(GroupElementIR(AND, [...]), ...)`.

**End-to-end runtime tests:**

- 2-param acc with and(), count (zero args): `acc(and(p : /persons, a : /addresses[city == p.city]), var count = count())` — count joined tuples. Verifies the no-extractor path with SubnetworkTuple handle.
- 2-param acc, single-binding extractor: `acc(and(p : /persons, t : /transactions[personId == p.id]), var total = sum(t.amount))` — sum referencing one source binding. Verifies tuple-based extraction even for single-binding expressions (handle.getObject() would return SubnetworkTuple, not the fact).
- 2-param acc, **cross-binding extractor**: `acc(and(p : /persons, t : /transactions[personId == p.id]), var weighted = sum(p.age * t.amount))` — extractor references bindings from both source patterns (exercises tuple-aware extraction with multi-declaration MVEL3 compilation)
- 3-param custom acc with and(): inline init/action/result over joined patterns. Action block references bindings from multiple source patterns.
- 5-param custom acc with and(): inline with reverse block over joined patterns. Verifies that tryReverse() correctly re-extracts from tuple.
- and() with single child pattern: `acc(and(p : /persons), var count = count())` — edge case. Drools rete builder unwraps single-child AND groups (`ge.isAnd() && ge.getChildren().size() == 1`), so this should behave identically to a single-source accumulate.
- Single-source regression: all existing accumulate tests pass unchanged
