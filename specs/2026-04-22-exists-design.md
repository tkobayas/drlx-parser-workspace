# `exists /pattern` + `exists(/a, /b)` Design

**Date:** 2026-04-22
**Status:** Approved — ready for implementation plan
**Issue:** #9
**Scope:** Add the `exists` group-element CE to DRLX: bare form `exists /pattern,` and parenthesised form `exists(/a[, /b, ...]),`. Wires through lexer, parser, IR, visitor, runtime, proto, and LSP tolerant visitor. Ships both forms (including multi-element) in a single landing.
**References:**
- `docs/DRLXXXX.md` §"'not' / 'exists'" (line 595-600)
- `specs/2026-04-22-multi-element-not-design.md` (prior NOT landing — template for this spec)
- `blog/2026-04-22-tk01-not-a-b-two-gotchas.md` (single-child constraint gotcha, AND-wrap fix)
- Drools `GroupElement.addChild` (drools-base) — enforces single-child for NOT **and** EXISTS

## One-liner

`exists` mirrors `not` structurally and at runtime. Both are `ScopeDelimiter.ALWAYS` in Drools (outer bindings leak in; inner bindings do not leak out), both enforce single-child on the Drools `GroupElement`, and both need the AND-wrap fix for multi-element input. The pipeline added for NOT is already prepared for N>=1 — this landing exposes `Kind.EXISTS` and reuses it.

## Decisions (locked)

| # | Decision |
|---|----------|
| 1 | **Both forms in one landing.** Ship bare (`exists /a`) and paren (`exists(/a)`, `exists(/a, /b)`) together. Infra is proven N>=1 from NOT; phasing would not mitigate any risk. |
| 2 | **Bindings forbidden inside `exists`.** `exists var p : /persons[...]` is a parse error — `var bind :` is a `rulePattern` prefix, not part of `oopathExpression`. Symmetric with the NOT inner-binding restriction. |
| 3 | **No trailing comma inside parens.** `exists(/a, /b,)` is a parse error — matches existing grammar convention. |
| 4 | **Empty `exists()` rejected by grammar.** The production requires at least one `oopathExpression`. |
| 5 | **Nested `existsElement` / `notElement` inside `exists(...)` rejected by grammar.** The inner position accepts only `oopathExpression`. No runtime check needed. |
| 6 | **Visitor factored into a shared helper.** `buildGroupElementFromOopaths(oopathCtxs, kind)` takes the oopath list and `GroupElementIR.Kind`. Both `buildNotElement` and `buildExistsElement` delegate to it. Minor duplication removal; no abstraction overhead. |
| 7 | **Test coverage: reduced mirror.** 5 tests (not a full 7-test mirror). Drops `protoRoundTrip_withExists` and `existsParens_singleElement` — both are ceremonial given proven NOT infra. |
| 8 | **Tolerant LSP visitor: first child only.** `TolerantDrlxToJavaParserVisitor.visitExistsElement` delegates to `ctx.oopathExpression(0)`, same rationale as `visitNotElement`. |

## Architecture overview

```
exists(/persons[age>=18], /orders[amount>1000])
        |
        v
ANTLR: existsElement           ← new production parallel to notElement
        |
        v
DrlxToRuleAstVisitor.buildExistsElement
        |                       (delegates to buildGroupElementFromOopaths — Decision #6)
        v
GroupElementIR(EXISTS,
  [PatternIR("", "", "persons", [age>=18], null, []),
   PatternIR("", "", "orders",  [amount>1000], null, [])])
        |
        v
DrlxRuleAstRuntimeBuilder.buildLhs  (AND-wrap condition generalised to
                                     match NOT or EXISTS with N>1 children)
        |
        v
GroupElement newExistsInstance() with 1 child (wrapped AND if multi)
        |
        v
Drools: EXISTS-with-AND semantics — fires once if >=1 matching combination,
                                     scope-delimiter ALWAYS
```

## Lexer change

`DrlxLexer.g4` — one new token.

```antlr
EXISTS : 'exists';
```

