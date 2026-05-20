---
layout: post
title: "#432 — fixing ++/-- in MVEL3's map evaluator"
date: 2026-05-20
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [mvel3, transpiler, bugfix, upstream]
---

# #432 — fixing ++/-- in MVEL3's map evaluator

The previous session's accumulate work (#51) hit a bug in MVEL3: `count++` in a map-context block silently left the map unchanged. I'd documented the root cause and workaround (`count = count + 1`), but the bug was upstream. This session I filed [mvel/mvel#432](https://github.com/mvel/mvel/issues/432) and fixed it.

## The missing write-back

MVEL3's transpiler rewrites assignments to context input variables with a write-back call. For `count = count + 1`, it generates:

```java
context.put("count", count = count + 1);
```

The `AssignExpr` case in `MVELToJavaRewriter.rewriteNode()` handles this. But the `UnaryExpr` case — which handles `++` and `--` — only dealt with BigDecimal/BigInteger literal folding. No write-back. The local variable incremented, the map never saw it, and nothing complained.

I brought Claude in for the implementation. The fix mirrors the `AssignExpr` pattern: after recursing into the operand, check if it's a `NameExpr` in `context.getInputs()`, then wrap with `context.put()` (standalone statements) or `MVEL.putMap()` (expression context). We added the same handling for List contexts.

## The postfix trap

The first implementation wrapped the unary expression directly — `context.put("count", count++)`. Four eval tests passed, two failed. The postfix tests wrote 0 to the map instead of 1.

The issue is Java semantics: `count++` returns the old value before incrementing. So `context.put("count", count++)` dutifully puts 0 into the map. The local variable becomes 1, but the map never sees it. The existing `AssignExpr` pattern (`context.put("count", count = count + 1)`) works because assignment returns the new value — mirroring it for postfix silently produces the wrong result.

The fix converts postfix to prefix before wrapping:

```java
private static void ensurePrefixForm(UnaryExpr unaryExpr) {
    if (unaryExpr.getOperator() == UnaryExpr.Operator.POSTFIX_INCREMENT) {
        unaryExpr.setOperator(UnaryExpr.Operator.PREFIX_INCREMENT);
    } else if (unaryExpr.getOperator() == UnaryExpr.Operator.POSTFIX_DECREMENT) {
        unaryExpr.setOperator(UnaryExpr.Operator.PREFIX_DECREMENT);
    }
}
```

For standalone statements (the common case), prefix and postfix are equivalent — the return value is discarded. For expression context like `return count++`, the conversion means the return value is the new value rather than the old, but the map stays consistent.

## What landed

| Item | Detail |
|------|--------|
| Issue | [mvel/mvel#432](https://github.com/mvel/mvel/issues/432) |
| PR | [mvel/mvel#433](https://github.com/mvel/mvel/pull/433) — merged |
| Tests | 746 MVEL tests pass (10 new), 255 drlx-parser tests pass |
| Garden | `GE-20260520-ee66d2` — postfix write-back gotcha |

The drlx-parser accumulate tests still use the `count = count + 1` workaround from #51. Now that the fix is merged upstream, those could switch to `count++` — but both forms work, so it's not urgent.
