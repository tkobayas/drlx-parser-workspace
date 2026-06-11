# #92 Plain Property Reactivity via DataStoreSupport

**Issue:** [#92](https://github.com/tkobayas/drlx-parser/issues/92)
**Date:** 2026-06-11
**Status:** Draft

## Problem

Property reactivity for `@PropertyReactive` classes does not work in DRLX
today. Both update paths use `AllSetBitMask` (all properties treated as
changed):

- **External:** `DataStore.update(DataHandle, T)` has no property-name
  parameter; the engine re-evaluates every rule that touches the type.
- **Consequence:** `DataStoreSupport.update()` hardcodes
  `AllSetBitMask.get()` regardless of which properties the consequence
  actually modified.

The underlying engine plumbing already works: `ListDataStore` has an internal
`update(dh, obj, BitMask, class, match)` method and
`NamedEntryPoint.calculateUpdateBitMask()` computes the correct mask from
property names. The gap is threading property names through `DataStoreSupport`.

## Approach

Keep all changes in drlx-parser. No modifications to the upstream drools
`DataStore` interface.

`DataStoreSupport` becomes the facade for both property-aware update paths.
`DrlxRuleUnitInstance` exposes the `InternalRuleBase` needed for BitMask
calculation.

## External Update

New `DataStoreSupport` method for use by test and application code:

```java
public static void update(DataStore<?> store, DataHandle handle, Object fact,
                          InternalRuleBase ruleBase, String... modifiedProperties) {
    InternalStoreCallback callback = (InternalStoreCallback) store;
    BitMask mask = modifiedProperties.length == 0
        ? AllSetBitMask.get()
        : calculateUpdateBitMask(ruleBase, fact, modifiedProperties);
    callback.update(handle, fact, mask, fact.getClass(), null);
}
```

`null` for `InternalMatch` — same convention as
`NamedEntryPoint.update(FactHandle, Object, String...)` for external updates.

`DrlxRuleUnitInstance` adds an accessor:

```java
public InternalRuleBase getRuleBase() {
    return (InternalRuleBase) reteEvaluator.getKnowledgeBase();
}
```

Calling pattern:

```java
withMyUnitInstance(rule, (instance, unit, listener) -> {
    DataHandle dh = unit.getReactiveEmployees().add(emp);
    instance.fire();
    emp.setSalary(7000);
    DataStoreSupport.update(unit.getReactiveEmployees(), dh, emp,
                            instance.getRuleBase(), "salary");
    instance.fire();
});
```

## Consequence-Side Update

### Getting `InternalRuleBase` into consequence code

`DrlxLambdaConsequence.evaluate()` adds `__ruleBase__` to the consequence
variable map:

```java
vars.put("__ruleBase__", ((ReteEvaluator) valueResolver).getKnowledgeBase());
```

`DrlxRuleAstRuntimeBuilder` registers the type:

```java
types.put("__ruleBase__", Type.type(InternalRuleBase.class));
```

### Enhanced `DataStoreSupport.update()`

```java
public static void update(DataStore<?> store, Object fact, InternalMatch match,
                          InternalRuleBase ruleBase, String storeName,
                          String... modifiedProperties) {
    InternalStoreCallback callback = (InternalStoreCallback) store;
    DataHandle handle = Objects.requireNonNull(callback.lookup(fact),
            "DataStore '" + storeName + "' has no DataHandle for the given fact");
    BitMask mask = modifiedProperties.length == 0
        ? AllSetBitMask.get()
        : calculateUpdateBitMask(ruleBase, fact, modifiedProperties);
    callback.update(handle, fact, mask, fact.getClass(), match);
}
```

The old method signature is replaced — its only callers are rewriter-generated
consequence code, which is updated simultaneously.

### `DataStoreUpdateRewriter` changes

The rewriter updates its generated call to include `__ruleBase__`:

```java
// Before: DataStoreSupport.update(alerts, t, __match__, "alerts")
// After:  DataStoreSupport.update(alerts, t, __match__, __ruleBase__, "alerts")
```

**CompactWithExpression property extraction:** When the rewriter handles a
CompactWithExpression (e.g., `alerts.update(t) { setSalary(7000) }`), it
extracts property names from setter method names in the compact-with block and
appends them:

```java
DataStoreSupport.update(alerts, t, __match__, __ruleBase__, "alerts", "salary")
```

**Plain setter-before-update:** When the consequence uses plain
`t.setSalary(7000); alerts.update(t)`, the rewriter does NOT try to detect
preceding setter calls. Detecting which setters precede an `update()` call
across arbitrary control flow (conditionals, loops) is fragile. The rewriter
omits property names, so `AllSetBitMask` applies — a safe default that
preserves correctness at the cost of unnecessary re-evaluations.

## Test Plan (Tests First)

Create `PropertyReactiveTest` extending `DrlxBuilderTestSupport`, using
`withMyUnitInstance`. Reuses existing `ReactiveEmployee` domain class
(`@PropertyReactive` with `salary`, `basePay`, `bonusPay`).

### External update tests

1. **`externalUpdate_firesOnConstraintProperty`** — insert a `ReactiveEmployee`
   with `salary > 5000`, fire, then update `salary` via
   `DataStoreSupport.update(..., "salary")`. Rule re-fires because `salary` is
   used in the constraint.

2. **`externalUpdate_doesNotFireOnUnrelatedProperty`** — same setup, but update
   `bonusPay` via `DataStoreSupport.update(..., "bonusPay")`. Rule does NOT
   re-fire because `bonusPay` is not read by the constraint.

3. **`externalUpdate_withoutPropertyNames_firesAlways`** — same setup, update
   via plain `DataStore.update(handle, obj)` (no property names). Rule re-fires
   because `AllSetBitMask` means all properties treated as changed. Confirms
   backward compatibility.

### Consequence-side tests

4. **`consequenceUpdate_compactWith_firesOnConstraintProperty`** — two rules:
   R1 modifies a property read by R2's constraint via CompactWithExpression.
   R2 re-fires because the rewriter extracts the property name.

5. **`consequenceUpdate_compactWith_doesNotFireOnUnrelatedProperty`** — two
   rules: R1 modifies a property NOT read by R2's constraint via
   CompactWithExpression. R2 does NOT re-fire.

6. **`consequenceUpdate_plainSetter_treatsAllPropertiesAsChanged`** — two
   rules: R1 modifies a property NOT read by R2's constraint via plain
   `t.setX(...); store.update(t)`. R2 re-fires anyway because `AllSetBitMask`
   is used. Test comment explains: detecting setters preceding `update()` in
   arbitrary control flow is fragile, so `AllSetBitMask` is the safe default.

## Files Changed

| File | Change |
|------|--------|
| `PropertyReactiveTest.java` (new) | 6 test cases |
| `DrlxRuleUnitInstance.java` | Add `getRuleBase()` accessor |
| `DataStoreSupport.java` | New external update method; enhance consequence method |
| `DrlxLambdaConsequence.java` | Add `__ruleBase__` to consequence vars |
| `DrlxRuleAstRuntimeBuilder.java` | Register `__ruleBase__` type |
| `DataStoreUpdateRewriter.java` | Pass `__ruleBase__`; extract properties from CompactWithExpression |

## Implementation Order (TDD)

1. Write all 6 tests (initially failing)
2. Add `getRuleBase()` to `DrlxRuleUnitInstance`
3. Add external update method to `DataStoreSupport` → external tests pass
4. Enhance consequence path (`__ruleBase__`, rewriter, `DataStoreSupport`) →
   consequence tests pass

## Out of Scope

- Plain setter-before-update property detection in rewriter (follow-up)
- Watch list tests (#91) — depends on this working first
- Drools `DataStore` API changes — not needed
