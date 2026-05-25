# #58 Recursive Queries (Transitive Closure) — Design Spec

**Issue:** #58
**Epic:** #26
**Date:** 2026-05-25

## Problem

Queries cannot reference themselves in their own LHS. The transitive closure
pattern from the DRLXXXX spec is not supported:

```drlx
rule Trusts(Object a, Object b) {
    or(/trusts(a, b),
       and(/trusts(a, var z),
           /trusts(z, b))
    )
}
```

The cause: `buildQuery()` passes `Map.of()` as the query registry to
`buildLhs()`, so no query lookups succeed during query LHS compilation.

## Scope

- **Self-recursion only** — a query referencing itself via its own entry point.
- Cross-query references (A→B→A) are out of scope and untested.
- No grammar changes. No new Drools APIs. No new classes.

## Design

### Compilation change

In `DrlxRuleAstRuntimeBuilder.build()`, the first-pass query loop currently:

1. Calls `buildQuery()` which internally creates QueryImpl and compiles LHS
   with an empty query registry
2. Registers the returned QueryImpl in `queryRegistry`

Change to:

1. Compute `entryPointName` and create `QueryImpl` up front
2. Register it in `queryRegistry` **before** LHS compilation
3. Call `buildQuery()` with the pre-created QueryImpl and the `queryRegistry`
4. `buildQuery()` passes `queryRegistry` to `buildLhs()` instead of `Map.of()`

This allows `buildLhs()` to find the query being compiled via the standard
`queryRegistry.get(entryPoint)` lookup path — the same path used for query
invocations from regular rules.

### Method signature change

`buildQuery()` changes from:

```java
private QueryImpl buildQuery(RuleIR parseResult, TypeResolver typeResolver,
                             Map<String, Class<?>> entryPointTypes, Class<?> unitClass)
```

to:

```java
private void buildQuery(QueryImpl query, RuleIR parseResult, TypeResolver typeResolver,
                        Map<String, Class<?>> entryPointTypes, Class<?> unitClass,
                        Map<String, QueryImpl> queryRegistry)
```

Returns void since the QueryImpl is pre-created by the caller.

### Self-reference disambiguation heuristic

Within a query body, `/trusts(args)` where `trusts` matches BOTH the current
query (self-reference via `queryRegistry`) AND a DataStore entry point creates
an ambiguity: should it compile to a **Pattern** (positional fact match — base
case) or a **QueryElement** (recursive query call)?

Old DRL disambiguates by name: `Trust(;a, b)` (class name → Pattern) vs
`trusts(;z, b)` (query name → QueryElement). DRLXXXX unifies both under
`/trusts(...)`.

**Heuristic:** For a self-referencing `/trusts(args)` in a query body, examine
each **input** positional argument (excluding `var` outputs and unbound
identifiers which are implicit outputs):

- If **all** input variable arguments are query definition parameters →
  compile as **Pattern** (positional match on the DataStore's element type).
  This is the base case — matching facts directly.
- If **any** input variable argument is NOT a query parameter (i.e., it was
  bound locally within the query body) → compile as **QueryElement**
  (recursive query call).

Applied to the spec example:

```
rule Trusts(Object a, Object b) {       // params: a, b
    or(/trusts(a, b),                    // a=param, b=param → Pattern (base case)
       and(/trusts(a, var z),            // a=param, z=output → Pattern (intermediate)
           /trusts(z, b)))              // z=local, b=param → QueryElement (recursive)
}
```

This produces the same RETE structure as old DRL:

```
OR:
  Pattern(Trust, positional [a, b])           ← base case fact match
  AND:
    Pattern(Trust, positional [a, z])         ← intermediate fact match, bind z
    QueryElement(trusts, [z, b])              ← recursive call
```

**Note:** This heuristic may be revisited in the future if more complex
recursive query patterns are needed. It is correct for all standard transitive
closure patterns (`or(base, and(one_hop, recursive))`).

### Runtime

No changes needed. Drools' `PhreakQueryNode` and `PhreakQueryTerminalNode`
already handle recursive query propagation via stack-based evaluation. The
`QueryElement` produced by `buildLhs()` is the same structure whether the
target query is self or another query.

## Test

A transitive closure test in `QueryTest.java`:

- **POJO:** `Trust` with `Object a` and `Object b` fields
- **Query:** `Trusts(Object a, Object b)` with
  `or(/trusts(a, b), and(/trusts(a, var z), /trusts(z, b)))`
- **Rule:** `R1` invokes `/trusts(a, var b)` and collects results
- **Data:** Insert trust chain A→B→C→D (3 Trust facts)
- **Assert:** From A, transitive results are {B, C, D}

## Files changed

| File | Change |
|------|--------|
| `DrlxRuleAstRuntimeBuilder.java` | Pass queryRegistry through to buildQuery/buildLhs; register query before LHS compilation |
| `QueryTest.java` | Add recursive query transitive closure test |
| Trust POJO (test) | New test helper class |
