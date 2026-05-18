# Accumulate MultiAccumulate folding — N×SingleAccumulate → one node (#49)

**Status:** ready for implementation plan
**Parent epic:** #26
**Follows:** #48 (MVEL3-backed extractor compile path shipped at project HEAD `27c528f`)
**DRLXXXX reference:** §Accumulate, network-shape note (line 827)

## Summary

v1 lowers the multi-function accumulate form into N independent
`SingleAccumulate`s, each cloning the source pattern. Drools'
canonical convention — confirmed in both `KiePackagesBuilder`
(executable model) and `MVELAccumulateBuilder` (legacy textual DRL) —
is to emit a single `MultiAccumulate` over one source pattern when
the function count exceeds 1, with the result projected through an
`Object[]` and per-binding `ArrayElementReader` declarations.

#49 aligns DRLX with that convention: one source pattern, one
`MultiAccumulate`, one wrapping `Object[]` result `Pattern` carrying
N declarations. Behaviour for the N=1 case is unchanged. The per-iteration
source-pattern clone in v1's `buildAccumulatePattern` loop (`idx++ == 0 ?
srcTemplate : srcTemplate.clone()`) is removed entirely — the new flow
has no loop over source patterns, so no clone call survives.

## Scope

**In:**

| Form | Status v1 (#48) | Status after #49 |
|---|---|---|
| `var x = avg(p.age)` (N=1) | `SingleAccumulate`, typed result Pattern | unchanged |
| `var minAge = min(p.age), var maxAge = max(p.age), var avgAge = avg(p.age)` (N≥2) | N `SingleAccumulate`s, N cloned source patterns | one `MultiAccumulate`, one source pattern, `Object[]` result Pattern |
| Mixed `count() , sum(p.age)` (N≥2 with null + non-null extractors) | N `SingleAccumulate`s | one `MultiAccumulate` (null extractor slot still works — `DrlxLambdaAccumulator.accumulate` already null-checks) |

**Out:**

- Outer-binding references inside extractor expressions
  (e.g. `sum(p.age * q.factor)`) — `MultiAccumulate.requiredDeclarations`
  is passed `new Declaration[0]` here, matching the Drools
  executable-model path (`KiePackagesBuilder.createAccumulate`).
  Drools' legacy textual paths (`MVELAccumulateBuilder`,
  `JavaAccumulateBuilder`) propagate accumulated required decls
  instead; #54 will decide which path DRLX adopts when outer-binding
  refs land. Owned by **#54**.
- GroupBy support and the +1 `arraySize` for the group key — owned by **#40**.
- `acc()` keyword forms — owned by **#51**. #49 applies to the
  `var`-prefixed forms only.
- Reflection fast-path for trivial `binding.property` — deferred until
  profiling justifies it.

## Architecture

`buildAccumulatePattern` in `DrlxRuleAstRuntimeBuilder` becomes a
uniform builder followed by a single dispatch on accumulator count.

### Top-level flow

```
buildAccumulatePattern(accPat):
  srcPattern  = buildPattern(accPat.source(), ...)        // ONE source, no cloning
  innerScope  = outerScope ∪ {sourceBinding}

  Accumulator[] accs = new Accumulator[N]
  for i in 0..N-1:
      accs[i] = buildSingleAccumulator(accumulators[i], srcClass, innerScope)
                // returns a configured DrlxLambdaAccumulator(fn, extractor)

  if N == 1:
      Declaration[] required = requiredFor(accumulators[0], innerScope)   // v1 logic preserved
      Accumulate single = new SingleAccumulate(srcPattern, required, accs[0])
      Pattern wrap     = wrapSingleResultPattern(accumulators[0], single)
  else:
      Accumulate multi = new MultiAccumulate(srcPattern, new Declaration[0], accs, N)
      Pattern wrap     = wrapMultiResultPattern(accumulators, multi)

  parent.addChild(wrap)
  for i in 0..N-1:
      Declaration decl = wrap.getDeclarations().get(accumulators[i].resultBindName())
      outerScope.put(accumulators[i].resultBindName(),
                     new BoundVariable(name, resultClassFor(acc_i), wrap, decl))
