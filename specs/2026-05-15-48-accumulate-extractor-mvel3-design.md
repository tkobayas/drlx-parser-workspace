# Accumulate value-extractor — MVEL3 compile path (#48)

**Status:** ready for implementation plan
**Parent epic:** #26
**Follows:** #39 (accumulate v1 shipped at project HEAD `538185b`)
**DRLXXXX reference:** §Accumulate (lines 820–890)

## Summary

Lift the v1 accumulate restriction that rejects anything beyond a simple
`<binding>.<property>` argument. After this change, arbitrary MVEL3
expressions over the source binding compile through `DrlxLambdaCompiler`
and become a `Function<Object, Object>` value-extractor on
`DrlxLambdaAccumulator`.

The reflection-based `buildSimpleExtractor` in
`DrlxRuleAstRuntimeBuilder` is deleted in full. No fast-path is retained
for trivial `binding.property` — defer that to a separate optimisation
issue if profiling justifies it.

## Scope

**In:** any MVEL3 expression that resolves entirely against the source
binding — arithmetic, property chains, method calls, references to
multiple properties of the same binding.

| Form | Status v1 | Status after #48 |
|---|---|---|
| `sum(p.age)` | accepted (reflection) | accepted (MVEL3 — uniform path) |
| `sum(p.age + 1)` | rejected at build | accepted |
| `sum(p.age * p.age)` (multiple refs to same binding) | rejected at build | accepted |
| `sum(p.name.length())` (method call) | rejected at build | accepted |
| `count()`, `count(p)`, `count(<anything>)` | accepted (extractor `null`; argument parsed but not validated) | unchanged |

**`count(expr)` validation:** the argument of `count` is parsed by the
visitor but its semantic correctness is not validated — the registry
flags `count` as `acceptsZeroArgs`, so `buildSingleAccumulate` skips
extractor construction entirely. As a result, `count(garbage)` and
`count(q.factor)` (where `q` is from outer scope) both parse and run as
plain `count()`. This matches v1 exactly; #48 does not change it.
Compile-validating `count`'s argument while still ignoring its runtime
value is a deliberate future improvement and is out of scope here.

**Out:** references to outer-scope bindings inside an extractor
expression (e.g. `sum(p.age * q.factor)` where `q` was bound earlier
in the rule). The `DrlxLambdaAccumulator.extractor` field stays
`Function<Object, Object>` — single-fact input only. This was
originally filed as #54 but later dropped — the use case is not in
the DRLXXXX.md language spec.

## Architecture

Three pieces, all isolated.

### 1. New compile method on `DrlxLambdaCompiler`

Public entry point `createValueExtractor`, parallel to
`createEvalExpression`:

```java
public DrlxValueExtractor createValueExtractor(String argExpr,
                                                Class<?> srcClass,
                                                String sourceBindingName);
```

Internals follow the three-mode pattern already used by
`createEvalExpression` / `createBetaLambdaConstraint`:

1. Bump `lambdaCounter`.
2. `tryLoadPreCompiled(counter, argExpr, "value extractor")` — hit
   returns a `DrlxValueExtractor` with the evaluator already bound.
3. Miss → batch path: build `CompilerParameters<Map<String,Object>,
   Void, Object>` via `MVEL.<Object>map(Declaration.of(sourceName,
   srcClass)).<Object>out(Object.class)`, register a `LambdaHandle`
   with `batchCompiler`, return a `DrlxValueExtractor` with a `null`
   evaluator and queue a `PendingLambda` for later late-binding.
4. Fire `onLambdaCreated(counter, argExpr)` so
   `DrlxPreBuildLambdaCompiler` records metadata.

`imports` field is passed into `CompilerParameters` the same way as
the other paths.

### 2. New class `DrlxValueExtractor`

Location: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxValueExtractor.java`.

Implements:
- `Function<Object, Object>` — slots into the existing
  `DrlxLambdaAccumulator.extractor` field with no change.
- `EvaluatorSink` — `bindEvaluator(Evaluator)` injects the compiled
  evaluator after `compileBatch(classLoader)` resolves the handle.

State: expression string, source-binding name, `Evaluator<Map<String,
Object>, Void, Object>` (initially `null` on the batch path; populated
on bind).

`apply(Object fact)`:
1. Build a 1-entry `HashMap` with `{sourceBindingName → fact}`.
2. Call `evaluator.eval(map)` and return.
3. Wrap any thrown exception in a `RuntimeException` carrying the
   expression string (read off the stored field), mirroring the existing
   error-wrap style in `DrlxLambdaAccumulator.accumulate`. The fact's
   actual class is reachable via `fact.getClass()` at the failure point
   if a future improvement wants to include it; not stored on the
   extractor.

### 3. Call-site change in `DrlxRuleAstRuntimeBuilder`

In `buildSingleAccumulate`, replace:

```java
extractor = buildSimpleExtractor(acc.argExpressions().get(0), srcClass);
```

with:

```java
extractor = lambdaCompiler.createValueExtractor(
        acc.argExpressions().get(0),
        srcClass,
        srcDecl.getIdentifier());
