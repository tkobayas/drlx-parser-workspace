# #40 — Group By

**Issue:** https://github.com/tkobayas/drlx-parser/issues/40
**Parent epic:** #78
**Follows:** #39 (Accumulate v1), #48 (MVEL3 extractors), #51 (acc keyword forms), #52 (multi-pattern source)
**DRLXXXX reference:** Lines 891–902 (§Group By)

## Summary

`groupBy` partitions accumulate source facts by a grouping expression and maintains separate accumulator state for each distinct group key. It "works exactly as per acc, but with an extra parameter for the group definition" (DRLXXXX spec). The rule fires once per group, with the accumulate result and optional group key binding available in the consequence.

## Syntax

### Built-in function form

```
// Unbound group key
groupBy(var p : /persons,
        p.status,
        var avgAge = avg(p.age))

// Bound group key — accessible in do-block
groupBy(var p : /persons,
        var g = p.status,
        var avgAge = avg(p.age))

// Multiple functions
groupBy(var p : /persons,
        var g = p.status,
        (var minAge = min(p.age),
         var maxAge = max(p.age)))

// Multi-pattern source
groupBy(and(var p : /persons,
            var cf : /cashflows[personId == p.id]),
        p.status,
        var total = sum(cf.amount))
```

### Custom accumulator form

```
// 3-param (init, action, result)
groupBy(var p : /persons,
        p.status,
        int s = 0;,
        s = s + p.age,
        int total = s)

// 5-param (init, action, reverse, result)
groupBy(var p : /persons,
        var g = p.status,
        { int count = 0; int total = 0; },
        { total = total + p.age; count = count + 1; },
        { total = total - p.age; count = count - 1; },
        double avg = (double) total / count)
```

### Group key parameter

The group key is the **second parameter** (after source, before the accumulate body). It takes one of two forms:

- **Unbound:** `p.status` — the expression is evaluated for grouping but not named. Not accessible in the consequence.
- **Bound:** `var g = p.status` — the value is named `g` and accessible in the do-block alongside accumulate results.

## Grammar

A new `groupByKeywordItem` rule parallel to `accKeywordItem`, reusing existing `accSource` and `accBody` rules:

```antlr
ruleItem
    : ...
    | groupByKeywordItem ','
    ;

groupByKeywordItem
    : identifier '(' accSource ',' groupByKey ',' accBody ')'
    ;

groupByKey
    : VAR identifier '=' expression    // bound: var g = p.status
    | expression                       // unbound: p.status
    ;
```

The visitor validates the `identifier` text is `groupBy` (same contextual-keyword pattern as `acc`). All accumulate body rules (`accBody`, `accFunctionList`, `accInitVars`, `accActionBlock`, `accResultBinding`) are reused unchanged.

## IR Model

Two new records added to `DrlxRuleAstModel`, both implementing `LhsItemIR`:

```java
public record GroupByAccumulateIR(
    LhsItemIR source,
    List<AccumulatorIR> accumulators,
    String groupKeyExpression,
    String groupKeyBindName,          // null if unbound
    List<String> groupKeyReferencedBindings
) implements LhsItemIR {}

public record GroupByCustomAccumulateIR(
    LhsItemIR source,
    List<InitVarIR> initVars,
    String actionBlock,
    String reverseBlock,
    String resultTypeName,
    String resultBindName,
    String resultExpression,
    List<String> referencedBindings,
    String groupKeyExpression,
    String groupKeyBindName,          // null if unbound
    List<String> groupKeyReferencedBindings
) implements LhsItemIR {}
```

The sealed `LhsItemIR` interface gains these two new permits. The visitor's `buildGroupByKeywordItem` method produces one or the other depending on whether the body uses built-in functions or custom init/action/result.

## Runtime: DrlxGroupByAccumulate

A new class `DrlxGroupByAccumulate extends Accumulate` in `org.drools.drlx.builder`. Structurally mirrors `LambdaGroupByAccumulate` from `drools-model-compiler` but uses drlx-parser's own lambda infrastructure rather than `FunctionN`/`Function1`.

### Fields

- `Accumulate innerAccumulate` — the wrapped `SingleAccumulate` or `MultiAccumulate`
- `Declaration[] groupingDeclarations` — declarations referenced by the grouping expression
- `Function<Object, Object> groupingFunction` — MVEL3-compiled lambda for group key (single-source)
- `DrlxValueExtractor groupingFunctionMulti` — for multi-source `and(...)` case

### Methods

All delegate to `innerAccumulate` after group key handling:

- `isGroupBy()` → `true`
- `accumulate(5-arg)` — the primary entry point called by `PhreakAccumulateNode.addMatch()`. Receives `GroupByContext` as `context`. Computes the group key, calls `GroupByContext.getGroup()` to find/create the group's `TupleListWithContext`, then calls `this.accumulate(6-arg)`.
- `accumulate(6-arg)` — called by the 5-arg method above AND directly by `PhreakGroupByNode.reaccumulateForLeftTuple()` (re-accumulating within a known group). Moves the tuple list to the propagate list, then delegates to `innerAccumulate.accumulate(5-arg)` with the group's `AccumulateContextEntry` as context. **Note:** `SingleAccumulate` and `MultiAccumulate` both throw `UnsupportedOperationException` on their 6-arg overload — it is only implemented by the groupBy wrapper.
- `tryReverse(...)` — finds group from match's memory (`TupleListWithContext`), delegates reverse to `innerAccumulate`, removes empty groups from `GroupByContext`
- `getResult(...)` — delegates to `innerAccumulate.getResult()`, returns null if group is empty (via `AccumulateContextEntry.isEmpty()`)
- `isMultiFunction()`, `supportsReverse()`, `getAccumulators()`, `createFunctionContext()`, `createWorkingMemoryContext()` — all delegate to `innerAccumulate`
- `init(...)` — no-op (initialization happens lazily when `GroupByContext.getGroup()` creates a new group)

