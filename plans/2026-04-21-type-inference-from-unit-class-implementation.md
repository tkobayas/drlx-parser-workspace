# Pattern-Type Inference from Unit Class Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Compile-time inference of DRLX pattern element types from a required unit class's `DataSource<T>` / `DataStore<T>` fields. Unblocks #8 / #9 and enables bare `var p : /persons[...]` end-to-end.

**Architecture:** Read `unit <Name>;` into the IR, resolve the unit class via imports at build time, reflectively scan `DataSource`-subtype fields for their generic `T`, build `Map<String, Class<?>>`, consult it during pattern-type resolution with precedence `castType > explicit (cross-checked) > inferred`. Runtime pipeline (KieSession, EntryPointId) unchanged.

**Tech Stack:** Java 17, Maven, ANTLR4, protobuf (bundled `protoc-25.5`), JUnit 5, AssertJ, Drools RuleUnit API (`org.drools.ruleunits.api.DataSource` / `DataStore`).

**Spec:** `/home/tkobayas/claude/public/drlx-parser/specs/2026-04-21-type-inference-from-unit-class-design.md` (commit `ebe3fbf`).

**Issue:** #14 (blocks #8). Parent epic: #4.

---

## File Structure

All paths are relative to `/home/tkobayas/usr/work/mvel3-development/drlx-parser/`.

| Path | Role |
|------|------|
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` | Add `unitName` to `CompilationUnitIR`. |
| `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` | Add `string unit_name = 5;` to `CompilationUnitParseResult`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java` | Regenerated. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` | Translate `unitName` through save/load. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Capture `unitName` from `ctx.unitDeclaration()`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` | Add `resolveUnitClass`, `buildEntryPointTypeMap`, `resolveDataSourceTypeArg`, `resolvePatternType`. Thread `entryPointTypes` map + `unitClass` through `buildRule` → `buildLhs` → `buildPattern`. |
| `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java` (CREATE) | Shared test-domain unit class with `DataStore<T>` fields for every entry point in the test corpus. |
| `drlx-parser-core/src/test/**/*.java` containing DRLX source | Add `import org.drools.drlx.ruleunit.MyUnit;` to each DRLX block. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java` (CREATE) | 9 tests (3 happy + 6 negative). |
| `drlx-parser-core/docs/DESIGN.md` | Mention `unitName` on `CompilationUnitIR` and the two new helper methods. |

**Maven rule:** after modifying `drlx-parser-core`, run `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install`. Do NOT `cd` — use `-f` / `-C` flags.

**Commit convention:** scoped imperative subject (`refactor(ir):`, `feat(runtime):`, `test(inference):`, `docs:`), HEREDOC body, `Refs #14` trailer. No `Co-Authored-By`. No emoji. Commit directly on `main` (no worktree).

---

## Task 1: IR + proto + visitor — add `unitName` field

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java` (regen)
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

No behaviour change; `unitName` populated but not yet consumed. All existing tests pass.

- [ ] **Step 1: Add `unitName` to `CompilationUnitIR`**

In `DrlxRuleAstModel.java`, change the record:

```java
public record CompilationUnitIR(String packageName,
                                String unitName,
                                List<String> imports,
                                List<RuleIR> rules) {
}
```

- [ ] **Step 2: Add `unit_name` to proto**

In `drlx_rule_ast.proto`, add field `5` to `CompilationUnitParseResult`:

```proto
message CompilationUnitParseResult {
  string source_hash = 1;
  string package_name = 2;
  repeated string imports = 3;
  repeated RuleParseResult rules = 4;
  string unit_name = 5;
}
```

- [ ] **Step 3: Regenerate proto**

```bash
/tmp/protoc-25.5/bin/protoc --java_out=/home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java \
    --proto_path=/home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/proto \
    /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/proto/drlx_rule_ast.proto
```

- [ ] **Step 4: Update `DrlxRuleAstParseResult`**

Translate `unitName` through save/load. In `save(...)`:

```java
DrlxRuleAstProto.CompilationUnitParseResult.Builder builder =
        DrlxRuleAstProto.CompilationUnitParseResult.newBuilder()
                .setSourceHash(hashSource(drlxSource))
                .setPackageName(data.packageName())
                .setUnitName(data.unitName() == null ? "" : data.unitName());
