---
layout: post
title: "#50 — inline-from: the shorthand that was secretly boring"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, accumulate, inline-from, antlr4]
---

# #50 — inline-from: the shorthand that was secretly boring

The bound accumulate form from #39/#48/#49 works but is verbose — you have to declare a source binding even when the oopath and the accumulator are one thought:

```
var p : /persons,
var avgAge = avg(p.age),
```

The DRLX spec calls the alternative "inline-from": `var avgAge = avg(/persons.age)`. The oopath lives inside the function call; the visitor desugars it into a synthetic source pattern (`$inline0`) and a rewritten extractor (`$inline0.age`). Downstream of the visitor, the IR shape is identical to what the bound form produces.

I expected the grammar to be the tricky part. It wasn't.

## The grammar that didn't fight back

The new `inlineFromOopath` rule slots in as the first alternative of `accumulateCall`:

```antlr
accumulateCall
    : qualifiedName '(' inlineFromOopath ')'
    | qualifiedName '(' (expression (',' expression)*)? ')'
    ;

inlineFromOopath
    : oopathExpression ('.' identifier)?
    ;
```

The spec predicted a possible ANTLR ambiguity warning — Java's `expression` never starts with `/` (division is infix), so the adaptive LL(\*) should dispatch cleanly on the first token. It did. No warning, no semantic predicate needed. The spec's fallback plan (`{_input.LT(1).getType() == DIV}?`) stayed in the drawer.

## Per-item, not per-source: why inline-from doesn't fold

Two adjacent inline-from items on the same source — `min(/persons.age), max(/persons.age)` — produce two separate `SingleAccumulate`s, each with its own synthetic source pattern (`$inline0`, `$inline1`). This is wasteful: two joins on `/persons` instead of one.

I chose this deliberately. Drools' `MVELAccumulateBuilder.isMultiFunction()` at line 144 folds functions inside one `accumulate(...)` block, but never merges two separate `from accumulate(...)` clauses even when their source patterns are textually identical. The #49 multi-function fold (which produces `MultiAccumulate`) maps to the DRL in-block case: the user explicitly groups functions under one declared source. Inline-from items are syntactically separate — each is its own `accumulateItem` in the grammar — so they map to the separate-clause case. Merging by source equivalence is an optimisation for a later issue.

## The visitor was 40 lines of dispatch

The `buildRule` loop gains one new branch: when `call.inlineFromOopath() != null`, flush any pending pattern, synthesise a `PatternIR` with a `$inlineN` binding, build an `AccumulatorIR` with the rewritten arg, and emit an `AccumulatePatternIR` immediately. Two new helper overloads — `buildPatternFromOopath(ctx, syntheticBindName)` and `buildAccumulator(ctx, srcBindName, finalDotIdent)` — are thin wrappers around the existing extraction logic.

The runtime builder sees nothing new. `$inline0` is a valid Drools binding character (`$` follows the same convention legacy DRL uses for synthetic declarations). `DrlxLambdaCompiler.createValueExtractor` compiles `$inline0.age` against the source class the same way it compiles `p.age`.

One guard lives at visitor time: `count(/persons.age)` gets rejected because `count` is a zero-arg function and the `.age` extractor would be silently dropped by `buildSingleAccumulator`. The error message points at the fix: `use 'count(/persons)' instead`.

## Codex caught two plan bugs before they compiled

The spec went through two Codex review rounds during the previous session. The implementation plan got its own Codex pass, which caught two things:

The structural tests in the plan called `single.getSourcePattern()` — an API that doesn't exist. `SingleAccumulate` inherits `Accumulate.getSource()`, which returns the source `RuleConditionElement`. The correct cast is `(Pattern) single.getSource()`. This would have been a compile error, not a runtime bug, but it would have stalled the implementation.

The compose-with-bound-pattern test asserted `hasSize(4)` plus `contains("Alice", "Bob", 2L)` — which doesn't prove there are two count results. `containsExactlyInAnyOrder("Alice", 2L, "Bob", 2L)` is the right assertion: it proves both firings produce name + count pairs.

## What landed

| | |
|---|---|
| Commits | 6 on `main` (`c9ba325`..`713d55f`) |
| Grammar | `inlineFromOopath` rule, first alt of `accumulateCall` |
| Visitor | ~40 lines of new dispatch in `buildRule`, 2 helper overloads |
| IR / runtime | unchanged |
| Tests | 10 new (3 parse, 5 behavioural, 2 structural, 3 negative) — 226 total |
| Issue | #50 under epic #26 |
