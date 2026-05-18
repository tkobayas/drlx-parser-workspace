# Accumulate inline-from form — `avg(/persons.age)` (#50)

**Status:** ready for implementation plan
**Parent epic:** #26
**Follows:** #49 (MultiAccumulate folding shipped at project HEAD `50049af`)
**DRLXXXX reference:** §Accumulate — "the path must end in a final '.' notation to make it clear it's accessing and returning a single value" (line ~855 area).

## Summary

v1 / #48 / #49 require an explicit bound source pattern in front of every
accumulate item:

```
var p : /persons,
var avgAge = avg(p.age),
do {...}
```

#50 admits the shorthand the DRLX spec names "inline-from":

```
var avgAge = avg(/persons.age),
do {...}
```

The oopath argument is desugared at visitor time into a synthetic source
pattern (`var $inline0 : /persons,`) plus an accumulator whose extractor
expression is rewritten against the synthetic binding (`avg($inline0.age)`).
The synthetic name is internal — never visible to the user — and lives only
in the accumulate's inner scope. Downstream of the visitor, the
`AccumulatePatternIR` shape is identical to what the bound form produces;
`DrlxRuleAstRuntimeBuilder` is unchanged.

Each inline-from accumulator produces its own `AccumulatePatternIR`
with its own synthetic source pattern — no fold by oopath equivalence.
This mirrors Drools' DRL convention: `MVELAccumulateBuilder.isMultiFunction()`
(`drools-mvel/.../MVELAccumulateBuilder.java:144`) only treats functions
inside one `accumulate(...)` block as multi-function; two separate
`from accumulate(...)` clauses with textually identical source patterns
are emitted as two independent `SingleAccumulate`s and not merged.
The bound multi-function form (#49) continues to fold because adjacency
to a single bound source is the DRLX analog of DRL's in-block grouping —
the user explicitly groups functions under one declared source.

## Scope

**In:**

| Input | Lowered to |
|---|---|
| `var avgAge = avg(/persons.age)` | `var $inline0 : /persons, var avgAge = avg($inline0.age)` → one `SingleAccumulate` |
| `long n = count(/persons)` (bare oopath, zero-arg function) | `var $inline0 : /persons, long n = count()` → one `SingleAccumulate` |
| `var minAge = min(/persons.age), var maxAge = max(/persons.age)` (two adjacent inline-from items) | two separate `AccumulatePatternIR`s, fresh synthetic source pattern each (`$inline0` for min, `$inline1` for max) → two `SingleAccumulate`s. Source patterns are NOT folded even when textually identical — matches DRL behavior for separate `from accumulate(...)` clauses. |
| `var personAvg = avg(/persons.age), var seniorAvg = avg(/seniors.age)` | same: two separate `AccumulatePatternIR`s, each with its own synthetic source |
| Inline-from with source-pattern constraints (`avg(/persons[age >= 40].age)`) | oopath's `[...]` constraints flow into the synthesised source pattern unchanged |

**Out (filed elsewhere):**

- `and(...)` source inside inline-from (`avg(and(l : /locations, /persons[locationId == l.id].age))`) — **#52**.
- `acc()` keyword forms — **#51**.
- Outer-binding references inside extractor expressions
  (`sum(/persons.age * q.factor)` where `q` is a prior bound pattern) — **#54**.
- Full expression continuation after final dot
  (`sum(/persons.age * 2)`) — deferred; the grammar accepts only
  `oopathExpression ('.' identifier)?`. If a use case appears, it gets
  its own issue.

**Composes with bound patterns** (not rejected):

A bound pattern immediately followed by an inline-from accumulator
(e.g. `var t : /thresholds, var n = count(/persons), do {...}`) flushes
the bound pattern as a normal LHS item and starts a separate synthetic
accumulate. Even the same-source case
(`var p : /persons, var avgAge = avg(/persons.age)`) parses — it
produces two separate `/persons` source-pattern joins, which is
wasteful but semantically correct. Recognising same-source equivalence
to merge them is out of scope for #50.

**Rejected (build error):**

- Final-dot extractor on a zero-arg function (`count(/persons.age)`).
  Without this check, today's runtime silently ignores the extractor
  (`buildSingleAccumulator` skips extractor construction for
  `acceptsZeroArgs` functions even when `argCount == 1`), masking
  typos. Visitor rejects with a precise message — see Error handling.
- Bare oopath without final-dot for non-zero-arg functions
  (`avg(/persons)`) — caught by the existing arity check inside
  `buildSingleAccumulator`.

## Architecture

```
                 DRLX source text
                       │
                       ▼
              ┌────────────────────┐
              │   DrlxParser.g4    │   accumulateCall now has two
              │   (grammar)        │   alternatives: inline-from
              └────────┬───────────┘   first, regular second
                       │
                       ▼
              ┌────────────────────┐
              │ DrlxToRuleAst-     │   for each inline-from item:
              │ Visitor.buildRule  │   - flush prior pending pattern
              │                    │   - synthesise PatternIR with $inlineN
              └────────┬───────────┘   - emit AccumulatePatternIR with one accumulator
                       │
                       ▼
              ┌────────────────────┐
              │ AccumulatePatternIR│   shape identical to bound form
              │   (existing record)│   — no IR-model change
              └────────┬───────────┘
                       │
                       ▼
              ┌────────────────────┐
              │ DrlxRuleAst-       │   unchanged: builds source Pattern
              │ RuntimeBuilder     │   from PatternIR, dispatches to
              │ .buildAccumulate-  │   SingleAccumulate per the existing
              │ Pattern            │   N=1 path                
              └────────────────────┘
```

The whole change is a grammar extension plus a visitor rewrite. No
changes to the IR model (`DrlxRuleAstModel`), no changes to the
runtime builder (`DrlxRuleAstRuntimeBuilder`), no changes to the
extractor compile path (`DrlxLambdaCompiler`).

## Grammar

One file: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`.

```antlr
// Accumulate result binding — DRLXXXX §Accumulate. The argument list is
// optional to admit `count()`; the inline-from alternative below admits
// the oopath-rooted shorthand `avg(/persons.age)` etc.
accumulateItem
    : (VAR | typeType) identifier '=' accumulateCall
    ;

// Two forms:
//   1. Inline-from: oopath-rooted argument (DRLXXXX §"final '.' notation"),
//      OR bare oopath for zero-arg functions (count). Synthesises a source
//      pattern at visitor time.
//   2. Regular: zero or more plain expressions; requires a preceding
//      bound pattern.
//
// ANTLR picks the first matching alternative. Inline-from comes first
// because its prefix ('/' or '?/') is disjoint from any Java `expression`
// prefix — no ambiguity, no predicate needed.
accumulateCall
    : qualifiedName '(' inlineFromOopath ')'
    | qualifiedName '(' (expression (',' expression)*)? ')'
    ;

// Inline-from argument: an oopath, optionally with a final-dot extractor
// (`.identifier`). Final dot REQUIRED for non-zero-arg functions; arity
// validation lives downstream in DrlxRuleAstRuntimeBuilder.
inlineFromOopath
    : oopathExpression ('.' identifier)?
    ;
```

**Dispatch correctness**: Java's `expression` rule never starts with `/`
(division is infix only) and never starts with `?/` (ternary is infix
and binds tighter than primary anyway). ANTLR adaptive LL(*) picks
alternative 1 on `/` / `?/`, alternative 2 otherwise. No left-factoring
or semantic predicate required.

**Why arity validation stays out of the grammar**: keeps a single source
of truth — `AccumulateFunctionRegistry.Resolution.acceptsZeroArgs`, used
by `buildSingleAccumulator`'s existing arity check. The grammar admits
the syntactic shape; semantic validation runs at IR-build time.
`avg(/persons)` parses, fails build with the existing arity error.
`count(/persons)` parses, builds fine.

## Visitor

One file: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`.

### One new piece of state in `buildRule`

The existing fold tracks one pending pattern and its accrued accumulators.
Inline-from adds only a per-rule counter for the synthetic binding name:

```java
PatternIR pendingPattern = null;
List<AccumulatorIR> pendingAccs = new ArrayList<>();
int     inlineCounter = 0;              // NEW: per-rule synthetic-name counter
```

No fold-state tracking — each inline-from accumulator flushes any pending
pattern and produces its own `AccumulatePatternIR` immediately. This is
mechanically the same shape as the bound form's single-function path.

### Per-item dispatch (inside the `for (RuleItemContext itemCtx : ...)` loop)

```
if itemCtx.accumulateItem != null:
    call = itemCtx.accumulateItem.accumulateCall
    if call.inlineFromOopath != null:               // alt 1 — inline-from
        oopathCtx     = call.inlineFromOopath.oopathExpression
        finalDotIdent = call.inlineFromOopath.identifier (or null)
        functionName  = call.qualifiedName.getText()

        // Zero-arg-function guard: count(/persons.age) etc would silently
        // ignore the .age extractor in buildSingleAccumulator — reject early.
        if finalDotIdent != null AND AccumulateFunctionRegistry.resolve(functionName).acceptsZeroArgs():
            throw "function '<functionName>' does not accept a final-dot extractor in rule '<R>'; "
                + "use '<functionName>(<oopath>)' instead"

        // Flush any prior pending pattern (synthetic or bound) — inline-from
        // always opens its own synthetic source. No fold.
        flushPending(lhs, pendingPattern, pendingAccs)

        String synthName    = "$inline" + inlineCounter++
        PatternIR synthSrc  = buildPatternFromOopath(oopathCtx, synthName)
        AccumulatorIR accIr = buildAccumulator(itemCtx.accumulateItem, synthName, finalDotIdent)

        // Emit immediately as its own AccumulatePatternIR — no pending state needed.
        lhs.add(new AccumulatePatternIR(synthSrc, List.of(accIr)))
        pendingPattern = null
        pendingAccs    = new ArrayList<>()
    else:                                           // alt 2 — regular
        if pendingPattern == null:
            throw "accumulate item without a preceding pattern in rule '<R>'"   // existing
        pendingAccs.add(buildAccumulator(itemCtx.accumulateItem))                // existing
    continue

// Non-accumulate item path: unchanged from today.
flushPending(lhs, pendingPattern, pendingAccs)
pendingPattern = null
pendingAccs    = new ArrayList<>()
... existing branches for rulePattern / notElement / etc ...
```

### Two small helpers

**`buildPatternFromOopath(OopathExpressionContext, String syntheticBindName) → PatternIR`**

Overload of the existing `buildPatternFromOopath(OopathExpressionContext)`.
The existing method returns `new PatternIR("", "", entryPoint, ...)` —
empty `typeName`, empty `bindName`, and the oopath root identifier as
`entryPoint`. The new overload differs in exactly one field:

- `typeName`  = `""` (unchanged — `DrlxRuleAstRuntimeBuilder.resolvePatternType`
  resolves the type via `entryPointTypes.get(p.entryPoint())` when
  `typeName` is blank; matches the existing unbound-oopath path used
  inside `not` / `exists` / `and` / `or`)
- `bindName` = `syntheticBindName` (e.g. `"$inline0"` — the new bit)
- `entryPoint`, `conditions`, `castTypeName`, `positionalArgs`,
  `passive`, `watchedProperties` are identical to the existing extraction.

**`buildAccumulator(AccumulateItemContext, String srcBindName, String finalDotIdent) → AccumulatorIR`**

New variant of the existing `buildAccumulator(AccumulateItemContext)`.
The existing method reads `argExpressions` from `call.expression()` and
derives `referencedBindings` from those expressions. The inline-from
variant builds them directly:

- If `finalDotIdent != null` (final-dot form):
  - `argExpressions     = [srcBindName + "." + finalDotIdent]`
    (e.g. `["$inline0.age"]`)
  - `referencedBindings = [srcBindName]`
- If `finalDotIdent == null` (bare-oopath / zero-arg form):
  - `argExpressions     = []`
  - `referencedBindings = []`
  - Arity validation in `buildSingleAccumulator` accepts this for
    `count()` (acceptsZeroArgs) and rejects it for everything else
    with the existing arity error.

`resultTypeName`, `resultBindName`, `functionName` come from
`AccumulateItemContext` identically to the existing builder.

### Why the runtime builder sees nothing new

The synthesised `PatternIR.bindName()` is `"$inline0"` instead of a
user-written `"p"`. `DrlxRuleAstRuntimeBuilder.buildPattern` threads
whatever string it receives as the `Pattern` identifier (`$` is a valid
Drools binding character — same convention legacy DRL uses for synthetic
declarations). The accumulator's `argExpressions[0]` is
`"$inline0.age"`, which `DrlxLambdaCompiler.createValueExtractor`
compiles against the source class using the synthetic binding as
`srcBindingName` exactly as it does for `"p.age"`.

The source binding lives only in the accumulate's inner scope —
identical to v1's handling of `p`. It never escapes to `outerScope`,
so the user never sees `$inline0` even if they tried to reference it
from a `do` block.

## Data flow

### `var avgAge = avg(/persons.age), do { results.add(avgAge); }`

```
Visitor builds:
  PatternIR    (typeName="", bindName="$inline0", entryPoint="persons",
                conditions=[], castTypeName=null, positionalArgs=[],
                passive=false, watchedProperties=[])
  AccumulatorIR(resultTypeName="var", resultBindName="avgAge",
                functionName="avg",
                argExpressions=["$inline0.age"],
                referencedBindings=["$inline0"])
  AccumulatePatternIR(srcPatternIR, [accumulatorIR])

Runtime builder (unchanged):
  srcPattern = Pattern(Person.class, "$inline0")
  innerScope = {$inline0 → BoundVariable("$inline0", Person, srcPattern, srcDecl)}
  acc = DrlxLambdaAccumulator(AvgFn, extractor("$inline0.age", Person, "$inline0"))
  single = SingleAccumulate(srcPattern, required, acc)
  wrap   = Pattern(Double.class, "avgAge")   // typed result pattern, identical to v1
  outerScope += {avgAge → ...}
```

### `var minAge = min(/persons.age), var maxAge = max(/persons.age)`

```
Visitor:
  // First accumulator: emit immediately with $inline0
  lhs.add(AccumulatePatternIR(
            PatternIR(typeName="", bindName="$inline0", entryPoint="persons", ...),
            [AccumulatorIR(min, ["$inline0.age"], ["$inline0"])]))

  // Second accumulator: emit immediately with $inline1 (fresh synthetic)
  lhs.add(AccumulatePatternIR(
            PatternIR(typeName="", bindName="$inline1", entryPoint="persons", ...),
            [AccumulatorIR(max, ["$inline1.age"], ["$inline1"])]))

Runtime builder (existing single-accumulate path, twice):
  For each AccumulatePatternIR:
    srcPattern = Pattern(Person.class, "$inlineN")
    acc        = DrlxLambdaAccumulator(MinFn or MaxFn, extractor)
    single     = SingleAccumulate(srcPattern, [], acc)
    wrap       = typed result Pattern named "minAge" or "maxAge"

Note: the Rete network has TWO source-pattern joins on /persons rather than
one — matches DRL's behavior for separate `from accumulate(...)` clauses.
Optimisation by source-equivalence is out of scope.
```

### `long n = count(/persons)`

```
Visitor:
  // finalDotIdent == null → bare-oopath form
  PatternIR     (typeName="", bindName="$inline0", entryPoint="persons", ...)
  AccumulatorIR (resultTypeName="long", resultBindName="n",
                 functionName="count", argExpressions=[], referencedBindings=[])

Runtime builder:
  srcPattern = Pattern(Person.class, "$inline0")
  buildSingleAccumulator sees acceptsZeroArgs=true, argCount=0, extractor=null
  acc = DrlxLambdaAccumulator(CountFn, null)
  single = SingleAccumulate(srcPattern, [], acc)
```

## Error handling

| Failure point | Where caught | Message |
|---|---|---|
| Final-dot extractor on a zero-arg function (e.g. `count(/persons.age)`) | `DrlxToRuleAstVisitor.buildRule` (new throw) | `function 'count' does not accept a final-dot extractor in rule '<R>'; use 'count(/persons)' instead` |
| Regular accumulate without preceding pattern | `DrlxToRuleAstVisitor.buildRule` (existing throw, unchanged) | `accumulate item without a preceding pattern in rule '<R>'` |
| Bare oopath for non-zero-arg function (e.g. `avg(/persons)`) | `DrlxRuleAstRuntimeBuilder.buildSingleAccumulator` (existing arity check, unchanged) | `function 'avg' requires exactly 1 argument, got 0` |
| Final-dot extractor references a property the source class doesn't have (e.g. `avg(/persons.nope)`) | `DrlxLambdaCompiler.createValueExtractor` via MVEL3 type-check (existing #48 path) | MVEL3's "unknown identifier" build-time exception, wrapped per #48 |
| Unknown accumulate function | `AccumulateFunctionRegistry.resolve` (existing) | unchanged |

The zero-arg-function guard happens at visitor time because the visitor
already knows whether a final-dot identifier was supplied and the
function-name string. Resolving against
`AccumulateFunctionRegistry.acceptsZeroArgs()` from the visitor is
fine — the registry is a static map. A user-imported custom function
(out of scope, #53) that turns out to accept zero args would not be
covered by this check; that's a deliberate gap to be revisited under
#53.

## Testing

All in `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/AccumulateTest.java`.

### New behavioural tests (`withSession` execution)

| Test | Rule fragment | Assertion |
|---|---|---|
| `inlineFromAvg` | `var avgAge = avg(/persons.age), do { results.add(avgAge); }` | `containsExactly(40.0)` for persons aged 20/40/60 — proves the synthetic source, the rewritten extractor `$inline0.age`, and the avg function wire up |
| `inlineFromCount` | `long n = count(/persons), do { results.add(n); }` | `containsExactly(2L)` for 2 persons inserted — proves the bare-oopath / zero-arg form |
| `inlineFromMultipleSameSource` | `var minAge = min(/persons.age), var maxAge = max(/persons.age), var avgAge = avg(/persons.age)` + consequence adding all three to `results` | `containsExactly(20, 60, 40.0)` — proves multiple inline-from items each produce an independent `SingleAccumulate` with its own synthetic source pattern, and all three result bindings are visible to the consequence |
| `inlineFromWithSourceConstraint` | `var totalSenior = sum(/persons[age >= 40].age)` | proves the oopath's `[...]` constraints carry into the synthesised source pattern — the `Person` test domain has `age`, not `location` |
| `inlineFromComposesWithBoundPattern` | `var p : /persons[age >= 18], var n = count(/persons), do { results.add(p.name); results.add(n); }` | proves a bound pattern adjacent to an inline-from coexists — bound `p` becomes a regular LhsItem, count(/persons) emits a separate AccumulatePatternIR. With persons "A"/20 and "B"/40 inserted: two firings, each adding the person's name and the count of all persons. |

### New structural tests (AST shape — no `KieSession`)

Same harness pattern as #49's `singleFunctionEmitsSingleAccumulate` /
`multiFunctionEmitsOneMultiAccumulate`. Walk the LHS via
`kieBase.getKiePackage(...).getRules() → Rule.getLhs() → first child Pattern
→ pattern.getSource()`.

| Test | Rule fragment | Asserts |
|---|---|---|
| `inlineFromSynthesisesSourcePattern` | `var avgAge = avg(/persons.age)` | LHS has one accumulate result Pattern; its `getSource()` is `SingleAccumulate`; the underlying source `Pattern.getDeclaration().getIdentifier()` starts with `$inline`; result Pattern has one declaration named `avgAge` |
| `inlineFromMultipleEmitsSeparateSingleAccumulates` | `var minAge = min(/persons.age), var maxAge = max(/persons.age)` | two distinct accumulate result Patterns under the rule LHS; each `getSource()` is `SingleAccumulate`; each underlying source `Pattern` has a different identifier (`$inline0` vs `$inline1`). Proves no fold-by-source-equivalence and matches DRL behavior for separate accumulate clauses. |
| `inlineFromDifferentSourceTypes` | `var personAvg = avg(/persons.age), var seniorAvg = avg(/seniors.age)` | same shape as above — two separate `SingleAccumulate`s; underlying source patterns have different `ObjectType`s |

### Negative tests

| Test | Rule fragment | Expects |
|---|---|---|
| `inlineFromCountWithFinalDotRejected` | `long n = count(/persons.age)` | build throws `RuntimeException` containing `"function 'count' does not accept a final-dot extractor"` |
| `inlineFromBareOopathRejectedForNonZeroArg` | `var avgAge = avg(/persons)` | build throws `RuntimeException` containing `"function 'avg' requires exactly 1 argument, got 0"` (existing arity error, surfaced on the inline-from path) |
| `inlineFromUnknownPropertyFailsAtBuild` | `var avgAge = avg(/persons.nope)` | MVEL3 build-time error from #48's extractor compile path |

### Existing tests (must stay green)

All `AccumulateTest` cases from #39 / #48 / #49 — the regular form is
untouched. The visitor's new dispatch is gated on
`call.inlineFromOopath != null`; the existing path is the second
alternative of `accumulateCall` and behaves identically.

### Intentional coverage gap

No new test for the pre-built (`DrlxPreBuildLambdaCompilerTest`)
round-trip path. The `AccumulatorIR` / `PatternIR` shapes are unchanged
— only the values of `bindName` differ (synthetic strings now possible).
`DrlxRuleAstParseResult` already serialises `PatternIR.bindName()`. If
protobuf round-trips break, the existing 217-test pre-build coverage
catches it. Adding a dedicated case is duplicate cost.

## Plan deviations (anticipated)

Low. One latent question: whether ANTLR's LL(*) actually picks the
inline-from alternative without ambiguity warnings against the
existing `expression` rule. The disjoint-prefix argument
(`/` / `?/` vs Java expression starts) is sound on paper but
`-Werror` or `-language=Java` quirks could surface a diagnostic. If
the generator complains, the fallback is a one-token semantic
predicate (`{_input.LT(1).getType() == DRLX_SLASH || _input.LT(1).getType() == QUESTION}?`)
in front of the inline-from alternative — costs one line.

## Out of scope (filed as separate issues under #26)

- **#52** — `and(...)` source in inline-from
- **#51** — `acc()` keyword forms (2/3/5-param)
- **#54** — Outer-binding refs in extractor expressions
- **#53** — Custom user-imported accumulate functions

## References

| Topic | Path |
|---|---|
| Issue | https://github.com/tkobayas/drlx-parser/issues/50 |
| Parent epic | https://github.com/tkobayas/drlx-parser/issues/26 |
| v1 spec | `specs/2026-05-13-drlx-accumulate-design.md` |
| #48 spec | `specs/2026-05-15-48-accumulate-extractor-mvel3-design.md` |
| #49 spec | `specs/2026-05-18-49-multiaccumulate-folding-design.md` |
| Grammar to extend | `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` (`accumulateCall`, line ~207) |
| Visitor to extend | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` (`buildRule` fold loop, ~line 99; `buildAccumulator`, ~line 265; `buildPatternFromOopath`, ~line 413) |
| Runtime builder (unchanged, for reference) | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` (`buildAccumulatePattern`, line 405) |
| Drools DRL precedent — no fold across separate `accumulate(...)` clauses | `~/usr/work/mvel3-development/drools/drools-mvel/src/main/java/org/drools/mvel/builder/MVELAccumulateBuilder.java:144` (`isMultiFunction()` dispatch — applies per `AccumulateDescr`, never across multiple Descrs) |
| Drools DRL `AccumulateDescr.isMultiFunction()` | `~/usr/work/mvel3-development/drools/drools-drl/drools-drl-ast/src/main/java/org/drools/drl/ast/descr/AccumulateDescr.java:223` |
| DRLXXXX §Accumulate | `docs/DRLXXXX.md` line ~855 |
