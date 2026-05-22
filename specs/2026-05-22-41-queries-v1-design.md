# #41 Queries v1 — Definition + Basic Invocation

**Issue:** https://github.com/tkobayas/drlx-parser/issues/41
**Epic:** #26 DrlxCompiler enhancement round 2
**DRLXXXX ref:** Lines 293-316, 904-978

## Scope

v1 implements the core query functionality:
- **Query definition** — rules with typed parameters (`rule Trusts(Object a, Object b) { ... }`)
- **Query invocation from rules** — positional argument syntax (`/trusts(a, var b)`)
- **Query invocation via API** — `executeQuery("QueryName", args...)`

### Deferred to follow-up issues
- Passive query invocation (`?/trusts(...)`)
- Result binding (`var t : /trusts(a, var b)`)
- `@Tabled` memoization
- Named access (`/trusts[a == subject, var object : b]`)
- `@Rule` / `@DataSource` annotations for custom name mapping
- Recursive / transitive closure queries
- `do` blocks inside queries
- Result POJO generation / custom result classes

## Approach

Mirror the drools compiler's query infrastructure (`PatternBuilderForQuery`, `QueryElementBuilder`), adapted to DRLX's direct-build pipeline (grammar → IR → visitor → runtime builder, bypassing Descr layer).

Drools runtime has full query support — `QueryImpl`, `QueryElement`, `PhreakQueryNode`, `QueryTerminalNode` — all in `drools-base` / `drools-core`. No drools-side changes needed.

## Design

### 1. Grammar

Add optional parameter list to `ruleDeclaration`. No existing rules modified.

```antlr
ruleDeclaration
    : annotation* RULE identifier ruleParameterList? '{' ruleBody '}'
    ;

ruleParameterList
    : '(' ruleParameter (',' ruleParameter)* ')'
    ;

ruleParameter
    : typeType identifier
    ;
```

Query invocation syntax (`/trusts(a, var b)`) already parses via the existing `oopathRoot` rule which supports positional args.

### 2. IR Model

Add `RuleParameterIR` record. Extend `RuleIR` with a parameters field.

```java
public record RuleParameterIR(String typeName, String paramName) { }

public record RuleIR(String name,
                     List<RuleAnnotationIR> annotations,
                     List<RuleParameterIR> parameters,  // empty for regular rules
                     List<LhsItemIR> lhs,
                     ConsequenceIR rhs) {
}
```

No new `LhsItemIR` variant needed — query invocations in the LHS parse as `PatternIR` with `positionalArgs`. The runtime builder distinguishes query calls from DataSource patterns at build time.

### 3. Visitor

In `DrlxToRuleAstVisitor.buildRule()`, extract parameters from the parse tree:

```java
List<RuleParameterIR> parameters = List.of();
if (ctx.ruleParameterList() != null) {
    parameters = ctx.ruleParameterList().ruleParameter().stream()
        .map(p -> new RuleParameterIR(p.typeType().getText(), p.identifier().getText()))
        .toList();
}
```

Pass `parameters` into `RuleIR`. No other visitor changes — LHS and consequence visitors remain identical.

### 4. Runtime Builder — Query Definition

When `RuleIR.parameters()` is non-empty, the runtime builder creates a `QueryImpl` and builds the drools prefix pattern. This mirrors `PatternBuilderForQuery` in drools-compiler.

**Prefix pattern:** A synthetic `Pattern` prepended to the query's LHS AND group, before any user-defined patterns. It:
1. Matches `DroolsQuery` objects by query name (via `QueryNameConstraint`)
2. Extracts argument values from `DroolsQuery.elements[]` into `Declaration` objects (via `ArrayElementReader`)

This makes query parameters available as bound variables for the rest of the LHS.

**Steps:**

1. Create `QueryImpl` (not `RuleImpl`).
2. Build prefix pattern on `ClassObjectType.DroolsQuery_ObjectType`.
3. Add `QueryNameConstraint` matching the query name.
4. For each parameter: create `ArrayElementReader` at index `i`, add `Declaration` to prefix pattern, resolve parameter type via `TypeResolver`.
5. Call `query.setParameters(declarations)`.
6. Insert prefix pattern as first child of the LHS AND group.
7. Pre-populate `boundVariables` map with parameter declarations so they're available during LHS compilation.
8. Skip consequence building (no `rule.setConsequence()`).

**API dependencies (all in drools-base, already a DRLX dependency):**
- `QueryImpl` — `org.drools.base.definitions.rule.impl.QueryImpl`
- `QueryNameConstraint` — `org.drools.base.rule.constraint.QueryNameConstraint`
- `ArrayElementReader` — `org.drools.base.base.extractors.ArrayElementReader`
- `ClassObjectType.DroolsQuery_ObjectType`

