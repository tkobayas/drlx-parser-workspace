# Design — `DrlxRuleUnitInstance` runtime wrapper for KieBase + RuleUnitData

**Issue:** #37 (precondition for testing) — DataStore CRUD operations on rule units
**Epic:** #26
**Date:** 2026-05-11
**Scope:** New runtime API in `drlx-parser-core/src/main`. Minor edits to test fixtures (`MyUnit`, `MyUnitWithRawField`) and a new test utility. No grammar, no IR, no proto changes.

## Motivation

Issue #37 introduces DRLX rules whose consequence calls `alerts.add(...)`, `alerts.remove(...)`, `alerts.update(...)` on a `DataStore` reference. To execute and verify such rules we need:

1. A `DataStore` instance available to the consequence — i.e. set as a session global of the same name as the unit field.
2. The same DataStore wired so its `add` propagates into the working memory (its named entry point).

The standard upstream entry point `RuleUnitProvider.get().createRuleUnitInstance(myUnit)` cannot be reused. It assumes a generated `RuleUnit<T>` class registered via `ServiceLoader`, produced by drools-ruleunits codegen scanning `.drl` files on the classpath. DRLX compiles rules from a string at runtime — no codegen, no service entry — so that path will not find the unit and will fail.

A direct refactor of drools-ruleunits-impl to accept a pre-built `KieBase` was rejected: it would entangle DRLX with drools core release cadence for what is, in practice, a six-line bind step. We instead vendor that bind step in DRLX as a public API.

## Scope

**In scope**
- Add `DrlxRuleUnitInstance<T extends RuleUnitData>` to `drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/`. Implements `org.drools.ruleunits.api.RuleUnitInstance<T>`.
- Promote test fixture `MyUnit` to `implements RuleUnitData` with each `DataStore<X>` field initialised via `DataSource.createStore()`.
- Add `TestDataObserver<T>` test utility in `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/` for inspecting consequence-side DataStore mutations.
- Migrate **no existing tests**. Wrapper is opt-in.

**Out of scope**
- No changes to drools-ruleunits-api, drools-ruleunits-impl, drools-core.
- No `RuleUnitProvider` integration, no `ServiceLoader` registration, no `@KieService` annotation on the wrapper.
- No changes to the DRLX compiler, grammar, or IR. The wrapper only operates on the `KieBase` the compiler already produces.
- No conversion of the existing 146 KieSession-based tests. Both patterns coexist.
- No `SessionUnit` integration. Single-instance, single-fire semantics only.

## Architecture

```
                  ┌──────────────────────────────────────────────┐
                  │  Test code                                   │
                  │                                              │
                  │  MyUnit unit = new MyUnit();                 │
                  │  unit.requests.add(new Request(...));        │
                  │                                              │
                  │  KieBase kb = drlxCompiler.build(src);       │
                  │  try (var i =                                │
                  │     DrlxRuleUnitInstance.create(kb, unit)) { │
                  │       var obs = TestDataObserver             │
                  │           .subscribeTo(unit.alerts);         │
                  │       i.fire();                              │
                  │       assertThat(obs.inserted()).hasSize(1); │
                  │  }                                           │
                  └────────────────────┬─────────────────────────┘
                                       │
                                       ▼
   ┌───────────────────────────────────────────────────────────────┐
   │ DrlxRuleUnitInstance<T extends RuleUnitData>                  │
   │   implements org.drools.ruleunits.api.RuleUnitInstance<T>     │
   │                                                               │
   │   create(KieBase, T) {                                        │
   │     reteEvaluator = new RuleUnitExecutorImpl(                 │
   │                       (InternalRuleBase) kieBase);            │
   │     for each public DataSource<?> field f on T:               │
   │       reteEvaluator.setGlobal(f.name, f.value);               │
   │       f.value.subscribe(new EntryPointDataProcessor(          │
   │           reteEvaluator.getEntryPoint(f.name)));              │
   │   }                                                           │
   │                                                               │
   │   fire(), fire(AgendaFilter), executeQuery(...), getClock(),  │
   │   close() → reteEvaluator.dispose()                           │
   └───────────────────────────────────────────────────────────────┘
```

### Bind protocol

Mirrors `org.drools.ruleunits.impl.AbstractRuleUnitInstance.bind` but walks **public declared fields** instead of `PropertyDescriptor`s, because DRLX unit classes use public fields without getters.

