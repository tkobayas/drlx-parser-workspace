---
issue: tkobayas/drlx-parser#12
status: design
date: 2026-04-27
depends_on: tkobayas/drlx-parser#23
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

**Out of scope (split to #22):** Form B — multiple per-branch consequences interleaving patterns and statements (DRLXXXX line 274). Form B requires either rule decomposition or auto-named consequences and is materially harder.

**Prerequisite (now first):** #23 — DRLXXXX `test` construct (`EvalIR` proto + IR + runtime + lambda compiler entry point + user-facing grammar). #12 was originally going to ship `EvalIR` internal-only and split the user-facing grammar to #23. After investigating drools' `EvalExpression` API, we restructured: #23 ships first as the full `test` feature (independently useful, smaller blast radius for the riskiest bridging code), and #12 becomes pure grammar + desugar to existing `or` + `and` + `EvalIR`.

## Runtime target

Form A maps to existing drools-core constructs only — **no drools-core changes**. The visitor desugars `if`/`else if`/`else` into the already-shipped `or` + `and` + `EvalIR` (from #23):

```
if (c1) { B1 } else if (c2) { B2 } else { B3 }
       ↓ (visitor desugar)
or(
  and( EvalIR(c1),                  B1... ),
  and( EvalIR(!(c1)), EvalIR(c2),    B2... ),
  and( EvalIR(!(c1)), EvalIR(!(c2)), B3... ),
)
```

Cumulative guards are split into multiple sequential `EvalIR` nodes (one per prior-condition negation + own condition), per the "multiple `EvalIR`s" decision in plan-phase open items. Simpler to build than compound `&&` expressions, equivalent at runtime.

## Architecture

**Approach: visitor-level desugar (sugar over `or` + `EvalIR`).**

Grammar parses `if`/`else`. Parse-tree → IR step in `DrlxToRuleAstVisitor` immediately desugars to `GroupElementIR(OR)` of `GroupElementIR(AND, [EvalIR, …, body…])` per branch.

What's added by this ticket:
- Grammar: new `conditionalBranch` production.
- Visitor: new `buildConditionalBranch` method (the desugar).

What is **not** added (lives in #23):
- `EvalIR` IR class.
- `EvalParseResult` proto message and round-trip.
- `EvalCondition` runtime mapping (in `DrlxRuleAstRuntimeBuilder`).
- `DrlxLambdaCompiler.createEvalExpression`.
- User-facing `test` grammar.

What is *also* not added: no first-class IR class for `if/else` (downstream sees the desugared `or`-tree); no new `RuleConditionElement` builder for branching; no drools-core changes.

**Trade-off accepted:** the user's `if/else` intent is lost after the visitor. Acceptable today — DRLX has no diagnostic surface that benefits from preserving the original form. If Form B or richer diagnostics need it later, promote to first-class IR then.

## Components

### 1. Grammar (`DrlxParser.g4`, `DrlxLexer.g4`)

Two new tokens: `IF`, `ELSE`. New production added as a `ruleItem` alternative:

```antlr
ruleItem
    : rulePattern
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','           // shipped by #23
    | conditionalBranch ','     // NEW (#12)
    | ruleConsequence
    ;

conditionalBranch
    : IF '(' expression ')' branchBody
      ( ELSE IF '(' expression ')' branchBody )*
      ( ELSE branchBody )?
    ;

branchBody
    : '{' ruleItem* '}'
    ;
```

Notes:
- `branchBody` reuses `ruleItem*` — branches accept patterns, group elements, nested `if/else`, and `test` automatically. Implicit AND between items handled by visitor.
- Trailing `,` after the close brace per DRLXXXX convention (uniform with all CE terminators).
- `expression` is the existing DRLX expression rule (`DrlxParser.g4:150`, inherited from MVEL3).
- `else if` is grammar-level chaining (the `( ELSE IF … )*` clause), not lexer-level — keeps the AST flat (a branch list) instead of right-nested else-trees.

### 2. Parse-tree → IR desugar (`DrlxToRuleAstVisitor`)

New method `buildConditionalBranch(ConditionalBranchContext)` that returns a `GroupElementIR(OR)` containing one `GroupElementIR(AND)` per branch. Algorithm:

1. Collect branches in document order: `(c1, B1), (c2, B2), …, (cn, Bn)`. Final `else`, if present, contributes `(null, Bfinal)`.
2. For each branch `(ci, Bi)` at index `i`:
   - Emit one `EvalIR("!(prior_j)")` for each prior condition, in order (j = 1..i-1).
   - Then emit `EvalIR(ci)` (own condition; omitted for the final-else branch).
   - Then recursively expand `Bi` as additional AND children.
   - Wrap all of the above in `GroupElementIR(AND, [...])`.
3. Wrap all branch ANDs in a single `GroupElementIR(OR)`.

Recursive expansion of `Bi` reuses existing visitor methods (`buildPattern`, `buildNotElement`, etc.), so nested `not`/`exists`/`and`/`or`, nested `if/else`, and `test` are handled by recursion without special cases.

`EvalIR.referencedBindings` is populated by extracting identifiers from the expression text (regex over `\b[A-Za-z_][A-Za-z0-9_]*\b`); the runtime builder filters them against the live `boundVariables` map (those that don't resolve are silently dropped, matching the existing `findReferencedBindings` behavior in `DrlxRuleAstRuntimeBuilder`).

### 3. Bindings & scope

Inherit existing `or` semantics — no new logic in #12:

- **Same name across branches** — if `var p` is referenced after the if-block, every branch must bind `p`. Drools' `or` decomposes into one subrule per branch; each subrule's `p` flows independently into the consequence. Asymmetric branches where `p` is referenced post-block → compile error from existing `or` checks.
- **Same type across branches** — required by drools-core `or`; we inherit.
- **Outer-scope bindings** — guards and branch bodies freely reference bindings from preceding rule items. The `EvalIR` carries those bindings as `referencedBindings`.
- **Branch-local bindings** — bindings introduced inside a branch but not in others are not visible post-block (standard `or` rule).

### 4. Edge cases

| Case | Behavior |
|---|---|
| `if (cond) { /a }` (no else) | Desugars to `or(and(EvalIR(cond), /a))`. Rule fires only when `cond` matches. No implicit else. |
| `if (c1) {…} else if (c2) {…}` (no final else) | N branches, cumulative guards. Rule doesn't fire if no branch matches. |
| Empty branch `if (cond) { }` | Reject at parse-tree → IR step. Error: `"empty branch body"`. |
| Branch with only group element `if (cond) { not(/a) }` | Allowed; guard `EvalIR` is the first child of the AND-group, the `not` IR follows. |
| Nested `if/else` inside a branch | Allowed; recursion handled naturally. Guards compose: outer's AND wraps inner's OR. |
| Side-effecting guard expression | No special guarantee — same as any DRLX constraint expression. Out of scope. |

### 5. Property reactivity

DRLX configures patterns with `PropertySpecificOption.ALWAYS` (`DrlxRuleAstRuntimeBuilder.java:105`), so all properties are watched by default. A guard like `if (c.creditRating == Rating.LOW) { … }` re-evaluates naturally on `creditRating` updates — no explicit watch list required.

This is the empirical behavior verified in #23's `TestElementTest.test_refiresOnPropertyUpdate` and inherited by #12. Note that drools-base's `EvalCondition.isPatternScopeDelimiter() == true` does *not* inhibit re-evaluation in this configuration; the pattern's ALWAYS-mode watched-property mask covers all properties regardless of where they're referenced.

If DRLX ever switches to `ALLOWED` mode (requiring `@PropertyReactive` annotations or explicit watch lists), this section will need revisiting — guard expressions would need to contribute their property reads to preceding patterns' masks (similar in spirit to #19's `Evaluator.getReadProperties()`).

## Testing

New test classes in `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/`:

**Parse-level (`IfElseParseTest`):**
- `if {…}` only
- `if/else`
- `if/else if/else`
- Nested if/else inside branch
- Trailing `,` enforced
- Parse failures: missing `{`, empty branch
- IR-shape assertion: verify the desugared `OR(AND(EvalIR, …, body), …)` structure

**Runtime semantics (`IfElseTest`, using `TrackingAgendaEventListener`):**
1. Binary if/else — DRLXXXX A-form example (LOW/HIGH, LOW/HIGH cases).
2. Else-if chain — three-rating example.
3. No-final-else — rule does not fire when no branch's guard holds.
4. Same-name bindings across branches — consequence reads the bound name regardless of branch.
5. Multi-element branch — implicit AND, cross-pattern join inside branch.
6. Nested if/else.
7. Group element inside branch — e.g., `not(/widget)` plus a pattern.
8. Property reactivity — guard re-evaluates on outer-scope binding's property update (covered by ALWAYS-mode default; no explicit watch list needed).

Target: ~8–10 runtime tests + parse tests. Smaller than the original #12 scope because EvalIR-direct tests and EvalExpression bridging tests live with #23.

## Resolved plan-phase items

- **`expression` reuse:** verified — `DrlxParser.g4:150` defines `expression` (inherited from MVEL3), already used by `drlxExpression`. Composes inside `IF '(' expression ')'` without ambiguity.
- **Guard cumulation form:** decided — split into multiple sequential `EvalIR`s (one per prior-condition negation + own condition). Simpler than compound `&&`. Equivalent at runtime since each `EvalCondition` becomes its own AND child.
- **`else if` token strategy:** decided — two separate tokens (`ELSE IF`), grammar-level chaining via `( ELSE IF … )*` repetition.
- **`EvalIR` runtime mapping:** moved to #23 (was item #2's `Plan-phase open items` — "lambda generation"). Verified API findings:
  - `EvalExpression` interface: `boolean evaluate(BaseTuple, Declaration[], ValueResolver, Object) throws Exception`, plus `Object createContext()`, `replaceDeclaration`, `clone`.
  - Tuple value extraction: `tuple.getObject(declaration)` (BaseTuple line 51).
  - MVEL3 compile pattern: mirror `DrlxLambdaCompiler.createBatchBetaConstraint` — `MVEL.map(...).out(Boolean.class).expression(...)` with `LambdaHandle`/`PendingLambda` deferred-resolution flow.

## Related issues

- **#22** — Form B (multi-consequence `if/else`). Requires rule decomposition or auto-named consequences. Postponed.
- **#23** — DRLXXXX `test` construct (now expanded scope: ships full `EvalIR` infra + user-facing grammar). Prerequisite for #12.

## References

- DRLXXXX § "if" — lines 727–751 in `docs/DRLXXXX.md`
- DRLXXXX § "test" — lines 720–725 in `docs/DRLXXXX.md`
- And/Or spec — `specs/2026-04-23-and-or-design.md`
- Group-element infrastructure — `specs/2026-04-21-group-element-and-not-design.md`
- Drools-core `ConditionalBranch` (NOT used — Form A only) — `drools-base/src/main/java/org/drools/base/rule/ConditionalBranch.java`
- `EvalCondition.isPatternScopeDelimiter` — `drools-base/src/main/java/org/drools/base/rule/EvalCondition.java:199-201`
