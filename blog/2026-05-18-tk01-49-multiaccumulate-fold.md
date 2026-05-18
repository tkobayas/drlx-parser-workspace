---
layout: post
title: "#49 — folding to one MultiAccumulate (with a Pattern API surprise)"
date: 2026-05-18
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, accumulate, multi-accumulate, java]
---

# #49 — folding to one MultiAccumulate (with a Pattern API surprise)

#49 was billed in the issue as a runtime-shape optimisation — semantically identical to v1, just fewer nodes. Drools' `KiePackagesBuilder.createAccumulate` was the template: when accumulator count exceeds 1, emit one `MultiAccumulate` over one source pattern instead of N×`SingleAccumulate` with N cloned source patterns. The two builders Drools ships (executable-model and legacy MVEL) both follow this pattern, so the convention was settled before I started.

What it took was 7 commits, an extension to the `BoundVariable` record, a widening of `collectPatternTypes`, and two rounds of Codex review catching one structural gap and one type-system mistake that would both have shipped silently.

## The reference shape was the easy part

Drools' `KiePackagesBuilder.createAccumulate` is the canonical multi-function lowering:

```java
if (accFunctions.length == 1) {
    return new SingleAccumulate(source, requiredDeclarationList.toArray(...), accumulators[0]);
}
return new MultiAccumulate(source, new Declaration[0], accumulators,
                           accumulators.length + (isGroupBy ? 1 : 0));
```

The wrapping result Pattern is `Object[].class`; declarations are per-slot `new ArrayElementReader(selfReader, i, perFnResultType)`. The MVEL textual DRL path (`MVELAccumulateBuilder.isMultiFunction()`) follows the same shape with a different `requiredDeclarations` convention — model passes `new Declaration[0]`, the textual paths propagate accumulated decls. For #49 the model-path convention applied because outer-binding refs (`sum(p.age * q.factor)`) live in a separate follow-up — filed earlier in the same session as #54, since the MVEL3 foundation #48 just landed makes that one a small delta.

The plan was straightforward: a `buildSingleAccumulator(acc, srcClass, srcBindingName)` helper shared by both paths; dispatch on N inside `buildAccumulatePattern`; N=1 stays on `SingleAccumulate` with v1's typed result Pattern; N>1 emits `MultiAccumulate` and a new `wrapMultiResultPattern`. The source-pattern clone in v1's loop disappears — the loop is gone.

## The binding record that had to grow

Codex flagged the first finding at spec review:

> High: the structural tests expect each per-binding Declaration reachable via `pattern.getDeclaration()`, but `BoundVariable` only stores Pattern. For an unnamed multi-result wrapper, `Pattern.getDeclaration()` returns null even though `getDeclarations()` contains minAge, maxAge, etc.

The DRLX binding record had always been `(name, type, pattern)`. Every downstream lookup that needed the Declaration went through `bv.pattern().getDeclaration()`. That works when each Pattern carries exactly one declaration. The multi-wrap Pattern is unnamed and carries N — `getDeclaration()` returns null and the bindings get silently dropped from required-decl arrays and consequence type maps. The `multiFunctionMinMaxAvg` test's consequence (`results.add(minAge); results.add(maxAge); results.add(avgAge)`) would have failed to type-check inside MVEL3.

The fix was to extend the record:

```java
public record BoundVariable(String name, Class<?> type,
                            Pattern pattern, Declaration declaration) {}
```

And to widen `collectPatternTypes` to iterate every declaration on each Pattern, typing each via `declaration.getExtractor().getExtractToClass()` rather than the Pattern's `getObjectType()`. Equivalent for every existing single-decl Pattern (verified against `SelfReferenceClassFieldReader` and `ArrayElementReader` — both return the construction class); correct for the new multi-wrap.

## `addDeclaration` overwrites the map but not the field

Codex's plan review caught a subtler variant of the same gravity:

> Medium: Task 1 keeps using `resultPattern.getDeclaration()` for the single-result accumulate binding. In `wrapResultPattern`, the constructor creates one declaration, then `addDeclaration(...)` installs another declaration with the same identifier and the intended `SelfReferenceClassFieldReader`. `Pattern.getDeclaration()` returns the constructor declaration, not necessarily the one in `getDeclarations()`.

Reading `Pattern.java` confirmed it. The constructor with an identifier seeds `this.declaration = new Declaration(name, PatternExtractor, ...)` AND puts the same declaration in `this.declarations`. Later `addDeclaration(new Declaration(name, SelfReferenceClassFieldReader, ...))` calls `this.declarations.put(name, newDecl)` — overwriting the map entry — but does NOT update `this.declaration`. So for the same identifier:

- `pattern.getDeclaration()` returns the original `PatternExtractor`-backed declaration.
- `pattern.getDeclarations().get(name)` returns the `addDeclaration`'d `SelfReferenceClassFieldReader`-backed one.

The runtime-resolvable one is the second. v1's single-result path had been "working" by luck — for most runtime purposes the two declarations are equivalent — but the moment you assert on `getExtractToClass()` in structural tests, or pass the declaration to anything that cares about the accessor type, they diverge.

The docs say nothing about this. Only `Pattern.java` lines 100-104 and 305-310 tell the story. Submitted to the garden.

## min/max return `Comparable`, not `Integer`

The other plan-review finding was a straight-line type-system mistake. The structural tests asserted `Integer.class` for `min(p.age)`:

```java
assertThat(((ClassObjectType) wrap.getObjectType()).getClassType())
        .isEqualTo(Integer.class);
```

`AccumulateFunctionRegistry` says otherwise:

```java
"min", new Resolution(MinAccumulateFunction.class, Comparable.class, false),
"max", new Resolution(MaxAccumulateFunction.class, Comparable.class, false),
"avg", new Resolution(AverageAccumulateFunction.class, Double.class,     false),
```

Of course — `min` and `max` work on anything `Comparable`, not just numbers. The runtime value of `min(p.age)` is an `Integer`, but the type at compile time has to be the broadest thing the function can return. I'd been carrying the wrong intuition from `multiFunctionMinMaxAvg`'s runtime assertion `[20, 60, 40.0]` — the numbers look like Integers, so I assumed the type was Integer.

(There was also a wrong import in the same draft — `org.drools.base.base.ReadAccessor` when the actual location is `org.drools.base.rule.accessor.ReadAccessor`. The first wouldn't have compiled. Codex caught both findings in one review pass.)

Easy fixes once seen. Easy to ship if not seen — the structural test would have failed at runtime with a confusing class-mismatch rather than at code-review time with "use Comparable.class".

## What landed

| Commit | Subject |
|---|---|
| `c27c80b` | `refactor: carry Declaration on BoundVariable (#49 prep)` |
| `5583879` | `refactor: type every Pattern declaration in collectPatternTypes (#49 prep)` |
| `5f60732` | `refactor: split buildSingleAccumulate into Accumulator + required helpers (#49 prep)` |
| `339c31f` | `test: structural baseline for SingleAccumulate (N=1) shape (#49)` |
| `d36b723` | `feat: fold multi-function accumulate into a single MultiAccumulate (#49)` |
| `1bf6f43` | `test: mixed null/non-null extractor in MultiAccumulate (#49)` |
| `50049af` | `test: MVEL3-compiled extractor per slot in MultiAccumulate (#49)` |

`drlx-parser-core` suite: 213 → 217 green. Reactor BUILD SUCCESS. #49 closed at `d36b723`. Remaining epic-#26 children: #50 (inline-from form), #51 (`acc()` keyword forms), #52 (multi-pattern source via `and(...)`), #53 (custom user-imported functions), #54 (outer-binding extractor refs).
