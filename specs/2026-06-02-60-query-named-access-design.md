# Query Named Access Design — Issue #60

## Overview

Support named (non-positional) access for query invocation:

```
rule R1 {
    var subject : /things[name == "a"],
    /trusts[a == subject, var object : b],
    do {...}
}
```

Instead of positional `/trusts(subject, var object)`, the caller maps arguments by query parameter name. This improves readability when queries have many parameters and makes argument order irrelevant.

**DRLXXXX reference:** Lines 920-923

## Design decisions

1. **No separate parse-time distinction.** The grammar already parses bracket contents as `drlxExpression`. The compiler reinterprets them as named query arguments when the entry point resolves to a query (Approach A).
2. **Only `==` for input arguments.** `paramName == expr` is the only valid input form. Other operators (`>=`, `!=`, etc.) are errors — named access is argument passing, not filtering.
3. **All parameters required.** Every query parameter must appear in the named access brackets. Missing parameters are an error with a message naming the missing param.
4. **Mixing positional and named is an error.** A query invocation uses `(...)` or `[...]`, not both.
5. **Self-referencing named access is an error.** Recursive queries must use positional syntax. Named access on a self-referencing query produces a clear error message.

## Syntax

Two forms inside brackets:

| Form | Meaning | Example |
|------|---------|---------|
| `paramName == expr` | Input: pass `expr` as query param `paramName` | `a == subject` |
| `var bindName : paramName` | Output: bind query param `paramName` to local name `bindName` | `var object : b` |

## Grammar change

One new alternative in `drlxExpression`:

```antlr
drlxExpression
    : VAR identifier ':' identifier    // NEW: output binding for named query access
    | identifier ':' expression
    | customConstraint
    | expression
    ;
```

The `VAR identifier ':' identifier` form is restricted to two identifiers — the bind name and the query parameter name.

## Visitor change

`extractConditions()` already collects `drlxExpression` nodes as strings via `getText()`. The new `VAR identifier ':' identifier` alternative serializes as `"var object : b"`. No visitor logic changes needed — the new alternative flows through as a condition string.

## Compiler changes

### Detection (in `buildLhsPatterns`)

Current logic at `DrlxRuleAstRuntimeBuilder.java:446-447`:

```java
QueryImpl targetQuery = queryRegistry.get(patternIr.entryPoint());
if (targetQuery != null && !patternIr.positionalArgs().isEmpty()) {
```

Extended to:

```
if targetQuery != null:
    if positionalArgs non-empty AND conditions non-empty → ERROR (mixing)
    if positionalArgs non-empty → existing positional path
    if conditions non-empty → named access path (new)
    if both empty → fall through to regular pattern
```

For self-reference: `targetQuery == currentQuery` on the named access path → ERROR.

### `buildNamedQueryArgs()` — new method

Transforms conditions into the same `List<String>` that positional access produces, ordered by parameter index.

**Input:** `conditions` (e.g., `["a == subject", "var object : b"]`), `queryParams` (e.g., `[Declaration("a"), Declaration("b")]`)

**Algorithm:**
1. Build name-to-index map from query parameters: `{"a" → 0, "b" → 1}`
2. Create `String[]` of size `queryParams.length`, initialized to null
3. For each condition:
   - Matches `"var <bindName> : <paramName>"` → look up `paramName`, set `args[index] = "var <bindName>"`
   - Matches `"<paramName> == <expr>"` → look up `paramName`, set `args[index] = "<expr>"`
   - Otherwise → error: "invalid named query argument: '<condition>'"
4. Validate: any null slot → error: "named access for query 'X' is missing parameter 'Y'"

**Output:** ordered `List<String>` identical to what positional parsing produces.

### Downstream — no changes

`buildQueryElement()` receives the ordered `List<String>` and processes it identically to positional access. Post-element variable registration (lines 477-509) also works unchanged. Result row binding (`var t : /query[...]`) is unaffected.

## Testing

All tests in `QueryTest.java`:

1. **Basic named access** — `PersonsByAge(int minAge, Person result)` invoked with `/personsByAge[minAge == 25, var p : result]`
2. **All inputs** — both params are input, verify query filtering
3. **Mixed input/output** — some params input, some output
4. **Parameter order independence** — named args in reverse order vs declaration order
5. **Error: missing parameter** — omit a parameter, expect descriptive error
6. **Error: unknown parameter name** — name not matching any query param
7. **Error: mixing positional and named** — `/query(a)[b == x]`
8. **Error: self-referencing named access** — recursive query using named syntax
9. **Error: non-`==` operator for input** — `/query[a >= x]`
10. **Named access with result binding** — `var t : /query[...]`, verify `t.paramName` access

## Files modified

| File | Change |
|------|--------|
| `DrlxParser.g4` | Add `VAR identifier ':' identifier` alternative to `drlxExpression` |
| `DrlxRuleAstRuntimeBuilder.java` | Add named access detection branch; add `buildNamedQueryArgs()` method |
| `QueryTest.java` | Add 10 test methods |
