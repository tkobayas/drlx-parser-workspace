# #53 — Custom User-Imported Accumulate Functions

**Issue:** https://github.com/tkobayas/drlx-parser/issues/53
**Parent epic:** #26
**Follows:** #39 (Accumulate v1), #48 (MVEL3-backed extractors)
**DRLXXXX reference:** §Accumulate — "Parent classes, that contain the functions, can be imported and then the functions used within that class."

## Summary

Enable user-defined accumulate functions via class-qualified names. The user imports a container class that exposes `AccumulateFunction` instances as `public static final` fields. Functions are invoked with the `Container.fieldName(expr)` syntax that the DRLXXXX spec shows.

## Syntax

```
import com.example.acc.Funcs;

rule R {
    var p : /persons,
    var s = Funcs.stddev(p.age),
    var v = Funcs.variance(p.age),
    do { ... }
}
```

No grammar changes — the parser already accepts qualified names via `qualifiedName()` and preserves them verbatim in `AccumulatorIR.functionName()`.

## Container Class Contract

The imported class exposes accumulate functions as `public static final` fields of type `AccumulateFunction`:

```java
public class Funcs {
    public static final AccumulateFunction stddev = new StddevAccumulateFunction();
    public static final AccumulateFunction variance = new VarianceAccumulateFunction();
}
```

Each `AccumulateFunction` implementation is stateless — all per-evaluation state lives in the context object produced by `createContext()`. Sharing the instance across rules is safe.

## Resolution Design

### Approach: split resolution

Built-in functions (no dot in name) continue through `AccumulateFunctionRegistry`. Qualified names (contains dot) are resolved by the runtime builder, which has access to `TypeResolver`.

### Resolution algorithm (qualified names)

Given function name `Funcs.stddev`:

1. Split on the last dot → class part `Funcs`, field part `stddev`.
2. Resolve `Funcs` via `TypeResolver` (which has the compilation unit's imports).
3. Look up a `public static` field named `stddev` on the resolved class.
4. Read the field value; validate it is an `AccumulateFunction` instance.
5. Call `getResultType()` on the instance for result type inference (`var` bindings).
6. Custom functions always require exactly 1 argument.

### Error messages

| Condition | Message |
|-----------|---------|
| Class not found via imports | `"cannot resolve accumulate function class 'Funcs' — ensure it is imported"` |
| No matching static field | `"class 'Funcs' has no static AccumulateFunction field named 'stddev'"` |
| Field exists but wrong type | `"field 'Funcs.stddev' is not an AccumulateFunction"` |

## Code Changes

### `AccumulateFunctionRegistry`

Remove the qualified-name rejection (`if (functionName.contains("."))` block) from `resolve()`. Callers now branch before calling the registry, so this code is never reached. The registry stays as a static map of built-in names — `resolve()` simply looks up unqualified names and throws "unknown accumulate function" for anything not in the map.

### `DrlxToRuleAstVisitor`

One change: the inline-from validation at ~line 114. Currently calls `AccumulateFunctionRegistry.resolve(functionName)` to check `acceptsZeroArgs()`, which rejects qualified names.

Fix: if `functionName` contains a dot, skip the `acceptsZeroArgs` check. Custom functions always require arguments, so the `finalDotIdent` extractor path is always valid for them.

```java
if (finalDotIdent != null && !functionName.contains(".")) {
    AccumulateFunctionRegistry.Resolution resolved =
            AccumulateFunctionRegistry.resolve(functionName);
    if (resolved.acceptsZeroArgs()) {
        throw new RuntimeException(...);
    }
}
```

### `DrlxRuleAstRuntimeBuilder`

Three methods call `AccumulateFunctionRegistry.resolve()`:

1. **`buildSingleAccumulator()`** — single-pattern source
2. **`buildSingleAccumulatorMulti()`** — multi-pattern source
3. **`resultClassFor()`** — result type for var-bindings

All three gain a branching check: if function name contains a dot, resolve via the custom path.

**Helper method:** Extract a common `resolveFunction(String functionName, TypeResolver typeResolver)` that returns a record holding either:
- A `Class<? extends AccumulateFunction>` (built-in, instantiate via no-arg constructor), or
- A ready-made `AccumulateFunction` instance (custom, read from static field)

Plus the result type and acceptsZeroArgs flag.

**Parameter threading:** `buildSingleAccumulator()`, `buildSingleAccumulatorMulti()`, and `resultClassFor()` don't currently receive `TypeResolver`. Add it as a parameter — their caller `buildAccumulatePattern()` already has it.

**Instantiation difference:**
- Built-in: `resolved.functionClass().getDeclaredConstructor().newInstance()` (unchanged)
- Custom: use the `AccumulateFunction` instance directly from the static field (stateless, safe to share)

## Test Plan

1. **Replace `rejectsQualifiedName`** unit test in `AccumulateFunctionRegistryTest` — the registry no longer sees qualified names, so this test becomes irrelevant. Remove it.
2. **Replace `qualifiedFunctionNameRejected`** integration test in `AccumulateTest` with a positive test: a rule using `Funcs.stddev(p.age)` where `Funcs` is a test container class.
3. **Error case tests:** class not found, field not found, field wrong type.
4. **Multi-accumulate test:** two custom functions from the same container in one rule.
5. **Mixed test:** built-in `avg(p.age)` and custom `Funcs.stddev(p.age)` in the same rule.
6. **Result type inference:** `var s = Funcs.stddev(p.age)` infers the type from `getResultType()`.

## Out of Scope

- Unqualified custom function names (e.g., `stddev(p.age)` without a class prefix) — all custom functions require the `Container.name` qualified syntax.
- `import acc` DRL syntax — DRLX uses standard Java imports, not Drools' `import acc` mechanism.
- Annotation-based discovery (e.g., `@AccumulateFunction(name = "stddev")`) — not needed; static fields are sufficient.
- Zero-arg custom accumulate functions — custom functions always require exactly 1 argument.
