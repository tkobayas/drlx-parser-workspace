# From-Expression Syntax: `/expression` for iterate-and-assign (#95)

**Issue:** [#95](https://github.com/tkobayas/drlx-parser/issues/95)
**Date:** 2026-08-17
**DRLXXXX reference:** Lines 286-290 ('from' section), 624-630 ('from' versus '/' section)

## Overview

DRLXXXX replaces the `from` keyword with `/expression` syntax for iterating over
arbitrary expressions. Currently `/` only works for DataStore references (`/persons`)
and query calls (`/trusts(a, b)`). This spec covers the from-expression form where
`/` precedes a constructor or method-call expression:

```
// Old DRL
i : Integer() from new int[] {1, 2, 3}

// DRLXXXX
var i : /new int[] {1, 2, 3}
```

## Scope

**In scope:**
- Constructor expressions: `var i : /new int[] {1, 2, 3}`
- Method calls: `var list : /Collections.emptyList()`
- Expressions referencing earlier bindings: `var p : /persons, var addrs : /p.getAddresses()`
- Explicit type bindings: `Integer i : /new int[] {1, 2, 3}`

**Out of scope:**
- Bare variable references as from-source (`/r` where `r` is a bound variable) — overlaps with DataStore lookup and the actors/async feature
- Constraint filters on from-expression results (`/expr[condition]`)
- Multi-segment navigation after from-expressions (`/expr/field`)

## Ambiguity Analysis

The `/` token is unambiguous because from-expressions only appear after a binding
(e.g., `var i :`). The parser sees `identifier identifier ':' '/'` and knows this
is either an OOPath pattern or a from-expression.

Both `rulePattern` (wrapping `boundOopath`) and `boundFromExpression` share the
prefix `identifier identifier (':' | '=') '/'`. ANTLR4's adaptive prediction
resolves ambiguity by looking ahead past the common prefix:

**Constructor expressions** (`/new int[] {1, 2, 3}`): `new` is a keyword, not an
identifier, so `oopathRoot` (which requires an identifier) cannot match. ANTLR4
unambiguously selects `boundFromExpression`.

**Dot-qualified method calls** (`/p.getAddresses()`, `/Collections.emptyList()`):
After `/`, the identifier (`p` or `Collections`) matches `oopathRoot`. But the
next token `.` is not valid in `oopathRoot` (`#`, `(`, `[` are the only valid
suffixes). So `rulePattern` would match `/p` but then cannot find the required `,`
(sees `.` instead). ANTLR4 prediction rules out `rulePattern` and selects
`boundFromExpression`.

**Bare identifier** (`/persons`): `oopathRoot` matches and `rulePattern` consumes
the trailing `,`. ANTLR4's first-match-wins selects `rulePattern`. Existing
DataStore patterns are unaffected.

**Query calls** (`/trusts(a, b)`): `oopathRoot` matches the identifier and
positional args. `rulePattern` consumes the trailing `,`. First-match-wins selects
`rulePattern`. Existing query patterns are unaffected.

**Known limitation — unqualified method calls with arguments:**
`/someMethod(42)` is syntactically identical to a query call `/someQuery(42)`.
ANTLR4's first-match-wins gives this to `rulePattern` (query call). This is
correct and expected — the parser cannot distinguish the two forms. Users needing
a from-expression with a method call must qualify it: `/Helper.someMethod(42)` or
`/this.someMethod(42)`. Zero-argument method calls (`/someMethod()`) ARE
unambiguous because `oopathRoot`'s positional syntax requires at least one
argument, so empty `()` does not match `oopathRoot`.

No constraint filters after from-expressions means `[]` within the expression is
always parsed as part of the expression (array access, array creation) — never as
an OOPath constraint filter.

## Design

### Approach: Separate grammar rule (Approach A)

A new `fromExpression` / `boundFromExpression` grammar rule is added alongside the
existing `oopathExpression` / `boundOopath`. This keeps the two paths cleanly
separated — from-expressions have no segments, constraints, positional args, or
watches, so sharing a grammar rule with OOPath would leave most fields unused.

### Layer 1: Grammar (DrlxParser.g4)

New rules:

```antlr
boundFromExpression
    : identifier identifier (':' | '=') fromExpression
    ;

fromExpression
    : '/' expression
    ;
```

Added to `ruleItem`, `groupChild`, and `branchItem` — all places that accept
`boundOopath`. In each, `boundOopath` alternatives remain first (first-match-wins
ensures DataStore patterns take priority).

`ruleItem` additions:

```antlr
ruleItem
    : rulePattern
    | boundFromExpression ','          // NEW
    | oopathExpression ','
    | ...
    ;
```

`groupChild` additions:

```antlr
groupChild
    : boundOopath
    | boundFromExpression              // NEW
    | oopathExpression
    | ...
    ;
```

`branchItem` additions:

```antlr
branchItem
    : boundOopath
    | boundFromExpression              // NEW
    | oopathExpression
    | ...
    ;
```

### Layer 2: IR (DrlxRuleAstModel)

New record in the `LhsItemIR` sealed hierarchy:

```java
public record FromExpressionIR(String typeName,
                                String bindName,
                                String fromExpression,
                                List<String> referencedBindings) implements LhsItemIR {
}
```

- `typeName` — `"var"` or an explicit type name
- `bindName` — the binding variable name
- `fromExpression` — raw expression text after `/`
- `referencedBindings` — identifiers from the expression that may reference earlier
  bound variables (over-collected by regex; the runtime builder filters against
  actual bound variables)

The `LhsItemIR` sealed interface permits list is extended to include
`FromExpressionIR`.

### Layer 3: Protobuf (drlx_rule_ast.proto)

New message:

```protobuf
message FromExpressionParseResult {
  string type_name = 1;
  string bind_name = 2;
  string from_expression = 3;
  repeated string referenced_bindings = 4;
}
```

Added to `LhsItemParseResult.oneof`:

```protobuf
message LhsItemParseResult {
  oneof kind {
    PatternParseResult pattern = 1;
    GroupElementParseResult group = 2;
    EvalParseResult eval = 3;
    AccumulatePatternParseResult accumulate_pattern = 4;
    CustomAccumulateParseResult custom_accumulate = 5;
    FromExpressionParseResult from_expression = 6;
  }
}
```

Serialization/deserialization in `DrlxRuleAstProtoConverter` follows the existing
pattern for other `LhsItemIR` variants.

### Layer 4: Visitor (DrlxToRuleAstVisitor)

New method:

```java
private FromExpressionIR buildFromExpression(DrlxParser.BoundFromExpressionContext ctx) {
    String typeName = ctx.identifier(0).getText();
    String bindName = ctx.identifier(1).getText();
    String fromExpr = getText(ctx.fromExpression().expression());
    List<String> referencedBindings = extractIdentifiers(fromExpr);
    return new FromExpressionIR(typeName, bindName, fromExpr, referencedBindings);
}
```

Dispatch additions in:
- The main `ruleItem` fold loop — handle `boundFromExpression` to produce
  `FromExpressionIR`, add to `lhs`
- `buildGroupChild()` — add `boundFromExpression` check
- `buildBranchItem()` — add `boundFromExpression` check

In all three, `boundOopath` checks remain first.

### Layer 5: Runtime Builder (DrlxRuleAstRuntimeBuilder)

**New class: `FromExpressionDataProvider`**

Modeled on `OopathFieldDataProvider`. Implements `DataProvider`:

```java
public class FromExpressionDataProvider implements DataProvider {
    private final Object evaluator;        // compiled lambda
    private Declaration[] declarations;    // required declarations from earlier bindings

    @Override
    public Iterator getResults(BaseTuple tuple, ValueResolver valueResolver,
                               Object providerContext) {
        // Resolve declaration values from the tuple
        // Invoke the compiled lambda with declaration values
        // Iterate result: handle Object[], Iterator, Iterable, single object
    }

    @Override
    public boolean isReactive() { return false; }
}
```

Iteration logic follows `LambdaDataProvider.getResults()`:
- `Object[]` → `Arrays.asList(result).iterator()`
- `Iterator` → return directly
- `Iterable` → `.iterator()`
- Single object → `Collections.singletonList(result).iterator()`

**New builder method: `buildFromExpressionPattern`**

1. Resolve the pattern type: if `typeName` is `"var"`, use `Object.class`; otherwise
   resolve via `typeResolver`
2. Create `Pattern` with `ObjectType` and `bindName`
3. Find referenced bindings by filtering `FromExpressionIR.referencedBindings`
   against the live `boundVariables` map
4. Compile the from-expression to a value-returning lambda using
   `DrlxLambdaCompiler` — similar to `createValueExtractor` (used for accumulate
   arguments), but the output type is `Object` rather than a specific numeric type
5. Create `FromExpressionDataProvider` with the compiled lambda and required
   `Declaration[]`
6. Set `pattern.setSource(new From(dataProvider))`
7. Register the binding in `boundVariables`

**Lambda compilation:** A new method `createFromExpressionLambda` on
`DrlxLambdaCompiler` compiles the from-expression text to an `Object`-returning
evaluator. It follows the same pattern as `createEvalExpression`
(batch-compiled via `MVELBatchCompiler`), but with `Object.class` as the output
type instead of `Boolean.class`. Referenced bindings are passed as MVEL
declarations so the expression can access earlier-bound variables.

## Testing

**Parser-level tests:**

1. Constructor from-expression: `var i : /new int[] {1, 2, 3},` → parses as
   `FromExpressionIR` with `fromExpression = "new int[] {1, 2, 3}"`
2. Method call: `var list : /Collections.emptyList(),` → `FromExpressionIR`
3. Method call with earlier binding: `var p : /persons, var addrs : /p.getAddresses(),`
   → second item is `FromExpressionIR` with `referencedBindings` containing `"p"`
4. Explicit type: `Integer i : /new int[] {1, 2, 3},` → `typeName = "Integer"`
5. DataStore regression: `var p : /persons[age > 18],` → still `PatternIR`

**Runtime-level tests (end-to-end rule execution):**

6. Iterate array: `var i : /new int[] {1, 2, 3}` — fires 3 times
7. Iterate collection: `var s : /java.util.List.of("a", "b", "c")` — fires 3 times
8. From-expression with earlier binding: `var p : /persons, var addr : /p.getAddresses()`
   — fires for each address of each person
9. Single-object result: `var s : /String.valueOf(42)` — fires once, binding is `"42"`

**Protobuf round-trip:**

10. Parse → serialize → deserialize → verify `FromExpressionIR` fields survive

## Files Modified

| File | Change |
|------|--------|
| `DrlxParser.g4` | Add `boundFromExpression`, `fromExpression` rules; update `ruleItem`, `groupChild`, `branchItem` |
| `DrlxRuleAstModel.java` | Add `FromExpressionIR` record; extend `LhsItemIR` permits |
| `drlx_rule_ast.proto` | Add `FromExpressionParseResult` message; extend `LhsItemParseResult` oneof |
| `DrlxRuleAstProtoConverter.java` | Serialize/deserialize `FromExpressionIR` |
| `DrlxToRuleAstVisitor.java` | Add `buildFromExpression()`; dispatch in fold loop, `buildGroupChild`, `buildBranchItem` |
| `DrlxRuleAstRuntimeBuilder.java` | Add `buildFromExpressionPattern()`; dispatch in LHS-item processing |
| `DrlxLambdaCompiler.java` | Add `createFromExpressionLambda()` method |
| `FromExpressionDataProvider.java` | New class — `DataProvider` for from-expressions |
