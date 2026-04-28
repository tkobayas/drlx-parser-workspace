# DRLX Syntax Implementation Candidates

Candidates for enhancing DrlxCompiler to cover more DRLX syntax, prioritized by implementation difficulty.

## Tier 1 — Truly easy (quick wins, no Rete topology changes)

| # | Feature | Why easy |
|---|---------|----------|
| 1 | **Inline cast** (`/objects#Car[speed > 80]`) | Grammar already parses `#identifier` in oopathChunk — just needs visitor to use it as a type filter |
| 2 | ~~**Rule annotations/attributes** (`@salience`, `@no-loop`, `@agenda-group`)~~ — **ON HOLD** | `docs/DRLXXXX.md` doesn't list these DRL7 attributes; it specifies a different set (`@Description`, `@ExistanceDriven`, `@Tabled`, `@DataSource`, field-level `@Position`). Pending clarification with spec designer on which annotations DRLX should support. |
| 3 | **Positional syntax** (`/locations("paris")`) | Grammar addition + visitor maps positional args to fields by index |

## Tier 2 — Medium (requires GroupElement infrastructure)

| # | Feature | Key challenge |
|---|---------|---------------|
| 4 | **Generalize ruleBody for nested group elements** | Prerequisite infra — redesign proto + visitor to support tree structure instead of flat list |
| 5 | **`not /pattern`** | Wraps pattern in `GroupElement(NOT)`, needs scope-aware binding tracking |
| 6 | **`exists /pattern`** | Same infra as `not`, wraps in `GroupElement(EXISTS)` |
| 7 | **Passive patterns** (`?/persons[...]`) | Easy grammar, but verify Drools `Pattern` supports a passive/non-reactive flag |

## Tier 3 — Harder

| # | Feature | Key challenge |
|---|---------|---------------|
| 8 | **`or()`/`and()` group CEs** | Requires subnetwork support and variable scoping |
| 9 | ~~**`if`/`else` branching in rule body** (Form A — single trailing consequence)~~ — **IMPLEMENTED in #12**. Form B (per-branch consequences) tracked in #22. | Conditional elements within rule body |
| 10 | **Property reactive** (`[][basePay, bonusPay]`) | Second `[]` block for watch list |

## Notes

- "Multiple constraints in `[]`" is already implemented (grammar + visitor handle comma-separated constraints).
- `not`/`exists` touch 4 layers: grammar, visitor, proto, and runtime builder — they should not be attempted before the GroupElement infrastructure (item 4) is in place.
- The `findReferencedBindings()` regex in `DrlxRuleAstRuntimeBuilder` will need scope-aware binding tracking once `not`/`exists` are added (variables bound inside `not` must not be visible outside).
- **`test` (DRLXXXX § "test", lines 720–725)** — IMPLEMENTED in #23. Eval-style guard, foundation for `if`/`else` (#12). Adds `EvalIR` to the LHS-item sealed hierarchy and bridges MVEL3 boolean lambdas to drools-base `EvalCondition`.