```

### Component splits

Four helpers, mechanically separated from the existing
`buildSingleAccumulate`:

1. **`buildSingleAccumulator(AccumulatorIR acc, Class<?> srcClass, Map<String,BoundVariable> innerScope) → Accumulator`**

   Pure per-function construction: validate arity against
   `AccumulateFunctionRegistry.Resolution`, build the extractor via
   `DrlxLambdaCompiler.createValueExtractor` (or leave it `null` for
   `acceptsZeroArgs`), instantiate the `AccumulateFunction` via
   reflection, return `new DrlxLambdaAccumulator(fn, extractor)`.

   This is the existing `buildSingleAccumulate` minus the
   `SingleAccumulate` allocation and minus the `required` decl
   array construction. The `required` computation moves to a small
   sibling helper used only by the N=1 dispatch branch (see below).

2. **`requiredFor(AccumulatorIR acc, Map<String,BoundVariable> innerScope) → Declaration[]`**

   Extracted from v1's existing `buildSingleAccumulate`: maps
   `acc.referencedBindings()` through `innerScope` to a
   `Declaration[]` via `bv.declaration()` (see the binding-model
   change below). Used only on the N=1 path to preserve v1
   semantics on `SingleAccumulate.requiredDeclarations`. The N>1
   path passes `new Declaration[0]`, matching the Drools
   executable-model path; outer-binding plumbing for the
   multi-function case is owned by #54.

3. **`wrapSingleResultPattern(AccumulatorIR acc, SingleAccumulate single) → Pattern`**

   Unchanged from v1: typed result `Pattern` named after
   `acc.resultBindName()`, one declaration via
   `SelfReferenceClassFieldReader`.

4. **`wrapMultiResultPattern(List<AccumulatorIR> accs, MultiAccumulate multi) → Pattern`** (new)

   ```
   ReadAccessor selfReader = new SelfReferenceClassFieldReader(Object[].class);
   Pattern p = new Pattern(0, new ClassObjectType(Object[].class));  // unnamed
   for i in 0..N-1:
       Class<?> rType = resultClassFor(accs.get(i));
       p.addDeclaration(new Declaration(
           accs.get(i).resultBindName(),
           new ArrayElementReader(selfReader, i, rType),
           p, true));
   p.setSource(multi);
   return p;
   ```

   Wrapping Pattern has no identifier — mirrors the Drools convention
   (`KiePackagesBuilder` does not name the multi-function wrapping
   pattern; declarations are referenced by their own identifiers).

### Binding-model change (required by multi-decl wrapper)

Today's `BoundVariable` record at `DrlxLambdaCompiler.java:113` is:

```java
public record BoundVariable(String name, Class<?> type, Pattern pattern) {}
```

The downstream lookup pattern is `bv.pattern().getDeclaration()`
(`DrlxRuleAstRuntimeBuilder.java:472` in `buildSingleAccumulate`'s
`required` computation, and `:525` in `buildEvalCondition`). This is
only sound when each Pattern carries exactly one declaration —
`Pattern.getDeclaration()` returns the single declaration installed
by the constructor's identifier argument.

The multi-function wrap Pattern is **unnamed** and carries N
declarations, so `Pattern.getDeclaration()` is `null` and looking up
any of `minAge` / `maxAge` / `avgAge` via the old idiom drops the
declaration silently. Result-binding references in subsequent
`EvalCondition` or future accumulate `required` arrays would then be
broken.

**Fix:** extend the record so each binding carries its specific
`Declaration`.

```java
public record BoundVariable(String name,
                            Class<?> type,
                            Pattern pattern,
                            Declaration declaration) {}
