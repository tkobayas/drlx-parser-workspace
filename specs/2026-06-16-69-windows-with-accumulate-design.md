# #69 — Windows with accumulate

**Issue:** https://github.com/tkobayas/drlx-parser/issues/69
**DRLXXXX reference:** Lines 1060–1068
**Epic:** #79 (Conditional branching & named windows)

## Summary

Verify and test that DRLX correctly combines CEP sliding windows with accumulate patterns. The existing pipeline (grammar → visitor → IR → runtime builder) already appears to support this combination — the work is confirming it end-to-end and adding test coverage.

## DRLX syntax

Two forms, per the DRLXXXX spec:

**Inline form** — window on pattern, accumulate function as a subsequent item:
```
w : /withdrawals | time[5s], a = avg(w),
```

**acc keyword form** — window on the source pattern inside `acc()`:
```
acc(w : /withdrawals | time[5s],
    a = avg(w)),
```

Both produce the same IR structure: an `AccumulatePatternIR` whose source `PatternIR` carries `windowType` and `windowParameter`.

## Why this should already work

The pipeline handles windows and accumulate independently, and their composition is natural:

1. **Grammar** — `boundOopath` has `windowFilter?`. Both `rulePattern` (inline form) and `accSource` (acc keyword form) use `boundOopath`, so both parse correctly.
2. **Visitor** — `buildPatternFromBoundOopath()` extracts window info into `PatternIR` fields. The inline fold logic and `buildAccKeywordItem()` both preserve the `PatternIR` as-is when wrapping into `AccumulatePatternIR`.
3. **Runtime builder** — `buildAccumulatePattern()` calls `buildPattern()` for the source, which applies window behaviors (`SlidingTimeWindow`/`SlidingLengthWindow`) at lines 1724–1731 regardless of context.

## Scope

**In scope:**
- Inline window + accumulate (time and length)
- acc keyword 2-param window + accumulate (time and length)
- acc keyword custom (3-param) with window
- Single and multiple accumulators with window
- Constraint before window with accumulate
- Visitor-level and session-level tests

**Out of scope:**
- Propagation delay (`delay.last(4s)`) — separate issue #71
- Named windows — separate issue #72
- GroupBy with windows — can be its own issue if needed

## Test matrix

### Visitor-level tests

Added to `AccumulateVisitorTest.java`. Each test parses a rule and asserts the IR structure.

| # | Form | Window | Accumulators | Rule snippet |
|---|------|--------|-------------|-------------|
| V1 | Inline | time | single (avg) | `w : /withdrawals \| time[5s], a = avg(w)` |
| V2 | Inline | length | single (avg) | `w : /withdrawals \| length[3], a = avg(w)` |
| V3 | Inline | time | multiple (avg + count) | `w : /withdrawals \| time[5s], a = avg(w), n = count()` |
| V4 | acc 2-param | time | single (avg) | `acc(w : /withdrawals \| time[5s], a = avg(w))` |
| V5 | acc 2-param | length | single (avg) | `acc(w : /withdrawals \| length[3], a = avg(w))` |
| V6 | acc 2-param | time | multiple (avg + count) | `acc(w : /withdrawals \| time[5s], (a = avg(w), n = count()))` |
| V7 | acc custom 3-param | time | custom | `acc(w : /withdrawals \| time[5s], int s, s = s + w.amount, int sum = s)` |
| V8 | Inline with constraint | time | single (avg) | `w : /withdrawals[amount > 100] \| time[5s], a = avg(w)` |

Assertions per test:
- LHS item is `AccumulatePatternIR` (V1–V6, V8) or `CustomAccumulateIR` (V7)
- Source `PatternIR` has correct `windowType` and `windowParameter`
- Accumulator details preserved: function name, bind name, args (V1–V6, V8); init vars, action block, result binding (V7)
- For V8: constraint list on the source pattern is non-empty

### Session-level tests

New file: `WindowAccumulateTest.java`. Each test uses a `@Role(Type.EVENT)` type, inserts events, and asserts accumulated results.

| # | Form | Window | Scenario |
|---|------|--------|----------|
| S1 | Inline | time | Insert events, advance pseudo clock past window, verify avg only includes events within window |
| S2 | Inline | length | Insert more events than window size, verify avg only includes last N events |
| S3 | acc 2-param | time | Same as S1, using acc keyword syntax |
| S4 | acc 2-param | length | Same as S2, using acc keyword syntax |

Session-level tests follow the pattern from `WindowTest.java` (pseudo clock, event insertion) and `AccumulateTest.java` (result verification via globals).

## Error handling

No new error handling. Existing validations cover:
- Non-event type with window → drools rejects at compile time
- Unknown window type → visitor throws `IllegalArgumentException`
- Accumulate without preceding pattern → visitor throws `RuntimeException`

## Drools runtime compatibility

Drools natively supports `accumulate(Pattern() over window:time/length(...), function(...))`. The runtime representation is a `Pattern` with `BehaviorDescr` inside an `AccumulateDescr` — exactly what our pipeline produces. Confirmed by `AccumulateCepTest` and `test_CEP_SimpleLengthWindow.drl` in the drools test suite.

## Implementation approach

1. Write visitor-level tests (V1–V8) → expect all to pass
2. Write session-level tests (S1–S4) → expect all to pass
3. If any test fails, investigate and fix the specific gap
4. Run the full drlx-parser test suite to confirm no regressions
