# `and(...)` + `or(...)` Group Conditional Elements Design

**Date:** 2026-04-23
**Status:** Approved — ready for implementation plan
**Issue:** #11
**Scope:** Add explicit `and(...)` and `or(...)` group conditional elements to DRLX. Extend NOT/EXISTS paren form to accept nested CEs (symmetry with DRL10). Wires through lexer, parser, IR, visitor, runtime, and proto. Grammar refactor: lifts the trailing CE `,` from individual CE rules up to `ruleItem`.
**References:**
- `docs/DRLXXXX.md` §"'and' / 'or' structures" (line 573-593)
- `docs/DRLXXXX.md` §"'not' / 'exists'" (line 595-600) — symmetry clause
- Drools `DRL10Parser.g4` `lhsNot` / `lhsExists` (line 395, 402) — both accept `LPAREN lhsExpression RPAREN`
- Drools `GroupElement.addChild` (drools-base line 81) — single-child restriction applies only to NOT/EXISTS
- Drools `GroupElement.pack()` (line 143-183) — handles single-child AND/OR collapse and NOT-containing-EXISTS unwrap at compile time
- `specs/2026-04-22-exists-design.md` (prior EXISTS landing — template for this spec)

## One-liner

AND/OR are structurally the same as NOT/EXISTS in the IR, but simpler at runtime — `GroupElement.addChild` accepts N children natively for AND/OR, so the single-child-wrap used for NOT/EXISTS is skipped. The real work is **allowing nested CEs as children**, which forces a grammar refactor: a unified `groupChild` non-terminal used by all four CE rules, with the CE terminator `,` lifted to `ruleItem`. NOT/EXISTS paren form is upgraded to accept `groupChild` at the same time (DRL10 symmetry), so `not(or(...))`, `exists(and(...))`, etc. become legal.

## Decisions (locked)

| # | Decision |
|---|----------|
| 1 | **Scope: intermediate with CE symmetry (option B2).** Ship `and(...)`, `or(...)`, and extend NOT/EXISTS paren form to accept nested CEs in the same PR. Bound patterns as children (`and(var l : /locations, ...)`), passive `?and(...)`, and subnetwork/OR-scoping assertions are explicitly deferred. |
| 2 | **Grammar shape: refactor comma out (option γ).** The trailing `,` moves from inside each CE rule up to `ruleItem : notElement ',' \| existsElement ',' \| ...`. A new `groupChild` non-terminal (no commas) is used by all four CE paren forms. Cleaner grammar, single source of truth, and future-proof for bindings. |
| 3 | **AND/OR require parentheses always.** `and /x` and `or /x` are parse errors. The paren-omission sugar stays NOT/EXISTS-only, per DRLXXXX §"'not' / 'exists'" ("except they also allow for parenthesis to be omitted if there is a single child element"). |
| 4 | **Empty forms rejected by grammar.** `and()`, `or()`, `not()`, `exists()` all fail to parse — the `groupChild (',' groupChild)*` production requires at least one child. Matches #9's handling of `exists()`. |
| 5 | **Single-child paren form allowed.** `and(/x)` and `or(/x)` parse and run; Drools `GroupElement.pack()` collapses them at compile time, which is harmless and produces the expected runtime shape. |
| 6 | **NOT/EXISTS bare form unchanged.** `not /x` and `exists /x` still parse via the bare-oopath alternative; the paren form upgrade only affects the `not(...)` / `exists(...)` syntax. |
| 7 | **Visitor helper renamed and widened.** `buildGroupElementFromOopaths(oopathCtxs, kind)` → `buildGroupElementFromChildren(childCtxs, kind)`. The new helper dispatches each child via `buildGroupChild`, which branches on child kind (oopath / nested NOT / nested EXISTS / nested AND / nested OR). |
| 8 | **Runtime builder: extend switch, leave single-child-wrap unchanged.** Two new arms (`AND`, `OR`) added to the `switch (group.kind())` in `DrlxRuleAstRuntimeBuilder.buildLhs`. The existing single-child-wrap block is already correctly gated on `kind == NOT \|\| kind == EXISTS` — AND/OR flow through the else branch and accept N children natively via `GroupElement.addChild`. |
| 9 | **Proto schema: two new enum values.** `Kind.AND = 2; Kind.OR = 3;` added to `drlx_rule_ast.proto`. Since the proto build is plugin-managed (migration from session 2026-04-22), this is a `.proto`-only edit; `DrlxRuleAstProto.java` regenerates on `mvn compile`. |
| 10 | **Test coverage: three new files.** `AndTest` (4), `OrTest` (4), `NestedGroupTest` (3) — see §5. Existing `NotTest` (7) and `ExistsTest` (5) stay green unchanged; the grammar refactor is behaviour-preserving for their inputs. |
| 11 | **Top-level OR expansion is Drools territory.** `LogicTransformer` (invoked by KieBase at compile) expands top-level `or` into multiple rule instances. The parser emits a correctly shaped `GroupElement(OR)` tree; `OrTest` validates branch firing counts via session execution, not tree shape after expansion. |
| 12 | **LSP tolerant visitor updates included.** `DrlxToJavaParserVisitor` (frozen parent) gets `visitAndElement` / `visitOrElement` that throw `UnsupportedOperationException`, matching the existing NOT/EXISTS message. `TolerantDrlxToJavaParserVisitor` gets silent-drop overrides for AND/OR, and its NOT/EXISTS overrides are updated to handle the new paren form (which now carries `groupChild` children, not `oopathExpression`). Strategy is "first child only, recursively silent-drop" — pick the first `groupChild`; if it's an oopath, wrap as a RulePattern stand-in; if it's a nested CE, recurse via the nested visitor. Keeps code completion alive for typing inside the innermost pattern. See §6. |
| 13 | **Non-goals.** Bound patterns as group children, passive `?and`, DRL5/DRL10 compatibility mode switch, LogicTransformer behaviour assertions, new `TolerantDrlxToJavaParserVisitorTest` coverage for AND/OR/nesting (added when LSP exercises these constructs). |