```java
for (Field f : ruleUnitData.getClass().getDeclaredFields()) {
    if (!Modifier.isPublic(f.getModifiers()) || Modifier.isStatic(f.getModifiers())) continue;
    Object v = f.get(ruleUnitData);
    if (v == null) continue;
    String name = f.getName();
    if (v instanceof DataSource<?> ds) {
        ds.subscribe(new EntryPointDataProcessor(reteEvaluator.getEntryPoint(name)));
    }
    try {
        reteEvaluator.setGlobal(name, v);
    } catch (RuntimeException ignored) {
        // global not declared in this rule unit — OK, skip
    }
}
```

### Why a `try`/`catch` on `setGlobal`

The unit class may declare DataStore fields that no rule in the current KieBase references. The upstream bind protocol swallows the resulting "global not declared" exception unconditionally; we do the same. The alternative — pre-inspecting the KieBase's declared globals — adds a KieBase walk per construction and changes nothing observable.

## Components

### 1. `DrlxRuleUnitInstance<T>` (new, ~80 LOC)

`drlx-parser-core/src/main/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstance.java`

```java
public final class DrlxRuleUnitInstance<T extends RuleUnitData>
        implements RuleUnitInstance<T> {

    private final T unitData;
    private final ReteEvaluator reteEvaluator;

    public static <T extends RuleUnitData> DrlxRuleUnitInstance<T> create(
            KieBase kieBase, T unitData) {
        return new DrlxRuleUnitInstance<>(kieBase, unitData);
    }

    private DrlxRuleUnitInstance(KieBase kieBase, T unitData) {
        this.unitData = unitData;
        this.reteEvaluator = new RuleUnitExecutorImpl((InternalRuleBase) kieBase);
        bind();
    }

    private void bind() { /* see "Bind protocol" above */ }

    @Override public int fire() { return reteEvaluator.fireAllRules(); }
    @Override public int fire(AgendaFilter filter) { return reteEvaluator.fireAllRules(filter); }
    @Override public QueryResults executeQuery(String name, Object... args) {
        fire();
        return reteEvaluator.getQueryResults(name, args);
    }
    @Override public <C extends SessionClock> C getClock() { return (C) reteEvaluator.getSessionClock(); }
    @Override public RuleUnit<T> unit() { return null; } // no RuleUnit instance — wrapper bypasses codegen
    @Override public T ruleUnitData() { return unitData; }
    @Override public void close() { reteEvaluator.dispose(); }
}
```

**`unit()` returns null** because there is no upstream `RuleUnit<T>` codegen for DRLX. Document this in the Javadoc; the field is provided purely to satisfy the `RuleUnitInstance` contract. Tests that need a RuleUnit reference can be rejected at this layer.

**No `RuleConfig` overload yet.** Event listeners can be attached directly to the underlying KieBase or via the `RuleUnitInstance` API in a future revision. YAGNI.

### 2. Test-fixture promotion (`MyUnit`)

`drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`

```java
public class MyUnit implements RuleUnitData {
    public DataStore<Person> persons = DataSource.createStore();
    public DataStore<Address> addresses = DataSource.createStore();
    public DataStore<Person> seniors = DataSource.createStore();
    // ... (all 14 existing fields, plus alerts/tasks/requests added per #37 work)
}
```

Existing 146 tests are unaffected: none currently call `new MyUnit()`, they only reference `MyUnit` via the `unit MyUnit;` declaration inside DRLX source strings, which is a parse-time symbol lookup that doesn't care whether the class implements `RuleUnitData`.

The benchmark-module copy at `drlx-parser-benchmark/src/main/java/org/drools/drlx/ruleunit/MyUnit.java` is **not** touched in this spec — benchmarks don't exercise #37 rules.

`MyUnitWithRawField` is **not** promoted: it is a parser-test fixture (single `public DataStore persons;` raw field), never instantiated at runtime in any test, and initialising it with `DataSource.createStore()` would inject an unchecked-assignment warning for no observable benefit.

### 3. `TestDataObserver<T>` (new test utility, ~30 LOC)

`drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/TestDataObserver.java`

