# #58 Recursive Queries Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable a query to reference itself in its own LHS, supporting transitive closure patterns like `or(/trusts(a, b), and(/trusts(a, var z), /trusts(z, b)))`.

**Architecture:** Thread the `queryRegistry` map into `buildQuery()` and all recursive `buildLhs()` calls. Register each query in the registry *before* compiling its LHS so the self-lookup succeeds. Eliminate the short `buildLhs` overload that defaults to `Map.of()`.

**Tech Stack:** Java 21, Drools 10.x runtime (QueryImpl, QueryElement, PhreakQueryNode)

---

### Task 1: Create Trust test POJO

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/domain/Trust.java`

- [ ] **Step 1: Create Trust.java**

```java
package org.drools.drlx.domain;

public class Trust {

    private Object a;
    private Object b;

    public Trust() {
    }

    public Trust(Object a, Object b) {
        this.a = a;
        this.b = b;
    }

    public Object getA() {
        return a;
    }

    public void setA(Object a) {
        this.a = a;
    }

    public Object getB() {
        return b;
    }

    public void setB(Object b) {
        this.b = b;
    }
}
```

- [ ] **Step 2: Add DataStore to MyUnit**

In `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`, add the import and field:

```java
import org.drools.drlx.domain.Trust;
```

```java
public DataStore<Trust> trusts = DataSource.createStore();
```

- [ ] **Step 3: Compile to verify**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test-compile`
Expected: BUILD SUCCESS

### Task 2: Write failing recursive query test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`

- [ ] **Step 1: Add the recursive query test**

Add this test method to `QueryTest.java`:

```java
@Test
void recursiveQueryTransitiveClosure() {
    String source = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Trust;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule Trusts(Object a, Object b) {
                or(Trust b1 : /trusts[a == a, b == b],
                   and(Trust b2 : /trusts[a == a],
                       /trusts(b2.b, var z),
                       z == b
                   )
                )
            }

            rule R1 {
                var a : /persons[name == "A"],
                /trusts(a.name, var b),
                do { results.add(b); }
            }
            """;

    KieBase kieBase = newBuilder().build(source);
    MyUnit unit = new MyUnit();
    unit.persons.add(new org.drools.drlx.domain.Person("A", 0));
    unit.trusts.add(new org.drools.drlx.domain.Trust("A", "B"));
    unit.trusts.add(new org.drools.drlx.domain.Trust("B", "C"));
    unit.trusts.add(new org.drools.drlx.domain.Trust("C", "D"));

    try (DrlxRuleUnitInstance<MyUnit> instance = DrlxRuleUnitInstance.create(kieBase, unit)) {
        instance.fire();

        assertThat(unit.results).containsExactlyInAnyOrder("B", "C", "D");
    }
}
```

Note: The exact DRLX syntax for the recursive query body may need adjustment during implementation. The key pattern is: `or` of a direct trust match and a recursive call via `/trusts(intermediate, var z)`. The test should fail at compile time because `buildLhs` can't find the query in the registry.

- [ ] **Step 2: Run the test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.QueryTest#recursiveQueryTransitiveClosure" -pl .`
Expected: FAIL — the `/trusts(...)` call inside the query body won't resolve because `queryRegistry` is `Map.of()`.

### Task 3: Thread queryRegistry through buildQuery and buildLhs

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Change the first-pass query loop in build() (lines 110–117)**

Replace:

```java
        for (RuleIR rule : parseResult.rules()) {
            if (!rule.parameters().isEmpty()) {
                QueryImpl query = buildQuery(rule, pkg.getTypeResolver(), entryPointTypes, unitClass);
                String entryPointName = Character.toLowerCase(rule.name().charAt(0)) + rule.name().substring(1);
                queryRegistry.put(entryPointName, query);
                pkg.addRule(query);
            }
        }
```

With:

```java
        for (RuleIR rule : parseResult.rules()) {
            if (!rule.parameters().isEmpty()) {
                String entryPointName = Character.toLowerCase(rule.name().charAt(0)) + rule.name().substring(1);
                QueryImpl query = new QueryImpl(rule.name());
                queryRegistry.put(entryPointName, query);
                buildQuery(query, rule, pkg.getTypeResolver(), entryPointTypes, unitClass, queryRegistry);
                pkg.addRule(query);
            }
        }
```

- [ ] **Step 2: Update buildQuery() signature and body (lines 379–421)**

Replace the entire `buildQuery` method:

```java
    private void buildQuery(QueryImpl query,
                            RuleIR parseResult,
                            TypeResolver typeResolver,
                            Map<String, Class<?>> entryPointTypes,
                            Class<?> unitClass,
                            Map<String, QueryImpl> queryRegistry) {
        lambdaCompiler.beginRule(parseResult.name());

        Pattern prefixPattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0,
                ClassObjectType.DroolsQuery_ObjectType, null);

        QueryNameConstraint nameConstraint = new QueryNameConstraint(null, query.getName());
        prefixPattern.addConstraint(nameConstraint);

        List<RuleParameterIR> params = parseResult.parameters();
        Declaration[] paramDecls = new Declaration[params.size()];
        for (int i = 0; i < params.size(); i++) {
            RuleParameterIR param = params.get(i);
            Class<?> paramType = resolveOrThrow(param.typeName(), typeResolver);
            Declaration decl = prefixPattern.addDeclaration(param.paramName());
            ArrayElementReader reader = new ArrayElementReader(
                    DroolsQueryElementsReader.INSTANCE, i, paramType);
            decl.setReadAccessor(reader);
            paramDecls[i] = decl;
        }
        query.setParameters(paramDecls);

        GroupElement root = GroupElementFactory.newAndInstance();
        root.addChild(prefixPattern);

        Map<String, BoundVariable> boundVariables = new LinkedHashMap<>();
        for (int i = 0; i < params.size(); i++) {
            RuleParameterIR param = params.get(i);
            Class<?> paramType = resolveOrThrow(param.typeName(), typeResolver);
            boundVariables.put(param.paramName(),
                    new BoundVariable(param.paramName(), paramType, prefixPattern, paramDecls[i]));
        }

        buildLhs(parseResult.lhs(), root, typeResolver, entryPointTypes, unitClass, boundVariables, queryRegistry);

        query.setLhs(root);
    }
```

- [ ] **Step 3: Thread queryRegistry through all recursive buildLhs calls**

Remove the short overload (lines 423–430):

```java
    // DELETE this method entirely:
    private void buildLhs(List<LhsItemIR> items,
                          GroupElement parent,
                          TypeResolver typeResolver,
                          Map<String, Class<?>> entryPointTypes,
                          Class<?> unitClass,
                          Map<String, BoundVariable> boundVariables) {
        buildLhs(items, parent, typeResolver, entryPointTypes, unitClass, boundVariables, Map.of());
    }
```

Then fix all call sites that used the short overload. Each needs `queryRegistry` added as the last argument:

**Line 496** (NOT/EXISTS multi-child wrapper):
```java
buildLhs(group.children(), andInstance, typeResolver, entryPointTypes, unitClass, innerScope, queryRegistry);
```

**Line 499** (normal group element):
```java
buildLhs(group.children(), ge, typeResolver, entryPointTypes, unitClass, innerScope, queryRegistry);
```

**Line 557** (accumulate multi-source):
```java
buildLhs(groupIr.children(), andGroup, typeResolver, entryPointTypes,
         unitClass, innerScope, queryRegistry);
```

**Line 644** (acc() multi-source):
```java
buildLhs(groupIr.children(), andGroup, typeResolver, entryPointTypes,
         unitClass, innerScope, queryRegistry);
```

For the accumulate call sites (557, 644), the `queryRegistry` parameter needs to be available. Check whether these methods (`buildAccumulate`, `buildAccKeywordForm`, etc.) already receive `queryRegistry` or need it added to their signatures. If they don't receive it, add it to their signatures and thread it from `buildLhs` which does receive it.

- [ ] **Step 4: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile`
Expected: BUILD SUCCESS

### Task 4: Run the recursive query test and iterate

**Files:**
- Possibly modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java`
- Possibly modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Run the recursive query test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="org.drools.drlx.builder.syntax.QueryTest#recursiveQueryTransitiveClosure" -pl .`

If the test passes, proceed to Step 3. If it fails, debug and fix (Step 2).

- [ ] **Step 2: Debug and fix**

The most likely issues:
1. **DRLX syntax for the recursive query body** — the `or`/`and` structure and how constraints bind to query parameters may need adjustment. Study the existing `or`/`and` group element tests to see the exact syntax the parser expects.
2. **Variable scoping** — `var z` output from the inner `/trusts(...)` call needs to be visible for the `z == b` constraint or the second `/trusts(z, b)` call. The `or`/`and` scope isolation in `buildLhs` (line 485-487) may need attention.
3. **QueryImpl not fully initialized** — since we pre-create QueryImpl and register it before `buildQuery` populates it, ensure the fields needed by `buildQueryElement` (namely `getParameters()` and `getName()`) are set by the time the self-referencing call is processed. `getName()` is set at construction. `getParameters()` is set at line `query.setParameters(paramDecls)` which happens before `buildLhs`, so this should be fine.

- [ ] **Step 3: Run full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`
Expected: All tests pass (existing + new recursive query test).

### Task 5: Commit

- [ ] **Step 1: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -DskipTests`

- [ ] **Step 2: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
  drlx-parser-core/src/test/java/org/drools/drlx/domain/Trust.java \
  drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/QueryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(query): support recursive query self-reference (#58)

Thread queryRegistry into buildQuery/buildLhs so a query can look
itself up during LHS compilation, enabling transitive closure patterns.

Refs #58"
```