```

**Callsite updates** (`bv.pattern().getDeclaration()` → `bv.declaration()`):

- `DrlxRuleAstRuntimeBuilder.java:472` (the `requiredFor` helper introduced above).
- `DrlxRuleAstRuntimeBuilder.java:525` (`buildEvalCondition`).

**Construction-site updates** (audit every `new BoundVariable(...)` for the new field):

- The existing pattern-builder paths that create regular
  `BoundVariable(name, type, pattern)` — pass
  `pattern.getDeclaration()` as the fourth arg (the single,
  primary declaration already on the Pattern).
- The accumulate result paths (both single and multi): pass the
  `Declaration` retrieved via `pattern.getDeclarations().get(name)`,
  not `pattern.getDeclaration()`. `wrapResultPattern` constructs
  the Pattern with an `identifier` (which seeds `this.declaration`
  with a `PatternExtractor`-backed declaration), then
  `addDeclaration(...)` overwrites the entry in the
  `declarations` map with a `SelfReferenceClassFieldReader`-backed
  declaration but does NOT update `this.declaration`. The
  runtime-resolvable accessor is the SelfReference one in the
  map. `wrapMultiResultPattern` skips the identifier and only
  uses `addDeclaration`, so its declarations live only in the
  map.

The N=1 case behaves identically to v1 after this change because
`pattern.getDeclaration()` for a single-decl Pattern is the same
declaration that lookups returned before.

### Consequence type collection (`collectPatternTypes`)

`DrlxLambdaCompiler.collectPatternTypes` at
`DrlxLambdaCompiler.java:468` currently reads only
`p.getDeclaration()` and types it from `p.getObjectType()`:

```java
Class<?> patternClass = ((ClassObjectType) p.getObjectType()).getClassType();
Declaration declaration = p.getDeclaration();
if (declaration != null) {
    types.put(declaration.getIdentifier(), Type.type(patternClass));
}
```

For the multi-function wrap Pattern this would skip every result
binding (declaration is null) and, even if not skipped, would type
each entry as `Object[].class`. The `multiFunctionMinMaxAvg`
consequence (`results.add(minAge); results.add(maxAge); results.add(avgAge);`)
would fail to type-check inside MVEL3.

**Fix:** iterate every declaration on the Pattern and use each
declaration's extractor type rather than the Pattern's object type.

```java
for (Declaration d : p.getDeclarations().values()) {
    types.put(d.getIdentifier(),
              Type.type(d.getExtractor().getExtractToClass()));
}
```

Equivalent to the old code for every existing pattern (a
single-decl pattern's lone declaration carries an extractor whose
`getExtractToClass()` matches the Pattern's object type — verified
for both `SelfReferenceClassFieldReader` and the standard property
extractors). New for the multi-wrap Pattern: each
`ArrayElementReader(selfReader, i, rType)` reports `rType` as
`getExtractToClass()`, so `minAge → Integer`, `maxAge → Integer`,
`avgAge → Double` populate correctly.

### Outer-scope additions

For both N=1 and N>1, every result binding still flows into the
caller's `outerScope` keyed by `acc.resultBindName()`. The
`BoundVariable.pattern` field points to the single wrapping `Pattern`
in either case; the `BoundVariable.declaration` field carries the
specific declaration for that binding. Downstream consumers (the
`do` block, subsequent patterns referencing accumulate results)
resolve by binding name and read the per-binding declaration —
shape-agnostic.

## Data flow

### Build time (`var minAge = min(p.age), var maxAge = max(p.age), var avgAge = avg(p.age)`)

```
buildAccumulatePattern(accPat with N=3)
  srcPattern = Pattern(Person.class, "p")
  accs = [
    new DrlxLambdaAccumulator(MinFn,  extractor("p.age", p, Person)),
    new DrlxLambdaAccumulator(MaxFn,  extractor("p.age", p, Person)),
    new DrlxLambdaAccumulator(AvgFn,  extractor("p.age", p, Person)),
  ]
  multi = new MultiAccumulate(srcPattern, Declaration[0], accs, 3)
  wrap  = Pattern(Object[].class)
          decls: [minAge → ArrayElementReader(self, 0, Comparable.class),
                  maxAge → ArrayElementReader(self, 1, Comparable.class),
                  avgAge → ArrayElementReader(self, 2, Double.class)]
          source = multi
  parent.addChild(wrap)
  outerScope += {minAge → wrap, maxAge → wrap, avgAge → wrap}