Confirmed safe: `exists` is not used as an identifier anywhere in the test corpus or existing grammar (only appears in a code comment in `DrlxParser.g4` and in `Files.exists` API calls in `DrlxCompiler`).

## Grammar change

`DrlxParser.g4` — one new `ruleItem` alternative and a new production.

```antlr
ruleItem
    : rulePattern
    | notElement
    | existsElement       // new
    | ruleConsequence
    ;

existsElement
    : EXISTS oopathExpression ','
    | EXISTS '(' oopathExpression (',' oopathExpression)* ')' ','
    ;
```

Structurally identical to `notElement`. Unlabelled alternatives (same rationale as NOT — avoids the ANTLR labelled-alternative Context-class split documented in `blog/2026-04-22-tk01-not-a-b-two-gotchas.md`).

## IR change

`DrlxRuleAstModel.GroupElementIR.Kind` — promote the reserved comment to a real slot.

```java
public enum Kind {
    NOT,
    EXISTS   // new — exposes the reservation from the NOT landing
}
```

## Visitor change

`DrlxToRuleAstVisitor.java` — extract helper, add `buildExistsElement`, dispatch from `visitRuleItem`.

```java
// New helper — shared by NOT and EXISTS; will accept future AND/OR the same way.
private GroupElementIR buildGroupElementFromOopaths(
        List<DrlxParser.OopathExpressionContext> oopathCtxs,
        GroupElementIR.Kind kind) {
    List<LhsItemIR> children = new ArrayList<>();
    for (var oopathCtx : oopathCtxs) {
        children.add(buildPatternFromOopath(oopathCtx));
    }
    return new GroupElementIR(kind, List.copyOf(children));
}

// Now 1-line wrappers:
private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx) {
    return buildGroupElementFromOopaths(ctx.oopathExpression(), GroupElementIR.Kind.NOT);
}

private GroupElementIR buildExistsElement(DrlxParser.ExistsElementContext ctx) {
    return buildGroupElementFromOopaths(ctx.oopathExpression(), GroupElementIR.Kind.EXISTS);
}
```

Dispatch in `visitRuleItem` — new `ExistsElementContext` branch parallel to the existing `NotElementContext` branch.

## Runtime change

`DrlxRuleAstRuntimeBuilder.buildLhs` — extend the switch and generalise the AND-wrap condition.

```java
GroupElement ge = switch (group.kind()) {
    case NOT    -> GroupElementFactory.newNotInstance();
    case EXISTS -> GroupElementFactory.newExistsInstance();
};
// Drools GroupElement.addChild enforces single-child for NOT and EXISTS alike.
// AND-wrap when the IR has multiple children so the group element has exactly one child.
if ((group.kind() == GroupElementIR.Kind.NOT || group.kind() == GroupElementIR.Kind.EXISTS)
        && group.children().size() > 1) {
    GroupElement andInstance = GroupElementFactory.newAndInstance();
    buildLhs(group.children(), andInstance, typeResolver, entryPointTypes, unitClass, boundVariables);
    ge.addChild(andInstance);
} else {
    buildLhs(group.children(), ge, typeResolver, entryPointTypes, unitClass, boundVariables);
}
parent.addChild(ge);
```

## Proto change

`drlx_rule_ast.proto` — additive enum value. No schema break; old `.pb` files (containing only `NOT`) still deserialise.

```proto
enum GroupElementKind {
  GROUP_ELEMENT_KIND_UNSPECIFIED = 0;
  GROUP_ELEMENT_KIND_NOT         = 1;
  GROUP_ELEMENT_KIND_EXISTS      = 2;  // new
}
```

Then:
- Regenerate `DrlxRuleAstProto.java` via `protoc`.
- Extend the two mapping switches in `DrlxRuleAstParseResult` (`fromProtoGroupKind` / `toProtoGroupKind`) to handle `EXISTS`.

## LSP / frozen-visitor changes

