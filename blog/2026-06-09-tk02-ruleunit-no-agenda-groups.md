---
layout: post
title: "RuleUnit doesn't do agenda groups — three annotations removed"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [annotations, epic-78, agenda-group, ruleunit]
---

# RuleUnit doesn't do agenda groups — three annotations removed

The earlier session filed #88 and #89: `@AgendaGroup`/`@AutoFocus`/`@RuleFlowGroup` annotations were correctly set on `RuleImpl` but never enforced at runtime. I asked Claude to pick up #88. Before writing any fix, we traced the execution path to understand why.

## The double bottleneck

`DrlxRuleUnitInstance` creates a `RuleUnitExecutorImpl`, which creates an `ActivationsManagerImpl`, which hardcodes `SimpleAgendaGroupsManager`. That's the no-op manager — it only knows the MAIN group, throws `UnsupportedOperationException` on `setFocus()`, and rejects any non-MAIN group.

But even if we swapped in the real manager (`StackedAgendaGroupsManager` via `DefaultAgenda`), there's a second problem: `ActivationsManagerImpl.createRuleAgendaItem()` always places rules in the MAIN group, ignoring the rule's `@AgendaGroup`. The full `DefaultAgenda` routes rules to `rtn.getRule().getAgendaGroup()` — `ActivationsManagerImpl` never does.

And `StackedAgendaGroupsManager` requires `InternalWorkingMemory`, not `ReteEvaluator` — a type that `RuleUnitExecutorImpl` doesn't implement. So there's no clean way to inject the full manager without either modifying drools or duplicating a large chunk of its internals.

## By design, not a bug

RuleUnit is the replacement for AgendaGroup and RuleFlowGroup. The concepts don't coexist — rule units partition work differently. Upstream drools doesn't support agenda groups in rule units, and trying to bolt them on would mean fighting the architecture.

I decided to remove all three annotations rather than carry dead code. The removal touched 9 files (3 annotation classes deleted, parser/model/proto/runtime cleaned up, 8 tests removed) and the diff was -301 lines. Closed #88 and #89.

## What remains from #6

`@ActivationGroup` still works — it's handled by `ActivationsManagerImpl` directly, no agenda group manager involved. The remaining annotations from epic #6 that we deferred (`@Duration`/`@Timer`, `@DateEffective`/`@DateExpires`) are tracked in #85 and #86.