## Architecture overview

```
or(and(/persons[age>=18], /orders[amount>1000]), /alerts[severity=="high"])
   |
   v
ANTLR: orElement        ← new production
         |
         +— groupChild (andElement)
         |     +— groupChild (oopathExpression — /persons[...])
         |     +— groupChild (oopathExpression — /orders[...])
         +— groupChild (oopathExpression — /alerts[...])
   |
   v
DrlxToRuleAstVisitor.buildOrElement
   |    (delegates to buildGroupElementFromChildren — Decision #7)
   v
buildGroupElementFromChildren(childCtxs, Kind.OR)
   |    iterates, dispatches each child via buildGroupChild
   v
GroupElementIR(Kind.OR, [GroupElementIR(Kind.AND, [PatternIR, PatternIR]), PatternIR])
   |
   v
DrlxRuleAstRuntimeBuilder.buildLhs
   |    switch arm → GroupElementFactory.newOrInstance()
   |    recurse into children (no single-child wrap; OR accepts N children)
   v
GroupElement(OR) { GroupElement(AND) { Pattern, Pattern }, Pattern }
   |
   v
rule.setLhs(root) → Drools KieBase.compile → LogicTransformer expansion
```

## §1 — Grammar changes (`DrlxParser.g4`)

**New non-terminal — unified CE child (no trailing comma):**
```antlr
groupChild
    : oopathExpression
    | notElement
    | existsElement
    | andElement
    | orElement
    ;
```

**Upgraded NOT/EXISTS paren forms (bare form unchanged; `,` removed from inside):**
```antlr
notElement
    : NOT oopathExpression
    | NOT '(' groupChild (',' groupChild)* ')'
    ;
existsElement
    : EXISTS oopathExpression
    | EXISTS '(' groupChild (',' groupChild)* ')'
    ;
```

**New AND/OR rules (parens required; `,` not included):**
```antlr
andElement
    : AND '(' groupChild (',' groupChild)* ')'
    ;
orElement
    : OR '(' groupChild (',' groupChild)* ')'
    ;
```

**`ruleItem` owns the terminator for all CEs:**
```antlr
ruleItem
    : rulePattern
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | ruleConsequence
    ;
```

**Lexer (`DrlxLexer.g4`):** add `AND : 'and' ;` and `OR : 'or' ;` if not already present. (Verify before editing — `and` / `or` might already be Java keywords reserved by a shared token set.)