### 5. Runtime Builder — Query Invocation from Rules

When a `PatternIR` entry point name matches a compiled `QueryImpl` in the query registry, the builder constructs a `QueryElement` instead of a `Pattern`.

**Detection:** The builder checks the query registry first. If a `QueryImpl` with the matching name exists, it's a query invocation. Otherwise, fall back to DataSource pattern building.

**Name mapping:** Query `Trusts` is registered in the query registry under the conventional entry point name `trusts` (lowercase first letter). This matches the OOPath entry point name directly, so lookup requires no case conversion at invocation time.

**Two-pass compilation:** The `build()` method processes rules in two passes:
1. First pass: compile all parameterized rules (queries) into `QueryImpl` objects, register them in the query registry under their entry point names.
2. Second pass: compile all remaining rules. Query invocations can now resolve against the registry.

**Build steps for invocation:**

1. Resolve the target `QueryImpl` from the registry.
2. Build `QueryArgument[]` from positional args:
   - Arg is a known bound variable → `QueryArgument.Declr(declaration)` (input)
   - Arg uses `var` prefix (e.g. `var b`) → `QueryArgument.VAR` (output variable)
   - Arg is a literal → `QueryArgument.Literal(value)`
3. Create result pattern (`Object[]` type) with `ArrayElementReader` declarations for each output variable.
4. Construct `QueryElement(resultPattern, queryName, arguments, variableIndexes, requiredDeclarations, openQuery=true, abductive=false)`.
5. Add `QueryElement` directly to the parent group element (`QueryElement` implements `RuleConditionElement`).
6. Register output bindings in `boundVariables` for downstream use.

**API dependencies (all in drools-base):**
- `QueryElement` — `org.drools.base.rule.QueryElement`
- `QueryArgument` — `org.drools.base.rule.QueryArgument` (Declr, Literal, Var)

### 6. Unit Class Integration

The unit class declares a `DataSource` field for the query's entry point:

```java
public class MyUnit {
    public DataSource<Trust> trusts;  // backs query "Trusts"
}
```

Naming convention: query `Trusts` → entry point `trusts` (lowercase first letter). No `@Rule` annotation support in v1.

The `DataSource` field provides type information and entry point registration. The compiled `QueryImpl` determines runtime behavior.

### 7. Complexity and Risk

| Area | Complexity | Risk to existing code |
|------|-----------|----------------------|
| Grammar | Low | None — additive only, no existing rules modified |
| IR Model | Low | Low — `RuleIR` signature change is compile-time visible |
| Visitor | Low | None — additive only |
| Runtime builder (definition) | Medium | Low — isolated in conditional branch, regular rule path untouched |
| Runtime builder (invocation) | Medium-High | Medium — two-pass compilation is a structural change to `build()` |
| Runtime builder (invocation) | Medium | Low — `QueryElement` construction follows drools blueprint |

**Main risk:** The two-pass compilation in `build()`. Currently rules are processed in a single `forEach`. Splitting into two passes (queries first, then rules) changes the iteration structure but doesn't affect per-rule logic.

**Mitigation:** The query registry is a simple `Map<String, QueryImpl>` populated during pass 1 and read during pass 2. If no queries exist, pass 1 is a no-op and behavior is identical to today.

## Testing

**Test domain:** Simple `Person`-based queries (not transitive closure — deferred).

**Test 1 — Query definition via API:**
```
rule PersonsByAge(int minAge, Person result) {
    result : /persons[age >= minAge]
}
```
Compile, call `executeQuery("PersonsByAge", 30, Variable.v)`, assert correct persons returned via result parameter.

**Test 2 — Query invocation from another rule:**
```
rule PersonsByAge(int minAge, Person result) {
    result : /persons[age >= minAge]
}

rule R1 {
    /personsByAge(25, var p),
    do { results.add(p); }
}
```
Insert persons, fire rules, verify R1 receives query output as bound variable `p`. Note: output variables (`var p`) correspond to query parameters — the caller marks which parameters are outputs via `var`.

**Test 3 — Multiple parameters:**
```
rule PersonsByAgeRange(int minAge, int maxAge, Person result) {
    result : /persons[age >= minAge, age <= maxAge]
}
```
Verify multi-parameter input binding and single output binding.

**Test 4 — Error cases:**
- Wrong argument count
- Incompatible argument types
- Name matching neither DataSource nor QueryImpl
