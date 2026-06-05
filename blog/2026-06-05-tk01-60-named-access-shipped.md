---
layout: post
title: "#60 — named query access shipped"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [queries, named-access, grammar, compiler, epic-77]
---

# #60 — named query access shipped

The design spec from last session laid out the five decisions and the `buildNamedQueryArgs` algorithm. This session turned it into working code: grammar, visitor, compiler, nine tests. The whole thing went smoothly because the design was right — named access reuses the existing positional path, and the compiler just reorders arguments before feeding them in.

## The ANTLR label surprise

The grammar change was one line — add `VAR identifier ':' identifier` as a new `drlxExpression` alternative. But two `identifier` references in one alternative would make ANTLR generate a list accessor for `identifier()`, breaking the existing visitor code that expects a single context. Element labels solved it:

```antlr
drlxExpression
    : VAR varBind=identifier ':' varParam=identifier
    | bind=identifier ':' expression
    | customConstraint
    | expression
    ;
```

Claude hit a second snag during the visitor update: ANTLR element labels generate public fields (`ctx.bind`), not accessor methods (`ctx.bind()`). The spec had assumed methods. A small thing, but the kind that wastes twenty minutes if you don't know it.

## `buildNamedQueryArgs` — string surgery that stays boring

The compiler change was the core of the feature. The detection logic in `buildLhsPatterns` gained two new branches: a mixing error check inside the existing positional path, and a new named access path after it.

`buildNamedQueryArgs` does the translation. It takes condition strings like `"minAge == 25"` and `"var p : result"`, maps each to its parameter index via `QueryImpl.getParameters()`, and produces the same ordered `List<String>` that positional parsing creates. The result feeds into `buildQueryElement` unchanged — no new IR types, no downstream changes.

```java
// "minAge == 25" → args[0] = "25"
// "var p : result" → args[1] = "var p"
// output: ["25", "var p"] — identical to positional parsing
```

The self-referencing check is a one-liner: `targetQuery == currentQuery` on the named access path throws immediately. Recursive queries must use positional syntax — named access adds no value there and the disambiguation heuristic doesn't support it yet.

## Nine tests, five of them error paths

The happy-path tests cover the basics: mixed input/output, all-input params, parameter order independence, and result row binding (`var t : /query[...]` with `t.result` access). Each test defines a query, invokes it with named syntax from a rule, fires the engine, and checks results.

The error tests verify every guardrail: missing parameter names the missing param, unknown parameter names the bad param, mixing `(...)` and `[...]` says so explicitly, self-referencing named access points to positional syntax, and non-`==` operators tell you to use `==`. All five produce messages specific enough to diagnose the problem without reading the compiler source.

## What landed

| Commit | Change |
|--------|--------|
| `c68a5de` | Grammar: `VAR` output-binding in `drlxExpression` |
| `10ae334` | Visitor: `visitDrlxExpression` for labeled accessors |
| `e11f8be` | Compiler: `buildNamedQueryArgs` + detection logic |
| `12510b5` | Tests: 9 named access tests |