**Edge cases rejected by grammar:**
- `and()` / `or()` / `not()` / `exists()` — empty child list.
- `and /x` / `or /x` — bare form not allowed.
- `and(/x,)` — trailing comma inside paren list.

**Edge cases accepted by grammar:**
- `and(/x)` / `or(/x)` — single-child paren form; `GroupElement.pack()` collapses at compile time.
- Arbitrary nesting: `and(not(/x), or(/y, /z))`, `not(or(/x, /y))`, etc.

## §2 — IR model (`DrlxRuleAstModel.java`)

```java
public record GroupElementIR(Kind kind, List<LhsItemIR> children) implements LhsItemIR {
    public enum Kind { NOT, EXISTS, AND, OR }
}
```

`children` is already `List<LhsItemIR>` — each element is either a `PatternIR` (oopath leaf) or a nested `GroupElementIR`. No further IR change needed.

**Proto (`drlx_rule_ast.proto`):** add `AND = 2; OR = 3;` to the `Kind` enum. Plugin-managed regeneration on `mvn compile`.

## §3 — Visitor changes (`DrlxToRuleAstVisitor.java`)

**(a) New visitor entry points:**
```java
private GroupElementIR buildAndElement(DrlxParser.AndElementContext ctx) {
    return buildGroupElementFromChildren(ctx.groupChild(), GroupElementIR.Kind.AND);
}

private GroupElementIR buildOrElement(DrlxParser.OrElementContext ctx) {
    return buildGroupElementFromChildren(ctx.groupChild(), GroupElementIR.Kind.OR);
}
```

**(b) Rename + widen helper:**
```java
private GroupElementIR buildGroupElementFromChildren(
        List<DrlxParser.GroupChildContext> childCtxs,
        GroupElementIR.Kind kind) {
    List<LhsItemIR> children = new ArrayList<>();
    for (DrlxParser.GroupChildContext c : childCtxs) {
        children.add(buildGroupChild(c));
    }
    return new GroupElementIR(kind, List.copyOf(children));
}

private LhsItemIR buildGroupChild(DrlxParser.GroupChildContext c) {
    if (c.oopathExpression() != null) return buildPatternFromOopath(c.oopathExpression());
    if (c.notElement() != null)       return buildNotElement(c.notElement());
    if (c.existsElement() != null)    return buildExistsElement(c.existsElement());
    if (c.andElement() != null)       return buildAndElement(c.andElement());
    if (c.orElement() != null)        return buildOrElement(c.orElement());
    throw new IllegalArgumentException("Unsupported group child: " + c.getText());
}
```

**(c) `buildNotElement` / `buildExistsElement` branch on bare vs paren:**
```java
private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx) {
    if (ctx.oopathExpression() != null) {
        return new GroupElementIR(GroupElementIR.Kind.NOT,
                List.of(buildPatternFromOopath(ctx.oopathExpression())));
    }
    return buildGroupElementFromChildren(ctx.groupChild(), GroupElementIR.Kind.NOT);
}
```
(Same shape for `buildExistsElement`.)

**(d) `buildRule` dispatch** — two new arms added to the `itemCtx` chain (~line 99-106):
```java
} else if (itemCtx.andElement() != null) {
    lhs.add(buildAndElement(itemCtx.andElement()));
} else if (itemCtx.orElement() != null) {
    lhs.add(buildOrElement(itemCtx.orElement()));
```

## §4 — Runtime builder (`DrlxRuleAstRuntimeBuilder.buildLhs`)

Two changes to the existing CE block (~line 237-255):

**(a) Extend the kind switch:**
```java
GroupElement ge = switch (group.kind()) {
    case NOT    -> GroupElementFactory.newNotInstance();
    case EXISTS -> GroupElementFactory.newExistsInstance();
    case AND    -> GroupElementFactory.newAndInstance();
    case OR     -> GroupElementFactory.newOrInstance();
};
```

**(b) Leave the single-child-wrap block unchanged.** The condition
```java
if ((group.kind() == GroupElementIR.Kind.NOT
        || group.kind() == GroupElementIR.Kind.EXISTS)
        && group.children().size() > 1) { ... }
```
is already correctly gated. AND/OR flow through the else branch, where `buildLhs` recurses into all children and `parent.addChild(ge)` attaches the group. Nested CEs are handled by the outer `for (LhsItemIR item : items)` loop — each nested `GroupElementIR` re-enters this same block.