### Group key computation

Key computation resolves declaration values from the tuple or handle (mirroring `LambdaGroupByAccumulate.getValue()`), then applies the compiled grouping function:

- **Single-source:** The fact is obtained from `handle.getObject()` (or via `declaration.getValue()` from the tuple if the declaration's tuple index is within range). The MVEL3-compiled `Function<Object, Object>` maps the fact to the key (e.g., `p -> p.getStatus()`). Compiled by `DrlxLambdaCompiler` from the key expression, same approach as accumulate extractor expressions.
- **Multi-source (`and(...)`):** Declarations for all referenced bindings are resolved from the tuple. The `DrlxValueExtractor` receives a bindings map and returns the key. Same infrastructure as multi-source accumulate extractors.

## Runtime Builder Wiring

### Visitor (`DrlxToRuleAstVisitor`)

A `buildGroupByKeywordItem` method, called when `ruleItem` matches `groupByKeywordItem`:

1. Validates the keyword is `groupBy`
2. Parses `accSource` — reuses `buildPatternFromBoundOopath` / `buildGroupElementFromChildren`
3. Parses `groupByKey` — extracts expression and optional bind name
4. Parses `accBody` — delegates to same body-parsing logic as `buildAccKeywordItem`
5. Returns `GroupByAccumulateIR` or `GroupByCustomAccumulateIR`

### Runtime builder (`DrlxRuleAstRuntimeBuilder`)

A `buildGroupByAccumulatePattern` method (for built-in functions) and `buildGroupByCustomAccumulatePattern` (for custom):

1. Builds source element (Pattern or GroupElement) — same as `buildAccumulatePattern`
2. Builds inner accumulate (SingleAccumulate or MultiAccumulate) — reuses existing accumulator-building logic
3. Compiles grouping expression via `DrlxLambdaCompiler`
4. Wraps inner accumulate in `DrlxGroupByAccumulate`
5. Constructs result pattern as `Object[].class` with `ArrayElementReader` positions

### Inner accumulate construction

- **Single function:** The inner accumulate is a `SingleAccumulate` — same as non-groupBy. No array size concerns since `PhreakGroupByNode.createResult()` wraps the scalar result in `new Object[]{result, key}`.
- **Multi function:** The inner accumulate is a `MultiAccumulate` constructed with **`n + 1` array slots** (not `n`). The extra slot is pre-allocated for the group key. `PhreakGroupByNode.createResult()` places the key at `array[array.length - 1]` in-place.

### Result pattern structure

`PhreakGroupByNode.createResult()` always produces an `Object[]`, regardless of whether the inner accumulate is single or multi:

- **Single function:** `Object[]{accResult, key}` — `ArrayElementReader(0)` for the acc result, `ArrayElementReader(1)` for the key
- **Multi function:** `Object[]{accResult0, accResult1, ..., key}` — `ArrayElementReader(i)` for acc result `i`, `ArrayElementReader(n)` for the key

The result pattern is **always `Object[].class`** (unlike non-groupBy single-acc which uses the function's result type). This is because `PhreakGroupByNode.createResult()` always wraps results in an array. Declarations are registered to `outerScope` so they're available in the consequence.

When the group key is bound (`var g = ...`), a Declaration backed by `ArrayElementReader` at the last position is added. When unbound, no key Declaration is created — the key is used internally for partitioning only.

## Testing

### Parser tests (`DrlxParserTest`)

- `groupBy` with unbound group key + single built-in function
- `groupBy` with bound group key (`var g = ...`) + single built-in function
- `groupBy` with multiple built-in functions (grouped in parentheses)
- `groupBy` with multi-pattern source (`and(...)`)
- `groupBy` with custom accumulator (3-param, 5-param)
- Rejection: missing group expression
- Rejection: `groupBy` with inline-from syntax (not supported)

### Runtime tests (`GroupByTest` in `syntax/`)

- Single function: per-group avg with bound key in consequence
- Multiple functions per group: min + max
- Multi-pattern source: `groupBy(and(p, cf), p.status, sum(cf.amount))`
- Custom accumulator form: init/action/result with grouping
- Custom accumulator with reverse: retraction updates correct group
- Retraction removes group entirely when last member removed
- Unbound key: verify accumulate results are correct without key in consequence

### AST structure tests

- Result pattern is `Object[].class` with correct `ArrayElementReader` positions
- `isGroupBy()` returns `true` on the accumulate source
- Group key declaration at last array position (when bound)

All tests extend `DrlxBuilderTestSupport` and use existing `MyUnit` with `persons`/`orders`/`cashflows` data stores.
