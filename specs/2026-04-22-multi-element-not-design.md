# Multi-element `not(/a, /b)` Design

**Date:** 2026-04-22
**Status:** Approved — ready for implementation plan
**Scope:** Extend the `notElement` grammar production to accept the parenthesised form `not(/a[, /b, ...])`, and wire the AST visitor to emit `GroupElementIR(NOT, [PatternIR...])` with `children.size() >= 1`.
**References:**
- `docs/DRLXXXX.md` §"'not' / 'exists'" (line 595-600)
- `specs/2026-04-21-group-element-and-not-design.md` (prior landing, single-element)
- `plans/2026-04-21-group-element-and-not-implementation.md`

## One-liner

Multi-element `not(/a, /b)` was deferred from #8 as a pure grammar extension. The infrastructure (tree-shape IR, proto, runtime NOT handling) was built for N children and has only been exercised with N=1. This landing exercises N>=1 via a second alternative on the `notElement` grammar rule plus a minimal loop in `buildNotElement`.

## Decisions (locked)

| # | Decision |
|---|----------|
| 1 | **Single-element-in-parens accepted.** Both `not /a` and `not(/a)` are legal. Spec line 597 reads permissively — "parens can be omitted when single" implies both forms are legal for N=1. |
| 2 | **Bindings still forbidden inside `not`.** The parenthesised form does not introduce a new binding-scope concept. `not(var p : /persons[...], ...)` stays a parse error (no `var bind :` prefix in `oopathExpression`), symmetric with the bare-form Decision #4 from the prior landing. Sibling correlation via inner bindings is explicitly out of scope here. |
| 3 | **No trailing comma.** `not(/a, /b,)` is a parse error — matches the existing trailing-comma policy elsewhere in the grammar. |
| 4 | **Empty `not()` rejected by grammar.** The production requires at least one `oopathExpression`. |
| 5 | **Nested `notElement` inside `not(...)` rejected by grammar.** The inner position accepts only `oopathExpression`, not another `notElement`. No runtime check needed. |
| 6 | **Tolerant LSP visitor: first child only.** `TolerantDrlxToJavaParserVisitor.visitNotElement` continues to delegate to `ctx.oopathExpression(0)` — the cursor is typically in one position at a time, and the `RulePattern` stand-in carries only one inner `OOPathExpr`. |

## Architecture overview

```
not(/persons[age<18], /orders[amount>1000])
        |
        v
ANTLR: notElement #parenNot     ← new alternative
        |
        v
DrlxToRuleAstVisitor.buildNotElement
        |                        (iterates ctx.oopathExpression())
        v
GroupElementIR(NOT,
  [PatternIR("", "", "persons", [age<18], null, []),
   PatternIR("", "", "orders",  [amount>1000], null, [])])
        |
        v
DrlxRuleAstRuntimeBuilder.buildLhs (unchanged — already recurses)
        |
        v
GroupElement newNotInstance() with 2 pattern children
        |
        v
Drools: NOT-with-AND semantics (standard)
```

## Grammar change

Current (prior landing):

```antlr
notElement
    : NOT oopathExpression
    ;
```

New:

```antlr
notElement
    : NOT oopathExpression                                          # singleNot
    | NOT '(' oopathExpression (',' oopathExpression)* ')'          # parenNot
    ;
```