```

### Runtime

Driven entirely by Drools' existing `MultiAccumulate.accumulate` /
`MultiAccumulate.getResult`. No new runtime classes:

```
For each Person fact entering the source:
  MultiAccumulate.accumulate(...)
    for i in 0..2:
      accs[i].accumulate(...)         → DrlxLambdaAccumulator.accumulate
                                        → extractor.apply(person) = person.age
                                        → fnCtx[i].accumulate(value)

At trigger time:
  Object[] results = MultiAccumulate.getResult(...)
  Each downstream Declaration on the wrap Pattern reads results[i] via
  ArrayElementReader.
```

`count()` interaction: `extractor == null` for count is preserved.
`DrlxLambdaAccumulator.accumulate` already null-checks:

```java
Object value = (extractor == null) ? handle.getObject() : extractor.apply(handle.getObject());
```

Mixed null/non-null extractor slots in the same `MultiAccumulate` work
without any new special case.

## Error handling

No new error paths.

| Failure point | Handling |
|---|---|
| Per-function arity / zero-arg / unknown-function validation | Existing checks inside `buildSingleAccumulator` — fire per slot, identical to v1. |
| Per-function extractor compile failure | Existing `DrlxLambdaCompiler.createValueExtractor` flow (per-slot MVEL3 build-time exception). |
| Runtime extractor failure | Existing `DrlxValueExtractor.apply` `RuntimeException` wrap, unchanged. |
| Pre-built metadata mismatch (.class drift) | Existing `handleMetadataMismatch` flow per slot. |

## Testing

All in `AccumulateTest.java` (no separate test class).

### Existing tests (must stay green)

- `singleAvgOverPersons`, `sumOverIntegerField`, `countWithNoArgument`,
  `sumOfPropertyAccess`, `avgOfPropertyAccess` — N=1, route through
  `SingleAccumulate` exactly as today.
- `multiFunctionMinMaxAvg` — assertion unchanged
  (`containsExactly(20, 60, 40.0)`); now satisfied by `MultiAccumulate`.
- All #48 extractor tests (`sumOfArithmeticExpression`,
  `sumOfMethodCallExpression`, `sumOfMultipleBindingRefs`,
  `avgOfExpression`, `extractorExpressionWithUnknownPropertyFailsAtBuild`)
  — N=1, unchanged.
- Negative tests (`qualifiedFunctionNameRejected`,
  `unknownFunctionRejected`, `sourceBindingNotVisibleAfterAccumulate`) —
  unchanged.

### New behavioural tests

| Test | Rule fragment | Coverage |
|---|---|---|
| `multiFunctionCountAndSum` | `long n = count(), var total = sum(p.age)` | mixed null-extractor + non-null inside one `MultiAccumulate` |
| `multiFunctionWithExpressionExtractors` | `var s1 = sum(p.age + 1), var s2 = sum(p.age * 2)` | every slot uses an MVEL3-compiled extractor (#48 path) |

`multiFunctionMinMaxAvg`'s existing consequence
(`results.add(minAge); results.add(maxAge); results.add(avgAge);`)
already exercises consequence-side type resolution for every
multi-accumulate result binding — the type-collection change
described above is covered by keeping that test green. No new test
class needed for that path; if it regresses, this test fails.

Both inserted into the existing `MyUnit` rule unit; assertions on
`results` list contents using the existing `withSession` helper.

### New structural tests (Drools-convention parity)

Both folded into `AccumulateTest.java`. Walk the LHS produced by
`DrlxRuleBuilder` and assert node shape directly — these are the only
guards against silently still emitting N×SingleAccumulate after the
refactor.

| Test | Rule fragment | Asserts |
|---|---|---|
| `singleFunctionEmitsSingleAccumulate` | `var minAge = min(p.age)` | exactly one accumulate result Pattern under the rule LHS; its `getSource()` is `SingleAccumulate`; pattern object type = `Comparable.class` (`AccumulateFunctionRegistry` maps `min`/`max` to `Comparable`); exactly one declaration named `minAge`. |
| `multiFunctionEmitsOneMultiAccumulate` | `var minAge = min(p.age), var maxAge = max(p.age), var avgAge = avg(p.age)` | exactly **one** accumulate result Pattern; its `getSource()` is `MultiAccumulate`; `getAccumulators().length == 3`; pattern object type = `Object[].class`; declarations at indices 0/1/2 named `minAge`/`maxAge`/`avgAge` with `ArrayElementReader` accessors. |

How to walk: `kieBase.getKiePackage("…").getRules()` → `Rule.getLhs()`
→ first child `Pattern` → `pattern.getSource()` is the `Accumulate`
instance. (`AccumulateTest` already builds via `DrlxRuleBuilder`; a
non-session-based build helper may need to be added if the existing
`withSession` flow doesn't expose the `Rule` directly — accept that
small addition or use reflection on `withSession`'s session if simpler.)

### Pre-build path

If `DrlxPreBuildLambdaCompilerTest` covers accumulate, add one
round-trip case for `min(p.age), max(p.age)` so the pre-built
metadata path is exercised for multi-function. Otherwise rely on the
existing 213-suite pre-build coverage; do not create a fresh test
class.

## Plan deviations (anticipated)

None expected. The change is a localised refactor of one method
(`buildAccumulatePattern`) plus one new helper (`wrapMultiResultPattern`),
guided directly by Drools' existing `KiePackagesBuilder.createAccumulate`
shape.

One latent question, to be confirmed at plan-writing time only if it
matters: whether the existing `withSession` test harness gives the
structural tests direct access to the parsed `Rule` / `Pattern` tree
without spinning up a `KieSession`. If not, expose a small
non-session build helper from the test support class — cheaper than
either reflection or building a session just to inspect AST shape.

## Out of scope (filed as separate issues under #26)

- #50 — Inline-from form
- #51 — `acc()` keyword forms
- #52 — Multi-pattern source via `and(...)`
- #53 — Custom user-imported accumulate functions
- #54 — Outer-binding refs in extractor expressions (this lands the
  `Declaration[]` plumbing on top of `MultiAccumulate.requiredDeclarations`)

## References

| Topic | Path |
|---|---|
| Issue | https://github.com/tkobayas/drlx-parser/issues/49 |
| Parent epic | https://github.com/tkobayas/drlx-parser/issues/26 |
| v1 spec | `specs/2026-05-13-drlx-accumulate-design.md` |
| #48 spec | `specs/2026-05-15-48-accumulate-extractor-mvel3-design.md` |
| Call site to refactor | `drlx-parser-core/.../DrlxRuleAstRuntimeBuilder.java:397` (`buildAccumulatePattern`), `:428` (`buildSingleAccumulate`), `:478` (`wrapResultPattern`) |
| Drools reference — model path | `drools-model-compiler/.../KiePackagesBuilder.java:878` (`createAccumulate`) |
| Drools reference — DRL path | `drools-mvel/.../MVELAccumulateBuilder.java:144` (`isMultiFunction()` branch) |
| Drools `MultiAccumulate` | `drools-base/src/main/java/org/drools/base/rule/MultiAccumulate.java` |
| Drools `ArrayElementReader` | `drools-base/src/main/java/org/drools/base/base/extractors/ArrayElementReader.java` |
