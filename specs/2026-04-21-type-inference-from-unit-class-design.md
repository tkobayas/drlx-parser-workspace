# DRLX Pattern-Type Inference from Unit Class — Design Spec

**Date:** 2026-04-21
**Status:** Approved — ready for implementation plan
**Scope:** Compile-time inference of the Java element type for each DRLX pattern from a required `unit <Name>;` class with `DataSource<T>` / `DataStore<T>` fields. Unblocks #8 (`not /pattern`) and #9 (`exists /pattern`), and enables bare `var p : /persons[...]` at runtime for the first time.

**References:**
- GitHub issues: #14 (this issue); blocked by → unblocks #8; follow-up runtime integration tracked as #15; parent epic #4.
- DRLXXXX.md §"'not' / 'exists'" (line 599) — canonical form is `not /persons[...]` with no explicit Type prefix.
- Drools RuleUnit API: `drools-ruleunits-api/src/main/java/org/drools/ruleunits/api/DataStore.java` (`DataStore<T> extends DataSource<T>`).
- Related spec: `2026-04-21-group-element-and-not-design.md` (#7/#8 — infrastructure + `not`).

## Background

Today the DRLX runtime has no way to map an entry-point name like `persons` to its Java class (`Person`). Every runtime test uses a `Type bind : /entry[...]` prefix to supply the class explicitly. `var p : /persons[...]` is accepted by the grammar but fails end-to-end because the visitor stamps `typeName = "var"` into the IR and `TypeResolver.resolveType("var")` throws `ClassNotFoundException`.

This blocks the canonical DRLX form of `not /persons[...]`, `exists /persons[...]`, and every future bare-OOPath construct. The compiler needs a source of truth for entry-point → Java-class mapping.

Drools' existing RuleUnit model already names that source of truth: a unit class with `DataSource<T>` / `DataStore<T>` fields. Read the class reflectively at build time, build a map, use it for inference.

## Decisions (locked)

| # | Decision |
|---|----------|
| 1 | **Unit class is required.** Every DRLX source must declare `unit <Name>;` AND supply a matching `import <fqn>.<Name>;` (or use an FQN in the declaration itself). Missing → fail loud. Existing tests migrate with a one-line import addition plus a shared `MyUnit.java` in `src/test/java/`. Option (A) from brainstorming. |
| 2 | **Runtime semantics unchanged.** MyUnit is a compile-time source of truth only. `KieSession.getEntryPoint(name)` and `EntryPointId(name)` stay untouched. MyUnit is never instantiated by drlx-parser. Full `RuleUnitInstance` integration tracked separately as #15. Option (a) from brainstorming. |
| 3 | **Explicit type must match inferred (β).** If a rule writes `Person p : /persons[...]` and the unit field is `DataStore<Wrong>`, fail loud. Explicit types stay supported (existing corpus keeps working) and the cross-check catches typos. |
| 4 | **Any `DataSource<T>` subtype** is recognised on the unit class. Reflection uses `DataSource.class.isAssignableFrom(field.getType())`. Covers `DataStore`, `SingletonStore`, and any future/user wrapper. Option (iii) from brainstorming. |
| 5 | **`getDeclaredFields()` only.** Inheritance on the unit class is ignored — parent-class DataSource fields are skipped. Flatten into the concrete unit class if inheritance is needed later. Option (a) from brainstorming. |
| 6 | **`var` as sentinel is in-scope.** When `PatternIR.typeName` equals `"var"` (grammar produces it from `var p :`), treat as "no explicit type" and use inference. Unlocks `var p : /persons[...]` end-to-end in this landing. Option (i) from brainstorming. |

## Architecture overview

```
DRLX source
  └─> ANTLR                        ← unitDeclaration already in grammar
        └─> DrlxToRuleAstVisitor   ← capture unitName into CompilationUnitIR
              └─> CompilationUnitIR(packageName, unitName, imports, rules)
                    ├─> proto round-trip (new unit_name field)
                    └─> DrlxRuleAstRuntimeBuilder
                          ├─> resolveUnitClass(unitName, imports, typeResolver)    ← fail loud if missing
                          ├─> buildEntryPointTypeMap(unitClass)                    ← reflection: DataSource fields → T
                          └─> buildLhs(…, entryPointTypes, …)
                                └─> buildPattern / resolvePatternType:
                                    castType → explicit (cross-check β) → inferred
```

No runtime pipeline change. `KieSession`, `EntryPointId`, `KieSession.getEntryPoint(…)` all untouched.

## IR / data model changes

`DrlxRuleAstModel.CompilationUnitIR` gains `unitName`:

```java
public record CompilationUnitIR(String packageName,
                                String unitName,
                                List<String> imports,
                                List<RuleIR> rules) {
}
```

`unitName` is the simple name from `unit <Name>;` (e.g. `"MyUnit"`). FQN resolution happens in the runtime builder using the `imports` list.

Proto mirror:

```proto
message CompilationUnitParseResult {
    string source_hash = 1;
    string package_name = 2;
    repeated string imports = 3;
    repeated RuleParseResult rules = 4;
    string unit_name = 5;   // NEW
}
```

Fresh field number; no reuse. Old cached `.pb` files without `unit_name` deserialize into `""` (proto3 default) and then fail loud at build time on the "missing unit declaration" check — same intentional cache-invalidation breakpoint as #7.

## Visitor changes (`DrlxToRuleAstVisitor`)

`visitDrlxCompilationUnit` reads the unit name:

```java
String unitName = "";
if (ctx.unitDeclaration() != null) {
    unitName = ctx.unitDeclaration().qualifiedName().getText();
}
return new CompilationUnitIR(packageName, unitName, List.copyOf(imports), List.copyOf(rules));
```

No other visitor changes — pattern construction still produces the same `PatternIR` (with `typeName` populated from `rulePattern`'s `identifier(0)` when present, including `"var"`).

## Runtime-builder changes (`DrlxRuleAstRuntimeBuilder`)

### Unit-class resolution

```java
public List<KiePackage> build(CompilationUnitIR parseResult) {
    KnowledgePackageImpl pkg = new KnowledgePackageImpl(parseResult.packageName());
    pkg.setClassLoader(Thread.currentThread().getContextClassLoader());
    parseResult.imports().forEach(imp -> pkg.addImport(new ImportDeclaration(imp)));

    Class<?> unitClass = resolveUnitClass(parseResult.unitName(),
                                         parseResult.imports(),
                                         pkg.getTypeResolver());
    Map<String, Class<?>> entryPointTypes = buildEntryPointTypeMap(unitClass);

    parseResult.rules().forEach(rule ->
            pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass)));

    return List.of(pkg);
}

private static Class<?> resolveUnitClass(String unitName,
                                         List<String> imports,
                                         TypeResolver typeResolver) {
    if (unitName == null || unitName.isEmpty()) {
        throw new RuntimeException(
                "DRLX compilation unit is missing a 'unit <Name>;' declaration");
    }
    if (unitName.contains(".")) {
        return resolveOrThrow(unitName, typeResolver);
    }
    for (String imp : imports) {
        String simple = imp.substring(imp.lastIndexOf('.') + 1);
        if (simple.equals(unitName)) {
            return resolveOrThrow(imp, typeResolver);
        }
    }
    throw new RuntimeException(
            "Unit class '" + unitName + "' not found — add `import <fqn>." + unitName
            + ";` to the DRLX source.");
}