```

In `load(...)`, after reading package name and imports, before the rules loop's final `return`:

```java
return new CompilationUnitIR(parseResult.getPackageName(),
        parseResult.getUnitName(),
        List.copyOf(parseResult.getImportsList()),
        List.copyOf(rules));
```

- [ ] **Step 5: Capture `unitName` in the visitor**

In `DrlxToRuleAstVisitor.visitDrlxCompilationUnit`, read the unit declaration:

```java
String unitName = "";
if (ctx.unitDeclaration() != null) {
    unitName = ctx.unitDeclaration().qualifiedName().getText();
}
```

Update the return statement:

```java
return new CompilationUnitIR(packageName, unitName, List.copyOf(imports), List.copyOf(rules));
```

- [ ] **Step 6: Full test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`, 50+7=57 tests pass (the 7 from the #7/#8 line — confirm actual baseline before this task).

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(ir): add unitName to CompilationUnitIR

Capture the simple name from 'unit <Name>;' and thread it through
IR, proto, and visitor. Not yet consumed by the runtime builder —
subsequent commits resolve the unit class and use its DataSource
fields for type inference.

Refs #14
EOF
)"
```

---

## Task 2: Create `MyUnit.java` test class + migrate existing DRLX sources

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`
- Modify: every test `.java` file containing DRLX source that uses `unit MyUnit;`

Required reference before starting: grep every DRLX source to inventory the entry points actually in use.

- [ ] **Step 1: Inventory entry points**

```bash
grep -rn "getEntryPoint\|EntryPoint\s*[a-z]*\s*=\s*" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test --include="*.java" | grep -oE 'getEntryPoint\("[a-zA-Z]+"\)' | sort -u
```

Also grep DRLX sources for `/identifier` patterns to catch any entry points that are only referenced in rules:

```bash
grep -rn 'rule [A-Za-z]' /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/test --include="*.java" -A 8 | grep -oE '/[a-z]+' | sort -u
```

Expected set based on current corpus: at minimum `persons`, `locations`. Add whatever else turns up.

- [ ] **Step 2: Create `MyUnit.java`**

Directory: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/`.

```java
package org.drools.drlx.ruleunit;

import org.drools.drlx.domain.Location;
import org.drools.drlx.domain.Person;
import org.drools.ruleunits.api.DataStore;

public class MyUnit {
    public DataStore<Person> persons;
    public DataStore<Location> locations;
    // Extend with more fields as the entry-point inventory from Step 1 demands.
}
```

If Step 1 reveals other entry points (e.g. `orders`, `requests`, `as`), add corresponding `DataStore<T>` fields and import their domain classes. If a domain class is missing (e.g. `Order`), create it with minimal fields (follow `Person`'s shape).

- [ ] **Step 3: Migrate DRLX sources in every test**

For each `.java` test file containing a DRLX source block with `unit MyUnit;`, add:

```
import org.drools.drlx.ruleunit.MyUnit;
```

to the DRLX source's imports. Place it alphabetically with the other DRLX-source imports (e.g. after `import org.drools.drlx.domain.Person;`).

Files to migrate (verify with grep `unit MyUnit` in test sources before starting):

- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PositionalTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/InlineCastTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleBuilderTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/tools/DrlxCompilerTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/tools/DrlxCompilerNoPersistTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxToJavaParserVisitorTest.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitorTest.java`

(Use grep to confirm the exact list.)

- [ ] **Step 4: Run the full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install 2>&1 | tail -10
```

Expected: all tests still pass. No behaviour change — the runtime builder still ignores `unitName`; the imports now resolve trivially but aren't used.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java \
    drlx-parser-core/src/test/java/org/drools/drlx

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: introduce MyUnit unit class and migrate DRLX imports

Creates a shared test-domain unit class with typed DataStore<T>
fields for each entry point the existing corpus references. Adds
the 'import org.drools.drlx.ruleunit.MyUnit;' line to every DRLX
source block. No behaviour change yet — the runtime builder still
ignores the unit class; subsequent commits consume it for type
inference.

Refs #14
EOF
)"
```

---

## Task 3: Runtime builder — `resolveUnitClass` (required-unit enforcement)

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

After this task, any DRLX source without a resolvable unit class fails at build time. Existing tests pass because MyUnit is now on the classpath and imported in every DRLX source (Task 2).

- [ ] **Step 1: Add imports**

At the top of `DrlxRuleAstRuntimeBuilder.java`:

```java
import org.drools.util.TypeResolver;
```

(If not already imported.)

- [ ] **Step 2: Add `resolveUnitClass` helper**

Place this as a new private static method in `DrlxRuleAstRuntimeBuilder`, near the other helpers:

```java
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

- [ ] **Step 3: Call `resolveUnitClass` at the start of `build(...)`**

In `DrlxRuleAstRuntimeBuilder.build(CompilationUnitIR)`, immediately after `pkg.addImport(...)` and before the `parseResult.rules().forEach(...)` loop:

```java
Class<?> unitClass = resolveUnitClass(parseResult.unitName(),
                                      parseResult.imports(),
                                      pkg.getTypeResolver());
```

For now, `unitClass` is unused in the loop. Task 4 uses it.

To avoid "unused variable" compile warnings temporarily, add a simple reference:

```java
Class<?> unitClass = resolveUnitClass(parseResult.unitName(),
                                      parseResult.imports(),
                                      pkg.getTypeResolver());
// unitClass is validated here; type-map construction lands in the next commit.
java.util.Objects.requireNonNull(unitClass);
```

(Remove the `requireNonNull` in Task 4 once it's used.)

- [ ] **Step 4: Full test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install 2>&1 | tail -15
```

Expected: all tests pass. If any test fails with "Unit class 'MyUnit' not found", that test's DRLX source didn't get the import in Task 2 — go back and fix it.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(runtime): require and resolve unit class on every build

Every DRLX compilation unit must declare 'unit <Name>;' and either
import the class by simple name or give its FQN in the declaration.
Missing declaration or unresolvable import fails at build time with
a clear message. Type-map construction and pattern resolution land
in the next commit.

Refs #14
EOF
)"
```

---

## Task 4: Runtime builder — `buildEntryPointTypeMap` + `resolvePatternType`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

Existing tests still pass after this task: their explicit types match MyUnit's field types (cross-check passes), and no test currently uses bare or `var` patterns at runtime.

- [ ] **Step 1: Add imports**

```java
import java.lang.reflect.Field;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import org.drools.ruleunits.api.DataSource;
```

- [ ] **Step 2: Add `buildEntryPointTypeMap` + `resolveDataSourceTypeArg`**

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
    if (DataSource.class.equals(rawClass)) {
        Type[] args = pt.getActualTypeArguments();
        return (args.length == 1 && args[0] instanceof Class<?> c) ? c : null;
    }
    // rawClass is a subtype — walk its generic interfaces to find DataSource<T>.
    return resolveAgainstSuperInterface(rawClass, pt.getActualTypeArguments());
}

private static Class<?> resolveAgainstSuperInterface(Class<?> rawClass, Type[] actualArgs) {
    // Map the subtype's type parameters to the concrete arguments we were given.
    java.lang.reflect.TypeVariable<?>[] typeParams = rawClass.getTypeParameters();
    Map<String, Type> bindings = new java.util.HashMap<>();
    for (int i = 0; i < typeParams.length && i < actualArgs.length; i++) {
        bindings.put(typeParams[i].getName(), actualArgs[i]);
    }
    // Find DataSource<T> amongst the declared generic interfaces.
    for (Type iface : rawClass.getGenericInterfaces()) {
        if (!(iface instanceof ParameterizedType ptIface)) continue;
        Type raw = ptIface.getRawType();
        if (!DataSource.class.equals(raw)) continue;
        Type tArg = ptIface.getActualTypeArguments()[0];
        // If T is a type variable on rawClass, substitute from bindings.
        if (tArg instanceof java.lang.reflect.TypeVariable<?> tv) {
            Type bound = bindings.get(tv.getName());
            return (bound instanceof Class<?> c) ? c : null;
        }
        if (tArg instanceof Class<?> c) {
            return c;
        }
    }
    // Also check generic superclass chain (e.g. a class extending AbstractDataSource<T>).
    Type superType = rawClass.getGenericSuperclass();
    if (superType instanceof ParameterizedType ptSuper
            && ptSuper.getRawType() instanceof Class<?> superClass) {
        return resolveAgainstSuperInterface(superClass, ptSuper.getActualTypeArguments());
    }
    return null;
}
```

- [ ] **Step 3: Add `resolvePatternType`**

