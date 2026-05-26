# #68 Constraint Before/After Window — Design Spec

**Issue:** https://github.com/tkobayas/drlx-parser/issues/68
**DRLXXXX reference:** Lines 1049-1057
**Depends on:** #67 (basic length/time windows) — completed
**Date:** 2026-05-26

## Problem

DRLX supports two semantically distinct ways to combine constraints with
sliding windows. The grammar, visitor, and runtime builder already handle
both forms, but there are no integration tests proving the runtime
difference. This spec defines the verification strategy.

## Two Forms

### Before-window constraint

```drlx
var w : /withdrawals[customer == "GOLD"] | length[3],
```

The OOPath inline constraint `[customer == "GOLD"]` filters facts **before**
they enter the window. Only matching facts are stored. The window always
contains up to 3 GOLD-customer withdrawals.

**Runtime path:** `PatternIR.conditions()` → `pattern.addConstraint()` →
drools `PatternBuilder` embeds alpha constraints in the `WindowNode`.

### After-window constraint

```drlx
var w : /withdrawals | length[3],
test w.customer == "GOLD",
```

All facts enter the window. The `test` element compiles to an
`EvalCondition` that sits **after** the pattern+window in the RETE network,
filtering the window's contents. The effective result set may be smaller
than the window size.

**Runtime path:** `EvalIR` → `buildEvalCondition()` → `EvalCondition`
added to `GroupElement` after the windowed `Pattern`.

Note: the bracket syntax `test w[customer == "GOLD"]` (issue #65) is out
of scope. This spec uses the already-supported dot-access form.

## Domain Change

Add a `customer` field to `Withdrawal`:

```java
// New field
private final String customer;

// New 3-arg constructor
public Withdrawal(String accountId, double amount, String customer)

// Backward-compatible 2-arg overload (defaults customer to null)
public Withdrawal(String accountId, double amount)

// Getter
public String getCustomer()
```

Existing tests use the 2-arg constructor and remain unchanged.

## Test Design

Both tests use identical event data with a `length[3]` window, producing
different fire counts to prove the semantic difference.

### Shared event data

| Order | accountId | amount | customer   |
|-------|-----------|--------|------------|
| 1     | A1        | 100    | GOLD       |
| 2     | A2        | 200    | STANDARD   |
| 3     | A3        | 300    | GOLD       |
| 4     | A4        | 400    | STANDARD   |
| 5     | A5        | 500    | GOLD       |

### Test 1: `constraintBeforeWindowFiltersEntries`

**DRLX:**
```drlx
var w : /withdrawals[customer == "GOLD"] | length[3],
do {}
```

**Expected:** Only GOLD events (A1, A3, A5) enter the window. Window holds
exactly 3. `fire()` = **3**.

### Test 2: `constraintAfterWindowFiltersContents`

**DRLX:**
```drlx
var w : /withdrawals | length[3],
test w.customer == "GOLD",
do {}
```

**Expected:** All 5 events enter the window of size 3. Final window holds
last 3: A3 (GOLD), A4 (STANDARD), A5 (GOLD). Test filters out A4.
`fire()` = **2**.

### Result

3 vs 2 proves the before/after semantic difference.

## Test Location

`WindowTest.java` — already has STREAM mode setup via
`buildWithStreamMode()` and uses `DrlxRuleUnitInstance`.

## Files Changed

| File | Change |
|------|--------|
| `Withdrawal.java` | Add `customer` field, 3-arg constructor, 2-arg overload, getter |
| `WindowTest.java` | Add `constraintBeforeWindowFiltersEntries()` and `constraintAfterWindowFiltersContents()` |

## Out of Scope

- Bracket-style test syntax `test w[...]` (issue #65)
- Time windows with constraints (mechanism is window-type-agnostic)
- Windows with accumulate (#69)
- Windows over group elements (#70)