private static Class<?> resolveOrThrow(String fqn, TypeResolver typeResolver) {
    try {
        return typeResolver.resolveType(fqn);
    } catch (ClassNotFoundException e) {
        throw new RuntimeException("Unit class '" + fqn + "' not on classpath", e);
    }
}
```

### Field scanning

```java
private static Map<String, Class<?>> buildEntryPointTypeMap(Class<?> unitClass) {
    Map<String, Class<?>> map = new LinkedHashMap<>();
    for (Field field : unitClass.getDeclaredFields()) {
        if (!DataSource.class.isAssignableFrom(field.getType())) {
            continue;
        }
        Class<?> element = resolveDataSourceTypeArg(field.getGenericType());
        if (element == null) {
            throw new RuntimeException(
                    "Field '" + unitClass.getName() + "." + field.getName()
                    + "' is a raw DataSource — must be parameterised (e.g. DataStore<Person>)");
        }
        map.put(field.getName(), element);
    }
    return map;
}

private static Class<?> resolveDataSourceTypeArg(Type type) {
    if (!(type instanceof ParameterizedType pt)) {
        return null;
    }
    if (!(pt.getRawType() instanceof Class<?> rawClass)) {
        return null;
    }
    if (!DataSource.class.isAssignableFrom(rawClass)) {
        return null;
    }
    // Walk the generic hierarchy until we find DataSource<T>, then return T.
    if (DataSource.class.equals(rawClass)) {
        Type[] args = pt.getActualTypeArguments();
        return (args.length == 1 && args[0] instanceof Class<?> c) ? c : null;
    }
    return resolveAgainstSuperInterface(pt);
}

