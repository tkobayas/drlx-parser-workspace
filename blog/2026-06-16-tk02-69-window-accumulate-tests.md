---
layout: post
title: "#69 — window + accumulate: twelve tests, zero production changes"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [windows, accumulate, cep, #69]
---

# #69 — window + accumulate: twelve tests, zero production changes

I expected this one to need grammar or visitor work. The issue description said "requires combining window AST nodes with accumulate AST nodes," which sounded like plumbing. Claude traced the code paths before we started and came back with a different read: the pipeline already handles it.

## The pipeline that was already there

The argument was simple. `boundOopath` already has `windowFilter?` as an optional suffix. Both `rulePattern` (inline form) and `accSource` (acc keyword form) use `boundOopath`. So both parse correctly without any grammar changes. The visitor's `buildPatternFromBoundOopath()` extracts window info into `PatternIR` fields — `windowType` and `windowParameter` — regardless of context. And `buildAccumulatePattern()` in the runtime builder calls `buildPattern()` for its source, which applies `SlidingTimeWindow` or `SlidingLengthWindow` behaviours unconditionally:

```java
if (parseResult.windowType() != null) {
    switch (parseResult.windowType()) {
        case "time" -> pattern.addBehavior(
                new SlidingTimeWindow(TimeUtils.parseTimeString(parseResult.windowParameter())));
        case "length" -> pattern.addBehavior(
                new SlidingLengthWindow(Integer.parseInt(parseResult.windowParameter())));
    }
}
```

The window gets attached to the source pattern of the accumulate. Drools handles this natively — `AccumulateCepTest` in drools already tests `accumulate(Pattern() over window:time(...), function(...))`. Our builder produces the same runtime structures.

## Proving it with tests

We added eight visitor-level tests confirming the IR structure: inline time/length, acc keyword time/length, multiple accumulators with window, custom 3-param acc with window, and constraint-before-window with accumulate. Every test parses a DRLX rule and asserts the `PatternIR` inside the `AccumulatePatternIR` carries the correct `windowType` and `windowParameter`. All passed on first run.

Four session-level tests went into a new `WindowAccumulateTest.java`. These exercise the full pipeline — parse, compile, fire with real CEP windowing — using `WithdrawalUnit` with a pseudo clock for time windows. The core assertion: insert events, advance the clock past the window boundary, insert one more, fire, and confirm the accumulated average reflects only events within the window.

```
var w : /withdrawals | time[5s],
var avgAmount = avg(w.amount),
do { results.add(avgAmount); }
```

Both inline and acc keyword forms produce identical results at runtime. The length window tests verify eviction — five events into a `length[3]` window yields an average of the last three.

## What landed

| Metric | Count |
|--------|-------|
| Visitor-level tests added | 8 |
| Session-level tests added | 4 |
| Production lines changed | 0 |
| Total test count | 473 |

Issue #69 closed. Two issues remain in epic #79: #72 (named windows, spec marked TODO) and the de-prioritised #71 (propagation delay).