```java
private static Class<?> resolvePatternType(PatternIR p,
                                           TypeResolver typeResolver,
                                           Map<String, Class<?>> entryPointTypes,
                                           Class<?> unitClass) {
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

    if (inferred == null) {
        throw new RuntimeException(
                "entry point '" + p.entryPoint() + "' is not declared on unit class '"
                + unitClass.getName() + "' — add `DataSource<T> " + p.entryPoint() + ";`");
    }

    Class<?> explicit;
    try {
        explicit = typeResolver.resolveType(p.typeName());
    } catch (ClassNotFoundException e) {
        throw new RuntimeException(
                "pattern type '" + p.typeName() + "' could not be resolved", e);
    }
    if (!inferred.equals(explicit)) {
        throw new RuntimeException(
                "pattern type '" + p.typeName() + "' on entry point '" + p.entryPoint()
                + "' does not match unit-class declaration '" + inferred.getName() + "'");
    }
    return explicit;
}
```

Note: `resolveOrThrow` was added in Task 3. Reuse it for cast resolution.

- [ ] **Step 4: Thread `entryPointTypes` + `unitClass` through the build pipeline**

In `build(CompilationUnitIR parseResult)`:

```java
Class<?> unitClass = resolveUnitClass(parseResult.unitName(),
                                      parseResult.imports(),
                                      pkg.getTypeResolver());
Map<String, Class<?>> entryPointTypes = buildEntryPointTypeMap(unitClass);

parseResult.rules().forEach(rule ->
        pkg.addRule(buildRule(rule, pkg.getTypeResolver(), entryPointTypes, unitClass)));
```

Remove the `java.util.Objects.requireNonNull(unitClass);` shim from Task 3.

Update `buildRule` signature and internal calls:

```java
private RuleImpl buildRule(RuleIR parseResult,
                           TypeResolver typeResolver,
                           Map<String, Class<?>> entryPointTypes,
                           Class<?> unitClass) {
    // ... unchanged annotation / setup code ...
    buildLhs(parseResult.lhs(), root, typeResolver, entryPointTypes, unitClass, boundVariables);
    // ... unchanged rhs / finalisation ...
}
```

Update `buildLhs`:

```java
private void buildLhs(List<LhsItemIR> items,
                      GroupElement parent,
                      TypeResolver typeResolver,
                      Map<String, Class<?>> entryPointTypes,
                      Class<?> unitClass,
                      Map<String, BoundVariable> boundVariables) {
    for (LhsItemIR item : items) {
        if (item instanceof PatternIR patternIr) {
            Pattern pattern = buildPattern(patternIr, typeResolver, entryPointTypes, unitClass, boundVariables);
            // ... unchanged post-processing ...
        } else if (item instanceof GroupElementIR group) {
            GroupElement ge = switch (group.kind()) {
                case NOT -> GroupElementFactory.newNotInstance();
            };
            buildLhs(group.children(), ge, typeResolver, entryPointTypes, unitClass, boundVariables);
            parent.addChild(ge);
        } else {
            throw new IllegalArgumentException("Unsupported LHS item: " + item.getClass().getName());
        }
    }
}
```

Update `buildPattern` signature and replace its type-resolution preamble:

```java
private Pattern buildPattern(PatternIR parseResult,
                             TypeResolver typeResolver,
                             Map<String, Class<?>> entryPointTypes,
                             Class<?> unitClass,
                             Map<String, BoundVariable> boundVariables) {
    Class<?> type = resolvePatternType(parseResult, typeResolver, entryPointTypes, unitClass);
    ObjectType objectType = new ClassObjectType(type, false);

    Pattern pattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0, objectType, parseResult.bindName(), false);
    pattern.setSource(new EntryPointId(parseResult.entryPoint()));

    // ... unchanged declarations, positionalArgs, conditions ...
    return pattern;
}
```

Delete the old `effectiveTypeName` line (the `parseResult.castTypeName() != null ? parseResult.castTypeName() : parseResult.typeName()` + inline `resolveType` + `try/catch`) — `resolvePatternType` now handles all three cases.

