---
layout: post
title: "#73 — runtime CEP: STREAM mode and pseudo clock"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [cep, windows, stream-mode, pseudo-clock]
---

# #73 — runtime CEP: STREAM mode and pseudo clock

Last session shipped window parsing (#67) — `| length[5]` and `| time[5s]` attached `SlidingLengthWindow`/`SlidingTimeWindow` behaviors to `Pattern`. But the tests only verified the RETE structure. Actually inserting events and firing rules threw `ClassCastException: RuleUnitDefaultFactHandle cannot be cast to DefaultEventHandle`. I filed #73 to track the runtime gap.

## Two fixes, one root cause each

I brought Claude in to trace the fact-handle creation path through drools. The chain is `ClassObjectTypeConf.isEvent` → factory → handle type. Two things were wrong.

First, `DrlxRuleBuilder.createKieBase()` hardcoded default `RuleBaseConfiguration` — CLOUD mode. Windows need STREAM. Rather than auto-detecting STREAM from the package contents, we added overloads that accept `KieBaseConfiguration` externally:

```java
public KieBase createKieBase(List<KiePackage> kiePackages, KieBaseConfiguration config) {
    RuleBase kBase = RuleBaseFactory.newRuleBase("myKBase", config);
    kBase.addPackages(kiePackages);
    return KnowledgeBaseFactory.newKnowledgeBase(kBase);
}
```

A matching `build(String, KieBaseConfiguration)` convenience method sits on top. Callers set `config.setOption(EventProcessingOption.STREAM)` when they need CEP.

Second, `buildPattern()` was setting `ClassObjectType.isEvent` based on `windowType != null`. That's the wrong signal — event-ness belongs to the domain class, not the pattern syntax. We replaced it with a `@Role` annotation check:

```java
Role roleAnnotation = type.getAnnotation(Role.class);
boolean isEvent = roleAnnotation != null && roleAnnotation.value() == Role.Type.EVENT;
```

This is consistent with how `TypeDeclaration.createTypeDeclarationForBean()` already works — the annotation was being picked up for the TypeDeclaration but ignored for the ClassObjectType.

## Pseudo clock on `DrlxRuleUnitInstance`

With the fixes in, we needed session-level tests. The length window test was straightforward — insert 7 events into `length[5]`, assert only 5 fire. But time windows need clock control. `DrlxRuleUnitInstance` was creating `RuleUnitExecutorImpl` with a single-arg constructor that defaults to realtime clock.

We added a `create(kieBase, unitData, sessionConfig)` overload. The test sets `ClockType.PSEUDO_CLOCK`, inserts two events at time 0, advances 6 seconds past the 5-second window, inserts a third, and fires once — only the third event matches:

```java
SessionPseudoClock clock = instance.getClock();
unit.withdrawals.add(new Withdrawal("A1", 100.0));
unit.withdrawals.add(new Withdrawal("A2", 200.0));
clock.advanceTime(6, TimeUnit.SECONDS);
unit.withdrawals.add(new Withdrawal("A3", 300.0));
assertThat(instance.fire()).isEqualTo(1);
```

Four tests in `WindowTest`: basic length, length eviction, basic time, and time expiry with pseudo clock. All green.

## What landed

| Commit | Change |
|--------|--------|
| `0d0f4c2` | `KieBaseConfiguration` overloads on `DrlxRuleBuilder` |
| `32f095d` | `isEvent` from `@Role` annotation |
| `5bf8cd0` | 3 session-level window tests |
| `0d0c9e4` | `SessionConfiguration` on `DrlxRuleUnitInstance` + pseudo clock test |
