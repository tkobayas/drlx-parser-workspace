# Design — Unit-class fields as DRLX globals (issue #37, part 1)

**Issue:** #37 (DataStore CRUD) — first iteration, minimum viable
**Epic:** #26
**Date:** 2026-05-11
**Scope:** Parser/builder change in `drlx-parser-core/src/main`. No runtime API change, no grammar change, no proto change. Re-enables a pinned probe test and adds happy-path tests for `add` / `remove(T)`.

## Motivation

The probe test `DataStoreAddProbeTest.consequenceCanCallDataStoreAdd` fails at parse time with `UnsolvedSymbol Unsolved symbol in persons1`. Cause: DRLX's `DrlxRuleAstRuntimeBuilder` walks unit-class fields only for the *entry-point* type map. It never registers them as **globals** on the package, so neither the MVEL3 consequence batch-compiler nor the runtime KieBase knows that names like `alerts` or `persons1` are bound.

Upstream rule-units codegen (`org.drools.model.codegen.execmodel.PackageModel.addRuleUnitVariable`, line 381) treats every unit-class field as a global, and DataSource-typed fields additionally as entry points. DRLX implements only the second half.

This iteration mirrors the upstream "every public field becomes a global" half-step. That alone is enough to make `do { alerts.add(...); }` and `do { alerts.remove(t); }` work end-to-end, since `DataStore` exposes `add(T)` and `remove(T)` directly. `update(T)` (no direct API) and the `with`-block compact-update syntax remain explicitly out of scope.

## Scope

**In scope**
- Add `DrlxRuleAstRuntimeBuilder.buildGlobalTypeMap(unitClass)` — symmetric to the existing `buildEntryPointTypeMap`. Walks public, non-static fields; collects `(field.getName(), field.getGenericType())`.
- In `DrlxRuleAstRuntimeBuilder.build`, register each entry as a global on the `KnowledgePackageImpl` via `pkg.addGlobal(name, type)` (`Type` is `java.lang.reflect.Type` — passed straight through).
- In `DrlxRuleAstRuntimeBuilder.buildRule`, merge unit-field types into the MVEL3 consequence type map (`Map<String, org.mvel3.Type<?>>`) *after* the LHS bindings, using `Type.type(rawClass)` (no generics — matches the existing pattern-type pattern in `DrlxLambdaCompiler.collectPatternTypes`, line 432).
- Re-enable `DataStoreAddProbeTest.consequenceCanCallDataStoreAdd` by removing `@Disabled`.
- Add new tests in the same file: `removeByObjectViaDataStore` and `consequenceCanReferenceMultipleUnitFields`.

**Out of scope**
- `update(T)` translation / handle lookup. The DataStore API has no single-arg `update`, only `update(DataHandle, T)`. Belongs in a follow-up sub-issue split from #37.
- `with`-block compact update syntax (`alerts.update(t{prop = val})`). Mvel3 + DRLX grammar work — explicitly experimental per DRLXXXX.md. Separate sub-issue.
- Non-DataSource global functionality beyond "the call site compiles". A `public Logger log` field becoming a usable global is not a test goal.
- Generic-aware MVEL3 type resolution. `alerts` is exposed as raw `Type.type(DataStore.class)`; calls to `add(Person)` will resolve via erasure to `add(Object)`. Matches how DRLX already handles pattern bindings (raw class only).
- Any change to `DataStore` API, drools-ruleunits, or drools-core.

## Architecture

```
                   DRLX source
                        │
                        ▼
            DrlxRuleAstParser → CompilationUnitIR
                        │
                        ▼
       DrlxRuleAstRuntimeBuilder.build(parseResult)
                        │
                        ▼
               resolveUnitClass()
                        │
             ┌──────────┼──────────────────────────────┐
             ▼          ▼                              ▼
   buildEntryPointTypeMap   buildGlobalTypeMap   (existing rule walk)
   (DataSource fields →     (all public, non-     ...
    element types)           static fields →
                             field name + Type)
                        │
                        ▼
         pkg.addGlobal(name, fieldType) for each
                        │
                        ▼
            for each rule: buildRule
                        │
                        ▼
        types = lambdaCompiler.getTypeMap(root)
        types.putAll(globalsAsMvelTypes)   ← NEW merge
                        │
                        ▼
   lambdaCompiler.createLambdaConsequence(rhs, types)
                        │
                        ▼
     MVEL3 resolves `alerts.add(person)` against
     the merged type map
```

### Bind protocol — symmetric to entry-point walk

```java
private static Map<String, java.lang.reflect.Type> buildGlobalTypeMap(Class<?> unitClass) {
    Map<String, java.lang.reflect.Type> map = new LinkedHashMap<>();
    for (Field field : unitClass.getDeclaredFields()) {
        int mods = field.getModifiers();
        if (!Modifier.isPublic(mods) || Modifier.isStatic(mods)) continue;
        map.put(field.getName(), field.getGenericType());
    }
    return map;
}
```

DataSource-specific generic-arg resolution stays in `buildEntryPointTypeMap`. The two walks are intentionally separate: the entry-point map needs the element type (`Person` from `DataStore<Person>`); the globals map only needs the field's own type (`DataStore<Person>` as-is).

### MVEL3 type-map merge

In `buildRule`, immediately before `lambdaCompiler.createLambdaConsequence`:

```java
Map<String, Type<?>> types = lambdaCompiler.getTypeMap(root);
for (Map.Entry<String, java.lang.reflect.Type> e : globalTypes.entrySet()) {
    Class<?> raw = erasure(e.getValue());
    if (raw != null) {
        types.put(e.getKey(), Type.type(raw));
    }
}
```

