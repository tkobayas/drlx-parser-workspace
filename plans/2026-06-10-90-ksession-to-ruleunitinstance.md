# Plan: Convert KieSession tests to DrlxRuleUnitInstance (#90)

## Context

The test suite mixes two session patterns: KieSession-based (via `withSession` helper and direct `kieBase.newKieSession()`) and DrlxRuleUnitInstance-based. Issue #90 proposes aligning all runtime tests to DrlxRuleUnitInstance for consistency.

**Scope:** ~188 test methods across ~22 files to convert. ~68 tests already use DrlxRuleUnitInstance (no change). ~50 tests are parse-only/metadata-only (no session, no change). 1 test (EvalIRBuilderTest) left as-is — tests low-level IR pipeline.

**Cannot convert:** PropertyReactiveWatchListTest (7 runtime tests) — uses `ep.update(fh, obj, "propertyName")` for property-name-aware updates. DataStore.update(DataHandle, T) has no property-name variant. Changing these tests would alter what they verify. Leave as KieSession.

## Step 1: Add `addEventListener` to DrlxRuleUnitInstance

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java`

Add one method:

```java
public void addEventListener(AgendaEventListener listener) {
    reteEvaluator.getAgendaEventSupport().addEventListener(listener);
}
```

The underlying `reteEvaluator.getAgendaEventSupport()` already supports this — it's a one-line delegation. Needed by nearly all converted tests for `TrackingAgendaEventListener`.

**Verify:** existing tests pass.

## Step 2: Fix CreditUnit and update DrlxBuilderTestSupport

**CreditUnit** (`ruleunit/CreditUnit.java`): Add `implements RuleUnitData`, initialize DataStore fields with `DataSource.createStore()`.

**DrlxBuilderTestSupport** (`builder/syntax/DrlxBuilderTestSupport.java`): Add `withInstance` helper:

```java
@FunctionalInterface
interface TriConsumer<A, B, C> { void accept(A a, B b, C c); }