// resolveAgainstSuperInterface: standard TypeToken-style resolution —
// substitute pt's actual type arguments into rawClass's generic-interface declarations,
// find DataSource<T>, return the concrete T. ~20 lines.
```

### Pattern type resolution

```java
private static Class<?> resolvePatternType(PatternIR p,
                                           TypeResolver typeResolver,
                                           Map<String, Class<?>> entryPointTypes,
                                           Class<?> unitClass) {
    // Inline cast beats everything else.
    if (p.castTypeName() != null) {
        return resolveOrThrow(p.castTypeName(), typeResolver);
    }

    Class<?> inferred = entryPointTypes.get(p.entryPoint());
    boolean isBare = p.typeName() == null || p.typeName().isBlank() || "var".equals(p.typeName());

    if (isBare) {
        if (inferred == null) {
            throw new RuntimeException(
                    "no type could be inferred for entry point '" + p.entryPoint()
                    + "' — declare `DataSource<T> " + p.entryPoint() + ";` on unit class '"
                    + unitClass.getName() + "'");
        }
        return inferred;
    }

    // Explicit type given (non-"var").
    if (inferred == null) {
        throw new RuntimeException(
                "entry point '" + p.entryPoint() + "' is not declared on unit class '"
                + unitClass.getName() + "' — add `DataSource<T> " + p.entryPoint() + ";`");
    }

    Class<?> explicit = resolveOrThrow(p.typeName(), typeResolver);
    if (!inferred.equals(explicit)) {
        throw new RuntimeException(
                "pattern type '" + p.typeName() + "' on entry point '" + p.entryPoint()
                + "' does not match unit-class declaration '" + inferred.getName() + "'");
    }
    return explicit;
}
```

Precedence: **cast** > **explicit (with cross-check)** > **inferred**. `"var"` counts as bare.

### Call-site wiring

`buildRule`, `buildLhs`, `buildPattern` all gain a `Map<String, Class<?>> entryPointTypes` parameter (and `Class<?> unitClass` for error messages). `buildPattern`'s existing `effectiveTypeName` line:

```java
String effectiveTypeName = parseResult.castTypeName() != null ? parseResult.castTypeName() : parseResult.typeName();
```

is replaced by:

```java
Class<?> type = resolvePatternType(parseResult, typeResolver, entryPointTypes, unitClass);
ObjectType objectType = new ClassObjectType(type, false);
```

## Test corpus migration

### New test-domain class

`drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`:

```java
package org.drools.drlx.ruleunit;

import org.drools.drlx.domain.Location;
import org.drools.drlx.domain.Person;
// plus whatever else the test corpus references
import org.drools.ruleunits.api.DataStore;

