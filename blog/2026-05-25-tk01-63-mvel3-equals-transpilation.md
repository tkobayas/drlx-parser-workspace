---
layout: post
title: "#63 — fixing MVEL3's == for reference types"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [mvel3, drlx-parser]
tags: [mvel3, transpiler, equals, objects-equals]
---

# #63 — fixing MVEL3's == for reference types

This one started as a loose thread from the recursive queries work (#58). I'd added
`.intern()` to `parseLiteral` and `.equals()` to `buildSelfReferencePattern` based
on a hunch that MVEL3 transpiles `==` to Java `==` — reference equality, not value
equality. The hunch was never verified. Time to find out.

## What the transpiler actually does

I asked Claude to trace the `==` path through MVEL3's transpiler. The answer was
clear: `MVELToJavaRewriter.rewrite()` delegates to `OverloadRewriter.overload()`,
which only has overloads registered for BigDecimal and BigInteger. Those get
`.compareTo() == 0`. Everything else — String, Object, any POJO — falls through
and keeps the raw Java `==`.

So `name == "John"` in MVEL becomes `_this.getName() == "John"` in Java. That works
for string literals only because the JVM interns compile-time string constants. A
runtime-constructed string would silently fail to match. No error, no warning, just
no results.

## The fix: `rewriteReferenceEquality()`

The fix goes in `MVELToJavaRewriter.rewrite()` — after the overloader returns null,
a new fallback method checks whether the operator is `==` or `!=` and at least one
side is a non-primitive reference type. If so, it rewrites to
`java.util.Objects.equals(left, right)` (or `!java.util.Objects.equals(...)` for `!=`).

Three exclusions keep existing behaviour correct:

- **Primitives** — Java `==` is already value equality
- **Enums** — singleton guarantee makes `==` idiomatic and correct
- **Null literals** — `x == null` stays as `==`

The enum detection turned out to be non-obvious. JavaParser's
`ResolvedReferenceType.isJavaLangEnum()` checks whether the type IS `java.lang.Enum`
itself, not whether it's an enum subclass. The actual check requires walking through
`type.asReferenceType().getTypeDeclaration().get().isEnum()`.

## What landed

| Item | Detail |
|------|--------|
| Branch | `equals-transpile` on `tkobayas/mvel` |
| PR | [mvel/mvel#434](https://github.com/mvel/mvel/pull/434) |
| Tests | 8 new in `ConstraintTranspilerTest`, 754 total pass |
| drlx-parser | 279 tests pass with updated MVEL3, no regressions |

The drlx-parser workarounds (`.intern()` in `parseLiteral`, `.equals()` in
`buildSelfReferencePattern`) are now redundant but harmless. They'll be cleaned up
in a follow-up once the MVEL3 fix is merged upstream.