- [ ] **Step 5: Full test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install 2>&1 | tail -20
```

Expected: all existing tests pass. The cross-check (β) silently validates that `Person p : /persons[...]` matches MyUnit's `DataStore<Person> persons;`, etc. If any test fails with "pattern type … does not match unit-class declaration", MyUnit's field type is wrong for that entry point — fix MyUnit.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(runtime): resolve pattern types via unit class

Reflectively scans the unit class's DataSource<T> fields, builds a
Map<entryPoint, Class>, and consults it during pattern resolution
with precedence castType > explicit (cross-checked) > inferred.
'var' is treated as 'no explicit type' and routes to the map.
Existing tests pass unchanged — the β cross-check silently confirms
their explicit types match MyUnit's declarations.

Refs #14
EOF
)"
```

---

## Task 5: Happy test — `bareVarPattern_inferredFromUnit` (RED + GREEN)

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java`

After Task 4, `var p : /persons[...]` already works. This task creates the first test that proves it and establishes the new test class.

- [ ] **Step 1: Create `TypeInferenceTest.java` with the happy test**

```java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.domain.Person;
import org.junit.jupiter.api.Test;
import org.kie.api.runtime.rule.EntryPoint;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TypeInferenceTest extends DrlxBuilderTestSupport {

    @Test
    void bareVarPattern_inferredFromUnit() {
        // `var p : /persons[...]` with no explicit type — element type comes from
        // MyUnit.persons' DataStore<Person> declaration.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule OnlyAdults {
                    var p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        withSession(rule, kieSession -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");
            persons.insert(new Person("Alice", 30));
            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
        });
    }
}
```

- [ ] **Step 2: Run the test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am test \
    -Dtest=TypeInferenceTest#bareVarPattern_inferredFromUnit 2>&1 | tail -10
```

Expected: passes. If it fails with `ClassNotFoundException: var`, the `"var"`-as-sentinel branch in `resolvePatternType` isn't firing — re-check the `isBare` condition in Task 4 Step 3.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(inference): bare var pattern inferred from unit class

First end-to-end proof that 'var p : /persons[...]' resolves to
Person via MyUnit.persons' DataStore<Person> declaration. The
'var' sentinel now routes through the entry-point type map.

Refs #14
EOF
)"
```

---

## Task 6: Happy test — `explicitTypeMatchesUnitField`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java`

- [ ] **Step 1: Append the test**

```java
    @Test
    void explicitTypeMatchesUnitField() {
        // Existing corpus behaviour — 'Person p : /persons[...]' with MyUnit.persons
        // as DataStore<Person> passes the cross-check silently.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule ExplicitMatches {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        withSession(rule, kieSession -> {
            final EntryPoint persons = kieSession.getEntryPoint("persons");
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
        });
    }
```

- [ ] **Step 2: Run the test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am test \
    -Dtest=TypeInferenceTest#explicitTypeMatchesUnitField 2>&1 | tail -10
```

Expected: passes.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(inference): explicit pattern type passes cross-check

Belt-and-braces — existing corpus form 'Person p : /persons[...]'
continues to work, with the β cross-check silently validating
against MyUnit.persons' DataStore<Person> declaration.

Refs #14
EOF
)"
```

---

## Task 7: Happy test — `inlineCastBeatsInference`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`
- Possibly create: `drlx-parser-core/src/test/java/org/drools/drlx/domain/Vehicle.java` and `Car.java`.

- [ ] **Step 1: Verify domain classes**

Look for `Car.java` / `Vehicle.java` under `drlx-parser-core/src/test/java/org/drools/drlx/domain/`. If missing, create the minimal shapes:

```java
package org.drools.drlx.domain;

public class Vehicle {
    private final String vin;
    public Vehicle(String vin) { this.vin = vin; }
    public String getVin() { return vin; }
}
```

```java
package org.drools.drlx.domain;

public class Car extends Vehicle {
    private final int speed;
    public Car(String vin, int speed) { super(vin); this.speed = speed; }
    public int getSpeed() { return speed; }
}
```

Note: the `/objects#Car[speed > 80]` existing inline-cast test (`InlineCastTest.java`) should already use matching domain shapes — cross-check there before adding or duplicating.

- [ ] **Step 2: Add `DataStore<Vehicle> objects` to MyUnit** (only if not already present from existing InlineCastTest migration)

In `MyUnit.java`:

```java
import org.drools.drlx.domain.Vehicle;
// ...
public DataStore<Vehicle> objects;
```

- [ ] **Step 3: Append the test**

