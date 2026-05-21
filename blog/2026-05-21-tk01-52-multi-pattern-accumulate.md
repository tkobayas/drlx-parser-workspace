---
layout: post
title: "#52 — multi-pattern accumulate: joined sources in acc()"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, accumulate, subnetwork, mvel3]
---

# #52 — multi-pattern accumulate: joined sources in acc()

The previous two sessions wrote and reviewed the spec for multi-pattern accumulate sources — `acc(and(p : /persons, o : /orders[customerId == p.age]), sum(o.amount))`. The key finding during spec review was that `handle.getObject()` returns a `SubnetworkTuple` in multi-pattern context, not a domain fact. I brought Claude in to execute the 11-task implementation plan, and we walked through it inline.

The feature widens the accumulate pipeline from single-source extraction (`handle.getObject()` → domain fact) to tuple-based extraction (`innerDecls` → multiple bound facts from a joined source).

## Widening the IR without breaking the pipeline

The core change is type-level: `AccumulatePatternIR.source` and `CustomAccumulateIR.source` go from `PatternIR` to `LhsItemIR`. A single-source accumulate still passes a `PatternIR`; a multi-pattern source passes a `GroupElementIR(AND, [...])`.

The ripple is mechanical but touches every layer. Protobuf schema fields change from `PatternParseResult` to `LhsItemParseResult`, serialization switches from `patternToProto()`/`patternFromProto()` to the existing `toProtoLhs()`/`fromProtoLhs()` pair. Test files that called `.source().bindName()` need casts to `(PatternIR)` — the sealed interface no longer exposes pattern-specific methods directly.

The runtime builder branches on `instanceof`: single source builds a `Pattern` as before; multi-source builds an AND `GroupElement` via `buildLhs()` and sets `multiSource = true`. Drools creates the subnetwork and `RightInputAdapterNode` automatically when it sees a `GroupElement` source — no engine changes needed.

## Two extraction paths in the accumulators

`DrlxLambdaAccumulator` gains a `multiSource` flag and a second constructor accepting a `DrlxValueExtractor` instead of a `Function<Object, Object>`. In `accumulate()`, the multi-source path iterates `innerDecls` to build a `Map<String, Object>` from the tuple, then passes it to `applyMulti()`. The single-source path is unchanged — `handle.getObject()` as before.

`DrlxCustomAccumulator` does the same for custom blocks. Action and reverse evaluators get source bindings injected from `innerDecls` instead of from `handle.getObject()`. The compiler adds multi-declaration overloads that build MVEL3 `Declaration[]` arrays from the source scope:

```java
public DrlxValueExtractor createValueExtractor(String argExpr,
                                               Map<String, BoundVariable> sourceScope) {
    // ...
    org.mvel3.transpiler.context.Declaration<?>[] decls = sourceScope.entrySet().stream()
            .map(e -> Declaration.of(e.getKey(), e.getValue().type()))
            .toArray(Declaration[]::new);
    // ...
}
```

The `sourceScope` is `innerScope` minus `outerScope` — only bindings introduced by the source patterns. Outer bindings must not leak into the MVEL3 compilation context because they aren't available in the runtime extraction map built from `innerDecls`.

## The single-child AND that broke at rete level

The edge-case test — `acc(and(var p : /persons), avg(p.age))` — hit a NPE: `Cannot invoke "TupleImpl.getIndex()" because "entry" is null`.

Drools rete builder unwraps single-child AND groups. When `GroupElement.isAnd()` and `children.size() == 1`, no subnetwork is created — the single pattern is inlined directly. But we'd set `multiSource = true` and the accumulator tried tuple-based extraction via `innerDecls`, which were empty.

The fix: detect single-child AND at IR builder level and unwrap it to a plain `PatternIR` path before building the runtime objects. This mirrors what Drools does at rete level and avoids the mismatch.

## What landed

| Metric | Value |
|--------|-------|
| Modified source files | 13 |
| New test methods | 9 (1 parser + 3 visitor + 1 protobuf + 4 integration) |
| End-to-end scenarios | count, sum single-binding, sum cross-binding, custom 3-param, custom 5-param with reverse, single-child AND edge case |

The cross-binding test is the critical one — `sum(p.age * o.amount)` exercises multi-declaration MVEL3 compilation with bindings from both source patterns, tuple-based extraction at runtime, and correct scope isolation. Issue #52 is ready to close.
