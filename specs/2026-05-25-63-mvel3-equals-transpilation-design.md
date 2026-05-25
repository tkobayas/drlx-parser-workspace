# #63 — MVEL3 `==` Transpilation for Reference Types

**Date:** 2026-05-25
**Issue:** tkobayas/drlx-parser#63
**Target codebase:** MVEL3 (`/home/tkobayas/usr/work/mvel3-development/mvel`)

## Problem

MVEL3's transpiler generates Java `==` (reference equality) for `==`/`!=` on all
reference types except BigDecimal and BigInteger, which get `.compareTo()` via
`OverloadRewriter`. This means an MVEL expression like `name == "John"` compiles
to `_this.getName() == "John"` — reference equality, not value equality.

This violates MVEL's semantic contract that `==` means value equality. It works
for string literals only by accident (Java interns compile-time string constants),
but fails for runtime-constructed strings and all other Object types.

## Solution

Add a fallback in `MVELToJavaRewriter.rewrite()` that rewrites `==`/`!=` on
reference types to `java.util.Objects.equals()`.

### Change 1: New method `rewriteReferenceEquality()`

A new private method in `MVELToJavaRewriter`:

```
private Expression rewriteReferenceEquality(BinaryExprTypes binExprTypes)
```

Logic:
1. Check operator is `EQUALS` or `NOT_EQUALS` — otherwise return `null`
2. Check at least one side is a non-primitive type — otherwise return `null`
3. Skip if either side is a `NullLiteralExpr` — `x == null` stays as Java `==`
4. Skip if the reference type side is an enum — enum `==` is correct in Java
5. Generate `java.util.Objects.equals(left, right)` for `==`
6. Generate `!java.util.Objects.equals(left, right)` for `!=`

### Change 2: Call site in `rewrite()`

After `overloader.overload()` returns `null`, call `rewriteReferenceEquality()`
as a fallback:

```java
Expression overloaded = overloader.overload(...);
if (overloaded == null) {
    overloaded = rewriteReferenceEquality(binExprTypes);
}
return overloaded;
```

### Exclusions (what stays as Java `==`)

| Type | Reason |
|------|--------|
| Primitives (`int`, `boolean`, etc.) | Java `==` is correct |
| Enums | Singleton guarantee, `==` is idiomatic |
| BigDecimal / BigInteger | Already handled by `OverloadRewriter` |
| Null literal on either side | `x == null` should stay as `==` |

## Key files

| File | Change |
|------|--------|
| `MVELToJavaRewriter.java` | Add `rewriteReferenceEquality()`, call from `rewrite()` |
| `ConstraintTranspilerTest.java` | New tests for String `==`/`!=`, Object `==`, enum `==` |

## Test cases

1. **String == String literal** — `name == "John"` → `java.util.Objects.equals(_this.getName(), "John")`
2. **String != String literal** — `name != "John"` → `!java.util.Objects.equals(_this.getName(), "John")`
3. **String == bound variable** — `name == $n` → `java.util.Objects.equals(_this.getName(), $n)`
4. **Nested property String ==** — `parent.name == "John"` → `java.util.Objects.equals(_this.getParent().getName(), "John")`
5. **Enum == (unchanged)** — `gender == Gender.MALE` → stays as `_this.getGender() == Gender.MALE`
6. **Null comparison (unchanged)** — `name == null` → stays as `_this.getName() == null`
7. **Primitive == (unchanged)** — `age == 30` → stays as `_this.getAge() == 30`
8. **Object == Object** — reference-typed field == bound variable → `java.util.Objects.equals()`

## drlx-parser follow-up (separate issue)

Once this MVEL3 fix is released, remove defensive workarounds in drlx-parser:
- `.intern()` in `DrlxRuleAstRuntimeBuilder.parseLiteral()`
- `.equals()` in `DrlxRuleAstRuntimeBuilder.buildSelfReferencePattern()` collision case