**No changes to `buildPattern`, `applyAnnotations`, or the outer rule-building scaffolding.**

## §5 — Test surface

Three new test files under `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/`, following the `ExistsTest` template (minimal fact model, DRLX string, KieBase build via `DrlxRuntime`, session fire, assert fire count).

### `AndTest` (4)
| Test | Input | Assertion |
|------|-------|-----------|
| `andAllowsMatch` | `and(/persons[age>18], /persons[name=="Bob"])` | Fires when an over-18 Bob exists. |
| `andSingleChild` | `and(/persons[age>18])` | Parses and fires; verifies pack() collapse is harmless. |
| `andEmpty_failsParse` | `and()` | Parser throws. |
| `andBare_failsParse` | `and /persons[age>18]` | Parser throws (no paren-omission for AND). |

### `OrTest` (4)
| Test | Input | Assertion |
|------|-------|-----------|
| `orFirstBranchFires` | `or(/as[x==1], /bs[x==1])` with only A matching | Fires once. |
| `orBothBranchesFire` | `or(/as[x==1], /bs[x==1])` with both matching | Fires once per matching branch (Drools OR-expansion). |
| `orBlocksWhenNeither` | Neither matches | No fire. |
| `orEmpty_failsParse` | `or()` | Parser throws. |

### `NestedGroupTest` (3)
| Test | Input | Assertion |
|------|-------|-----------|
| `orOfAnds` | `or(and(/as, /bs), and(/cs, /ds))` | Fires for each satisfied branch. DRLXXXX §16 canonical example. |
| `andContainingNot` | `and(/as, not(/bs))` | Fires only when some A exists and no B exists. Verifies AND accepts nested NOT. |
| `notContainingOr` | `not(or(/as, /bs))` | Fires only when neither A nor B exists. Verifies NOT paren form accepts nested OR. |

**Existing tests unchanged:** `NotTest` (7) and `ExistsTest` (5) stay green. The `,` refactor is behaviour-preserving for their inputs; the `groupChild` widening for NOT/EXISTS is purely additive.

**Total after landing:** 73 → 84 passing.

## §6 — LSP tolerant visitor (`TolerantDrlxToJavaParserVisitor`)

Two visitor classes are affected:

### `DrlxToJavaParserVisitor` (frozen parent)

Add two methods that throw, mirroring the existing NOT/EXISTS handling at line 402-416:
```java
@Override
public Node visitAndElement(DrlxParser.AndElementContext ctx) {
    throw new UnsupportedOperationException(
            "`and` is not supported in DrlxToJavaParserVisitor — "
            + "use DrlxToRuleAstVisitor for DRLX→RuleImpl. "
            + "Note: this visitor is frozen for new DRLX syntax.");
}

@Override
public Node visitOrElement(DrlxParser.OrElementContext ctx) { /* same for 'or' */ }
```

This prevents ANTLR's default tree-walk from producing garbage when the frozen visitor encounters AND/OR constructs.

### `TolerantDrlxToJavaParserVisitor` (LSP)

**(a) Update `visitNotElement` / `visitExistsElement`** to branch on bare vs paren. The paren form now exposes `ctx.groupChild()` instead of `ctx.oopathExpression()`:
```java
@Override
public Node visitNotElement(DrlxParser.NotElementContext ctx) {
    DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
    if (oopathCtx != null) {
        // bare form: not /x
        return buildRulePatternStandIn(oopathCtx);
    }
    // paren form: not ( groupChild, ... ) — silent-drop, keep first child only
    return visitFirstGroupChild(ctx.groupChild());
}
```

**(b) Add matching overrides for AND/OR:**
```java
@Override
public Node visitAndElement(DrlxParser.AndElementContext ctx) {
    return visitFirstGroupChild(ctx.groupChild());
}

@Override
public Node visitOrElement(DrlxParser.OrElementContext ctx) {
    return visitFirstGroupChild(ctx.groupChild());
}
```