```

Delete `buildSimpleExtractor`, `isIdentifier`, `findGetter` in full —
the entire reflection path goes away.

`DrlxLambdaAccumulator` is unchanged. The `Declaration[] required`
plumbing for `SingleAccumulate` is unchanged.

## Data flow

**Build time** (once per accumulate site):

```
buildSingleAccumulate
  argExpr="p.age + 1", srcClass=Person.class, sourceName="p"
    → DrlxLambdaCompiler.createValueExtractor
        → MVEL.<Object>map(Declaration.of("p", Person.class))
              .<Object>out(Object.class)
              .expression("p.age + 1")
              .imports(imports)
              .classManager(batchCompiler.getClassManager())
              .generatedClassName("GeneratorEvaluator__")
              .build()
        → batchCompiler.add(params) → LambdaHandle
        → new DrlxValueExtractor("p.age + 1", "p", null)
        → pendingLambdas.add(PendingLambda(handle, extractor))
        → onLambdaCreated(counter, argExpr)
```

Later, `compileBatch(classLoader)` resolves all pending handles and
fires `extractor.bindEvaluator(...)`.

**Runtime** (every accumulate call):

```
DrlxLambdaAccumulator.accumulate(... handle ...)
  extractor.apply(handle.getObject())
    → map = new HashMap<>(1); map.put("p", fact)
    → return evaluator.eval(map)
  accFunction.accumulate(context, value)
```

One HashMap allocation per call. Same profile as
`DrlxLambdaBetaConstraint.buildEvalMap`.

## Error handling

| Failure point | Handling |
|---|---|
| Compile-time (bad property, syntax error, type mismatch) | Raised by `batchCompiler.compile(classLoader)` — bubbles out of `DrlxRuleBuilder.build()` as the MVEL3-native exception. No new error code. |
| Runtime (extractor throws on bad fact, NPE inside expression) | Wrapped in `RuntimeException` inside `DrlxValueExtractor.apply`, carrying the expression string. |
| Pre-built metadata mismatch (stale .class for the expression) | Existing `handleMetadataMismatch` flow — `FAIL_FAST` by default, optional `FALLBACK`. Log line tags the kind as `"value extractor"`. |

The visible `"v1 accumulate supports only simple <binding>.<property>"`
message is deleted along with `buildSimpleExtractor`.

## Testing

All tests in `AccumulateTest.java` unless noted.

**Renamed / flipped:**
- `complexExtractorExpressionRejectedAsV1Limitation` →
  `sumOfArithmeticExpression`. Body changes from
  `assertThatThrownBy(...).hasMessageContaining("v1 accumulate supports only simple")`
  to inserting three persons and asserting `MyUnit.results` carries
  the expected `sum(p.age + 1)` total.

**New positive tests** (use existing fields on `org.drools.drlx.domain.Person` — `name:String`, `age:int`, `address:Address` with `city:String`):

| Test | Expression | Coverage |
|---|---|---|
| `sumOfArithmeticExpression` | `sum(p.age + 1)` | binary arithmetic, constant operand |
| `sumOfMethodCallExpression` | `sum(p.name.length())` | method call on a navigated property |
| `sumOfMultipleBindingRefs` | `sum(p.age * p.age)` | two refs to the same binding within one expression |
| `avgOfExpression` | `avg(p.age + 1)` | confirms non-sum functions pick up the same extractor |

**New negative test:**
- `extractorExpressionWithUnknownPropertyFailsAtBuild` — `sum(p.notAField)`
  throws at build. Assert on `RuntimeException` only; do NOT assert on
  MVEL3's exact message (brittle across MVEL3 versions).

**Regression coverage:**
- Full `drlx-parser-core` suite (currently 209 green per the 2026-05-15
  handover). Existing `sumOfPropertyAccess`, `avgOfPropertyAccess` etc.
  now go through the MVEL3 path instead of reflection — must stay green.
- `countOfBinding`, `countWithoutArgs` — extractor stays `null`,
  unchanged.
- Error-path tests for unknown function / qualified name / source-scope
  isolation — still relevant; only the v1-limit test changes.

**Pre-build path:** if an accumulate-aware test exists in
`DrlxPreBuildLambdaCompilerTest`, add one round-trip case for
`sum(p.age + 1)` (write metadata, reload, assert the extractor binds
and runs). Otherwise rely on the existing 209-suite pre-build
coverage — don't add a fresh test class just for this.

## Plan deviations (anticipated)

None expected. The compile-path pattern is firmly established by
`createEvalExpression` and `createBetaLambdaConstraint`; this is a
copy-and-adapt with `Boolean` swapped for `Object` in the output type.

## Out of scope (filed as separate issues under #26)

- #49 — `MultiAccumulate` folding
- #50 — Inline-from form
- #51 — `acc()` keyword forms
- #52 — Multi-pattern source via `and(...)`
- #53 — Custom user-imported accumulate functions
- ~~Outer-binding refs inside accumulate args (e.g. `sum(p.age * q.factor)`
  where `q` is from outer scope) — was filed as #54, dropped because
  the use case is not in the DRLXXXX.md language spec.~~
- Reflection fast-path for trivial `binding.property` — defer until
  profiling justifies it.

## References

| Topic | Path |
|---|---|
| Issue | https://github.com/tkobayas/drlx-parser/issues/48 |
| Parent epic | https://github.com/tkobayas/drlx-parser/issues/26 |
| v1 spec | `specs/2026-05-13-drlx-accumulate-design.md` |
| v1 plan | `plans/2026-05-13-drlx-accumulate-implementation.md` (Task 7.5) |
| Call site to change | `drlx-parser-core/.../DrlxRuleAstRuntimeBuilder.java:449` |
| Reference compile method | `DrlxLambdaCompiler.createEvalExpression` (line 209) |
| Reference runtime wrapper | `DrlxLambdaBetaConstraint.buildEvalMap` |
| v1-limit test (to flip) | `AccumulateTest.complexExtractorExpressionRejectedAsV1Limitation` (line 214) |
