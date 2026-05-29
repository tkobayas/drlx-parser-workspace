# #56 Passive Query Invocation (`?/queryName(...)`)

**Issue:** https://github.com/tkobayas/drlx-parser/issues/56
**Epic:** #77 Query enhancements
**DRLXXXX ref:** Lines 935-948
**Depends on:** #41 (v1 query infrastructure — completed)

## Scope

Support passive (non-reactive) query invocation using `?/` prefix:

```
rule R1 {
    var a : /things[name == "a"],
    ?/trusts(a, var b),
    do { ... }
}
```

`?/` marks the query call as passive — results are fetched once at evaluation time, but the query does not reactively wake the rule when new matching facts are inserted.

### Included

- Thread the existing `passive` flag from `PatternIR` into `QueryElement` construction (`openQuery = !passive`)
- Compile-time validation: reject passive invocation of queries that have an agenda-based `do` block
- Tests for passive query invocation

### Out of scope

- Result binding (`var t : /queryName(...)`) — issue #57
- Named access (`/query[a == x, var b : y]`) — issue #60
- `do` blocks inside queries — not yet implemented; the validation is a forward guard

## Approach

The entire pipeline from grammar through IR already supports the `?` prefix:
- **Grammar:** `oopathExpression : QUESTION? '/' ...` already parses `?/`
- **Visitor:** `DrlxToRuleAstVisitor` already extracts `ctx.oopathExpression().QUESTION() != null` into `PatternIR.passive`
- **IR:** `PatternIR` record already has a `passive` field

The only gap is in the runtime builder: `DrlxRuleAstRuntimeBuilder.buildQueryElement()` hardcodes `openQuery = true` at line 1012 instead of using the `passive` flag.

## Design

### 1. Fix `openQuery` in `buildQueryElement`

**File:** `DrlxRuleAstRuntimeBuilder.java`, method `buildQueryElement`

Change the `QueryElement` construction (line 1006-1013) from:

```java
return new QueryElement(
        resultPattern,
        targetQuery.getName(),
        arguments,
        varIndexArray,
        requiredDeclarations.toArray(new Declaration[0]),
        true,     // hardcoded
        false);
```

To:

```java
return new QueryElement(
        resultPattern,
        targetQuery.getName(),
        arguments,
        varIndexArray,
        requiredDeclarations.toArray(new Declaration[0]),
        !patternIr.passive(),
        false);
```

`QueryElement(openQuery=true)` means reactive (live updates propagate). `openQuery=false` means passive (one-shot evaluation). The `?/` prefix sets `passive=true` on `PatternIR`, so `openQuery = !passive`.

### 2. Validation: passive invocation of queries with agenda `do` blocks

**File:** `DrlxRuleAstRuntimeBuilder.java`, method `buildQueryElement`

Before the `QueryElement` construction, add:

```java
if (patternIr.passive() && targetQuery.getConsequence() != null) {
    throw new RuntimeException(
            "Cannot passively invoke query '" + targetQuery.getName()
            + "': the query has an agenda-based 'do' block which is incompatible with passive invocation");
}
```

**Rationale:** DRLXXXX §935-948 states "an exception will be thrown if a passive query is called and an agenda 'do' is found." Agenda-based `do` blocks require reactive propagation to schedule on the agenda; passive invocation (one-shot fetch) cannot trigger agenda execution.

Currently `buildQuery()` never calls `setConsequence()` on `QueryImpl`, so this guard cannot trigger today. It will activate when query `do` blocks are implemented in the future.

### 3. Tests

Add test method(s) to `QueryTest.java`:

**Test: passive query invocation works**
- Define a query `PersonsByAge(int minAge, Person result)` that matches persons by age
- Invoke it passively: `?/personsByAge(25, var p)`
- Insert facts AFTER rule activation to verify that the passive query does NOT re-fire (unlike an active/reactive query which would)
- Pattern: insert reactive-side fact to trigger rule, then insert query-matching facts, verify rule fires only for what was present at evaluation time

This follows the same verification pattern used in `PassivePatternTest`: passive-side insertions alone do not wake the rule; only prior reactive data triggers evaluation.

## Runtime behavior

| Syntax | `PatternIR.passive` | `QueryElement.openQuery` | Behavior |
|--------|---------------------|--------------------------|----------|
| `/trusts(a, var b)` | `false` | `true` | Reactive — query re-evaluates on changes |
| `?/trusts(a, var b)` | `true` | `false` | Passive — one-shot fetch, no reactive updates |