`erasure` extracts the raw `Class<?>` from a `java.lang.reflect.Type` (handles `Class` directly, `ParameterizedType` via `getRawType`, returns null for `TypeVariable` / `WildcardType` which we can't sensibly project into MVEL3 today). Null entries are silently skipped — they would not have produced a runnable consequence anyway.

### Why "after LHS bindings"

The merge `types.putAll(globalsAsMvelTypes)` happens *after* `lambdaCompiler.getTypeMap(root)`. In the unlikely case a pattern binding shares a name with a unit field, the global wins. This is the safer default: a unit-field global is package-wide; a pattern binding is rule-local. If we ever want to flip the precedence (LHS wins for the consequence's lexical scope), the merge order is trivial to reverse and gets its own test.

In practice the names should not collide. If they do, the conflict is a code smell in the DRLX source.

## Components

### 1. New helper `buildGlobalTypeMap` (~10 LOC)

`drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`. Sits next to `buildEntryPointTypeMap`. See "Bind protocol" snippet above.

### 2. Global registration in `build` (~5 LOC)

After `Map<String, java.lang.reflect.Type> globalTypes = buildGlobalTypeMap(unitClass);`:

```java
globalTypes.forEach(pkg::addGlobal);
```

`KnowledgePackageImpl.addGlobal(String, java.lang.reflect.Type)` is the exact signature it expects (line 421 of that file; import `java.lang.reflect.Type` at line 27).

### 3. Type-map merge in `buildRule` (~5 LOC)

`Map<String, org.mvel3.Type<?>>` lives in `DrlxRuleAstRuntimeBuilder.buildRule`. Add the merge described above. Pass `globalTypes` down from `build` as an extra parameter to `buildRule` — keeps `DrlxLambdaCompiler` stateless about globals.

### 4. Probe test re-enabled

`drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/DataStoreAddProbeTest.java`:

- Drop the `@Disabled` annotation.
- Update the class-level Javadoc to remove the "Status: Currently disabled" note.
- Optionally rename the class from `DataStoreAddProbeTest` to `DataStoreCrudTest` to reflect that it's no longer a one-shot probe and will grow with each sub-piece of #37. This is cosmetic; either name is fine.

### 5. New happy-path tests

In the same file:

```java
@Test
void removeByObjectViaDataStore() {
    String rule = """
            package test;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                Person p : /persons[ age > 30 ],
                do { persons.remove(p); }
            }
            """;
    KieBase kieBase = new DrlxRuleBuilder().build(rule);

    MyUnit unit = new MyUnit();
    Person alice = new Person("Alice", 40);
    unit.persons.add(alice);

    try (var instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        var obs = TestDataObserver.subscribeTo(unit.persons);
        assertThat(instance.fire()).isEqualTo(1);
        assertThat(obs.removed()).hasSize(1);
    }
}

@Test
void consequenceCanReferenceMultipleUnitFields() {
    String rule = """
            package test;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                Person p : /persons[ age > 30 ],
                do { persons1.add(p); persons2.add(p); }
            }
            """;
    KieBase kieBase = new DrlxRuleBuilder().build(rule);

    MyUnit unit = new MyUnit();
    unit.persons.add(new Person("Alice", 40));

    try (var instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        var obs1 = TestDataObserver.subscribeTo(unit.persons1);
        var obs2 = TestDataObserver.subscribeTo(unit.persons2);
        assertThat(instance.fire()).isEqualTo(1);
        assertThat(obs1.inserted()).hasSize(1);
        assertThat(obs2.inserted()).hasSize(1);
    }
}
```

### 6. Unit test for `buildGlobalTypeMap`

`drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilderTest.java` (may not exist yet — create if absent). Fixture class with `public DataStore<Person>`, `public int counter`, `public static String IGNORED`, `private DataStore<Address> hidden`. Assertions:

- Map contains `persons` (with `ParameterizedType DataStore<Person>`) and `counter` (with `Class int`).
- Map does NOT contain `IGNORED` (static skipped) or `hidden` (private skipped).

This is the only test that exercises the helper in isolation; the end-to-end probe tests exercise the full path.

## Error handling

- **Pattern binding name collides with unit field** — current spec: global wins (merge order). No diagnostic. Future work if it bites in practice.
- **Field type unprojectable to MVEL3** (TypeVariable / WildcardType) — silently skipped in the consequence type map but still registered as a KieBase global with its full reflect Type. MVEL3 will then fail to resolve uses of that symbol with the existing `UnsolvedSymbol` error — informative enough.
- **MVEL3 still fails on something we expected to work** — bubbles up unchanged. No new wrapping.

## What's deliberately not in this round

(Repeated here so a future reader of just the spec body sees the line.)

- `update(T)` and the `update(DataHandle, T)` coercion.
- `with`-block compact update syntax.
- Non-DataSource global functional verification.
- Generic-aware MVEL3 type resolution.

## Testing strategy

- Unit test on `buildGlobalTypeMap` (fixture-class introspection — fast, no rule build).
- Three end-to-end tests in `DataStoreCrudTest` (or `DataStoreAddProbeTest` if rename is deferred): add, remove(T), multi-field reference.
- Full module test suite must remain green: `mvn -pl drlx-parser-core -am test`.

## Open questions / follow-ups

None blocking. After this lands:

- File a sub-issue under #26 for `update(T)` with handle-lookup coercion.
- File a sub-issue under #26 for `with`-block compact update.
- Consider whether to add tests for non-DataSource globals once a real use case appears.
