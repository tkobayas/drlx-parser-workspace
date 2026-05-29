# Query Result Binding — `var t : /queryName(...)`

**Issue:** #57
**Parent epic:** #77
**Date:** 2026-05-29

## Summary

Support binding the entire query invocation result to a variable, enabling named access (`t.a`), indexed access (`t[0]`), and method access (`t.objects()`).

## DRLXXXX Reference

Lines 920-938. Example:

```
rule R1 {
    var a : /things[name == "a"],
    var t : /trusts(a, var b),
    allEqual(t.a, t[0])
}
```

- `var t :` binds the query result row to `t`
- `t.a` returns the value of query parameter `a` (named access)
- `t[0]` returns the first element (indexed access)
- `t.a` and `t[0]` return the same value (coerced to Object)

## Approach

**QueryResultRow extends AbstractMap<String, Object>** — leverages MVEL3's native Map property rewriting:

- `t.a` → MVEL3 `maybeRewriteToGetter()` detects Map type → rewrites to `t.get("a")`
- `t[0]` → MVEL3 `rewriteArrayAccessExpr()` detects non-array → rewrites to `t.get(0)`
- `t.handles()`, `t.objects()` → normal method call resolution

No changes to MVEL3 or the expression compiler are needed.

## Components

### 1. QueryResultRow class

**Location:** `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRow.java`

Extends `AbstractMap<String, Object>`, implements `Iterable<Object>`.

**Fields:**
- `Object[] values` — query parameter values from the result pattern's Object[]
- `Map<String, Integer> nameToIndex` — parameter name → array index mapping

**Key methods:**
- `get(Object key)` — dispatches on key type:
  - `String` → look up index via `nameToIndex`, return `values[index]`
  - `Integer` → return `values[intValue]` directly
- `objects()` → returns the `values` array
- `handles()` → throws `UnsupportedOperationException` (deferred to #82)
- `entrySet()` → builds entry set from nameToIndex + values (required by AbstractMap)
- `iterator()` → iterates over `values`
- `size()` → returns `values.length`

### 2. QueryResultRowReader

**Location:** `drlx-parser-core/src/main/java/org/drools/drlx/runtime/QueryResultRowReader.java`

A `ReadAccessor` implementation that wraps the result `Object[]` into a `QueryResultRow`.

Implements `ReadAccessor` directly, following the same pattern as `DroolsQueryElementsReader`. Receives the `Object[]` fact object from the result pattern and returns a `QueryResultRow` wrapping it.

**Fields:**
- `Map<String, Integer> nameToIndex` — passed at construction time from the query's parameter declarations

**Key method:**
- `getValue(ValueResolver, Object)` — casts `object` to `Object[]`, returns `new QueryResultRow(values, nameToIndex)`

### 3. Runtime builder changes

**File:** `DrlxRuleAstRuntimeBuilder.java`, in `buildLhs()` around lines 472-494.

After building the QueryElement and binding individual output variables, add:

```
if (patternIr.bindName() != null) {
    // Build name→index map from target query's parameter declarations
    Map<String, Integer> nameToIndex = new LinkedHashMap<>();
    Declaration[] queryParams = targetQuery.getParameters();
    for (int i = 0; i < queryParams.length; i++) {
        nameToIndex.put(queryParams[i].getIdentifier(), i);
    }
    
    // Create QueryResultRowReader
    QueryResultRowReader reader = new QueryResultRowReader(nameToIndex);
    
    // Add Declaration for bindName on the result pattern
    Pattern resultPattern = queryElement.getResultPattern();
    Declaration decl = new Declaration(patternIr.bindName(), reader, resultPattern);
    resultPattern.addDeclaration(decl);
    
    // Register in boundVariables as QueryResultRow type
    boundVariables.put(patternIr.bindName(),
        new BoundVariable(patternIr.bindName(), QueryResultRow.class, resultPattern, decl));
}
```

### 4. No parsing changes

The existing grammar and `DrlxToRuleAstVisitor` already produce the correct `PatternIR` for `var t : /trusts(a, var b)`:
- `bindName = "t"`
- `entryPoint = "trusts"`
- `positionalArgs = ["a", "var b"]`

The runtime builder currently ignores `bindName` on query element paths. This change makes it meaningful.

## Deferred

- **Handle access** (`t.handles[0]`, `t.handles().a`) — filed as #82. ReadAccessor doesn't receive the tuple, so exposing FactHandles requires a different mechanism.

## Test Plan

1. **Named access** — `var t : /queryName(a, var b)`, verify `t.a` and `t.b` resolve to correct values in an eval constraint
2. **Indexed access** — verify `t[0]` and `t[1]` return same values as `t.a` and `t.b`
3. **Mixed eval** — `allEqual(t.a, t[0])` passes, confirming equivalence
4. **objects() method** — call `t.objects()` in consequence, verify it returns the full Object array
5. **Iteration** — verify QueryResultRow is iterable