- **`TolerantDrlxToJavaParserVisitor`** — new `visitExistsElement` override delegating to `ctx.oopathExpression(0)`. Same rationale as `visitNotElement` (Decision #8).
- **`DrlxToJavaParserVisitor`** — new `visitExistsElement` throwing `UnsupportedOperationException`. Matches the frozen-visitor convention for `visitNotElement`.

## Errors / negative handling

| Input | Layer | Behaviour |
|-------|-------|-----------|
| `exists /a` | Grammar | Accepted — bare form |
| `exists(/a)` | Grammar | Accepted — paren single |
| `exists(/a, /b)` | Grammar | Accepted — paren multi |
| `exists()` | Grammar | Parse error — first child required (test #5) |
| `exists(/a,)` | Grammar | Parse error — no trailing comma |
| `exists var p : /a` | Grammar | Parse error — `var bind :` is not in `oopathExpression` (test #4) |
| `exists(exists /a)` | Grammar | Parse error — inner position accepts only `oopathExpression` |
| `exists /a` (no terminating `,`) | Grammar | Parse error — `,` terminates `ruleItem` |

All error surfacing is pure ANTLR output. No visitor-side checks, no warning-level behaviour.

## Testing scope

TDD-light. Each test as its own commit. New file `ExistsTest.java` extending `DrlxBuilderTestSupport`.

| # | Type | Name | Description |
|---|------|------|-------------|
| 1 | happy | `existsAllowsMatch` | `exists /persons[age>=18]` — rule fires only when an adult is present. Insert no one → 0 fires. Insert an adult → fires once. Inverse of `notSuppressesMatch`. |
| 2 | happy | `existsWithOuterBinding` | `Person p : /persons[...], exists /orders[customerId == p.age]` — outer binding `p` referenced inside the EXISTS constraint. Proves beta-join from outer pattern into EXISTS group element. |
| 3 | happy | `existsMultiElement_crossProduct` | `exists(/persons[age>=18], /orders[amount>1000])` — fires while at least one `(adult, high-value order)` combination exists. Exercises the generalised AND-wrap path for multi-child EXISTS. |
| 4 | negative | `existsWithInnerBinding_failsParse` | `exists var p : /persons[...]` → parse error. Guards Decision #2 for EXISTS. |
| 5 | negative | `existsEmpty_failsParse` | `exists()` → parse error. Guards Decision #4. |

**Dropped from full mirror (Decision #7):**
- `protoRoundTrip_withExists` — proto plumbing already proven for NOT; the new `GROUP_ELEMENT_KIND_EXISTS` enum value is a trivial additive change and doesn't warrant a per-kind round-trip regression test at this layer.
- `existsParens_singleElement` — the multi-element test (#3) exercises the paren form; a dedicated single-in-parens test would be purely ceremonial given the bare-form happy-path already covers N=1.

**Regression guard — NotTest unchanged.** All 7 NOT tests still pass. EXISTS has its own test file; no edits to NotTest.

**Expected end state:** 68 (current) + 5 (new) = **73 tests**, all passing.

## Non-goals / deferred

Explicitly out of scope:

- **`and(...)` / `or(...)` group CEs** — #11. `Kind.AND` / `Kind.OR` slots remain unused after this landing.
- **`exists` nested inside `not(...)` or vice-versa** — no grammar support; covered implicitly under the group-CE follow-ups.
- **Inner bindings inside `exists(...)`** — symmetric with NOT Decision #4. Revisit only when a real correlation-inside-EXISTS use case surfaces.
- **Proto round-trip test for EXISTS** — see Decision #7.
- **Cache migration** — no schema break; the `GROUP_ELEMENT_KIND_EXISTS` enum value is additive.
- **`drlx-lsp` cross-repo changes** — none. Tolerant override handles the new form structurally (same `RulePattern` stand-in; dispatch to first inner oopath). LSP consumes transparently.

## Follow-ups this enables

- **`and(...)` / `or(...)`** (#11) — will reuse `buildGroupElementFromOopaths` once `Kind.AND` / `Kind.OR` are exposed. The AND-wrap condition in `buildLhs` may need a different generalisation (AND/OR are not single-child), but the IR shape is ready.
