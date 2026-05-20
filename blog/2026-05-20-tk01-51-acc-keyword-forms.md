---
layout: post
title: "#51 — acc() keyword forms: spec to code in one session"
date: 2026-05-20
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, accumulate, acc, mvel3, antlr4]
---

# #51 — acc() keyword forms: spec to code in one session

The spec for `acc()` keyword forms came out of the previous session — six rounds of review, all findings addressed, ready for implementation. I wanted to see how fast the pipeline could move when the spec was already solid.

Eight commits later, three accumulate arities work end-to-end: 2-param (function sugar), 3-param (custom with optional reverse), and 5-param (custom with explicit reverse). I brought Claude in at the start for the implementation plan, and we executed it inline.

## Ten files, one pipeline

The spec defined the full pipeline: grammar → IR → visitor → protobuf → runtime → compiler → builder → tests. We walked through it in order:

The grammar adds eight new rules anchored by `accKeywordItem`. `acc` is contextual — parsed as an identifier, validated at visitor level — so it stays available as a variable name elsewhere:

```antlr
accKeywordItem
    : identifier '(' accSource ',' accBody ')'
    ;

accBody
    : accFunctionList
    | accInitVars ',' accActionBlock ',' accResultBinding
    | accInitVars ',' accActionBlock ',' accActionBlock ',' accResultBinding
    ;
```

The 2-param form (`acc(source, functions)`) lowers to the existing `AccumulatePatternIR` — no new runtime code needed. The 3/5-param forms introduce `CustomAccumulateIR` and `InitVarIR` records on the IR side, and `DrlxCustomAccumulator` on the runtime side.

`DrlxCustomAccumulator` uses MVEL3's map-based evaluator pattern. A `Map<String, Object>` carries init vars across the accumulate lifecycle. Action and reverse blocks get the source fact injected under its binding name, execute, then remove it in a `finally` block. The result expression sees only init vars — the source binding is gone by `getResult()` time.

## Fifteen validations before the code runs

The visitor carries the bulk of the semantic checking. Fifteen validation paths, each with a test:

- `var` rejected in init vars and result bindings (explicit types required for `Declaration.of()`)
- Init var names checked against source binding (collision would cause `accumulate()` to overwrite the holder)
- Duplicate init var names rejected
- Literal initializer type compatibility (widening ok, narrowing rejected, null only for reference types)
- Complex initializers like `new Foo()` rejected — literals only for now
- Paired `(action, reverse)` rejected in 5-param form (it has separate positions)
- Source binding reference in result expression rejected
- Outer-binding references caught at runtime-builder time with a pointer to #54

Multi-declarators (`int a = 0, b = 1;`) split into separate `InitVarIR` entries. Missing initializers map to Java type defaults.

## The `Infinity` that found an MVEL3 bug

The 5-param avg test was the first to use multi-statement braced blocks:

```
acc(var p : /persons,
    { int count = 0; int total = 0; },
    { total += p.age; count++; },
    { total -= p.age; count--; },
    double avgAge = (double) total / count)
```

Expected: `40.0`. Got: `Infinity`.

`Infinity` from `(double) total / count` means `count` is zero — the increments aren't sticking. We wrote a diagnostic test isolating the four assignment forms in MVEL3's map evaluator:

| Operator | Map write-back | Result |
|---|---|---|
| `count = count + 1` | works | 1 |
| `count += 1` | works | 1 |
| `count++` | broken | 0 |
| `++count` | broken | 0 |

The root cause is in `MVELToJavaRewriter.rewriteNode()`. The `AssignExpr` case wraps assignments in `context.put("name", name = value)` for map write-back. The `UnaryExpr` case handles BigDecimal literal folding and bitwise complement — but has no write-back logic for `PREFIX_INCREMENT`, `POSTFIX_INCREMENT`, `PREFIX_DECREMENT`, or `POSTFIX_DECREMENT`. The increment executes on the local variable; the map never sees it.

The workaround is `count = count + 1` instead of `count++`. The tests use this form. The MVEL3 fix is straightforward — add `context.put()` wrapping in the `UnaryExpr` case — but it's a separate issue.

## What landed

| Metric | Count |
|--------|-------|
| Commits | 8 |
| New/modified source files | 7 |
| New test methods | 23 (16 visitor + 5 integration + 2 protobuf round-trip) |
| Total test suite | 255 tests, 0 failures |

The `acc()` keyword is the first DRLX syntax that compiles user-written MVEL3 blocks (not just expressions) through the batch compiler. The `EvaluatorSink` pattern — three inner classes (`ActionSink`, `ReverseSink`, `ResultSink`) each binding one slot on the parent accumulator — extends naturally from how constraints and consequences already work.

Issue #51 is ready to close. The MVEL3 `++`/`--` bug gets its own upstream issue.
