# DRLX Accumulate (#39) — v1 Design

**Issue:** tkobayas/drlx-parser#39
**Parent epic:** #26 (DrlxCompiler enhancement round 2)
**Spec reference:** DRLXXXX §Accumulate, lines 820–890
**Date:** 2026-05-13

## Scope

DRLXXXX defines five accumulate forms. v1 ships only the two simplest:

1. **Simple function** — `var p : /persons[...], var avgAge = avg(p.age), do {...}`
2. **Multi-function** — `var p : /persons[...], var avgAge = avg(p.age), var minAge = min(p.age), var maxAge = max(p.age), do {...}`

Forms 3–5 (the `acc(...)` keyword with 2/3/5 parameters), the inline `avg(/persons.age)` "from" notation, custom Holder objects, and multi-pattern sources via `and(...)` are **out of scope** for v1 and tracked as fast-follows under #26.

Built-in functions covered: `avg`, `sum`, `min`, `max`, `count`. Each maps to the corresponding Drools `AccumulateFunction` subclass (`AverageAccumulateFunction`, `SumAccumulateFunction`, `MinAccumulateFunction`, `MaxAccumulateFunction`, `CountAccumulateFunction`).

Lowering: **N × SingleAccumulate**. Each accumulator in the multi-function form becomes its own `SingleAccumulate` sharing a clone of the source pattern. Folding into `MultiAccumulate` (matching Drools DRL's optimised path) is deliberately deferred.

## Semantics

**Shared rules** (apply to both v1 forms):

- The preceding pattern is consumed as the accumulate's source. Its binding (`p`) is **internal** — not visible to later patterns or to the consequence.
- Each result binding (`avgAge`, etc.) **is** visible to subsequent patterns and to the consequence. Its type is the accumulator's return type (`Double` for `avg`, `Long` for `count`, etc.).
- A typed binding (`int sum = sum(p.age),` or `BigDecimal total = sum(p.amount),`) declares the result variable with that explicit type. `var` infers from the accumulator function.

**Lowering — single-function form** (`var p : /persons[...], var avgAge = avg(p.age),`): exactly **one** `SingleAccumulate`, whose `source` is the inner Pattern bound to `p`, wrapped by one outer result `Pattern` carrying the `avgAge` declaration.

**Lowering — multi-function form** (`var p : /persons[...], var avgAge = avg(p.age), var minAge = min(p.age), var maxAge = max(p.age),`): **N independent `SingleAccumulate`s** (one per accumulator function) in v1. Each accumulator gets its own clone of the inner Pattern as its source, and its own outer result `Pattern`. The three result Patterns all become direct children of the enclosing GroupElement; the inner Pattern template itself is never added to that GroupElement.

This is functionally equivalent to a single `MultiAccumulate` over a shared source, but uses more beta-memory at runtime. Folding adjacent accumulators sharing one source into a single `MultiAccumulate` (matching Drools DRL's optimised path) is a deliberate fast-follow under #26.

The shared-rules above match Drools' `SingleAccumulate` shape and stay consistent with how the 2-param `acc(p : /persons, var avgAge = avg(p.age))` form nests its source. The DRLXXXX remark about "pushing up the child nodes" (line 827) is a network-optimisation aside, not a semantic divergence.

## Grammar

Add one new `ruleItem` alternative and two new rules in `DrlxParser.g4`. No lexer changes — `VAR` and `typeType` are already defined in the imported JavaLexer/JavaParser.

```antlr
ruleItem
    : rulePattern
    | accumulateItem ','              // NEW
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','
    | conditionalBranch ','
    | ruleConsequence
    ;

// Type may be `var` or any Java type (incl. generics): int, double, BigDecimal, List<Integer>, ...
accumulateItem
    : (VAR | typeType) identifier '=' accumulateCall
    ;

accumulateCall
    : qualifiedName '(' (expression (',' expression)*)? ')'
    ;
```

The argument list is optional to admit `count()` (DRL's `count` takes no argument; we accept the same shape).

ANTLR disambiguation versus `rulePattern` (which contains `boundOopath` with `(':' | '=')`):

| Source                                | First post-`=` token | Alternative chosen |
|---------------------------------------|----------------------|--------------------|
| `var p : /persons[...]`               | (uses `:`)            | rulePattern        |
| `var p = /persons[...]`               | `/`                  | rulePattern        |
| `var avgAge = avg(p.age)`             | `IDENTIFIER (`       | accumulateItem     |

`boundOopath` (used by `rulePattern`) keeps its existing `identifier identifier (':' | '=') oopathExpression` shape. Tightening `boundOopath` to `(VAR | typeType) identifier ...` would catch the same class of mistakes there but ripples into every pattern test — left as a follow-up under #26.

## IR

Add new variants in `DrlxRuleAstModel`:

```java
public sealed interface LhsItemIR
        permits PatternIR, GroupElementIR, EvalIR, AccumulatePatternIR { }

public record AccumulatePatternIR(PatternIR source,
                                  List<AccumulatorIR> accumulators) implements LhsItemIR {
    public AccumulatePatternIR {
        accumulators = List.copyOf(accumulators);
    }
}

public record AccumulatorIR(String resultTypeName,      // "var", "int", "java.lang.Double", ...
                            String resultBindName,      // "avgAge"
                            String functionName,        // "avg" or "Func.avg"
                            List<String> argExpressions,        // ["p.age"] (1 entry for v1 builtins)
                            List<String> referencedBindings) {  // ["p"]
    public AccumulatorIR {
        argExpressions     = List.copyOf(argExpressions);
        referencedBindings = List.copyOf(referencedBindings);
    }
}
```

Rationale for bundling source + accumulators in `AccumulatePatternIR`:

- The accumulate's source is always the immediately preceding pattern in v1. A flat sibling-IR shape would force the runtime builder to do list look-back. Bundling expresses the relationship at the IR layer.
- For multi-function, the N accumulators share one source. Bundling makes that explicit and makes the future MultiAccumulate folding a one-spot change.

`argExpressions` is a list to leave room for multi-arg accumulators (DRLXXXX line 856 shows `max(p1.age, p2.age)` once multi-pattern source lands). v1 builtins are all single-arg; the list will hold one entry.

## Protobuf

Add a new `oneof` arm and two messages in `drlx_rule_ast.proto`:

```protobuf
message LhsItemParseResult {
  oneof kind {
    PatternParseResult pattern  = 1;
    GroupElementParseResult group = 2;
    EvalParseResult eval        = 3;
    AccumulatePatternParseResult accumulate_pattern = 4;   // NEW
  }
}

message AccumulatePatternParseResult {
  PatternParseResult source = 1;
  repeated AccumulatorParseResult accumulators = 2;
}

message AccumulatorParseResult {
  string result_type_name = 1;
  string result_bind_name = 2;
  string function_name    = 3;
  repeated string arg_expressions     = 4;
  repeated string referenced_bindings = 5;
}
```

No wire-compat constraints — DRLX is pre-1.0 experimental.

## Visitor

`DrlxToRuleAstVisitor` gains one transformation: when walking `ruleBody`, fold a `boundOopath` followed by one or more `accumulateItem`s into a single `AccumulatePatternIR`.

```java
List<LhsItemIR> foldAccumulates(List<LhsItemIR> rawItems) {
    List<LhsItemIR> out = new ArrayList<>();
    for (int i = 0; i < rawItems.size(); i++) {
        LhsItemIR item = rawItems.get(i);
        if (item instanceof PatternIR src && peekAccumulators(rawItems, i + 1)) {
            List<AccumulatorIR> accs = new ArrayList<>();
            int j = i + 1;
            while (j < rawItems.size() && rawItems.get(j) instanceof PendingAccumulatorIR a) {
                accs.add(a.toAccumulator());
                j++;
            }
            out.add(new AccumulatePatternIR(src, accs));
            i = j - 1;
        } else {
            out.add(item);
        }
    }
    return out;
}
```

(The `PendingAccumulatorIR` is a transient visitor-local type — never reaches `DrlxRuleAstModel`.)

After folding, the visitor populates each `AccumulatorIR.referencedBindings` by scanning argument expressions for identifiers — using the same regex-based identifier scanner that `EvalIR` calls today. **No scope checking happens at visitor time.** The visitor does not validate that referenced identifiers exist; it does not register/deregister any bindings; it does not emit "unknown function" diagnostics. All such validation is build-time (see next section). This is consistent with the rest of DRLX today — `EvalIR` similarly defers identifier validity to the lambda compiler.

The visitor's only obligation is that the IR it emits is structurally well-formed: a `boundOopath` followed by one or more `accumulateItem`s collapses into one `AccumulatePatternIR`, regardless of what's inside the args.

## Runtime builder

`DrlxRuleAstRuntimeBuilder.buildLhs` gains one branch. Two scopes are maintained explicitly:

- **outer scope** — the existing `boundVariables` map. Contains bindings visible to later LHS items and to the consequence. The accumulate result bindings are added here. The source pattern's binding (`p`) is **not** added here.
- **inner scope** — a builder-local `Map<String,BoundVariable>` constructed at the accumulate site. It is a *copy* of the outer scope, *plus* the source pattern's binding (`p` → its `Pattern` and class). The lambda compiler uses this map when compiling extractor expressions like `p.age`. The inner scope is discarded as soon as the accumulate finishes — it never escapes the branch.

```java
} else if (item instanceof AccumulatePatternIR accPat) {
    Pattern innerTemplate = buildPattern(accPat.source(), ...);
    Map<String, BoundVariable> innerScope = new LinkedHashMap<>(boundVariables);
    Declaration srcDecl = innerTemplate.getDeclaration();
    Class<?> srcClass = ((ClassObjectType) innerTemplate.getObjectType()).getClassType();
    innerScope.put(srcDecl.getIdentifier(),
                   new BoundVariable(srcDecl.getIdentifier(), srcClass, innerTemplate));

    for (AccumulatorIR acc : accPat.accumulators()) {
        Pattern innerClone = clonePattern(innerTemplate, ...);
        SingleAccumulate single = buildSingleAccumulate(acc, innerClone, innerScope, ...);
        Pattern resultPattern   = wrapResultPattern(acc, single, ...);
        parent.addChild(resultPattern);
        boundVariables.put(acc.resultBindName(),
                new BoundVariable(acc.resultBindName(),
                                  resultClassFor(acc),
                                  resultPattern));
    }
}
```

The inner pattern template is **not** added to `parent` — each `SingleAccumulate` owns its own clone. The parent GroupElement sees only the N result Patterns. After the loop, `innerScope` is dropped; `boundVariables` (outer scope) now contains the result bindings but not `p`.

Subsequent LHS items or the consequence that reference `p` produce a build-time "unknown binding" error from the lambda compiler — this is the same error path as any other undefined identifier and requires no new diagnostic machinery.

### Function-name resolution

```java
private static final Map<String, Class<? extends AccumulateFunction>> BUILTIN_ACC = Map.of(
    "avg",   AverageAccumulateFunction.class,
    "sum",   SumAccumulateFunction.class,
    "min",   MinAccumulateFunction.class,
    "max",   MaxAccumulateFunction.class,
    "count", CountAccumulateFunction.class);
```

Resolution rules:

1. **Unqualified name** (e.g., `avg`) — look up in `BUILTIN_ACC`. If absent, build-time error: `unknown accumulate function 'X' — built-ins are: avg, sum, min, max, count`.
2. **Qualified name** (e.g., `Func.avg`) — **rejected in v1** with: `qualified accumulate function names ('X.y') are not yet supported — use unqualified built-ins (avg, sum, min, max, count)`. The grammar still accepts the form (so copy-pastes from DRLXXXX line 829 parse rather than syntax-erroring), but resolution fails fast and explicitly. When custom accumulate functions land, this rejection lifts and the qualified name resolves against imports.

The earlier alternative — silently treating `Func.avg` as an alias for `avg` — is rejected because `Random.avg(p.age)` or `Stats.avg(p.age)` would then quietly resolve to the built-in `avg` and hide a real bug.

Custom accumulate functions imported by the user are out of v1. The resolution table is keyed by name, so adding them later is "drop a new entry into the map (or extend the lookup to walk imports)".

### Value-extractor lambda

For an argument like `"p.age"`, the existing `lambdaCompiler` (the same path that compiles `EvalIR.expression`) produces a small MVEL3 lambda that takes the `p` object and returns `p.age`. This lambda becomes the value-extractor passed into a Drools `Accumulator` wrapper.

`count(...)` accepts either zero arguments (`count()`) or one argument that it ignores (`count(p)`); Drools' `CountAccumulateFunction` increments per tuple regardless. In both cases the wrapper skips extractor construction.

### Accumulator wrapper

DRLX writes a thin `DrlxLambdaAccumulator implements Accumulator` that holds:
- the `AccumulateFunction` instance (e.g. `new AverageAccumulateFunction()`),
- the value-extractor lambda (or `null` for `count`),
- the `Declaration[]` of in-scope bindings the extractor needs.

This avoids pulling `drools-model-compiler` into the DRLX runtime path. (Drools' own `LambdaAccumulator` is in that module; we write our own equivalent inside `drlx-parser-core`.)

### Result pattern assembly

```java
Pattern wrapResultPattern(AccumulatorIR acc, SingleAccumulate single, ...) {
    Class<?> resultType = resultClassFor(acc);                  // Double / Long / Number / ...
    Pattern p = new Pattern(declarationOffset++,
                            new ClassObjectType(resultType),
                            null);
    p.addDeclaration(new Declaration(acc.resultBindName(),
                                     new SelfReferenceClassFieldReader(resultType),
                                     p, true));
    p.setSource(single);
    return p;
}
```

Result-type inference for `resultTypeName == "var"` is a two-step lookup:
1. A static table keyed by built-in function (e.g., `avg → Double`, `count → Long`, `sum → Number`, `min/max → comparable element type`).
2. For `sum`/`min`/`max` whose result type depends on the *argument's* type, the lambda compiler already knows the argument type from its type-resolution pass; the builder threads that through.

For explicit types (`int avgAge = avg(p.age)`), validate assignment-compatibility against the function's inferred result type. Reject mismatches at build time.

## Testing

Five layers, mirroring how recent features (#37, #45) were tested.

1. **Grammar — `DrlxParserTest`**
   - `var x = avg(p.age),` parses
   - `int x = sum(p.age),` parses
   - `BigDecimal x = avg(p.amount),` parses
   - `long n = count(),` parses (zero-arg form)
   - `long n = count(p),` parses (one-arg form)
   - `Func.avg(p.age)` qualified form parses (rejection is build-time, not parse-time)
   - Malformed forms (missing `,`, missing `=`, missing parens) reject with grammar errors

2. **Visitor / IR — `DrlxToJavaParserVisitorTest`**
   - One pattern + one accumulate folds into `AccumulatePatternIR(1 source, 1 accumulator)`
   - One pattern + three accumulates folds into `AccumulatePatternIR(1 source, 3 accumulators)`
   - `referencedBindings` populated by identifier-scan of the arg expressions (e.g., `["p"]` for `avg(p.age)`, `[]` for `count()`)
   - **No scope-checking tests at this layer** — the visitor emits IR without validating identifier references. Scope errors are exercised at end-to-end layer 5.

3. **Proto roundtrip — `DrlxRuleAstParseResultTest`**
   - Write an `AccumulatePatternIR` (single + multi accumulator) to proto, read back, assert structural equality
   - Include a zero-arg accumulator (`count()`) and verify `arg_expressions` is empty on the wire

4. **Runtime builder — `DrlxRuleAstRuntimeBuilderTest`**
   - Synthetic IR → assert the produced `RuleImpl` has the expected outer `Pattern` (right `ObjectType`, single `Declaration` named after `resultBindName`), whose `source` is a `SingleAccumulate` whose own `source` is the inner Pattern
   - One parameterised case per built-in (avg / sum / min / max / count, including `count()` zero-arg)
   - **Inner scope test**: extractor compilation succeeds for `p.age` even when `p` is absent from the outer `boundVariables` map (i.e., the inner scope mechanism actually plugs `p` in for the extractor)
   - **Outer scope test**: after the accumulate branch runs, `boundVariables` contains the result binding but not `p`
   - Build-time error when `functionName` doesn't resolve (unknown unqualified) — specific message
   - Build-time error when `functionName` is qualified (e.g., `Func.avg`) — specific message naming the v1 restriction

5. **End-to-end — `syntax/AccumulateTest`**
   - **Single function**: avg/sum/min/max/count over a known `/persons` dataset, assert consequence receives the expected value
   - **count() with no argument**: `var n = count(), do {...}` — assert `n` equals dataset size
   - **Multi function**: `var avg = avg(p.age), var max = max(p.age), var min = min(p.age)` — three SingleAccumulates sharing one source; consequence sees all three bindings
   - **Result chains to later pattern**: `var sum = sum(p.amount), var t : /thresholds[value < sum], do {...}`
   - **Negative: qualified name rejected**: `var x = Func.avg(p.age),` → build-time error mentioning the v1 restriction
   - **Negative: unknown function**: `var x = bogus(p.age),` → build-time error with the built-ins list
   - **Negative: arg references undefined binding**: `var x = avg(q.age),` (no `q` bound) → build-time error from lambda compiler
   - **Negative: source binding `p` used after the accumulate**: subsequent pattern or consequence references `p` → build-time error from lambda compiler ("unknown binding `p`")

Build-cache parity (proto roundtrip through `DrlxRuleAstParseResult`) is exercised by the existing cache test infrastructure once layer 3 passes.

## Out of scope (tracked under #26)

- The `acc(...)` keyword with 2/3/5 parameters (DRLXXXX lines 842–890).
- Inline `avg(/persons.age)` "from" form (DRLXXXX line 834).
- Multi-pattern source via `and(...)` (DRLXXXX lines 839, 856).
- Custom Holder objects (DRLXXXX line 883).
- Custom accumulate functions imported by user code.
- MultiAccumulate folding (network optimisation).
- Tightening `boundOopath` to `(VAR | typeType) identifier ...` for consistency.

## Open questions

None blocking implementation. Two minor details to finalise in the implementation plan:

- Exact API of the `DrlxLambdaAccumulator` wrapper — depends on which Drools `Accumulator` constructor we target.
- Whether the result-type table is hand-maintained or derived from reflecting on each `AccumulateFunction.createContext()`.
