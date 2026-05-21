---
layout: post
title: "#53 — custom accumulate functions: container classes and qualified names"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, accumulate, custom-functions]
---

# #53 — custom accumulate functions: container classes and qualified names

Accumulate v1 hardcoded five built-in functions — avg, sum, min, max, count. The grammar already accepted qualified names like `Funcs.stddev(p.age)`, but the registry rejected them at compile time with a clear "not yet supported" message. This session opens that gate.

## The container class model

The DRLXXXX spec says "parent classes, that contain the functions, can be imported and then the functions used within that class." I wanted to follow that literally — import a container class, call functions as `Container.fieldName(expr)`. I brought Claude in to brainstorm the resolution model.

The design we landed on: the container exposes `AccumulateFunction` instances as `public static final` fields. Since `AccumulateFunction` is stateless — all per-evaluation state lives in the context object — sharing the field instance across rules is safe.

```java
public class Funcs {
    public static final AccumulateFunction stddev = new StddevAccumulateFunction();
    public static final AccumulateFunction variance = new VarianceAccumulateFunction();
}
```

The rule syntax reads naturally:

```
import com.example.acc.Funcs;

rule R {
    var p : /persons,
    var s = Funcs.stddev(p.age),
    var v = Funcs.variance(p.age),
    do { ... }
}
```

No grammar changes needed — the parser already preserves qualified names verbatim in the IR.

## Split resolution: built-in vs. custom

The key architectural choice was where to resolve qualified names. `AccumulateFunctionRegistry` is a static map of five built-ins — bolting import-aware resolution onto it would have meant threading `TypeResolver` into a class that didn't need it.

Instead we split: unqualified names (no dot) go through the registry as before. Qualified names (contains a dot) are resolved by the runtime builder, which already has a `TypeResolver` populated with the compilation unit's imports. The resolution algorithm splits on the last dot, resolves the class part via imports, then reads the static field via reflection. `getResultType()` on the `AccumulateFunction` instance handles `var` type inference.

Three methods in the runtime builder needed the new path — `buildSingleAccumulator()`, `buildSingleAccumulatorMulti()`, and `resultClassFor()`. We extracted a common `resolveFunction()` helper that returns a `ResolvedFunction` record holding the instantiated function, result type, and zero-arg flag. The built-in path instantiates via no-arg constructor; the custom path reads the instance directly from the static field.

## What landed

| Metric | Value |
|--------|-------|
| Commits | 6 |
| New test methods | 6 (3 positive, 3 error cases) |
| Scenarios | single custom function, multi-accumulate (two custom), mixed built-in + custom, class not found, field not found, field wrong type |

The accumulate subsystem now has 43 tests. All accumulate issues from epic #26 are complete — what remains is Group By, Queries, and the medium/low priority non-accumulate features.
