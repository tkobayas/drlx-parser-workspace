---
issue: tkobayas/drlx-parser#22
status: design
date: 2026-06-11
depends_on: tkobayas/drlx-parser#12
---

# Form B: `if`/`else` with per-branch consequences — design

## Scope

Implements the **multi-consequence form (Form B)** of DRLXXXX `if`/`else` (DRLXXXX line 274): each branch interleaves patterns *and* action statements, with no shared trailing `do`.

```
rule EmailPeople {
    var l : /locations,
    if (l.location == "paris") {
        var p : /persons[locationId == l.id],
        do Email.to(p.emailAddress).body("paris message").send()
    } else if (l.location == "london") {
        var p : /persons[locationId == l.id],
        do Email.to(p.emailAddress).body("london message").send()
    }
}
```

Per DRLXXXX, `do` is optional for single-statement consequences:

```
rule EmailPeople {
    var l : /locations,
    if (l.location == "paris") {
        var p : /persons[locationId == l.id],
        Email.to(p.emailAddress).body("paris message").send()
    } else if (l.location == "london") {
        var p : /persons[locationId == l.id],
        Email.to(p.emailAddress).body("london message").send()
    }
}
```

**Prerequisite:** Form A (#12) — landed.

## Compiler strategy: rule decomposition

Form B compiles to N synthetic `RuleImpl` objects, one per branch. Each synthetic rule gets the common LHS prefix, cumulative EvalIR guards, branch-specific patterns, and the branch's own consequence.

```
rule EmailPeople { ... if (paris) { P1, do C1 } else if (london) { P2, do C2 } }
    ↓ compiles to
RuleImpl "EmailPeople$0":
  LHS = [/locations, EvalIR("l.location == \"paris\""),              P1]
  RHS = C1

RuleImpl "EmailPeople$1":
  LHS = [/locations, EvalIR("!(l.location == \"paris\")"),
         EvalIR("l.location == \"london\""),                         P2]
  RHS = C2
```

### Why rule decomposition

Three approaches were evaluated:

1. **Rule decomposition** — synthesize N `RuleImpl` objects. No drools-core changes.
2. **OR desugar + ConditionalBranch hybrid** — reuse Form A's OR desugar for patterns, add `ConditionalBranch` for consequence dispatch. Each guard is evaluated twice (once in EvalIR for LHS matching, once in ConditionalBranch for dispatch). Needs new IR types and complex interaction between OR decomposition and ConditionalBranch terminal nodes.
3. **"Fat" ConditionalBranch** — extend drools-core's `ConditionalBranch` to support embedded sub-networks (branch-specific patterns). Significant drools-core changes.

Rule decomposition was chosen because:
- No drools-core changes — keeps the dependency boundary clean.
- Reuses all existing infrastructure (cumulative guards, EvalIR, ConsequenceIR).
- Each synthetic rule is standard — nothing novel at runtime.
- Consistent with how drools itself decomposes `or` into subrules.
- RETE node sharing automatically optimizes the duplicated common prefix.

Trade-off accepted: `no-loop` applies per synthetic rule, not per original rule. If branch 0's consequence modifies a fact making branch 1's guard true, branch 1 could fire. This scenario (consequence modifying the guard fact to trigger a different branch) is unusual and the behavior is documented via a code comment in the visitor.

## Architecture

### 1. Grammar (`DrlxParser.g4`)

Extend `branchItem` to accept consequences:

```antlr
branchItem
    : boundOopath
    | oopathExpression
    | notElement
    | existsElement
    | andElement
    | orElement
    | testElement
    | conditionalBranch
    | branchConsequence          // NEW
    ;

branchConsequence
    : DO statement               // explicit: do Email.send(p)
    | statement                  // bare:     Email.send(p)
    ;
```

Disambiguation (ANTLR4 adaptive LL(\*), 2-4 token lookahead):
- `boundOopath` requires `identifier identifier (':' | '=')` — two identifiers then `:` or `=`.
- `oopathExpression` starts with `/` or `?/`.
- Keyword-led elements start with `NOT`, `EXISTS`, `IF`, `TEST`, `DRLX_AND`, `DRLX_OR`.
- `branchConsequence` with `DO` starts with `DO`.
- `branchConsequence` bare starts with `identifier '.'` or `identifier '('` — never matches the pattern prefix.

`branchBody` rule unchanged — `,` as separator, `{}` delimiters.

### 2. Form detection

The visitor inspects each branch body's items. If ANY branch body contains a `branchConsequence`, the entire `conditionalBranch` is Form B. Otherwise Form A (existing desugar to OR).

### 3. Visitor: rule decomposition (`DrlxToRuleAstVisitor`)

New method `buildConditionalBranchFormB(ConditionalBranchContext, List<LhsItemIR> commonPrefix, List<RuleAnnotationIR> annotations, List<RuleParameterIR> parameters, String ruleName)` returns `List<RuleIR>`.

Algorithm:
1. Collect branches in document order: `(guard_i, body_i)` for `i = 0..N-1`. Final `else`, if present, has `guard = null`.
2. For each branch `i`:
   - Partition `body_i` items into LHS items (patterns, group elements, nested conditionals) and RHS items (`branchConsequence`s). Validate that all RHS items come after all LHS items.
   - Build cumulative guards — same logic as Form A (`buildConditionalBranch`): one `EvalIR("!(prior_j)")` per prior condition, then `EvalIR(guard_i)` for own condition (omitted for final else).
   - Build LHS: `commonPrefix + guards + branch LHS items`.
   - Build RHS: combine all `branchConsequence` items into one `ConsequenceIR` block.
   - Construct `RuleIR("{ruleName}${i}", annotations, parameters, lhs, rhs)`.
3. Return the list of synthetic `RuleIR` objects.

The existing `buildConditionalBranch` (Form A) is unchanged — Form A detection falls through to the existing code path.

**Caller change:** `buildRule()` (line 118) currently returns `RuleIR`. It changes to return `List<RuleIR>`. The calling code in `visitDrlxCompilationUnit` (line 107) changes from `rules.add(buildRule(...))` to `rules.addAll(buildRule(...))`. Form A rules return a singleton list — no behavioral change for existing rules.

### 4. IR model (`DrlxRuleAstModel`)

No new types. `RuleIR` and `ConsequenceIR` are unchanged. Rule decomposition creates multiple `RuleIR` instances from one parse tree node using existing record types.

### 5. Runtime builder (`DrlxRuleAstRuntimeBuilder`)

No changes. Each synthetic `RuleIR` flows through the existing `buildRule()` method (line 397) independently. `CompilationUnitIR.rules()` already holds `List<RuleIR>` — it naturally accommodates the additional synthetic rules.

### 6. Naming & attributes

**Naming:** `{originalName}${branchIndex}` — e.g., `EmailPeople$0`, `EmailPeople$1`. The `$` mirrors Java's synthetic inner-class convention.

**Rule attributes:** All annotations from the original rule are duplicated to every synthetic rule (salience, agenda-group, no-loop, etc.).

## Validation rules

| Condition | Error |
|---|---|
| Form B branch with no `branchConsequence` | "Form B branch must contain at least one action statement; use Form A with a trailing `do` for pattern-only branches" |
| Form B rule with trailing `ruleConsequence` (rule-level `do`) | "Rule with per-branch consequences cannot also have a trailing `do` consequence" |
| `branchConsequence` before LHS items in a branch | "Action statements must follow all patterns and group elements in a branch body" |
| Nested Form B `conditionalBranch` inside a Form B branch | "Nested per-branch consequences are not supported; extract the inner if/else into a separate rule" |
| Empty branch body | Rejected (same as Form A) |

## Edge cases

| Case | Behavior |
|---|---|
| No final `else` | Only matching branches generate synthetic rules. No match = no rule fires. |
| Single `if` (no else) | One synthetic rule with guard. |
| Multiple `branchConsequence` items in one branch | Combined into a single `ConsequenceIR` block (semicolon-separated). |
| Bare expression + `do` in same branch | Both are `branchConsequence` items; combined. |
| Nested Form B inside Form B | Deferred — compile error. Recursive decomposition (inner Form B produces `List<RuleIR>` that the outer branch must multiply with its own guards) adds significant complexity for a rare use case. Revisit if real demand arises. |
| Nested Form A inside Form B | Inner Form A desugars to OR (existing). The OR group becomes an LHS item in each synthetic rule. |
| Items after the `conditionalBranch` in `ruleItem` list | Error — Form B's `conditionalBranch` must be the last LHS item. |

## Testing

### Parse-level (`IfElseFormBParseTest`)

1. Two-branch Form B — verify 2 `RuleIR` objects with correct names (`R$0`, `R$1`)
2. `if`/`else if`/`else` — three synthetic rules, cumulative guards
3. Single `if` (no else) — one synthetic rule
4. Bare expression consequence (no `do`) — parses correctly
5. Explicit `do` consequence — parses correctly
6. Mixed bare + `do` in same branch — combined into one `ConsequenceIR`
7. Multiple consequence statements in one branch — combined correctly
8. LHS/RHS partitioning — patterns and group elements correctly separated from consequences
9. Error: Form B branch with no consequence
10. Error: Form B + trailing `do` at rule level
11. Error: consequence before pattern in branch

### Runtime (`IfElseFormBTest`, `TrackingAgendaEventListener`)

1. Binary if/else — correct branch fires, correct consequence executes
2. Else-if chain — three branches, correct dispatch per guard
3. No-final-else — rule doesn't fire when no branch matches
4. Branch-specific bindings — consequence accesses both common-prefix and branch-local bindings
5. Multiple consequence statements — all execute in order
6. Bare expression consequence (no `do`) — fires correctly
7. Property reactivity — guard re-evaluates on fact update (ALWAYS-mode default)
8. Rule attributes — salience/agenda-group honored on synthetic rules
9. Nested Form B inside Form B — compile error with clear message

## Related issues

- **#12** — Form A (single-consequence `if/else`). Prerequisite — landed.
- **#23** — `test` construct (`EvalIR` infra). Prerequisite — landed.

## References

- Form A spec: `specs/2026-04-27-if-else-branching-design.md`
- DRLXXXX line 274 — multi-consequence example
- DRLXXXX line 159 — "nested `do`", named consequences dropped
- DRLXXXX §"if" lines 727-751
