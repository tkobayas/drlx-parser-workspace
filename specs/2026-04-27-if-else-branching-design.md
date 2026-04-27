---
issue: tkobayas/drlx-parser#12
status: design
date: 2026-04-27
---

# `if` / `else` branching in rule body — design (Form A)

## Scope

Implements the **single-consequence form (Form A)** of DRLXXXX `if`/`else` (DRLXXXX § "if", lines 727–751):

```
rule R1 {
   var c : /customer,
   if (c.creditRating == Rating.LOW) {
      var p : /products[rate == Rates.HIGH]
   } else if (c.creditRating == Rating.MEDIUM) {
      var p : /products[rate == Rates.MEDIUM]
   } else {
      var p : /products[rate == Rates.LOW]
   },
   do System.out.println(c + " " + p);
}
```

Branches are LHS pattern blocks; a single trailing `do` consequence is the only RHS.

**Out of scope (split to a separate issue):** Form B — multiple per-branch consequences interleaving patterns and statements (DRLXXXX line 274). Form B requires either rule decomposition or auto-named consequences and is materially harder; tracked separately.

## Runtime target

Form A maps to existing drools-core constructs only — **no drools-core changes**. The visitor desugars `if`/`else if`/`else` into the already-shipped `or` + `and` + (new internal) eval-style IR:

```
if (c1) { B1 } else if (c2) { B2 } else { B3 }
       ↓ (visitor desugar)
or(
  and( test(c1),                B1... ),
  and( test(!c1 && c2),         B2... ),
  and( test(!c1 && !c2),        B3... ),
)
```

The `test` IR is the planned DRLXXXX equivalent of legacy `eval` (DRLXXXX § "test", line 720). #12 ships the IR + proto + runtime mapping for `test` (internal-only — used by the if/else desugar). User-facing grammar exposure for `test` is split to a sibling issue.

## Architecture

**Approach: visitor-level desugar (sugar over `or` + `test`).**

Grammar parses `if`/`else`. Parse-tree → IR step in `DrlxToRuleAstVisitor` immediately desugars to `GroupElementIR(OR)` of `GroupElementIR(AND, [test, body…])` per branch.

What's added by this ticket:
- Grammar: new `conditionalBranch` production.
- Visitor: new `buildConditionalBranch` method (the desugar).
- IR + proto + runtime: new `EvalIR` (the `test` construct from DRLXXXX § "test"). Internal-only — used by the desugar; user-facing grammar exposure is split to a sibling issue.

What is *not* added: no first-class IR class for `if/else` itself (downstream sees the desugared `or`-tree); no new `RuleConditionElement` builder for branching; no drools-core changes.

**Trade-off accepted:** the user's `if/else` intent is lost after the visitor. Acceptable today — DRLX has no diagnostic surface that benefits from preserving the original form. If Form B or richer diagnostics need it later, promote to first-class IR then.

## Components

### 1. Grammar (`DrlxParser.g4`)

Two new tokens: `IF`, `ELSE`. New production added as a `ruleItem` alternative:

```antlr
ruleItem
    : rulePattern
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | conditionalBranch ','     // NEW
    | ruleConsequence
    ;

conditionalBranch
    : IF LPAREN expression RPAREN branchBody
      ( ELSE IF LPAREN expression RPAREN branchBody )*
      ( ELSE branchBody )?
    ;

branchBody
    : LBRACE ruleItem* RBRACE
    ;
```

Notes:
- `branchBody` reuses `ruleItem*` — branches accept patterns, group elements, and nested `if/else` automatically. Implicit AND between items handled by visitor.
- Trailing `,` after the close brace per DRLXXXX convention (uniform with all CE terminators).
- `expression` is the existing DRLX expression rule (same one used inside `[…]` constraints).
- `else if` is grammar-level chaining (the `( ELSE IF … )*` clause), not lexer-level — keeps the AST flat (a branch list) instead of right-nested else-trees.

### 2. Parse-tree → IR desugar (`DrlxToRuleAstVisitor`)

New method `buildConditionalBranch(ConditionalBranchContext)` that returns a `GroupElementIR(OR)` containing one `GroupElementIR(AND)` per branch. Algorithm:

1. Collect branches in document order: `(c1, B1), (c2, B2), …, (cn, Bn)`. Final `else`, if present, contributes `(null, Bfinal)`.
2. For each branch `(ci, Bi)` at index `i`:
   - Compute the cumulative guard: `(NOT c1) AND (NOT c2) AND … AND (NOT c_{i-1}) AND ci` (final-else branch omits the trailing `ci` term).
   - Emit `GroupElementIR(AND, [ EvalIR(guard), …recursive build of Bi… ])`.
3. Wrap all branch ANDs in a single `GroupElementIR(OR)`.

Recursive expansion of `Bi` reuses existing visitor methods, so nested `not`/`exists`/`and`/`or` and nested `if/else` are handled by recursion without special cases.

### 3. `test` (eval) IR — new internal construct

Three small additions, used internally by the if/else desugar; **not exposed in user grammar** in this ticket.

**a. Proto message** (`drlx_rule_ast.proto`):

```protobuf
message EvalParseResult {
  string expression = 1;        // raw expression text
  repeated string referenced_bindings = 2;  // declarations consumed by the expression
}

message LhsItemParseResult {
  oneof kind {
    PatternParseResult pattern = 1;
    GroupElementParseResult group = 2;
    EvalParseResult eval = 3;   // NEW
  }
}
```

