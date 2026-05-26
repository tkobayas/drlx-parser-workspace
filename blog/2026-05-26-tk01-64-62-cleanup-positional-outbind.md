---
layout: post
title: "#64 + #62 — cleanup and positional out-binding"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [positional, out-binding, cleanup, mvel3]
---

# #64 + #62 — cleanup and positional out-binding

Short session. The MVEL3 equals PR ([mvel/mvel#434](https://github.com/mvel/mvel/pull/434))
merged overnight, so I started by closing #63 and pulling the fix into the local
snapshot. That unlocked #64 — removing the two workarounds we'd put in last session.

## Stripping the band-aids (#64)

Two changes in `DrlxRuleAstRuntimeBuilder`:

1. `.intern()` in `parseLiteral()` — was interning string literals so Java `==` would
   match. No longer needed since MVEL3 now generates `Objects.equals()`.
2. The `.equals()` ternary in `buildSelfReferencePattern()` — a null-safe workaround
   for query positional args that collided with pattern field names. Replaced with the
   standard `fieldName == (alias)` synthesis, matching the non-collision path.

Both were four-line deletes. We ran the full suite — 279 tests, all green.

## Positional out-binding (#62)

With the cleanup done, I picked #62 from epic #26. The spec (DRLXXXX §POSL) shows
this syntax:

```
var l : /locations("paris", var postCode)
```

`"paris"` constrains position 0 (city). `var postCode` binds position 1 (district)
to a variable without constraining it.

The grammar already had `positionalArg : VAR identifier | expression` — added during
the queries work (#41). The visitor already prefixed `"var "` to those args. And
`buildSelfReferencePattern` (the query path) already handled the prefix correctly,
creating a `Declaration` binding instead of a constraint.

The gap was `buildPattern` — the regular pattern path. It treated every positional
arg as a constraint, synthesizing `fieldName == (argExpr)` unconditionally. The fix
adds a `"var "` prefix check before the synthesis loop:

```java
if (argExpr.startsWith("var ")) {
    String varName = argExpr.substring(4).trim();
    Declaration decl = pattern.addDeclaration(varName);
    java.lang.reflect.Method getter = findGetterForField(patternClass, fieldName);
    Class<?> fieldType = getter.getReturnType();
    decl.setReadAccessor(new DrlxBeanFieldReader(getter, fieldType));
    boundVariables.put(varName, new BoundVariable(varName, fieldType, pattern, decl));
    continue;
}
```

Same logic as `buildSelfReferencePattern`, just in the right place.

## What landed

| Item | Detail |
|------|--------|
| #64 | `.intern()` and `.equals()` workarounds removed |
| #62 | Positional `var` out-binding on regular patterns |
| Tests | 282 total (279 existing + 3 new for #62) |
| Commits | `9b5079a` (#64), `9199925` (#62) |
