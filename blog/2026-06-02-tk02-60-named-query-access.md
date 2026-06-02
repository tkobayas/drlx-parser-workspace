---
layout: post
title: "#60 — designing named query access"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [queries, named-access, grammar, compiler, epic-77]
---

# #60 — designing named query access

Two issues left in epic #77 after #82 and #83 landed: #60 (named access) and #61 (@Rule/@DataSource annotations). I picked #60 — it's the more interesting parser/compiler feature.

Positional query invocation already works: `/trusts(subject, var object)`. Named access replaces position with parameter names: `/trusts[a == subject, var object : b]`. The DRLXXXX spec (lines 920-923) shows the syntax but doesn't say much about semantics. The design decisions were ours to make.

## The bracket reuse that almost worked for free

I brought Claude in to survey the current query infrastructure. It mapped the full path: grammar (`oopathRoot` with positional `(...)` and constraint `[...]`), visitor (`extractPositionalArgs` / `extractConditions`), IR (`PatternIR.positionalArgs` and `conditions` as string lists), and compiler (`buildQueryElement` turning positional strings into `QueryArgument[]`).

The key insight: brackets already parse. The grammar's `drlxExpression` handles `a == subject` as an expression and `object : b` as `identifier ':' expression`. Named access reuses the same bracket syntax — the compiler just reinterprets conditions as query arguments when the entry point resolves to a query.

One snag: `var object : b` doesn't parse. The `VAR` keyword isn't recognized inside `drlxExpression`. The grammar needs one new alternative:

```antlr
drlxExpression
    : VAR identifier ':' identifier    // output binding for named query access
    | identifier ':' expression
    | customConstraint
    | expression
    ;
```

That's the only grammar change. Everything else lives in the compiler.

## Five decisions that narrowed the design

The design came down to five choices:

1. **Compiler reinterpretation over parser distinction** — the grammar already handles the bracket syntax. The compiler knows which entry points are queries (via `queryRegistry`). No need to make the parser query-aware.

2. **Only `==` for inputs** — `paramName == expr` is argument passing, not filtering. Other operators would be confusing. If you write `a >= subject` in named access, that's an error.

3. **All parameters required** — every query parameter must appear. No implicit output variables for unnamed params. Explicit is safer; `var x : paramName` is easy to write.

4. **Mixing positional and named is an error** — `(...)` or `[...]`, not both. Brackets on a query with positional args would be ambiguous.

5. **Self-referencing named access is an error (for now)** — recursive queries need the disambiguation heuristic that currently works with positional args. Named access adds complexity there without clear benefit yet. Positional syntax works fine for recursion.

## How `buildNamedQueryArgs()` bridges to the existing path

The compiler's new method transforms named conditions into the same ordered `List<String>` that positional parsing produces. The algorithm:

1. Build a name-to-index map from `QueryImpl.getParameters()`: `{"a" → 0, "b" → 1}`
2. Parse each condition — `"a == subject"` becomes `args[0] = "subject"`, `"var object : b"` becomes `args[1] = "var object"`
3. Validate all slots are filled

The result feeds into `buildQueryElement()` unchanged. No new IR types, no visitor changes beyond what the grammar produces naturally.

The spec is at `specs/2026-06-02-60-query-named-access-design.md`. Next session: review the spec and write the implementation plan.