```java
    @Test
    void inlineCastBeatsInference() {
        // `/objects#Car[speed > 80]` — the #Car cast wins over the unit declaration
        // `DataStore<Vehicle> objects;`. No mismatch error despite Car != Vehicle.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Car;
                import org.drools.drlx.domain.Vehicle;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule FastCars {
                    var v : /objects#Car[ speed > 80 ],
                    do { System.out.println(v); }
                }
                """;

        withSession(rule, kieSession -> {
            final EntryPoint objects = kieSession.getEntryPoint("objects");
            objects.insert(new org.drools.drlx.domain.Car("ABC", 120));
            objects.insert(new org.drools.drlx.domain.Car("XYZ", 40));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
        });
    }
```

- [ ] **Step 4: Run the test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am test \
    -Dtest=TypeInferenceTest#inlineCastBeatsInference 2>&1 | tail -10
```

Expected: passes. If it fails with a mismatch error, `resolvePatternType`'s cast precedence is wrong — cast should short-circuit before any inferred lookup.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java \
    drlx-parser-core/src/test/java/org/drools/drlx/domain

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(inference): inline cast beats unit-class inference

Precedence check — '/objects#Car[...]' with MyUnit.objects declared
as DataStore<Vehicle> resolves to Car (the inline cast), no
mismatch error against Vehicle. Confirms cast short-circuits before
any inferred lookup.

Refs #14
EOF
)"
```

---

## Task 8: Negative tests — missing / malformed unit class

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java`

Three tests in one commit (all about the unit class itself not being resolvable).

- [ ] **Step 1: Append `missingUnitDeclaration_failsLoud`**

```java
    @Test
    void missingUnitDeclaration_failsLoud() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                rule NoUnit {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("missing a 'unit <Name>;' declaration");
    }
```

- [ ] **Step 2: Append `missingUnitImport_failsLoud`**

```java
    @Test
    void missingUnitImport_failsLoud() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                unit MyUnit;

                rule NoImport {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("Unit class 'MyUnit' not found")
                .hasMessageContaining("import");
    }
```

- [ ] **Step 3: Append `unitClassNotOnClasspath_failsLoud`**

```java
    @Test
    void unitClassNotOnClasspath_failsLoud() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.NotARealUnit;

                unit NotARealUnit;

                rule BadUnit {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("not on classpath");
    }
```

- [ ] **Step 4: Run the three tests**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am test \
    -Dtest='TypeInferenceTest#missingUnitDeclaration_failsLoud+TypeInferenceTest#missingUnitImport_failsLoud+TypeInferenceTest#unitClassNotOnClasspath_failsLoud' 2>&1 | tail -10
```

Expected: 3 tests pass. Adjust `hasMessageContaining` argument if the actual message wording differs slightly.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(inference): fail loud on missing / unresolvable unit class

Three negative cases — no 'unit <Name>;' declaration, missing import
for the declared unit, and import spelled but class not on classpath.
Each surfaces a distinct actionable message from resolveUnitClass.

Refs #14
EOF
)"
```

---

## Task 9: Negative tests — field-scan and pattern-resolution mismatches

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnitWithRawField.java` and `MyUnitWithExtraField.java` for scoped failure cases.

Three tests: raw DataSource field, entry point not on unit, explicit-type mismatch.

- [ ] **Step 1: Create the two helper unit classes**

`drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnitWithRawField.java`:

```java
package org.drools.drlx.ruleunit;

import org.drools.ruleunits.api.DataStore;

public class MyUnitWithRawField {
    @SuppressWarnings("rawtypes")
    public DataStore persons;   // deliberately raw
}
```

(No other helper needed — `MyUnit` can be reused for the remaining cases since they don't need extra fields.)

- [ ] **Step 2: Append `rawDataSourceField_failsLoud`**

```java
    @Test
    void rawDataSourceField_failsLoud() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnitWithRawField;

                unit MyUnitWithRawField;

                rule RawField {
                    Person p : /persons[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("raw DataSource")
                .hasMessageContaining("persons");
    }
```

- [ ] **Step 3: Append `entryPointNotOnUnit_failsLoud`**

```java
    @Test
    void entryPointNotOnUnit_failsLoud() {
        // MyUnit has no 'strangers' field — rule references unknown entry point.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule UnknownEntry {
                    Person p : /strangers[ age > 18 ],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("entry point 'strangers' is not declared")
                .hasMessageContaining("MyUnit");
    }
```

- [ ] **Step 4: Append `explicitTypeMismatchesUnitField_failsLoud`**

This requires a type that's on the classpath but not equal to `Person`. Use an already-imported domain class (e.g. `Location`).

```java
    @Test
    void explicitTypeMismatchesUnitField_failsLoud() {
        // MyUnit.persons is DataStore<Person>; the rule declares 'Location p'
        // which is a real class but does not match the unit field.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Location;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule Mismatch {
                    Location p : /persons[],
                    do { System.out.println(p); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("does not match unit-class declaration");
    }
```

- [ ] **Step 5: Run the three new tests**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am test \
    -Dtest='TypeInferenceTest#rawDataSourceField_failsLoud+TypeInferenceTest#entryPointNotOnUnit_failsLoud+TypeInferenceTest#explicitTypeMismatchesUnitField_failsLoud' 2>&1 | tail -10
```

Expected: 3 tests pass. If actual error messages differ, update `hasMessageContaining` accordingly.

- [ ] **Step 6: Full suite sanity check**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install 2>&1 | tail -15
```

Expected: all existing tests + 9 new `TypeInferenceTest` tests pass. Verify `TypeInferenceTest` shows "Tests run: 9, Failures: 0, Errors: 0" in the surefire output.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/TypeInferenceTest.java \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnitWithRawField.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(inference): field-scan and pattern-resolution negatives

Three failures covered — raw DataSource field, entry point absent
from the unit class, and explicit pattern type contradicting the
unit-class declaration (β cross-check). Introduces a scoped
MyUnitWithRawField for the raw-field case.

Refs #14
EOF
)"
```

---

## Task 10: Update `docs/DESIGN.md`

**Files:**
- Modify: `/home/tkobayas/usr/work/mvel3-development/drlx-parser/docs/DESIGN.md`

- [ ] **Step 1: Locate relevant sections**

```bash
grep -n "CompilationUnitIR\|unitName\|DrlxRuleAstRuntimeBuilder\|buildPattern" \
    /home/tkobayas/usr/work/mvel3-development/drlx-parser/docs/DESIGN.md | head -20
```

Two spots likely need updating:
1. The IR table (around line 170 based on prior sessions) — update the `DrlxRuleAstModel` row to mention the new `RuleIR(name, annotations, lhs, rhs)` + `CompilationUnitIR(packageName, unitName, imports, rules)`.
2. A paragraph somewhere in the build-pipeline section describing how patterns resolve their types — add a note about unit-class-driven inference.

- [ ] **Step 2: Apply updates**

Replace the `DrlxRuleAstModel` description row (example — adjust to the current exact text):

**Before:**
```markdown
| `DrlxRuleAstModel` | In-memory IR. Records: `CompilationUnitIR`, `RuleIR(name, annotations, lhs, rhs)`, `RuleAnnotationIR`, `PatternIR`, `ConsequenceIR`, `GroupElementIR`. Sealed interface `LhsItemIR` permits `PatternIR | GroupElementIR` for tree-shape LHS. |
```

**After:**
```markdown
| `DrlxRuleAstModel` | In-memory IR. Records: `CompilationUnitIR(packageName, unitName, imports, rules)`, `RuleIR(name, annotations, lhs, rhs)`, `RuleAnnotationIR`, `PatternIR`, `ConsequenceIR`, `GroupElementIR`. Sealed interface `LhsItemIR` permits `PatternIR | GroupElementIR`. `unitName` is the simple name from `unit <Name>;`; the runtime builder resolves it against imports. |
```

Add a short note in the type-resolution area:

```markdown
Pattern types resolve with precedence `castType > explicit (cross-checked) > inferred`.
The inferred path consults a `Map<entryPoint, Class<?>>` built reflectively from the
unit class's `DataSource<T>` / `DataStore<T>` fields at build time. `'var'` counts as
"no explicit type" and routes through inference. Missing unit class, undeclared entry
points, or explicit-type mismatches all fail loud at build time.
```

(Exact placement depends on existing structure — use best judgement.)

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add docs/DESIGN.md

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
docs: note unit-class type inference in DESIGN.md

Adds unitName to the CompilationUnitIR description and a short
paragraph on the cast > explicit > inferred resolution order
introduced by this issue.

Refs #14
EOF
)"
```

---

## Final verification

- [ ] **Step 1: Full install + test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install 2>&1 | tail -25
```

Expected: `BUILD SUCCESS`. Class-level counts (compare against output):

- `InlineCastTest`: 2
- `PositionalTest`: 8
- `RuleAnnotationsTest`: 9
- `NotTest`: 1 (the RED test from #8, still failing in isolation — will flip GREEN once #8 Task 5 is re-run after #14 lands)
- `TypeInferenceTest`: **9**
- `DrlxRuleBuilderTest`: 5
- `DrlxCompilerTest`: 6
- `DrlxToJavaParserVisitorTest`: 4
- `TolerantDrlxToJavaParserVisitorTest`: 7
- `DrlxParserTest`: 4
- `DrlxCompilerNoPersistTest`: 5

Note on `NotTest`: it was committed as RED in #8 Task 4 and is not yet GREEN. #14 does not fix it. After #14 lands, resume #8 at Task 5 — the visitor wiring will now run against the new type-inference path and the test will go GREEN.

**Expected grand total: 50 (existing migrated) + 1 (RED NotTest from #8 Task 4) + 9 (new TypeInferenceTest) = 60 tests. 1 failure (NotTest#notSuppressesMatch) is expected and acceptable until #8 resumes.**

If you want a clean green run during #14 verification, temporarily disable `NotTest#notSuppressesMatch` with `@org.junit.jupiter.api.Disabled("re-enable once #8 Task 5 lands")`. Remove the annotation as the first step of #8 Task 5 resumption. **Skip if comfortable with the known RED**.

- [ ] **Step 2: Git log sanity check**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser log --oneline main..HEAD
```

Expected: roughly 10 commits on `main` starting with `feat(ir): add unitName to CompilationUnitIR` and ending with `docs: note unit-class type inference in DESIGN.md`, each trailing `Refs #14`.

- [ ] **Step 3: Close / update GitHub issues**

```bash
gh issue comment 14 --repo tkobayas/drlx-parser --body "Landed. Unit-class type inference + MyUnit test-corpus migration + 9 new TypeInferenceTest cases. Runtime RuleUnitInstance model still tracked as #15."
gh issue close 14 --repo tkobayas/drlx-parser

gh issue comment 8 --repo tkobayas/drlx-parser --body "Unblocked — #14 landed. Ready to resume #8 execution from Task 5."
```

- [ ] **Step 4: Update HANDOFF.md**

Update `/home/tkobayas/claude/public/drlx-parser/HANDOFF.md` via the `handover` skill or by hand. Key content: commit range for this landing, test count, #14 closed, #8 unblocked (resume at Task 5), and the now-known RED `NotTest#notSuppressesMatch` that will flip GREEN during #8 resumption.

---

## Appendix — notes for the implementer

- **Task 2 is the migration boundary.** Every DRLX test block needs the `import MyUnit;` line before Task 3 lands; otherwise Task 3's `resolveUnitClass` throws on those tests. Audit with `grep "unit MyUnit;" -A 0` vs `grep "import.*MyUnit"`.
- **Task 4's `resolveAgainstSuperInterface`** is the only non-trivial reflection in this plan. For the common case (field declared as `DataStore<Person>`), `DataStore.getGenericInterfaces()` yields `DataSource<T>` and the substitution table maps `T` → `Person` directly. The `getGenericSuperclass()` walk handles the less-common case of custom DataSource subclasses — probably never exercised by this project's corpus but cheap to include.
- **`"var"` sentinel.** Java has no reserved class name `var` (`var` is a restricted identifier, not a keyword), so collision with a user class is effectively impossible. The string comparison `"var".equals(p.typeName())` is safe.
- **Cache invalidation.** Old `.pb` files that saved under the #7 schema but before this change have `unit_name = ""`. At load time they deserialize into `unitName = ""` and Task 3's `resolveUnitClass` throws. This is intentional — the cache regenerates from source on next build.
- **Runtime is still untouched.** No `KieSession`-facing API changes. `kieSession.getEntryPoint("persons").insert(new Person(...))` keeps working exactly as today.
- **#8 resumption.** After this plan closes #14, resume the #8 plan (`plans/2026-04-21-group-element-and-not-implementation.md`) starting at Task 5. Tasks 1-4 of #8 already landed (commits `dcaa791`, `77724b1`, `2f5d9af`, `307bc45`, `cd63875` — note the `307bc45` trailing-comma grammar fix appended between Task 3 and Task 4).
