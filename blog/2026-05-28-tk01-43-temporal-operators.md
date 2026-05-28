---
layout: post
title: "#43 — temporal operators close out epic #26"
date: 2026-05-28
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [temporal-operators, cep, allen-interval, drools-api]
---

# #43 — temporal operators close out epic #26

Epic #26 had one item left: temporal operators for CEP. The 13 Allen interval operators — `after`, `before`, `coincides`, `during`, and nine more — let rules correlate events by timing. The DRLXXXX spec shows the syntax as `this after a` inside pattern brackets, optionally parameterised with duration ranges.

I brought Claude in for the design. The core question was how to represent temporal constraints in a system where all constraints are currently MVEL3 expression strings. `this after a` is not a valid MVEL3 expression — there's no infix word-operator in Java or MVEL3. So we needed grammar-level support.

## A generic grammar rule

Rather than adding 13 lexer keywords, we added a single `customConstraint` rule:

```antlr
customConstraint
    : (THIS | identifier) NOT? customOp expression
    ;
```

The operator name stays an `identifier`, validated at visitor level. This keeps the grammar extensible — fuzzy operators (#66) can reuse the same rule later without touching the lexer. Using `THIS` on the left side makes it unambiguous: `this after a` can never parse as a valid Java expression.

## Two Interval classes and a missing interface

The runtime design reuses drools' `TemporalPredicate` implementations (`AfterPredicate`, `BeforePredicate`, etc.) — already in the classpath via `drools-canonical-model`. A new `DrlxTemporalConstraint` extracts timestamps from `EventHandle` and delegates evaluation.

Claude caught a gap during the drools API review: the RETE builder's `BuildUtils.gatherTemporalRelationships()` checks `constraint instanceof IntervalProviderConstraint` before extracting temporal intervals. Without implementing this interface, event expiration and negative-pattern delay timers would silently break. The fix was straightforward — implement `IntervalProviderConstraint` and convert between two `Interval` types (model layer vs base layer, same fields, different packages).

The predicate constructors also vary per operator. Six operators (`finishes`, `finishedby`, `meets`, `metby`, `starts`, `startedby`) only accept zero or one parameter — passing two throws. `CoincidesPredicate` has no no-arg constructor at all. The factory validates per-operator and gives clear errors.

## What landed

The `Withdrawal` test domain was already annotated `@Role(Type.EVENT)` from the window work, so the integration tests slot in naturally — pseudo-clock, stream mode, insert events at different times, assert firing counts.

| Item | Detail |
|------|--------|
| Commits | 8 on `main` (316e0cf..297d04d) |
| New files | `DrlxTemporalConstraint`, `TemporalPredicateFactory`, 3 test classes |
| Tests added | 17 (5 visitor, 6 factory, 6 integration) |
| Issue | #43 closed |
| Epic | #26 fully complete — all sub-issues closed |
