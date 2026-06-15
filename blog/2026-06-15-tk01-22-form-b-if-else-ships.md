---
layout: post
title: "#22 — Form B if/else ships: one rule in, N rules out"
date: 2026-06-15
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [if-else, form-b, rule-decomposition, antlr4, #22]
---

# #22 — Form B if/else ships: one rule in, N rules out

Form A landed weeks ago — pattern-only branches with a single trailing `do`. Form B is the other half: each branch carries its own consequence, and the compiler decomposes one DRLX rule into N synthetic `RuleImpl` objects. The spec was approved in the previous session; this one was pure execution.

## Rule decomposition in the visitor

The strategy avoids touching drools-core entirely. `buildConditionalBranchFormB` takes the common LHS prefix, walks each branch to partition items into patterns and consequences, builds cumulative negation guards (same as Form A), and emits one `RuleIR` per branch:

```
rule EmailPeople { ... if (paris) { P1, do C1 } else if (london) { P2, do C2 } }
    ↓
RuleIR "EmailPeople$0":  LHS = [prefix, EvalIR("paris"),           P1]  RHS = C1
RuleIR "EmailPeople$1":  LHS = [prefix, EvalIR("!(paris)"), EvalIR("london"), P2]  RHS = C2
```

No new IR types. No runtime builder changes. Each synthetic rule flows through the existing `buildRule` pipeline — RETE node sharing handles the duplicated prefix automatically. The `$` naming mirrors Java's synthetic inner-class convention.

`buildRule` now returns `List<RuleIR>` (singleton for Form A, N-element for Form B). The caller in `visitDrlxCompilationUnit` changed from `add` to `addAll` — one line.

## Two grammar surprises

The `branchConsequence` rule needed `expression` for the bare form, not `statement`. The DRLXXXX examples show `Email.to(...).send()` without a semicolon, but Java's `statement` rule requires one for expression statements. The `do` form keeps `statement` to support blocks.

```antlr
branchConsequence
    : DO statement
    | expression
    ;
```

The second surprise: `ruleItem` had `conditionalBranch ','` with a mandatory comma. Form A needs the comma to separate the conditional from the trailing `do`, but Form B's conditional branch is the last item — no comma follows. Changing to `','?` resolved it without affecting Form A parsing.

## What landed

Five validations reject malformed Form B rules: branch with no consequence, trailing `do` after per-branch consequences, consequence before pattern, nested Form B inside Form B, and items after the conditional branch.

| Metric | Value |
|--------|-------|
| Tests added | 24 (13 parse-level + 11 runtime) |
| Baseline | 415 → 439 |
| Commits | 5 on `main` |
| Files changed | 3 (grammar, visitor, 2 new test classes) |

Issue #22 is closed. The if/else feature set is now complete — both Form A (shared consequence) and Form B (per-branch consequences) are shipping.