- `#singleNot` — unchanged bare form `not /persons[...]`.
- `#parenNot` — `not(/a)` (single in parens, Decision #1), `not(/a, /b)` (two), `not(/a, /b, /c)` (N).
- Star quantifier `(',' oopathExpression)*` allows any N >= 1 elements inside parens.
- `not()` is a parse error (first `oopathExpression` not optional).
- Labelled alternatives are idiomatic in this parser (e.g. `oopathRoot`, `ruleItem`).

## Visitor change

`DrlxToRuleAstVisitor.buildNotElement` currently assumes a single `oopathExpression`:

```java
private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx) {
    DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
    ...
    PatternIR inner = new PatternIR("", "", entryPoint, conditions, castTypeName, positionalArgs);
    return new GroupElementIR(GroupElementIR.Kind.NOT, List.of(inner));
}
```

New:

```java
private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx) {
    List<LhsItemIR> children = new ArrayList<>();
    for (DrlxParser.OopathExpressionContext oopathCtx : ctx.oopathExpression()) {
        children.add(buildPatternFromOopath(oopathCtx));
    }
    return new GroupElementIR(GroupElementIR.Kind.NOT, List.copyOf(children));
}

private PatternIR buildPatternFromOopath(DrlxParser.OopathExpressionContext oopathCtx) {
    String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
    String castTypeName = extractCastType(oopathCtx);
    List<String> conditions = extractConditions(oopathCtx);
    List<String> positionalArgs = extractPositionalArgs(oopathCtx);
    return new PatternIR("", "", entryPoint, conditions, castTypeName, positionalArgs);
}
```

ANTLR's generated `ctx.oopathExpression()` returns `List<OopathExpressionContext>` because the production references `oopathExpression` more than once (via the `(...)*`). The single-element form's call site still works — with one entry the loop runs once. No `ctx.LPAREN()` branching needed.

## Runtime / proto / other

No changes:

- `DrlxRuleAstRuntimeBuilder.buildLhs` already recurses through `GroupElementIR.children()`. Drools' `GroupElementFactory.newNotInstance()` accepts N pattern children and implements NOT-with-AND.
- Proto schema already has `repeated LhsItem children` in `GroupElementProto`. No schema change, no `protoc` regen.
- `DrlxToJavaParserVisitor.visitNotElement` — still throws `UnsupportedOperationException`. Frozen-visitor policy unchanged; dispatch already goes through `visitRuleItem`.
- `TolerantDrlxToJavaParserVisitor.visitNotElement` — delegates to `ctx.oopathExpression(0)` (first element). See Decision #6.

## Errors / negative handling

| Input | Layer | Behaviour |
|-------|-------|-----------|
| `not(/a, /b, /c)` | Grammar | Accepted — N=3 |
| `not(/a)` | Grammar | Accepted — N=1 (Decision #1) |
| `not()` | Grammar | Parse error — first child required |
| `not(/a,)` | Grammar | Parse error — no trailing comma |
| `not(var p : /a, /b)` | Grammar | Parse error — `var bind :` is a `rulePattern` prefix, not part of `oopathExpression` |
| `not(not /a, /b)` | Grammar | Parse error — inner position accepts only `oopathExpression` |
| `not(/a` (unclosed) | Grammar | Parse error — missing `)` |

No visitor-side checks. No warning-level behaviour.

## Testing scope

TDD-light. Each test as its own commit, matching the cadence of the prior `#7 #8` landing.

| # | Type | Name | Description |
|---|------|------|-------------|
| 1 | happy | `notParens_singleElement` | `not(/persons[age<18])` behaves identically to the bare form — rule fires only when no under-18 present. Proves Decision #1. |
| 2 | happy | `notMultiElement_crossProduct` | `not(/persons[age<18], /orders[amount>1000])` — fires only while no `(under-18 person, high-value order)` combination exists. Insert one of each, rule stops firing; remove either, rule re-fires. Proves NOT-with-AND semantics over N=2. |
| 3 | negative | `notEmpty_failsParse` | `not()` → parse error. Asserts grammar requires at least one inner oopath. |

**Flipped test:** `notMultiElementForm_failsParse` — delete (was asserting the parser rejects `not(/a, /b)`; that behaviour is now obsolete). Remove the corresponding code block from `NotTest.java`.

**Unchanged — regression guard:**
- `notSuppressesMatch` — single-element bare form.
- `notWithOuterBinding` — outer binding referenced inside the (bare) NOT.
- `notWithInnerBinding_failsParse` — still guards Decision #4 (inner `var` binding rejected in both bare and paren forms).
- `protoRoundTrip_withNot` — N=1 proto round-trip.
- `javaParserVisitor_throwsOnNot` — frozen visitor still throws.
- `incompleteRule_WithNot` — LSP tolerance unchanged.

**Expected end state:** 66 − 1 (removed) + 3 (new) = **68 tests**, all passing.

## Non-goals / deferred

Explicitly out of scope:

- **`exists` (bare or paren forms)** — #9. Same `GroupElementIR(EXISTS)` shape; the `Kind.EXISTS` enum slot is still reserved, not exposed.
- **`and(...)` / `or(...)` group CEs** — #11. `Kind.AND` / `Kind.OR` slots anticipated.
- **Inner bindings inside `not(...)`** — Decision #4 from the prior landing holds. Revisit only when a real correlation-inside-NOT use case surfaces; would require scope-frame push/pop in `boundVariables` / `findReferencedBindings`.
- **Nested group elements inside `not(...)`** — e.g. `not(not /a, /b)`. No grammar support planned; covered implicitly under the group-CE follow-ups.
- **`drlx-lsp` cross-repo changes** — none. Tolerant override already handles the paren form structurally (same `RulePattern` stand-in; dispatch to first inner oopath). LSP consumes transparently.
- **Cache migration** — no proto schema change, no migration shim needed. Old cached `.pb` files still deserialize cleanly.

## Follow-ups this enables

- **`exists`** (#9) — the `GroupElementIR(EXISTS, [children])` shape can reuse `buildNotElement`'s structure once `Kind.EXISTS` is exposed. Single- and multi-element form can both land at once (this landing proves the infra handles N >= 1).
- **Group CEs** (#11) — `and(...)`, `or(...)` as siblings alongside `not`/`exists`. Same grammar pattern, same visitor shape.