**(c) New helper** (single implementation, shared by all four CE overrides):
```java
private Node visitFirstGroupChild(List<DrlxParser.GroupChildContext> children) {
    DrlxParser.GroupChildContext first = children.get(0);
    if (first.oopathExpression() != null) {
        return buildRulePatternStandIn(first.oopathExpression());
    }
    // Nested CE — delegate to the appropriate override, which recursively
    // reaches the innermost oopath or returns its own RulePattern stand-in.
    return visit(firstNonNullChild(first));
}

private RulePattern buildRulePatternStandIn(DrlxParser.OopathExpressionContext oopathCtx) {
    SimpleName type = new SimpleName("var");
    SimpleName bind = new SimpleName("_");
    OOPathExpr expr = (OOPathExpr) visit(oopathCtx);
    RulePattern pattern = new RulePattern(null, type, bind, expr);
    type.setParentNode(pattern);
    bind.setParentNode(pattern);
    expr.setParentNode(pattern);
    return pattern;
}
```

**Rationale.** The tolerant visitor's job is to keep `tokenIdJPNodeMap` populated for completion tokens even when the user has an in-progress CE construct. Picking the first child is sufficient — the user's cursor is overwhelmingly inside that child during typing. Full multi-child traversal is not needed for completion.

**Test coverage.** None added to `TolerantDrlxToJavaParserVisitorTest` in this PR (per Decision #13). The existing NOT/EXISTS tests exercise the bare-form paths and continue to pass. Coverage for AND/OR paren forms can be added when LSP features actually use them.

## §7 — Proto schema (`drlx_rule_ast.proto`)

```proto
message GroupElementIR {
    enum Kind {
        NOT = 0;
        EXISTS = 1;
        AND = 2;       // new
        OR = 3;        // new
    }
    Kind kind = 1;
    repeated LhsItemIR children = 2;
}
```

No other schema changes. Plugin-managed regeneration; no system `protoc` needed.

## §8 — Implementation order

1. Grammar (`DrlxParser.g4` + `DrlxLexer.g4` if needed) — verify existing parse tests stay green after the `,` refactor.
2. IR (`DrlxRuleAstModel.Kind`) + proto schema — compile the enum additions.
3. Visitor helper rename + widen (`buildGroupElementFromChildren`, `buildGroupChild`).
4. Rule-AST visitor dispatch — new `buildAndElement` / `buildOrElement`; update `buildNotElement` / `buildExistsElement` for paren form; add dispatch arms to `buildRule`.
5. Runtime builder switch arms.
6. LSP visitors — frozen-parent throws for AND/OR; tolerant-subclass silent-drop for AND/OR and updated NOT/EXISTS paren form.
7. Tests: `AndTest`, `OrTest`, `NestedGroupTest`.
8. Full-suite run (`mvn clean install`) — confirm 84 passing.

Each step should compile and leave existing tests green. The grammar refactor (step 1) is the largest single change; everything after is additive.

## §9 — Known gotchas

- **Stale `.tokens` files** — already auto-mitigated by `maven-clean-plugin` (session 2026-04-22). If IDE-only build bypasses Maven, fall back to `mvn clean install`.
- **`and` / `or` lexer tokens** — verify these aren't already reserved or in use for another purpose (e.g., Java-expression `&&` / `||` tokens in oopath condition context). If a conflict exists, solve via lexer mode or parser-level keyword disambiguation, not by renaming.
- **Nested `not(exists(...))` and `exists(not(...))`** — `GroupElement.pack()` rewrites these at Drools compile time (line 167-181). Tests should fire through the session to ensure the rewrite is exercised; parser just emits the literal tree.
- **Top-level `or(...)`** — `LogicTransformer` in Drools expands this into multiple rule instances. Our `OrTest` assertions on fire counts will reflect the expanded behaviour; if fire counts look surprising, the culprit is likely LogicTransformer semantics, not our IR.

## §10 — Explicit non-goals

- Bound patterns as group children (`and(var l : /locations, ...)`) — future issue; requires widening `groupChild` to accept `rulePattern`-like bindings, which is a meaningfully larger grammar and visitor change.
- Passive `?and(...)` / `?or(...)` markers.
- OR variable scoping (CONSENSUS-rule enforcement) — moot without bindings in scope.
- Explicit subnetwork / LogicTransformer behaviour assertions.
- New `TolerantDrlxToJavaParserVisitorTest` coverage for AND/OR and nested paren forms — LSP-layer tests land when LSP features actually use these constructs.
