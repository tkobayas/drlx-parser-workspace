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
- `accumulate(...)` — computes group key, calls `GroupByContext.getGroup()` to find/create the group, delegates to `innerAccumulate.accumulate()` via the 6-arg overload
- `tryReverse(...)` — finds group from match's memory, delegates reverse, removes empty groups
- `getResult(...)` — delegates to `innerAccumulate.getResult()`, returns null if group is empty
- `isMultiFunction()`, `supportsReverse()`, `getAccumulators()`, `createFunctionContext()`, `createWorkingMemoryContext()` — all delegate to `innerAccumulate`
- `init(...)` — no-op (initialization happens when a group is first created)

### Group key computation

- **Single-source:** The grouping function receives the source fact and returns the key. Compiled by `DrlxLambdaCompiler` from the key expression, same approach as accumulate extractor expressions.
- **Multi-source (`and(...)`):** The `DrlxValueExtractor` receives a bindings map and returns the key. Same infrastructure as multi-source accumulate extractors.

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

### Result pattern structure

`PhreakGroupByNode.createResult()` always produces an `Object[]`:

- **Single function:** `Object[]{accResult, key}` — `ArrayElementReader(0)` for the acc result, `ArrayElementReader(1)` for the key
- **Multi function:** `Object[]{accResult0, accResult1, ..., key}` — `ArrayElementReader(i)` for acc result `i`, `ArrayElementReader(n)` for the key

The result pattern is always `Object[].class` (unlike non-groupBy single-acc which uses the function's result type). Declarations are registered to `outerScope` so they're available in the consequence.

When the group key is bound (`var g = ...`), a Declaration is added at the last array position. When unbound, no key Declaration is created — the key is used internally for partitioning only.

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
