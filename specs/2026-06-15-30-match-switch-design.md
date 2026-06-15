# #30 Match (switch) conditional element

**Issue:** #30
**Epic:** #79 (Conditional branching & named windows)
**DRLXXXX reference:** Lines 754–776

## Summary

Add `match` as a switch-like conditional element that desugars to
an if/else-if chain of synthetic rules (Form B only — per-case consequences).
No new Drools runtime APIs required.

## Design decisions

| Decision | Choice |
|----------|--------|
| Keyword | `match` (new DrlxLexer token) |
| Form | Form B only (per-case consequences) |
| Case body forms | Block `{ ... }` and single-expression (`do stmt` / bare expr) |
| Type-match desugaring | Eval-based (`instanceof` + cast in constraint expressions) |
| Catch-all keyword | `default` only (no `else`) |
| Approach | Dedicated `buildMatchBranch` visitor method (parallel to `buildConditionalBranchFormB`) |

## Grammar

New token in `DrlxLexer.g4`:

```
MATCH : 'match';
```

New rules in `DrlxParser.g4`:

```antlr
matchBranch
    : MATCH '(' expression ')' matchCase+ matchDefault?
    ;

matchCase
    : CASE matchPattern matchCaseBody
    ;

matchDefault
    : DEFAULT matchCaseBody
    ;

matchPattern
    : HASH identifier ('[' drlxExpression (',' drlxExpression)* ']')?
    | expression
    ;

matchCaseBody
    : '{' (branchItem (',' branchItem)*)? '}'
    | DO statement
    | expression
    ;
```

Integration into existing rules:

```antlr
ruleItem
    : ...existing alternatives...
    | matchBranch ','?
    ;

branchItem
    : ...existing alternatives...
    | matchBranch
    ;
```

## Visitor — condition generation

Each `matchCase` generates an equality or instanceof condition
from the match subject expression and the case pattern.

**Value match:** `case Rating.LOW` with subject `c.creditRating`
→ `EvalIR("c.creditRating == Rating.LOW")`

**Type match:** `case #Train` with subject `o`
→ `EvalIR("o instanceof Train")`

**Type match with constraints:** `case #Automobile[value > 30]` with subject `o`
→ `EvalIR("o instanceof Automobile && ((Automobile)o).value > 30")`

**Multiple constraints:** `case #Automobile[value > 30, color == "red"]` with subject `o`
→ `EvalIR("o instanceof Automobile && ((Automobile)o).value > 30 && ((Automobile)o).color == \"red\"")`

**Default:** No condition — just cumulative negation of all prior cases.

The match subject expression text is captured once via `getText()` and
reused across all cases.

## Visitor — rule decomposition

`buildMatchBranch` produces a `List<RuleIR>`, one synthetic rule per case,
following the same pattern as `buildConditionalBranchFormB`.

### Example

```
rule R1 {
    var c : /customers,
    match (c.creditRating)
        case Rating.LOW {
            var p : /products[rate == Rates.HIGH],
            do System.out.println("Low")
        }
        case Rating.MEDIUM {
            var p : /products[rate == Rates.MED],
            do System.out.println("Med")
        }
        default { do System.out.println("Other") }
}
```

Produces three synthetic rules:

- **R1$0:** `commonPrefix` + `EvalIR("c.creditRating == Rating.LOW")` +
  branch patterns → consequence `System.out.println("Low");`

- **R1$1:** `commonPrefix` + `EvalIR("!(c.creditRating == Rating.LOW)")` +
  `EvalIR("c.creditRating == Rating.MEDIUM")` +
  branch patterns → consequence `System.out.println("Med");`

- **R1$2:** `commonPrefix` + `EvalIR("!(c.creditRating == Rating.LOW)")` +
  `EvalIR("!(c.creditRating == Rating.MEDIUM)")` → consequence
  `System.out.println("Other");`

### Validation rules

- Each case body must contain at least one consequence.
- Match must be the last `ruleItem` (no items after it).
- Cannot coexist with a trailing `do` consequence.

## Integration points

**`buildRule` in `DrlxToRuleAstVisitor`:**
Add a `matchBranch` branch alongside the existing `conditionalBranch`
handling. Detect it, collect common prefix, delegate to `buildMatchBranch`.

**`buildBranchItem` in `DrlxToRuleAstVisitor`:**
Add a `matchBranch` case for nesting match inside if/else branches.

## Testing

### Parse tests (`MatchParseTest`)

- Value match (single-expression and block forms)
- Type match (`#Type`)
- Type match with constraints (`#Type[constraint]`)
- Default clause
- No default
- Trailing comma optional

### IR tests (`MatchIRTest`)

- Correct equality/instanceof conditions
- Cumulative negation guards
- Correct number of synthetic rules
- Validation errors (empty body, trailing `do` conflict)

### Runtime tests (`MatchTest`)

- Value match selects correct case
- Type match with instanceof
- Default fires when no case matches
- Multiple patterns within a case body