public class MyUnit {
    public DataStore<Person> persons;
    public DataStore<Location> locations;
    // other entry points added during migration as test coverage demands
}
```

### DRLX source migration

Every existing test rule adds one import:

```
import org.drools.drlx.ruleunit.MyUnit;
```

Rules keep their existing `Type bind : /entry[...]` form. The new cross-check confirms the explicit type matches MyUnit's field types — catching any test-code drift.

### Migration inventory (verify during implementation)

- `PositionalTest.java` — `locations`
- `RuleAnnotationsTest.java` — `persons`
- `InlineCastTest.java` — mixed
- `DrlxRuleBuilderTest.java` — `persons`/others
- `DrlxCompilerTest.java` / `DrlxCompilerNoPersistTest.java`
- Parser-only tests (`DrlxParserTest.java`, `DrlxToJavaParserVisitorTest.java`, `TolerantDrlxToJavaParserVisitorTest.java`) — may or may not hit the runtime builder; add imports defensively.

## Error / negative cases

Fail loud on every malformed configuration — no warnings, no silent fallbacks. Reuses `RuntimeException` style established by `@Salience` / `@Description` work.

| Input | Where | Message |
|-------|-------|---------|
| Missing `unit <Name>;` declaration | `resolveUnitClass` (empty `unitName`) | `DRLX compilation unit is missing a 'unit <Name>;' declaration` |
| `unit MyUnit;` with no matching import | `resolveUnitClass` | `Unit class 'MyUnit' not found — add 'import <fqn>.MyUnit;' to the DRLX source.` |
| Import spelled but class not on classpath | `resolveOrThrow` | `Unit class '<fqn>' not on classpath` (wraps `ClassNotFoundException`) |
| Raw `DataSource` / `DataStore` field (no type parameter) | `buildEntryPointTypeMap` | `Field '<unit>.<field>' is a raw DataSource — must be parameterised (e.g. DataStore<Person>)` |
| Bare pattern references entry point not on unit | `resolvePatternType` | `no type could be inferred for entry point '<name>' — declare 'DataSource<T> <name>;' on unit class '<unit>'` |
| Explicit pattern type references entry point not on unit | `resolvePatternType` | `entry point '<name>' is not declared on unit class '<unit>' — add 'DataSource<T> <name>;'` |
| Explicit type doesn't match unit field type | `resolvePatternType` | `pattern type '<explicit>' on entry point '<name>' does not match unit-class declaration '<inferred>'` |

## Testing scope

New test class: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java`.

| # | Kind | Test | Proves |
|---|------|------|--------|
| 1 | happy (bare) | `bareVarPattern_inferredFromUnit` | `var p : /persons[...]` compiles and resolves to `Person`. First end-to-end `var` binding. |
| 2 | happy (cross-check) | `explicitTypeMatchesUnitField` | `Person p : /persons[...]` passes cross-check. Belt-and-braces. |
| 3 | happy (cast precedence) | `inlineCastBeatsInference` | `/objects#Car[...]` with `objects` declared as `DataStore<Vehicle>` — cast type `Car` wins, no mismatch error. |
| 4 | negative | `missingUnitDeclaration_failsLoud` | DRLX without `unit …;` → clear error. |
| 5 | negative | `missingUnitImport_failsLoud` | `unit MyUnit;` without matching import → error points at the missing import. |
| 6 | negative | `unitClassNotOnClasspath_failsLoud` | Import spelled but class absent → clear error. |
| 7 | negative | `rawDataSourceField_failsLoud` | Field declared as raw `DataStore` → clear error. |
| 8 | negative | `entryPointNotOnUnit_failsLoud` | Rule uses `/strangers` but MyUnit has no field → clear error. |
| 9 | negative | `explicitTypeMismatchesUnitField_failsLoud` | `Wrong p : /persons[...]` with `DataStore<Person> persons;` → mismatch error. |

Plus the existing ~50-test corpus gets migrated (import added, MyUnit field coverage). Their passing is the regression guard for the migration path.

**Expected end state:** 50 (migrated) + 9 new = **59 tests**. Once #14 lands, #8 unblocks and adds 7 more → 66 at the end of #8.

## Non-goals / deferred

Explicitly out of scope for #14:

- **Runtime `RuleUnitInstance` model** — tracked as #15.
- **Unit-class inheritance** — `getDeclaredFields()` only; Decision 5.
- **Field-level annotations** (e.g. `@Entry`, `@Watch`) — field name is the entry-point name; no extra metadata.
- **DRLX-declared data sources** inside the `unit` block (`unit MyUnit { DataStore<Person> persons; }`) — separate feature.
- **Multiple unit classes per compilation unit** — exactly one.
- **`@DataSource(...)` annotation on rules** (DRLXXXX.md line 920+) — separate feature.
- **DESIGN.md update** — lines mentioning `CompilationUnitIR` get updated with the new `unitName` field and the two new builder helpers (`resolveUnitClass`, `buildEntryPointTypeMap`, `resolvePatternType`). Applied during implementation via `java-update-design`.
