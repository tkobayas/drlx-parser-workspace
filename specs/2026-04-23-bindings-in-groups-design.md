# Bound Patterns as Group Children Design

**Date:** 2026-04-23
**Status:** Approved — ready for implementation plan
**Issue:** #17
**Scope:** Widen `groupChild` (introduced in #11 refactor γ) to accept bound patterns (`var l : /foo` and `Type l = /foo`) inside all four CE paren forms (`and`, `or`, `not`, `exists`). Bindings are branch-local inside `or(...)`. No IR, runtime, or proto changes — purely grammar + visitor + tests.
**References:**
- Issue #17 — https://github.com/tkobayas/drlx-parser/issues/17
- `specs/2026-04-23-and-or-design.md` (#11) — refactor γ (unified `groupChild`), §10 non-goal flags this ticket
- `blog/2026-04-23-tk01-and-or-and-the-and-that-ate-double-ampersand.md` — "next ticket" callout
- `docs/DRLXXXX.md` §"'and' / 'or' structures" (line 573-593) — target of §6 updates
- Drools `DRL10Parser.g4` `lhsPatternBind` (line 161) — syntactic reference
- Drools `GroupElement.pack()` (drools-base line 143-183) — runtime binding-scope authority (DRLX defers to it)

## One-liner

Bindings already work at top level (`var l : /foo,`) because `rulePattern`'s grammar carries type and name, and `PatternIR` already has `typeName` / `bindName` fields. This ticket makes the same bound form a legal `groupChild`, so `and(var l : /locations, /persons[locationId == l.id])` parses, emits correctly, and lets Drools resolve the join at compile time. OR branches get branch-local scoping — we do not implement the Drools consensus rule.

## Decisions (locked)

| # | Decision |
|---|----------|
| 1 | **All four CE paren forms accept bindings.** `and(var l : ...)`, `or(var l : ...)`, `not(var l : ...)`, `exists(var l : ...)` all parse. Keeps the `groupChild`-is-single-source-of-truth invariant from refactor γ. |
| 2 | **OR bindings are branch-local.** A binding introduced inside `or(...)` is visible only within its own branch and below. Never visible outside the `or(...)` group, even if the same name is used on every branch. Drools' consensus rule (`l : (Foo() or Bar())` with both branches binding `l`) is explicitly deferred; may be revisited as a separate ticket. |
| 3 | **Grammar: factor out `boundOopath`.** New non-terminal `boundOopath : identifier identifier (':' | '=') oopathExpression ;`. `rulePattern` delegates to it (with trailing `,`); `groupChild` gains an arm that uses it (without `,`). Both `var l :` and `Type l =` binding forms work uniformly at top level and inside groups. |
| 4 | **No binding-resolution at the DRLX layer.** `l.id`-style references inside right-sibling conditions stay opaque expression text; the binding table is Drools' responsibility at KieBase compile time. DRLX emits `PatternIR(typeName, bindName, ...)` correctly and stops there. |
| 5 | **NOT/EXISTS bindings parse and emit; runtime semantics follow Drools.** Inside `not(var l : /a, /bs[locationId == l.id])`, `l` is a within-group join; outside the group it has no scope. DRLX does not police this — Drools errors surface at KieBase build if misused. |
| 6 | **IR, runtime, proto, LSP-core are unchanged.** Only the LSP tolerant visitor gets a thin `visitBoundOopath` override to preserve the in-progress-typing invariant. |
| 7 | **Non-goals.** OR-branch consensus-rule enforcement; shadowing detection across nesting; passive `?/` bindings (part of #10); new `TolerantDrlxToJavaParserVisitorTest` coverage; validation-error catalogue improvements. |

## Architecture overview

```
and(var l : /locations, /persons[locationId == l.id])
   |
   v
ANTLR: andElement
         +— groupChild (boundOopath — var l : /locations)
         +— groupChild (oopathExpression — /persons[locationId == l.id])
   |
   v
DrlxToRuleAstVisitor.buildAndElement
   |    → buildGroupElementFromChildren → buildGroupChild
   v
buildGroupChild dispatches:
   +— boundOopath   → buildPatternFromBoundOopath  → PatternIR("var", "l", ...)
   +— oopathExpression → buildPatternFromOopath   → PatternIR("", "", ...)
   |
   v
GroupElementIR(AND, [PatternIR("var","l",...), PatternIR("","",...)])
   |
   v
DrlxRuleAstRuntimeBuilder.buildLhs  (UNCHANGED)
   |    → GroupElementFactory.newAndInstance()
   |    → Drools Pattern builder resolves `l` by name during compile
   v
GroupElement(AND) { Pattern(Location, bind=l), Pattern(Person, condition=locationId==l.id) }
```

## §1 — Grammar changes (`DrlxParser.g4`)

**New non-terminal — factored binding form (no trailing comma):**
```antlr
boundOopath
    : identifier identifier (':' | '=') oopathExpression
    ;
```

**`rulePattern` delegates:**
```antlr
rulePattern
    : boundOopath ','
    ;
```

**`groupChild` widened (bound arm first, see §9 gotcha on alternative ordering):**
```antlr
groupChild
    : boundOopath          // new — must be before oopathExpression
    | oopathExpression
    | notElement
    | existsElement
    | andElement
    | orElement
    ;
```

**No changes to `notElement`, `existsElement`, `andElement`, `orElement`, `ruleItem`.** These already delegate to `groupChild`; bindings thread through transparently.

**Edge cases accepted (new):**
- `and(var l : /a, /bs[locationId == l.id])` — single-type join
- `and(Person p = /persons, /orders[personId == p.id])` — explicit-type form
- `or(var l : /as, Person p = /bs)` — mixed binding forms in one group; both branch-local
- `not(var l : /a, /bs[ownerId == l.id])` — within-group join inside negated group
- `and(var l : /a, not(/bs[x == l.y]))` — outer binding visible to nested-CE right sibling (subject to Drools `pack()` semantics; see §9)

**Edge cases rejected (unchanged from #11):**
- `and()` / `or()` / `not()` / `exists()` — empty list
- `and(var l : /a,)` — trailing comma inside group
- `and /x` / `or /x` — no paren-omission sugar for AND/OR

**Edge cases that parse but behaviour documented:**
- `and(var l : /a)` / `or(var l : /a)` — single-child bound paren form; `GroupElement.pack()` collapses at Drools compile time (same as #11 Decision #5 but now with binding)

## §2 — Visitor changes (`DrlxToRuleAstVisitor.java`)

**(a) New helper — single source of truth for bound-pattern construction:**
```java
private PatternIR buildPatternFromBoundOopath(DrlxParser.BoundOopathContext ctx) {
    String typeName = ctx.identifier(0).getText();
    String bindName = ctx.identifier(1).getText();
    DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
    String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
    String castTypeName = extractCastType(oopathCtx);
    List<String> conditions = extractConditions(oopathCtx);
    List<String> positionalArgs = extractPositionalArgs(oopathCtx);
    return new PatternIR(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs);
}
```

**(b) Existing `buildPattern(RulePatternContext)` collapses to a delegator:**
```java
private PatternIR buildPattern(DrlxParser.RulePatternContext ctx) {
    return buildPatternFromBoundOopath(ctx.boundOopath());
}
```

This replaces the inline extraction currently at `DrlxToRuleAstVisitor.java:261-270`. The old body is deleted, not commented out.

**(c) `buildGroupChild` gains the bound-pattern arm (first, per grammar order):**
```java
private LhsItemIR buildGroupChild(DrlxParser.GroupChildContext c) {
    if (c.boundOopath() != null)      return buildPatternFromBoundOopath(c.boundOopath());
    if (c.oopathExpression() != null) return buildPatternFromOopath(c.oopathExpression());
    if (c.notElement() != null)       return buildNotElement(c.notElement());
    if (c.existsElement() != null)    return buildExistsElement(c.existsElement());
    if (c.andElement() != null)       return buildAndElement(c.andElement());
    if (c.orElement() != null)        return buildOrElement(c.orElement());
    throw new IllegalArgumentException("Unsupported group child: " + c.getText());
}
```

**No changes to** `buildAndElement`, `buildOrElement`, `buildNotElement`, `buildExistsElement`, `buildGroupElementFromChildren`, `buildRule`, or any of the oopath-extraction helpers.

## §3 — IR, runtime, proto — no changes

| Layer | Reason no change is needed |
|-------|---------------------------|
| IR (`DrlxRuleAstModel.java`) | `PatternIR(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs)` already encodes bindings. Bare = `("", "", ...)`, bound = `("var", "l", ...)`. Same record, different data. |
| Runtime (`DrlxRuleAstRuntimeBuilder.java`) | `buildPattern` already reads `pattern.typeName()` / `pattern.bindName()` and constructs a Drools `Declaration` + `Pattern` when they are non-empty. Unchanged paths handle bindings whether they came from top-level `rulePattern` or from a `groupChild`. |
| Proto (`drlx_rule_ast.proto`) | `PatternIR` fields are already in the schema; serialization round-trips bindings today for top-level patterns. |
| LSP core (`DrlxToJavaParserVisitor`) | Frozen parent; the existing `visitRulePattern` path still covers top-level bound patterns. No new throw-methods needed — `visitBoundOopath` is reached only via the tolerant subclass. |

## §4 — LSP tolerant visitor (`TolerantDrlxToJavaParserVisitor`)

Add a `visitBoundOopath` override that produces a real `RulePattern` stand-in using the bound type and name — not the `"var"/"_"` placeholder used by bare oopaths. This preserves the "tolerant visitor matches parser" invariant for in-progress typing inside group bindings.

```java
@Override
public Node visitBoundOopath(DrlxParser.BoundOopathContext ctx) {
    SimpleName type = new SimpleName(ctx.identifier(0).getText());
    SimpleName bind = new SimpleName(ctx.identifier(1).getText());
    OOPathExpr expr = (OOPathExpr) visit(ctx.oopathExpression());
    RulePattern pattern = new RulePattern(null, type, bind, expr);
    type.setParentNode(pattern);
    bind.setParentNode(pattern);
    expr.setParentNode(pattern);
    return pattern;
}
```

**`visitFirstGroupChild` (from #11) needs no change.** When the first child is a `BoundOopathContext`, the default `visit` dispatches into `visitBoundOopath`. Before this ticket, `firstNonNullChild` wasn't reachable with a bound child; after it, the override above handles the new case.

**Test coverage.** None added to `TolerantDrlxToJavaParserVisitorTest` this PR. Rationale is identical to #11 Decision #13: LSP-layer tests land when LSP features actually exercise these constructs.

## §5 — Test surface

Three new test files and one extension. Total: 84 → **94** passing.

### `BindingInGroupTest` — new file (4 tests)

Primary AC driver for #17.

| Test | Input | Assertion |
|------|-------|-----------|
| `andBindingVisibleToRightSibling` | `and(var l : /locations, /persons[locationId == l.id])` | Fires when a Person's `locationId` matches a Location's `id`. |
| `andBindingExplicitType` | `and(Location l = /locations, /persons[locationId == l.id])` | Same semantics, explicit-type form. Confirms `boundOopath` DRY worked. |
| `andBothChildrenBound` | `and(var l : /locations, var p : /persons[locationId == l.id])` | Both bindings emit to IR; `l` visible to `p`. |
| `bareOopathStillWorks` | `and(/locations, /persons)` | Regression — existing bare-oopath group children parse and fire after grammar change. |

### `OrBindingScopeTest` — new file (3 tests)

Validates Decision #2 (branch-local).

| Test | Input | Assertion |
|------|-------|-----------|
| `orBranchBindingLocal` | `or(and(var l : /as, /bs[aid == l.id]), and(var l : /cs, /ds[cid == l.id]))` | Each branch's `l` joins within that branch; top-level fires once per satisfied branch. |
| `orDirectBindingBranchLocal` | `or(var l : /as, var l : /bs)` | Both branches parse; each `l` is a distinct scope (no unification). Fires once per matching branch. Documents branch-local stance. |
| `orBindingNotVisibleAfterGroup` — parse-or-build-error | `or(var l : /as), /bs[x == l.id]` | **Observation test.** Either the KieBase build throws with "unresolved `l`" (or similar), or the session fires zero times. Assert whichever Drools produces, and leave an inline comment recording the observed behaviour. Documents branch-local stance at the system boundary. |

### `NotExistsBindingTest` — new file (2 tests)

Covers Decision #5.

| Test | Input | Assertion |
|------|-------|-----------|
| `notBindingWithinGroup` | `not(var l : /locations, /persons[locationId == l.id])` | Fires when no (Location, Person) pair joins. `l` is within-group. |
| `existsBindingWithinGroup` | `exists(var l : /locations, /persons[locationId == l.id])` | Fires when at least one matching (Location, Person) pair exists. |

### `NestedGroupTest` — extend (1 new test, 3 → 4)

| Test | Input | Assertion |
|------|-------|-----------|
| `andWithNestedNotCarryingBinding` | `and(var l : /locations, not(/persons[locationId == l.id]))` | Fires when some Location has no matching Person. `l` from outer `and` visible to nested `not(...)` pattern. |

### Existing tests — all stay green unchanged

`AndTest` (4), `OrTest` (4), `NestedGroupTest` (existing 3), `NotTest` (7), `ExistsTest` (5), top-level rulePattern tests. The `boundOopath` factor-out is behaviour-preserving.

## §6 — DRLXXXX §16 updates

Promote bindings from the "non-goal" block and add a scoping subsection.

**New canonical example:**
```
rule "match persons to their location" {
  and(
    var l : /locations,
    /persons[locationId == l.id]
  )
  do { ... }
}
```

**Explicit-type form:**
```
and(
  Location l = /locations,
  /persons[locationId == l.id]
) do { ... }
```

**Branch-local OR:**
```
or(
  and(var l : /as, /bs[aid == l.id]),
  and(var l : /cs, /ds[cid == l.id])
) do { ... }
```

**New subsection: "Binding scope in group CEs":**

> Bindings introduced inside `or(...)` are visible only within their own branch. DRLX does not unify same-named bindings across OR branches; use `and(...)` to make a binding visible across an OR. Bindings inside `not(...)` and `exists(...)` are visible only within the negated group.

## §7 — Implementation order

1. **Grammar (`DrlxParser.g4`)** — add `boundOopath`; refactor `rulePattern` to delegate; add bound arm to `groupChild` (as first alternative). Run `mvn -pl drlx-parser-core -am install` + full test suite. Expect 84 green (grammar refactor is behaviour-preserving).
2. **Visitor helper factor-out (`DrlxToRuleAstVisitor`)** — introduce `buildPatternFromBoundOopath`; collapse `buildPattern` to delegate; extend `buildGroupChild` with bound arm. Re-run — expect 84 green.
3. **LSP tolerant visitor** — add `visitBoundOopath` override. Re-run.
4. **New tests** — `BindingInGroupTest`, `OrBindingScopeTest`, `NotExistsBindingTest`, `NestedGroupTest#andWithNestedNotCarryingBinding`. Re-run — expect 94 green.
5. **DRLXXXX §16 updates** — promote bindings from non-goals; add "Binding scope in group CEs" subsection.
6. **Full-suite `mvn clean install`** — final sanity.

Each step compiles independently; tests run cleanly at 1, 2, 3, and 4.

## §8 — Known gotchas

- **ANTLR alternative ordering.** `boundOopath` must be the first alternative of `groupChild`. ANTLR picks first-match; putting `oopathExpression` first would mean bound children never match (a bare oopath cannot start with an identifier, but an identifier-starting input would fail the oopath parse and need error recovery). Inline grammar comment will document this.
- **`identifier` production — `$`-prefixed names.** DRL10 conventionally uses `$l` for bindings; DRLX's `identifier` production may not include `$`. This ticket does **not** widen it. Tests use plain names (`l`, `p`). If `$`-prefixed names are observed to fail in user-facing DRLX code post-landing, file as a follow-up; do not expand scope here.
- **Drools binding errors surface at `KieBase.newKieBase()`, not at parse.** The `orBindingNotVisibleAfterGroup` test may throw on build rather than run. Wrap in try/catch, assert on error text or on zero-fire count. Inline-comment the observed behaviour once discovered.
- **`GroupElement.pack()` and nested bindings.** When `l` is introduced in outer `and(...)` and referenced inside nested `not(...)`, Drools' `pack()` may rewrite the tree (drools-base `GroupElement.java:143-183`). The `andWithNestedNotCarryingBinding` test catches regression. If `pack()` drops the binding, document the observed behaviour and consider tightening the test or the docs.
- **Token collision (carryover from #11).** `identifier`, `var`, and type names all flow through the shared Java lexer. Adding new keyword-like tokens is out of scope here, but verify there are no name clashes when a user types `Person p = /persons` — `Person` must be an `identifier`, not any reserved DRLX token.

Older gotchas from #11 (lexer token name collision, ANTLR post-import line numbers) remain applicable to any further grammar work.

## §9 — Explicit non-goals

- OR-branch consensus-rule enforcement (same-name on every branch → outside-visible).
- Shadowing detection / warning when same name binds at two nesting levels.
- Passive `?/` bindings (`and(var l : ?/locations, ...)`) — part of #10.
- New `TolerantDrlxToJavaParserVisitorTest` coverage for bound group children.
- Validation-error catalogue improvements — DRLX continues to defer to Drools errors.
- `$`-prefixed identifier support (DRL10 convention) — separate ticket if the need arises.