**b. IR class** (`EvalIR`): mirrors `PatternIR`'s shape — holds the expression string and the set of binding names it references. Implements the same `LhsItemIR` interface so it slots into `GroupElementIR.children`.

**c. Runtime builder** (`DrlxRuleAstRuntimeBuilder`): when a branch's children include an `EvalIR`, emit drools-model `D.expr(declarations, lambda)` (the executable-model API for eval-style guards). The lambda is generated from the expression string the same way `DrlxLambdaCompiler` already generates lambdas for pattern constraints, except it returns `boolean` and takes the referenced bindings as input.

A standalone unit test (`EvalIRBuilderTest`) verifies the runtime path independently of if/else, since the if/else flow is the only user of this IR until the sibling `test`-grammar ticket lands.

### 4. Bindings & scope

Inherit existing `or` semantics — no new logic in #12:

- **Same name across branches** — if `var p` is referenced after the if-block, every branch must bind `p`. Drools' `or` decomposes into one subrule per branch; each subrule's `p` flows independently into the consequence. Asymmetric branches where `p` is referenced post-block → compile error from existing `or` checks.
- **Same type across branches** — required by drools-core `or`; we inherit.
- **Outer-scope bindings** — guards and branch bodies freely reference bindings from preceding rule items. The `EvalIR` carries those bindings as `referencedBindings`.
- **Branch-local bindings** — bindings introduced inside a branch but not in others are not visible post-block (standard `or` rule).

### 5. Edge cases

| Case | Behavior |
|---|---|
| `if (cond) { /a }` (no else) | Desugars to `or(and(test(cond), /a))`. Rule fires only when `cond` matches. No implicit else. |
| `if (c1) {…} else if (c2) {…}` (no final else) | N branches, cumulative guards. Rule doesn't fire if no branch matches. |
| Empty branch `if (cond) { }` | Reject at parse-tree → IR step. Error: `"empty branch body"`. |
| Branch with only group element `if (cond) { not(/a) }` | Allowed; guard `EvalIR` is the first child of the AND-group, the `not` IR follows. |
| Nested `if/else` inside a branch | Allowed; recursion handled naturally. Guards compose: outer's AND wraps inner's OR. |
| Side-effecting guard expression | No special guarantee — same as any DRLX constraint expression. Out of scope. |
| `test` IR with no referenced bindings | Valid — pure boolean expression like `test true`. Lambda has no inputs. |

## Testing

New test class `IfElseTest` in `drlx-parser-core/src/test/java/…`, mirroring `OrTest` / `NotTest`:

**Parse-level (`IfElseParseTest`):**
- `if {…}` only
- `if/else`
- `if/else if/else`
- Nested if/else inside branch
- Trailing `,` enforced
- Parse failures: missing `{`, empty branch

**Round-trip:** Verify desugared IR serializes/deserializes correctly via proto.

**Runtime semantics (`IfElseTest`, using `TrackingAgendaEventListener`):**
1. Binary if/else — DRLXXXX A-form example.
2. Else-if chain — three-rating example.
3. No-final-else — rule does not fire when no branch's guard holds.
4. Same-name bindings across branches — consequence reads the bound name regardless of branch.
5. Asymmetric bindings referenced post-block — compile error.
6. Outer-scope binding referenced in guard.
7. Multi-element branch — implicit AND, cross-pattern join inside branch.
8. Nested if/else.
9. Group element inside branch — e.g., `not(/widget)` plus a pattern.
10. Property-reactive interaction — watch lists propagate through branch patterns.

**`EvalIR` direct (`EvalIRBuilderTest`):**
- Construct an `EvalIR` programmatically (no grammar), build a rule, fire — verifies the runtime mapping in isolation.

Target: ~10–12 runtime tests + parse tests. Volume comparable to the and/or spec round.

## Plan-phase open items

- **`expression` reuse** — confirm the existing `expression` grammar rule is fully composable inside `IF LPAREN expression RPAREN` without left-recursion / ambiguity issues. Verify with a small ANTLR test before committing the grammar.
- **`EvalIR` lambda generation** — pick the cleanest reuse of `DrlxLambdaCompiler` for boolean-returning lambdas. Likely a new entry point that takes the expression text and a binding-set, returns a `Function<…, Boolean>` builder fragment.
- **Guard cumulation expression form** — desugar emits cumulative `!c1 && !c2 && c3` as one expression, or splits into multiple chained `EvalIR` nodes (`test(!c1), test(!c2), test(c3)`)? Multiple `EvalIR`s are simpler to build (no expression-tree rewriting) and equivalent at runtime; pick this in the plan unless a benchmark shows a meaningful cost.
- **`else if` token strategy** — emit `ELSE IF` as two separate tokens (current grammar) or introduce an `ELSE_IF` lexer rule? Two tokens is simpler; revisit only if grammar disambiguation needs it.

## Sibling issue (to open)

`test` user-facing grammar — expose `test <expression>` as a `ruleItem` for direct use in rule bodies. IR + runtime already shipped by #12; this ticket adds the grammar + visitor branch + parse and runtime tests. Small (~½ day).

## References

- DRLXXXX § "if" — lines 727–751 in `docs/DRLXXXX.md`
- DRLXXXX § "test" — lines 720–725 in `docs/DRLXXXX.md`
- And/Or spec — `specs/2026-04-23-and-or-design.md`
- Group-element infrastructure — `specs/2026-04-21-group-element-and-not-design.md`
- Drools-core `ConditionalBranch` (NOT used — Form A only) — `drools-base/src/main/java/org/drools/base/rule/ConditionalBranch.java`
