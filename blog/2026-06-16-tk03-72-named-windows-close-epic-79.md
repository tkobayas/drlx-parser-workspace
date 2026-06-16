---
layout: post
title: "#72 — named windows close out epic #79"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [windows, cep, named-windows, #72, #79]
---

# #72 — named windows close out epic #79

Named windows were the last issue in the "conditional branching & named windows" epic. The DRLXXXX spec had two lines on them — "like queries but without parameters" and a single syntax example. I wanted a minimal implementation matching exactly that.

## Drools already had the runtime

Claude surveyed the drools codebase before we started designing. The runtime has `WindowDeclaration` (name + compiled `Pattern`), `WindowReference` (a `PatternSource` for rules), and `InternalKnowledgePackage.addWindowDeclaration()`. All wired up and tested in classic DRL with the `declare window ... end` / `from window` syntax. Zero new drools APIs needed — we just needed drlx-parser to emit the right objects.

## The syntax decisions

I wanted the declaration to follow the pattern from the spec:

```drlx
window WithdrawalWindow {
    /withdrawals | time[10s]
}
```

For referencing, I settled on the same convention as queries: the window name becomes an OOPath root with the first letter lowered. Constraints apply via square brackets as usual:

```drlx
rule R1 {
    var w : /withdrawalWindow[amount > 1000],
    do { ... }
}
```

PascalCase declaration, camelCase reference — identical to how queries work. The runtime builder matches by lowercasing the first letter of the declaration name.

## Five layers, one feature

The implementation touched every layer of the pipeline:

1. **Grammar** — `WINDOW` keyword in the lexer, `windowDeclaration` rule reusing existing `oopathExpression` and `windowFilter`
2. **IR** — `WindowDeclarationIR` record, `CompilationUnitIR` gains a `windowDeclarations` list
3. **Visitor** — `buildWindowDeclaration()` extracts the OOPath body into a `PatternIR`
4. **Runtime builder** — compiles `WindowDeclaration` objects before rules, detects references in `buildLhs` by checking `entryPoint` against the window registry
5. **Protobuf** — `WindowDeclarationParseResult` message for serialization round-trips

The reference detection in the runtime builder was the interesting part. When `buildLhs` encounters a `PatternIR` whose `entryPoint` matches a declared window name, it creates a `Pattern` with `WindowReference` as its source and resolves the type from the declaration's pattern — then applies any additional constraints from the rule.

## What landed

| | |
|---|---|
| **Branch** | `main` at `a418bb6` |
| **Tests** | 485 pass (7 new: 3 visitor-level, 4 session-level) |
| **Issue** | #72 closed |
| **Epic** | #79 closed — all 4 sub-issues complete (#22 if/else, #30 match, #69 window+accumulate, #72 named windows) |