```java
public final class TestDataObserver<T> implements DataProcessor<T> {

    private final List<T> inserted = new ArrayList<>();
    private final List<T> updated  = new ArrayList<>();
    private final List<DataHandle> removed = new ArrayList<>();

    public static <T> TestDataObserver<T> subscribeTo(DataSource<T> ds) {
        TestDataObserver<T> obs = new TestDataObserver<>();
        ds.subscribe(obs);
        return obs;
    }

    @Override public FactHandle insert(DataHandle handle, T object) {
        inserted.add(object);
        return null;
    }
    @Override public void update(DataHandle handle, T object) { updated.add(object); }
    @Override public void delete(DataHandle handle) { removed.add(handle); }

    public List<T> inserted() { return List.copyOf(inserted); }
    public List<T> updated()  { return List.copyOf(updated); }
    public List<DataHandle> removed() { return List.copyOf(removed); }
}
```

**Two subscribers on one DataStore.** When a test calls `TestDataObserver.subscribeTo(unit.alerts)`, the bind step has *also* subscribed an `EntryPointDataProcessor` to the same DataStore. The DataStore fans out to both subscribers — observer captures the event, EntryPointDataProcessor pushes the fact into the named entry point so subsequent rules can match on it. Verified by upstream tests using the same pattern (`drools-ruleunits-impl` test `DataStoreTest` subscribes a custom processor alongside the auto-bound one).

## Coupling cost (acknowledged, not mitigated)

`DrlxRuleUnitInstance` imports two `impl` classes from drools-ruleunits-impl:

- `org.drools.ruleunits.impl.EntryPointDataProcessor`
- `org.drools.ruleunits.impl.sessions.RuleUnitExecutorImpl`

Plus `org.drools.core.impl.InternalRuleBase` from drools-core (the runtime type of `KieBase`).

These are not API packages. If a future Drools release repackages or hides them, DRLX will not compile. The same coupling would exist whether the wrapper is in src/main or src/test — moving it to src/main only makes the coupling part of DRLX's public contract.

**Mitigation:** none for this iteration. Pinned to the same Drools version DRLX already depends on. If breakage occurs, the fix is local to one file.

## Testing strategy

A single happy-path test verifies the bind protocol end-to-end. Lives in `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DrlxRuleUnitInstanceTest.java`.

```java
@Test
void dataStoreAddInConsequenceFiresAndCaptures() {
    String src = """
        package test;
        import org.drools.drlx.domain.Person;
        import org.drools.drlx.domain.Alert;
        import org.drools.drlx.ruleunit.MyUnit;
        unit MyUnit;

        rule R {
            var p : /persons[ age > 30 ],
            do alerts.add(new Alert(p.getName()))
        }
        """;
    KieBase kb = new DrlxCompiler().build(src);

    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 40));

    try (var instance = DrlxRuleUnitInstance.create(kb, unit)) {
        var alertObs = TestDataObserver.subscribeTo(unit.alerts);
        assertThat(instance.fire()).isEqualTo(1);
        assertThat(alertObs.inserted()).hasSize(1);
        assertThat(alertObs.inserted().get(0).getName()).isEqualTo("Alice");
    }
}
```

This test exists **only** to validate the wrapper. The bulk of #37's verification logic lives in dedicated `DataStoreCrudTest` (or similar) tests written under the #37 work item — that work is out of scope here.

**Domain prerequisites for this validation test:** an `Alert` class will be added under `org.drools.drlx.domain` as part of #37 implementation. If this spec is implemented before #37 starts, use an existing domain class (e.g. `Person`) for the wrapper validation test and switch when `Alert` lands.

## Migration / rollout

- **Existing tests:** no changes. They continue using `kieBase.newKieSession().getEntryPoint(name).insert(obj)`.
- **New tests for #37 and any future rule-unit feature:** use `DrlxRuleUnitInstance` + `TestDataObserver`.
- **No deprecation flag.** Both patterns are valid: the wrapper is required only when consequence code references DataStore methods (`add`/`remove`/`update`).

## Open questions

None blocking. Possible follow-ups (not part of this spec):

- Add a `RuleConfig` overload once a test needs agenda/runtime event listeners through this surface.
- Consider whether `MyUnit` in the benchmark module should be promoted symmetrically once benchmarks exercise #37 rules.
- If multiple unit classes (`MyUnit` vs `MyUnitWithRawField` vs future siblings) start diverging in DataStore membership, document the convention.
