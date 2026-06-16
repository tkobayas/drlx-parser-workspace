---
layout: post
title: "#30 — match lands: switch-style branching via eval desugaring"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [match, switch, conditional-branching, eval, #30]
---

# #30 — match lands: switch-style branching via eval desugaring

Form B if/else shipped yesterday. Today's `match` is the switch-style companion — same decomposition strategy, different surface syntax. The spec was already approved, so I brought Claude in and we went straight to implementation.

## The desugaring strategy

`match` reuses the Form B architecture wholesale. Each `case` becomes a synthetic rule with an `EvalIR` guard; `default` gets only the cumulative negation of all prior cases. The visitor method `buildMatchBranch` mirrors `buildConditionalBranchFormB` almost line for line:

```
rule R { var c : /customers,
    match (c.creditRating)
        case Rating.LOW    { P1, do C1 }
        case Rating.MEDIUM { P2, do C2 }
        default            { do C3 }
}
    ↓
RuleIR "R$0":  LHS = [prefix, EvalIR("c.creditRating == Rating.LOW"), P1]       RHS = C1
RuleIR "R$1":  LHS = [prefix, EvalIR("!(…LOW)"), EvalIR("…== Rating.MEDIUM"), P2]  RHS = C2
RuleIR "R$2":  LHS = [prefix, EvalIR("!(…LOW)"), EvalIR("!(…MEDIUM)")]          RHS = C3
```

No new IR types, no runtime builder changes — same as Form B.

## Type match via eval strings

The interesting part is `#Type` patterns. Value match is a simple equality (`subject == pattern`), but type match generates `instanceof` plus cast expressions as raw eval strings:

```java
// case #Car[speed > 80]  with subject "o"
// → "o instanceof Car && ((Car)o).speed > 80"
```

Multiple constraints chain with `&&`, each getting its own cast prefix. This keeps the visitor simple — no new AST nodes, no special handling in the runtime builder. The MVEL3 transpiler already handles `instanceof` and casts in eval expressions.

## Three body forms

The grammar supports block (`{ patterns, do consequence }`), `do statement`, and bare expression. All three desugar identically — the `processCaseBody` helper handles the dispatch, and `combineConsequences` merges multiple actions into a single consequence string.

## What landed

| Metric | Value |
|--------|-------|
| Tests added | 22 (12 parse/IR + 10 runtime) |
| Baseline | 439 → 461 |
| Commits | 3 on `main` |
| Files changed | 5 (lexer, parser, visitor, 2 new test classes) |

Issue #30 is closed. The conditional branching set — if/else Form A, if/else Form B, and now match — is complete.