protected static void withInstance(String rule,
        TriConsumer<DrlxRuleUnitInstance<MyUnit>, MyUnit, TrackingAgendaEventListener> test) {
    KieBase kieBase = new DrlxRuleBuilder().build(rule);
    MyUnit unit = new MyUnit();
    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
        instance.addEventListener(listener);
        test.accept(instance, unit, listener);
    }
}
```

Also add a `withCreditInstance` variant for IfElseTest (uses CreditUnit with `customers`/`products`).

**Verify:** compiles, existing tests pass.

## Step 3: Convert simple syntax tests (~95 tests, 12 files)

Mechanical conversion pattern:

| Before | After |
|--------|-------|
| `withSession(rule, (kieSession, listener) -> {` | `withInstance(rule, (instance, unit, listener) -> {` |
| `kieSession.getEntryPoint("persons").insert(x)` | `unit.persons.add(x)` |
| `kieSession.fireAllRules()` | `instance.fire()` |
| `listener.getAfterMatchFired()` | unchanged |

**Files:** AndTest(4), ArrayAccessTest(3), BindingInGroupTest(4), ExistsTest(5), InlineCastTest(2), NestedGroupTest(4), NotExistsBindingTest(2), NotTest(7), OrBindingScopeTest(3), OrTest(4), PassivePatternTest(3), TypeInferenceTest(9)

**Verify:** `mvn -f <pom> test` after each batch.

## Step 4: Convert AccumulateTest (43 tests)

Same as Step 3, plus globals:

| Before | After |
|--------|-------|
| `List<Object> observed = new ArrayList<>()` | (removed — use `unit.results`) |
| `kieSession.setGlobal("results", observed)` | (removed — auto-bound by DrlxRuleUnitInstance.bind()) |
| `assertThat(observed)` | `assertThat(unit.results)` |

**2 retraction tests** (`accKeyword3ParamSumWithReverse`, `accKeyword5ParamWithRetraction`): Use DataStore:

```java
DataHandle h1 = unit.persons.add(new Person("A", 20));
// ...fire...
unit.results.clear();
unit.persons.remove(h1);
instance.fire();
```

**Verify:** all 43 AccumulateTest tests pass.

## Step 5: Convert IfElseTest (10 tests) and TestElementTest (4 tests)

**IfElseTest:** Use `withCreditInstance`:

```java
withCreditInstance(rule, (instance, unit, listener) -> {
    unit.customers.add(new Customer("Alice", Rating.LOW));
    unit.products.add(new Product("luxury", Rates.HIGH));
    assertThat(instance.fire()).isEqualTo(1);
});
```

One test (`ifElse_refiresOnOuterScopePropertyUpdate`) uses `update(handle, obj)` — use `DataStore.update(DataHandle, T)`.

**TestElementTest:** 3 simple + 1 with `update(handle, obj)` via `DataStore.update(DataHandle, T)`.

**Verify:** all pass.

## Step 6: Convert remaining syntax tests

- **PositionalTest** (4 withSession tests): simple insert-fire
- **QueryTest** (2 remaining withSession tests): convert to `withInstance`

**Verify:** all pass.

## Step 7: Convert RuleAnnotationsTest session-firing tests

- **1 `withSession` test** (`testSalienceAffectsFiringOrder`): convert to `withInstance`
- **4 timer/duration pseudo-clock tests**: replace `pseudoClockSession(kieBase)` with `DrlxRuleUnitInstance.create(kieBase, unit, sessionConfig)` + `instance.getClock()`. Path already proven by TemporalOperatorTest.
- **8 date-effective/date-expires pseudo-clock tests**: same pattern, `clock.setStartupTime()`.
- **~36 metadata-only tests**: no session, no change.
- **4 already DrlxRuleUnitInstance tests**: no change.

Remove `pseudoClockSession()` private helper when empty.

**Verify:** all 49 RuleAnnotationsTest tests pass.

## Step 8: Convert non-syntax tests

- **DrlxRuleBuilderTest** (7 tests): replace `newKieSession()` with `DrlxRuleUnitInstance.create()`
- **DrlxCompilerTest** (6 tests): same
- **DrlxCompilerNoPersistTest** (5 tests): same (runs with `-Dmvel3.compiler.lambda.persistence=false`)
- **DrlxLambdaBoundaryTest** (1 test, D1): trivially replace null-check

**EvalIRBuilderTest** (1 test): leave as-is — tests programmatic IR pipeline.

**Verify:** all pass.

## Step 9: Remove withSession and clean up

- Delete `withSession` from DrlxBuilderTestSupport
- Remove unused imports across all converted files

**Verify:** full test suite passes.

## Tests left unconverted

| File | Tests | Reason |
|------|-------|--------|
| PropertyReactiveWatchListTest | 7 runtime tests | Requires `ep.update(fh, obj, "propName")` — DataStore lacks property-name-aware update |
| EvalIRBuilderTest | 1 test | Tests low-level IR builder pipeline, not the DrlxRuleBuilder API |

## Commit strategy

5 commits, each independently green:

1. **API + helpers:** DrlxRuleUnitInstance.addEventListener, CreditUnit fix, DrlxBuilderTestSupport withInstance (Steps 1-2)
2. **Simple syntax tests:** 12 files, ~95 tests (Step 3)
3. **AccumulateTest + IfElseTest + TestElementTest + remaining syntax:** ~59 tests (Steps 4-6)
4. **RuleAnnotationsTest:** ~13 tests (Step 7)
5. **Non-syntax tests + cleanup:** ~19 tests + remove withSession (Steps 8-9)

## Verification

After each commit:
```
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test
```

Final: confirm no `withSession` calls remain (except maybe PropertyReactiveWatchListTest), no `kieBase.newKieSession()` in tests (except EvalIRBuilderTest and PropertyReactiveWatchListTest), and total test count unchanged.
