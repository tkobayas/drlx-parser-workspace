---
layout: post
title: "#6 — seven annotations, three runtime surprises"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [annotations, epic-78, noloop, agenda-group, runtime]
---

# #6 — seven annotations, three runtime surprises

Epic #78 (rule metadata & syntax sugar) is underway. First task: extend the annotation pipeline from three annotations to ten. `@NoLoop`, `@LockOnActive`, `@AutoFocus`, `@Disabled`, `@AgendaGroup`, `@ActivationGroup`, `@RuleFlowGroup` — all mapping to existing `RuleImpl` setters, no drools-side changes needed.

## ArgShape makes the resolver data-driven

The existing `extractAnnotationLiteral` method switched on individual `Kind` values — one case per annotation. That was fine for three, but ten cases with four of them doing the identical "reject any argument" check felt wrong. We refactored `Kind` from a plain enum to one carrying an `ArgShape` field:

```java
public enum Kind {
    SALIENCE(ArgShape.INT),
    DESCRIPTION(ArgShape.STRING),
    NO_LOOP(ArgShape.NONE),
    AGENDA_GROUP(ArgShape.STRING),
    // ...
    ;
    enum ArgShape { NONE, INT, STRING }
    final ArgShape argShape;
    Kind(ArgShape argShape) { this.argShape = argShape; }
}
```

The resolver now switches on `kind.argShape` — three cases instead of ten. Adding a future annotation is a one-line `Kind` entry. The `kindDisplayName` method needed updating too: the old version just lowercased after the first letter, so `NO_LOOP` became `No_loop`. The new version splits on `_`, capitalises each segment, and joins. Claude caught that `RULEFLOW_GROUP` (two segments) produced `RuleflowGroup` instead of `RuleFlowGroup` — we renamed the enum value to `RULE_FLOW_GROUP`.

## @Disabled instead of @Enabled

One design decision worth noting: boolean annotations are marker-style (presence means "on", no arguments). That makes `@Enabled` pointless — rules are enabled by default. I went with `@Disabled` instead. Clean intent, and the pipeline rejects `@Disabled(false)` with "takes no arguments."

## The runtime tests that exposed the real gaps

Metadata tests passed on the first try. `RuleImpl.isNoLoop()` returns true, `getAgendaGroup()` returns `"g1"`, everything looks wired up. But when we switched the happy-path tests from KieSession-based stubs to actual `DrlxRuleUnitInstance` runtime tests — `instance.fire(100)` with real facts and real DataStore updates — three categories of failure appeared:

**@NoLoop fires 100 times.** `persons.update(p)` goes through `DataStore.update(DataHandle, T)` — the simple 2-arg version that doesn't pass `InternalMatch`. The drools no-loop check in `PhreakRuleTerminalNode` relies on `PropagationContext.getTerminalNodeOrigin()` to know which rule triggered the update. Without it, the check always passes and the rule refires to the limit. Filed as #87.

**@AgendaGroup ignored entirely.** `DrlxRuleUnitInstance` uses a `ReteEvaluator` with `SimpleAgendaGroupsManager`, which is a no-op implementation. Groups don't exist, `setFocus()` throws `UnsupportedOperationException`, and every rule fires in the default group. `@AutoFocus` and `@RuleFlowGroup` are affected too. Filed as #88 and #89.

Six tests are `@Disabled` with issue references. The annotation pipeline itself — parse, IR, proto, resolve, apply to `RuleImpl` — works correctly across all seven annotations. The gaps are in the engine/session layer, not the pipeline.

## What landed

| Item | Detail |
|------|--------|
| Annotations | 7 new (`@NoLoop`, `@LockOnActive`, `@AutoFocus`, `@Disabled`, `@AgendaGroup`, `@ActivationGroup`, `@RuleFlowGroup`) |
| Tests | 22 pass, 6 `@Disabled` (runtime bugs) |
| Issues created | #85, #86 (deferred Timer/Date annotations), #87, #88, #89 (runtime enforcement bugs) |
| Commits | 10 on `main` |
