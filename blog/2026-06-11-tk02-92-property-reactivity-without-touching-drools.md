---
layout: post
title: "#92 — Property reactivity without touching drools"
date: 2026-06-11
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [property-reactivity, DataStoreSupport, rewriter, #92]
---

# #92 — Property reactivity without touching drools

The previous entry left PropertyReactiveWatchListTest on `KieSession` because `DataStore.update(DataHandle, T)` has no property-name parameter. I wanted to fix that, but adding `update(DataHandle, T, String...)` to the drools `DataStore` interface means an upstream API change. I started looking at whether we could avoid it entirely.

## DataStoreSupport as the facade

The plumbing already exists. `ListDataStore` has an internal `update(dh, obj, BitMask, class, match)` method, and `NamedEntryPoint.calculateUpdateBitMask()` computes the correct mask from property names. The gap was just threading those names through from the caller.

`DataStoreSupport` — the static helper we already use for consequence-side updates — was the natural place. We added a new overload for external updates:

```java
// Removable once drools DataStore.update() supports property-name-aware updates
public static void update(DataStore<?> store, DataHandle handle, Object fact,
                          InternalRuleBase ruleBase, String... modifiedProperties) {
    // ...
    callback.update(handle, fact, mask, fact.getClass(), null);
}
```

`DrlxRuleUnitInstance` got a `getRuleBase()` accessor so test code can pass the `InternalRuleBase` in. The test pattern reads naturally:

```java
DataStoreSupport.update(unit.reactiveEmployees, dh, emp,
        instance.getRuleBase(), "salary");
```

No drools API changes. Everything stays in drlx-parser.

## CompactWithExpression knows its properties

The consequence side was more interesting. `DataStoreSupport.update()` had been hardcoding `AllSetBitMask` — every property treated as changed, property reactivity defeated. The fix needed two things: the `InternalRuleBase` (for BitMask calculation) and the actual property names.

For the rule base, we injected `__ruleBase__` into the consequence variable context alongside the existing `__match__`, sourced from `valueResolver.getRuleBase()`.

For property names, the `CompactWithExpression` AST node already knows them. When a DRLX consequence writes `reactiveEmployees.update(e{salary = 9999})`, the `CompactWithExpression` has `getAssignments()` returning a `NodeList<AssignExpr>` — each assignment target is the property name. The rewriter now extracts them:

```java
private static List<String> extractPropertyNames(CompactWithExpression compactWith) {
    return compactWith.getAssignments().stream()
            .map(assign -> assign.getTarget().toString())
            .toList();
}
```

The generated code goes from `DataStoreSupport.update(alerts, t, __match__, "alerts")` to `DataStoreSupport.update(alerts, t, __match__, __ruleBase__, "alerts", "salary")`. Plain `store.update(obj)` without compact-with still gets `AllSetBitMask` — detecting setters across arbitrary control flow is fragile, so we documented it as a safe default in the test.

## The silent hang

The consequence tests hung during the first run. No error, no timeout, no output — the test runner just never came back. The issue: R1's consequence updates a fact with `AllSetBitMask`, the engine thinks `basePay` changed, R1 re-fires, loops forever. `instance.fire()` blocks until the agenda empties, which never happens. Switching to `instance.fire(10)` capped the iterations and let the assertions check the actual state. Self-terminating rule constraints (adding `bonusPay < 9000` so R1 stops after one firing) were needed too, to make the test verify the right thing instead of just capping an infinite loop.

## What landed

| Item | Detail |
|------|--------|
| Files changed | 7 (+267 -13) |
| Tests | 6 new (`PropertyReactiveTest`), 403 total, 0 failures |
| Commits | 6 on `main` |
| Issue | #92 closed; #91 (watch list) unblocked |
