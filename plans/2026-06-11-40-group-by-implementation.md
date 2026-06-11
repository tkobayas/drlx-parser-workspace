# #40 Group By Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Support `groupBy(...)` keyword that partitions accumulate source facts by a grouping expression and maintains separate accumulator state per group.

**Architecture:** `groupBy` reuses the existing `acc` body grammar rules and accumulate compilation pipeline. A new `DrlxGroupByAccumulate extends Accumulate` wraps an inner `SingleAccumulate`/`MultiAccumulate` with a grouping key function compiled via MVEL3. The drools runtime dispatches to `PhreakGroupByNode` when `accumulate.isGroupBy()` returns `true`.

**Tech Stack:** ANTLR4 grammar, Java 17 records, drools-base Accumulate API, MVEL3 batch compiler, JUnit 5 + AssertJ

**Spec:** `specs/2026-06-11-40-group-by-design.md`

---

## File Map

| Action | File | Responsibility |
|--------|------|----------------|
| Modify | `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` | Add `groupByKeywordItem` and `groupByKey` grammar rules |
| Modify | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` | Add `GroupByAccumulateIR` and `GroupByCustomAccumulateIR` records |
| Modify | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Add `buildGroupByKeywordItem` visitor method |
| Create | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxGroupByAccumulate.java` | GroupBy accumulate wrapper (extends `Accumulate`) |
| Modify | `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` | Add `buildGroupByAccumulatePattern` / `buildGroupByCustomAccumulatePattern` |
| Modify | `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java` | Parser-level groupBy tests |
| Create | `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/GroupByTest.java` | Runtime + AST structure tests |

---

### Task 1: Grammar — add `groupByKeywordItem` and `groupByKey` rules

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`

- [ ] **Step 1: Add `groupByKeywordItem` alternative to `ruleItem`**

In `DrlxParser.g4`, add a new alternative to `ruleItem` (after the `accKeywordItem ','` line):

```antlr
ruleItem
    : rulePattern
    | oopathExpression ','
    | accumulateItem ','
    | accKeywordItem ','
    | groupByKeywordItem ','
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','
    | conditionalBranch ','
    | ruleConsequence
    ;
```

- [ ] **Step 2: Add `groupByKeywordItem` and `groupByKey` rules**

After the `accResultBinding` rule at the end of the acc section, add:

```antlr
// groupBy(...) keyword form — DRLXXXX §Group By.
// `groupBy` is contextual: parsed as an identifier, validated at visitor level.
groupByKeywordItem
    : identifier '(' accSource ',' groupByKey ',' accBody ')'
    ;

groupByKey
    : VAR identifier '=' expression
    | expression
    ;
```

- [ ] **Step 3: Regenerate parser and verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources -q`

Expected: BUILD SUCCESS. ANTLR generates new parser with `groupByKeywordItem` and `groupByKey` rules.

- [ ] **Step 4: Commit**

```
feat(#40): add groupBy grammar rules

Refs #40
```

---

### Task 2: IR Model — add GroupBy IR records

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`

- [ ] **Step 1: Add `GroupByAccumulateIR` record**

After the `CustomAccumulateIR` record (line ~128), add:

```java
public record GroupByAccumulateIR(
    LhsItemIR source,
    List<AccumulatorIR> accumulators,
    String groupKeyExpression,
    String groupKeyBindName,
    List<String> groupKeyReferencedBindings
) implements LhsItemIR {
    public GroupByAccumulateIR {
        accumulators = List.copyOf(accumulators);
        groupKeyReferencedBindings = List.copyOf(groupKeyReferencedBindings);
    }
}
```

- [ ] **Step 2: Add `GroupByCustomAccumulateIR` record**

After `GroupByAccumulateIR`, add:

```java
public record GroupByCustomAccumulateIR(
    LhsItemIR source,
    List<InitVarIR> initVars,
    String actionBlock,
    String reverseBlock,
    String resultTypeName,
    String resultBindName,
    String resultExpression,
    List<String> referencedBindings,
    String groupKeyExpression,
    String groupKeyBindName,
    List<String> groupKeyReferencedBindings
) implements LhsItemIR {
    public GroupByCustomAccumulateIR {
        initVars = List.copyOf(initVars);
        referencedBindings = List.copyOf(referencedBindings);
        groupKeyReferencedBindings = List.copyOf(groupKeyReferencedBindings);
    }
}
```

- [ ] **Step 3: Update the sealed `LhsItemIR` permits clause**

Change the sealed interface from:

```java
public sealed interface LhsItemIR permits PatternIR, GroupElementIR, EvalIR, AccumulatePatternIR, CustomAccumulateIR {
```

to:

```java
public sealed interface LhsItemIR permits PatternIR, GroupElementIR, EvalIR, AccumulatePatternIR, CustomAccumulateIR, GroupByAccumulateIR, GroupByCustomAccumulateIR {
```

- [ ] **Step 4: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
feat(#40): add GroupBy IR records to DrlxRuleAstModel

Refs #40
```

---

### Task 3: Parser tests — verify groupBy syntax parses

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java`

- [ ] **Step 1: Add parser test for groupBy with unbound key + single function**

```java
@Test
void parsesGroupByUnboundKeySingleFunction() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R {
                groupBy(var p : /persons,
                        p.name,
                        var avgAge = avg(p.age)),
                do {}
            }
            """;
    var tree = parseDrlxCompilationUnitAsAntlrAST(drlx);
    assertThat(tree).isNotNull();
    var groupBy = tree.ruleDeclaration(0).ruleBody().ruleItem(0)
            .groupByKeywordItem();
    assertThat(groupBy).isNotNull();
    assertThat(groupBy.identifier().getText()).isEqualTo("groupBy");
    assertThat(groupBy.accSource().boundOopath()).isNotNull();
    assertThat(groupBy.groupByKey().VAR()).isNull();
    assertThat(groupBy.groupByKey().expression()).isNotNull();
}
```

- [ ] **Step 2: Add parser test for groupBy with bound key**

```java
@Test
void parsesGroupByBoundKey() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R {
                groupBy(var p : /persons,
                        var g = p.name,
                        var avgAge = avg(p.age)),
                do {}
            }
            """;
    var tree = parseDrlxCompilationUnitAsAntlrAST(drlx);
    assertThat(tree).isNotNull();
    var groupByKey = tree.ruleDeclaration(0).ruleBody().ruleItem(0)
            .groupByKeywordItem().groupByKey();
    assertThat(groupByKey.VAR()).isNotNull();
    assertThat(groupByKey.identifier().getText()).isEqualTo("g");
}
```

- [ ] **Step 3: Add parser test for groupBy with multiple functions**

```java
@Test
void parsesGroupByMultipleFunctions() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R {
                groupBy(var p : /persons,
                        var g = p.name,
                        (var minAge = min(p.age),
                         var maxAge = max(p.age))),
                do {}
            }
            """;
    var tree = parseDrlxCompilationUnitAsAntlrAST(drlx);
    assertThat(tree).isNotNull();
    var accBody = tree.ruleDeclaration(0).ruleBody().ruleItem(0)
            .groupByKeywordItem().accBody();
    assertThat(accBody.accFunctionList()).isNotNull();
    assertThat(accBody.accFunctionList().accumulateItem()).hasSize(2);
}
```

- [ ] **Step 4: Add parser test for groupBy with and() source**

```java
@Test
void parsesGroupByWithAndSource() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R {
                groupBy(and(var p : /persons, var o : /orders[customerId == p.age]),
                        p.name,
                        var total = sum(o.amount)),
                do {}
            }
            """;
    var tree = parseDrlxCompilationUnitAsAntlrAST(drlx);
    assertThat(tree).isNotNull();
    var accSource = tree.ruleDeclaration(0).ruleBody().ruleItem(0)
            .groupByKeywordItem().accSource();
    assertThat(accSource.andElement()).isNotNull();
    assertThat(accSource.andElement().groupChild()).hasSize(2);
}
```

- [ ] **Step 5: Add parser test for groupBy with custom accumulator**

```java
@Test
void parsesGroupByCustomAccumulator() {
    String drlx = """
            package p;
            unit MyUnit;
            rule R {
                groupBy(var p : /persons,
                        p.name,
                        int s = 0;,
                        s = s + p.age,
                        int total = s),
                do {}
            }
            """;
    var tree = parseDrlxCompilationUnitAsAntlrAST(drlx);
    assertThat(tree).isNotNull();
    var accBody = tree.ruleDeclaration(0).ruleBody().ruleItem(0)
            .groupByKeywordItem().accBody();
    assertThat(accBody.accInitVars()).isNotNull();
    assertThat(accBody.accResultBinding()).isNotNull();
}
```

- [ ] **Step 6: Run parser tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -pl . -Dtest=DrlxParserTest -q`

Expected: All new tests PASS (grammar parses but visitor not wired yet — parser tests only check ANTLR parse tree).

- [ ] **Step 7: Commit**

```
test(#40): parser tests for groupBy syntax

Refs #40
```

---

### Task 4: Visitor — add `buildGroupByKeywordItem`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

- [ ] **Step 1: Add dispatch for `groupByKeywordItem` in the ruleItem loop**

In the `buildRule` method (around line 176, after the `accKeywordItem` block), add:

```java
if (itemCtx.groupByKeywordItem() != null) {
    flushPending(lhs, pendingPattern, pendingAccs);
    pendingPattern = null;
    pendingAccs = new ArrayList<>();
    lhs.add(buildGroupByKeywordItem(itemCtx.groupByKeywordItem()));
    continue;
}
```

- [ ] **Step 2: Add the `buildGroupByKeywordItem` method**

After the `buildAccKeywordItem` method, add:

```java
private LhsItemIR buildGroupByKeywordItem(DrlxParser.GroupByKeywordItemContext ctx) {
    String keyword = ctx.identifier().getText();
    if (!"groupBy".equals(keyword)) {
        throw new RuntimeException(
                "expected 'groupBy' keyword but found '" + keyword + "' at "
                + ctx.getStart().getLine() + ":" + ctx.getStart().getCharPositionInLine());
    }

    LhsItemIR source;
    if (ctx.accSource().boundOopath() != null) {
        source = buildPatternFromBoundOopath(ctx.accSource().boundOopath());
    } else {
        source = buildGroupElementFromChildren(
                ctx.accSource().andElement().groupChild(),
                GroupElementIR.Kind.AND);
    }

    java.util.Set<String> sourceBindNames;
    if (source instanceof PatternIR pat) {
        sourceBindNames = java.util.Set.of(pat.bindName());
    } else if (source instanceof GroupElementIR group) {
        sourceBindNames = group.children().stream()
                .filter(PatternIR.class::isInstance)
                .map(c -> ((PatternIR) c).bindName())
                .collect(java.util.stream.Collectors.toSet());
    } else {
        sourceBindNames = java.util.Set.of();
    }

    // Parse group key
    DrlxParser.GroupByKeyContext keyCtx = ctx.groupByKey();
    String groupKeyExpression;
    String groupKeyBindName;
    if (keyCtx.VAR() != null) {
        groupKeyBindName = keyCtx.identifier().getText();
        groupKeyExpression = getText(keyCtx.expression());
    } else {
        groupKeyBindName = null;
        groupKeyExpression = getText(keyCtx.expression());
    }
    List<String> groupKeyRefs = extractIdentifiers(groupKeyExpression);

    // Parse body — same logic as buildAccKeywordItem
    DrlxParser.AccBodyContext body = ctx.accBody();

    if (body.accFunctionList() != null) {
        List<AccumulatorIR> accumulators = new ArrayList<>();
        for (DrlxParser.AccumulateItemContext accItemCtx : body.accFunctionList().accumulateItem()) {
            accumulators.add(buildAccumulator(accItemCtx));
        }
        return new GroupByAccumulateIR(source, accumulators,
                groupKeyExpression, groupKeyBindName, groupKeyRefs);
    }

    // Custom accumulator form
    List<DrlxParser.AccActionBlockContext> actionBlocks = body.accActionBlock();
    boolean is5Param = actionBlocks.size() == 2;

    List<InitVarIR> initVars = buildInitVars(body.accInitVars(), sourceBindNames);

    String actionBlock;
    String reverseBlock;

    if (is5Param) {
        actionBlock = extractActionBlockText(actionBlocks.get(0), true);
        reverseBlock = extractActionBlockText(actionBlocks.get(1), true);
    } else {
        DrlxParser.AccActionBlockContext actionCtx = actionBlocks.get(0);
        if (actionCtx.expression().size() == 2 && actionCtx.getChild(0).getText().equals("(")) {
            actionBlock = getText(actionCtx.expression(0));
            reverseBlock = getText(actionCtx.expression(1));
        } else {
            actionBlock = extractActionBlockText(actionCtx, false);
            reverseBlock = null;
        }
    }

    DrlxParser.AccResultBindingContext resultCtx = body.accResultBinding();
    String resultTypeName = resultCtx.typeType().getText();
    String resultBindName = resultCtx.identifier().getText();
    String resultExpression = getText(resultCtx.expression());

    if (source instanceof PatternIR pat) {
        validateResultExpression(resultExpression, pat.bindName());
    }

    java.util.LinkedHashSet<String> refs = new java.util.LinkedHashSet<>();
    refs.addAll(extractIdentifiers(actionBlock));
    if (reverseBlock != null) {
        refs.addAll(extractIdentifiers(reverseBlock));
    }
    refs.addAll(extractIdentifiers(resultExpression));

    return new GroupByCustomAccumulateIR(source, initVars, actionBlock, reverseBlock,
            resultTypeName, resultBindName, resultExpression, List.copyOf(refs),
            groupKeyExpression, groupKeyBindName, groupKeyRefs);
}
```

- [ ] **Step 3: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
feat(#40): add buildGroupByKeywordItem visitor method

Refs #40
```

---

### Task 5: Runtime class — create `DrlxGroupByAccumulate`

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxGroupByAccumulate.java`

- [ ] **Step 1: Create the `DrlxGroupByAccumulate` class**

```java
package org.drools.drlx.builder;

import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;
import java.util.Map;
import java.util.function.Function;

import org.drools.base.base.ValueResolver;
import org.drools.base.reteoo.AccumulateContextEntry;
import org.drools.base.reteoo.BaseTuple;
import org.drools.base.rule.Accumulate;
import org.drools.base.rule.Declaration;
import org.drools.base.rule.GroupElement;
import org.drools.base.rule.RuleConditionElement;
import org.drools.base.rule.accessor.Accumulator;
import org.drools.core.common.ReteEvaluator;
import org.drools.core.reteoo.AccumulateNode.GroupByContext;
import org.drools.core.reteoo.TupleImpl;
import org.drools.core.util.index.TupleListWithContext;
import org.kie.api.runtime.rule.FactHandle;

public class DrlxGroupByAccumulate extends Accumulate {

    private Accumulate innerAccumulate;
    private Function<Object, Object> groupingFunction;
    private DrlxValueExtractor groupingFunctionMulti;
    private boolean multiSource;

    public DrlxGroupByAccumulate() {}

    public DrlxGroupByAccumulate(Accumulate innerAccumulate,
                                  Function<Object, Object> groupingFunction) {
        super(innerAccumulate.getSource(), innerAccumulate.getRequiredDeclarations());
        this.innerAccumulate = innerAccumulate;
        this.groupingFunction = groupingFunction;
        this.groupingFunctionMulti = null;
        this.multiSource = false;
    }

    public DrlxGroupByAccumulate(Accumulate innerAccumulate,
                                  DrlxValueExtractor groupingFunctionMulti) {
        super(innerAccumulate.getSource(), innerAccumulate.getRequiredDeclarations());
        this.innerAccumulate = innerAccumulate;
        this.groupingFunction = null;
        this.groupingFunctionMulti = groupingFunctionMulti;
        this.multiSource = true;
    }

    private Object getKey(BaseTuple tuple, FactHandle handle, ValueResolver valueResolver) {
        if (multiSource) {
            Declaration[] innerDecls = innerAccumulate.getInnerDeclarationCache();
            java.util.HashMap<String, Object> bindings = new java.util.HashMap<>(innerDecls.length);
            for (Declaration d : innerDecls) {
                bindings.put(d.getIdentifier(), d.getValue(valueResolver, tuple));
            }
            return groupingFunctionMulti.applyMulti(bindings);
        }
        return groupingFunction.apply(handle.getObject());
    }

    @Override
    public boolean isGroupBy() {
        return true;
    }

    @Override
    public Accumulator[] getAccumulators() {
        return innerAccumulate.getAccumulators();
    }

    @Override
    public Object createFunctionContext() {
        return innerAccumulate.createFunctionContext();
    }

    @Override
    public Object init(Object workingMemoryContext, Object accContext,
                       Object funcContext, BaseTuple leftTuple, ValueResolver valueResolver) {
        return funcContext;
    }

    @Override
    public Object accumulate(Object workingMemoryContext, Object context,
                             BaseTuple match, FactHandle handle,
                             ValueResolver valueResolver) {
        GroupByContext groupByContext = (GroupByContext) context;
        TupleImpl leftTupleMatch = (TupleImpl) match;
        TupleListWithContext<AccumulateContextEntry> tupleList =
                groupByContext.getGroup(workingMemoryContext, innerAccumulate,
                        leftTupleMatch,
                        getKey(leftTupleMatch, handle, valueResolver),
                        (ReteEvaluator) valueResolver);
        return accumulate(workingMemoryContext, match, handle, groupByContext, tupleList, valueResolver);
    }

    @Override
    public Object accumulate(Object workingMemoryContext, BaseTuple match,
                             FactHandle childHandle, Object groupByContext,
                             Object tupleList, ValueResolver valueResolver) {
        @SuppressWarnings("unchecked")
        TupleListWithContext<AccumulateContextEntry> list =
                (TupleListWithContext<AccumulateContextEntry>) tupleList;
        ((GroupByContext) groupByContext).moveToPropagateTupleList(list);
        return innerAccumulate.accumulate(workingMemoryContext, list.getContext(),
                match, childHandle, valueResolver);
    }

    @Override
    public boolean tryReverse(Object workingMemoryContext, Object context,
                              BaseTuple leftTuple, FactHandle handle,
                              BaseTuple match, ValueResolver valueResolver) {
        TupleImpl tupleMatch = (TupleImpl) match;
        @SuppressWarnings("unchecked")
        TupleListWithContext<AccumulateContextEntry> memory =
                (TupleListWithContext<AccumulateContextEntry>) tupleMatch.getMemory();
        AccumulateContextEntry entry = memory.getContext();
        boolean reversed = innerAccumulate.tryReverse(workingMemoryContext, entry,
                leftTuple, handle, match, valueResolver);
        if (reversed) {
            GroupByContext groupByContext = (GroupByContext) context;
            groupByContext.moveToPropagateTupleList(memory);
            memory.remove(tupleMatch);
            if (memory.isEmpty()) {
                groupByContext.removeGroup(entry.getKey());
                memory.getContext().setEmpty(true);
            }
        }
        return reversed;
    }

    @Override
    public Object getResult(Object workingMemoryContext, Object context,
                            BaseTuple leftTuple, ValueResolver valueResolver) {
        AccumulateContextEntry entry = (AccumulateContextEntry) context;
        return entry.isEmpty() ? null :
                innerAccumulate.getResult(workingMemoryContext, context, leftTuple, valueResolver);
    }

    @Override
    public boolean supportsReverse() {
        return innerAccumulate.supportsReverse();
    }

    @Override
    public Accumulate clone() {
        if (multiSource) {
            return new DrlxGroupByAccumulate(innerAccumulate.clone(), groupingFunctionMulti);
        }
        return new DrlxGroupByAccumulate(innerAccumulate.clone(), groupingFunction);
    }

    @Override
    public Object createWorkingMemoryContext() {
        return innerAccumulate.createWorkingMemoryContext();
    }

    @Override
    public boolean isMultiFunction() {
        return innerAccumulate.isMultiFunction();
    }

    @Override
    public void replaceAccumulatorDeclaration(Declaration declaration, Declaration resolved) {
        innerAccumulate.replaceAccumulatorDeclaration(declaration, resolved);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        super.readExternal(in);
        this.innerAccumulate = (Accumulate) in.readObject();
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        super.writeExternal(out);
        out.writeObject(innerAccumulate);
    }
}
```

- [ ] **Step 2: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```
feat(#40): create DrlxGroupByAccumulate runtime class

Refs #40
```

---

### Task 6: Runtime builder — wire groupBy into `DrlxRuleAstRuntimeBuilder`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Add dispatch in `collectPatternClasses`**

In `collectPatternClasses` (around line 213), after the `CustomAccumulateIR` block, add:

```java
} else if (item instanceof GroupByAccumulateIR gbAcc) {
    if (gbAcc.source() instanceof PatternIR p) {
        classes.add(resolvePatternType(p, typeResolver, entryPointTypes, unitClass));
    } else if (gbAcc.source() instanceof GroupElementIR g) {
        collectPatternClasses(g.children(), classes, typeResolver, entryPointTypes, unitClass);
    }
} else if (item instanceof GroupByCustomAccumulateIR gbCustom) {
    if (gbCustom.source() instanceof PatternIR p) {
        classes.add(resolvePatternType(p, typeResolver, entryPointTypes, unitClass));
    } else if (gbCustom.source() instanceof GroupElementIR g) {
        collectPatternClasses(g.children(), classes, typeResolver, entryPointTypes, unitClass);
    }
```

- [ ] **Step 2: Add dispatch in `buildLhs`**

In `buildLhs` (around line 635), after the `CustomAccumulateIR` block, add:

```java
} else if (item instanceof GroupByAccumulateIR gbAcc) {
    buildGroupByAccumulatePattern(gbAcc, parent, typeResolver, entryPointTypes,
                                  unitClass, boundVariables, queryRegistry, currentQuery);
} else if (item instanceof GroupByCustomAccumulateIR gbCustom) {
    buildGroupByCustomAccumulatePattern(gbCustom, parent, typeResolver, entryPointTypes,
                                        unitClass, boundVariables, queryRegistry, currentQuery);
```

- [ ] **Step 3: Add `buildGroupByAccumulatePattern` method**

After `buildCustomAccumulatePattern`, add the method for built-in function groupBy. This method builds the inner accumulate (SingleAccumulate or MultiAccumulate), compiles the grouping key, wraps in DrlxGroupByAccumulate, and creates the Object[] result pattern:

```java
private void buildGroupByAccumulatePattern(GroupByAccumulateIR gbAcc,
                                            GroupElement parent,
                                            TypeResolver typeResolver,
                                            Map<String, Class<?>> entryPointTypes,
                                            Class<?> unitClass,
                                            Map<String, BoundVariable> outerScope,
                                            Map<String, QueryImpl> queryRegistry,
                                            QueryImpl currentQuery) {
    LhsItemIR srcIr = gbAcc.source();
    Map<String, BoundVariable> innerScope = new java.util.LinkedHashMap<>(outerScope);
    org.drools.base.rule.RuleConditionElement srcElement;
    boolean multiSource;

    if (srcIr instanceof GroupElementIR groupIr
            && groupIr.children().size() == 1
            && groupIr.children().get(0) instanceof PatternIR) {
        srcIr = groupIr.children().get(0);
    }

    if (srcIr instanceof PatternIR patIr) {
        Pattern srcPattern = buildPattern(patIr, typeResolver, entryPointTypes, unitClass, outerScope);
        srcElement = srcPattern;
        multiSource = false;
        Declaration srcDecl = srcPattern.getDeclaration();
        if (srcDecl != null) {
            Class<?> srcClass = ((ClassObjectType) srcPattern.getObjectType()).getClassType();
            innerScope.put(srcDecl.getIdentifier(),
                    new BoundVariable(srcDecl.getIdentifier(), srcClass, srcPattern, srcDecl));
        }
    } else if (srcIr instanceof GroupElementIR groupIr) {
        GroupElement andGroup = GroupElementFactory.newAndInstance();
        buildLhs(groupIr.children(), andGroup, typeResolver, entryPointTypes,
                 unitClass, innerScope, queryRegistry, currentQuery);
        srcElement = andGroup;
        multiSource = true;
    } else {
        throw new IllegalArgumentException("Unsupported groupBy source: " + srcIr.getClass());
    }

    List<AccumulatorIR> accumulators = gbAcc.accumulators();
    int n = accumulators.size();

    Map<String, BoundVariable> sourceScope = null;
    if (multiSource) {
        sourceScope = new java.util.LinkedHashMap<>();
        for (Map.Entry<String, BoundVariable> e : innerScope.entrySet()) {
            if (!outerScope.containsKey(e.getKey())) {
                sourceScope.put(e.getKey(), e.getValue());
            }
        }
    }

    org.drools.base.rule.accessor.Accumulator[] accs =
            new org.drools.base.rule.accessor.Accumulator[n];
    for (int i = 0; i < n; i++) {
        if (multiSource) {
            accs[i] = buildSingleAccumulatorMulti(accumulators.get(i), sourceScope, typeResolver);
        } else {
            Declaration srcDecl = ((Pattern) srcElement).getDeclaration();
            Class<?> srcClass = ((ClassObjectType) ((Pattern) srcElement).getObjectType()).getClassType();
            String srcBindingName = srcDecl != null ? srcDecl.getIdentifier() : null;
            accs[i] = buildSingleAccumulator(accumulators.get(i), srcClass, srcBindingName, typeResolver);
        }
    }

    // Build inner accumulate
    Accumulate innerAccumulate;
    if (n == 1) {
        Declaration[] required = multiSource
                ? new Declaration[0]
                : requiredFor(accumulators.get(0), innerScope);
        innerAccumulate = new SingleAccumulate(srcElement, required, accs[0]);
    } else {
        innerAccumulate = new MultiAccumulate(srcElement, new Declaration[0], accs, n + 1);
    }

    // Compile grouping key
    DrlxGroupByAccumulate groupByAccumulate;
    if (multiSource) {
        DrlxValueExtractor keyExtractor = lambdaCompiler.createValueExtractor(
                gbAcc.groupKeyExpression(), sourceScope);
        groupByAccumulate = new DrlxGroupByAccumulate(innerAccumulate, keyExtractor);
    } else {
        Class<?> srcClass = ((ClassObjectType) ((Pattern) srcElement).getObjectType()).getClassType();
        String srcBindingName = ((Pattern) srcElement).getDeclaration().getIdentifier();
        DrlxValueExtractor keyExtractor = lambdaCompiler.createValueExtractor(
                gbAcc.groupKeyExpression(), srcClass, srcBindingName);
        groupByAccumulate = new DrlxGroupByAccumulate(innerAccumulate, keyExtractor);
    }

    // Build Object[] result pattern
    ReadAccessor selfReader = new SelfReferenceClassFieldReader(Object[].class);
    Pattern wrap = new Pattern(0, new ClassObjectType(Object[].class));

    if (n == 1) {
        Class<?> rType = resultClassFor(accumulators.get(0), typeResolver);
        wrap.addDeclaration(new Declaration(
                accumulators.get(0).resultBindName(),
                new ArrayElementReader(selfReader, 0, rType),
                wrap, true));
    } else {
        for (int i = 0; i < n; i++) {
            Class<?> rType = resultClassFor(accumulators.get(i), typeResolver);
            wrap.addDeclaration(new Declaration(
                    accumulators.get(i).resultBindName(),
                    new ArrayElementReader(selfReader, i, rType),
                    wrap, true));
        }
    }

    // Add group key declaration if bound
    int keyIndex = (n == 1) ? 1 : n;
    if (gbAcc.groupKeyBindName() != null) {
        wrap.addDeclaration(new Declaration(
                gbAcc.groupKeyBindName(),
                new ArrayElementReader(selfReader, keyIndex, Object.class),
                wrap, true));
    }

    wrap.setSource(groupByAccumulate);
    parent.addChild(wrap);

    // Register result bindings to outer scope
    for (int i = 0; i < n; i++) {
        AccumulatorIR acc = accumulators.get(i);
        Class<?> resultClass = resultClassFor(acc, typeResolver);
        Declaration decl = wrap.getDeclarations().get(acc.resultBindName());
        outerScope.put(acc.resultBindName(),
                new BoundVariable(acc.resultBindName(), resultClass, wrap, decl));
    }
    if (gbAcc.groupKeyBindName() != null) {
        Declaration keyDecl = wrap.getDeclarations().get(gbAcc.groupKeyBindName());
        outerScope.put(gbAcc.groupKeyBindName(),
                new BoundVariable(gbAcc.groupKeyBindName(), Object.class, wrap, keyDecl));
    }
}
```

- [ ] **Step 4: Add `buildGroupByCustomAccumulatePattern` method**

After `buildGroupByAccumulatePattern`, add the method for custom accumulator groupBy:

```java
private void buildGroupByCustomAccumulatePattern(GroupByCustomAccumulateIR gbCustom,
                                                   GroupElement parent,
                                                   TypeResolver typeResolver,
                                                   Map<String, Class<?>> entryPointTypes,
                                                   Class<?> unitClass,
                                                   Map<String, BoundVariable> outerScope,
                                                   Map<String, QueryImpl> queryRegistry,
                                                   QueryImpl currentQuery) {
    LhsItemIR srcIr = gbCustom.source();

    if (srcIr instanceof GroupElementIR groupIr
            && groupIr.children().size() == 1
            && groupIr.children().get(0) instanceof PatternIR) {
        srcIr = groupIr.children().get(0);
    }

    Map<String, BoundVariable> innerScope = new java.util.LinkedHashMap<>(outerScope);
    org.drools.base.rule.RuleConditionElement srcElement;
    boolean multiSource;

    if (srcIr instanceof PatternIR patIr) {
        Pattern srcPattern = buildPattern(patIr, typeResolver, entryPointTypes, unitClass, outerScope);
        srcElement = srcPattern;
        multiSource = false;
        Declaration srcDecl = srcPattern.getDeclaration();
        if (srcDecl != null) {
            Class<?> srcClass = ((ClassObjectType) srcPattern.getObjectType()).getClassType();
            innerScope.put(srcDecl.getIdentifier(),
                    new BoundVariable(srcDecl.getIdentifier(), srcClass, srcPattern, srcDecl));
        }
    } else if (srcIr instanceof GroupElementIR groupIr) {
        GroupElement andGroup = GroupElementFactory.newAndInstance();
        buildLhs(groupIr.children(), andGroup, typeResolver, entryPointTypes,
                 unitClass, innerScope, queryRegistry, currentQuery);
        srcElement = andGroup;
        multiSource = true;
    } else {
        throw new IllegalArgumentException("Unsupported groupBy source: " + srcIr.getClass());
    }

    // Build custom accumulator (same as buildCustomAccumulatePattern)
    DrlxCustomAccumulator accumulator;
    if (multiSource) {
        Map<String, BoundVariable> sourceScope = new java.util.LinkedHashMap<>();
        for (Map.Entry<String, BoundVariable> e : innerScope.entrySet()) {
            if (!outerScope.containsKey(e.getKey())) {
                sourceScope.put(e.getKey(), e.getValue());
            }
        }
        accumulator = lambdaCompiler.createCustomAccumulator(gbCustom, sourceScope);
    } else {
        Class<?> srcClass = ((ClassObjectType) ((Pattern) srcElement).getObjectType()).getClassType();
        String srcBindingName = ((Pattern) srcElement).getDeclaration() != null
                ? ((Pattern) srcElement).getDeclaration().getIdentifier() : null;
        accumulator = lambdaCompiler.createCustomAccumulator(gbCustom, srcClass, srcBindingName);
    }

    Declaration[] required = new Declaration[0];
    SingleAccumulate innerAccumulate = new SingleAccumulate(srcElement, required, accumulator);

    // Compile grouping key
    DrlxGroupByAccumulate groupByAccumulate;
    if (multiSource) {
        Map<String, BoundVariable> sourceScope = new java.util.LinkedHashMap<>();
        for (Map.Entry<String, BoundVariable> e : innerScope.entrySet()) {
            if (!outerScope.containsKey(e.getKey())) {
                sourceScope.put(e.getKey(), e.getValue());
            }
        }
        DrlxValueExtractor keyExtractor = lambdaCompiler.createValueExtractor(
                gbCustom.groupKeyExpression(), sourceScope);
        groupByAccumulate = new DrlxGroupByAccumulate(innerAccumulate, keyExtractor);
    } else {
        Class<?> srcClass = ((ClassObjectType) ((Pattern) srcElement).getObjectType()).getClassType();
        String srcBindingName = ((Pattern) srcElement).getDeclaration().getIdentifier();
        DrlxValueExtractor keyExtractor = lambdaCompiler.createValueExtractor(
                gbCustom.groupKeyExpression(), srcClass, srcBindingName);
        groupByAccumulate = new DrlxGroupByAccumulate(innerAccumulate, keyExtractor);
    }

    // Build Object[] result pattern — custom acc always has 1 result + key
    Class<?> resultClass = resolveCustomResultType(gbCustom.resultTypeName(), typeResolver);
    ReadAccessor selfReader = new SelfReferenceClassFieldReader(Object[].class);
    Pattern wrap = new Pattern(lambdaCompiler.nextPatternId(), new ClassObjectType(Object[].class));
    wrap.addDeclaration(new Declaration(gbCustom.resultBindName(),
            new ArrayElementReader(selfReader, 0, resultClass), wrap, true));

    if (gbCustom.groupKeyBindName() != null) {
        wrap.addDeclaration(new Declaration(gbCustom.groupKeyBindName(),
                new ArrayElementReader(selfReader, 1, Object.class), wrap, true));
    }

    wrap.setSource(groupByAccumulate);
    parent.addChild(wrap);

    Declaration decl = wrap.getDeclarations().get(gbCustom.resultBindName());
    outerScope.put(gbCustom.resultBindName(),
            new BoundVariable(gbCustom.resultBindName(), resultClass, wrap, decl));
    if (gbCustom.groupKeyBindName() != null) {
        Declaration keyDecl = wrap.getDeclarations().get(gbCustom.groupKeyBindName());
        outerScope.put(gbCustom.groupKeyBindName(),
                new BoundVariable(gbCustom.groupKeyBindName(), Object.class, wrap, keyDecl));
    }
}
```

- [ ] **Step 5: Add required imports**

Add these imports at the top of `DrlxRuleAstRuntimeBuilder.java` if not already present:

```java
import org.drools.drlx.builder.DrlxRuleAstModel.GroupByAccumulateIR;
import org.drools.drlx.builder.DrlxRuleAstModel.GroupByCustomAccumulateIR;
```

- [ ] **Step 6: Handle `createCustomAccumulator` for `GroupByCustomAccumulateIR`**

The `DrlxLambdaCompiler.createCustomAccumulator` currently accepts `CustomAccumulateIR`. Since `GroupByCustomAccumulateIR` has the same fields for the custom accumulator body, add overloads to `DrlxLambdaCompiler` that accept `GroupByCustomAccumulateIR`, or extract an interface. The simplest approach: create a helper in the runtime builder that converts the relevant fields:

In `buildGroupByCustomAccumulatePattern`, replace the calls to `lambdaCompiler.createCustomAccumulator(gbCustom, ...)` with:

```java
CustomAccumulateIR asCustom = new CustomAccumulateIR(
        gbCustom.source(), gbCustom.initVars(),
        gbCustom.actionBlock(), gbCustom.reverseBlock(),
        gbCustom.resultTypeName(), gbCustom.resultBindName(),
        gbCustom.resultExpression(), gbCustom.referencedBindings());

// Then use:
accumulator = lambdaCompiler.createCustomAccumulator(asCustom, sourceScope);
// or:
accumulator = lambdaCompiler.createCustomAccumulator(asCustom, srcClass, srcBindingName);
```

- [ ] **Step 7: Verify compilation**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`

Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
feat(#40): wire groupBy into DrlxRuleAstRuntimeBuilder

Refs #40
```

---

### Task 7: Runtime tests — single function groupBy

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/GroupByTest.java`

- [ ] **Step 1: Create test class with single-function groupBy test (bound key)**

```java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.domain.Person;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class GroupByTest extends DrlxBuilderTestSupport {

    @Test
    void singleFunctionBoundKey() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule R {
                    groupBy(var p : /persons,
                            var g = p.name,
                            var avgAge = avg(p.age)),
                    do { results.add(g); results.add(avgAge); }
                }
                """;

        withMyUnitInstance(rule, (instance, unit, listener) -> {
            unit.persons.add(new Person("Alice", 20));
            unit.persons.add(new Person("Alice", 40));
            unit.persons.add(new Person("Bob", 30));
            instance.fire();
            // Two groups: Alice (avg 30.0) and Bob (avg 30.0)
            assertThat(unit.results).hasSize(4);
            assertThat(unit.results).contains("Alice", "Bob", 30.0);
        });
    }
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=GroupByTest#singleFunctionBoundKey -q`

Expected: PASS

- [ ] **Step 3: Add unbound key test**

```java
@Test
void singleFunctionUnboundKey() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                groupBy(var p : /persons,
                        p.name,
                        var avgAge = avg(p.age)),
                do { results.add(avgAge); }
            }
            """;

    withMyUnitInstance(rule, (instance, unit, listener) -> {
        unit.persons.add(new Person("Alice", 20));
        unit.persons.add(new Person("Alice", 40));
        unit.persons.add(new Person("Bob", 30));
        instance.fire();
        assertThat(unit.results).hasSize(2);
        assertThat(unit.results).containsExactlyInAnyOrder(30.0, 30.0);
    });
}
```

- [ ] **Step 4: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=GroupByTest#singleFunctionUnboundKey -q`

Expected: PASS

- [ ] **Step 5: Commit**

```
test(#40): single-function groupBy runtime tests

Refs #40
```

---

### Task 8: Runtime tests — multi-function, multi-pattern, and custom groupBy

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/GroupByTest.java`

- [ ] **Step 1: Add multi-function groupBy test**

```java
@Test
void multiFunctionGroupBy() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                groupBy(var p : /persons,
                        var g = p.name,
                        (var minAge = min(p.age),
                         var maxAge = max(p.age))),
                do { results.add(g); results.add(minAge); results.add(maxAge); }
            }
            """;

    withMyUnitInstance(rule, (instance, unit, listener) -> {
        unit.persons.add(new Person("Alice", 20));
        unit.persons.add(new Person("Alice", 40));
        unit.persons.add(new Person("Bob", 30));
        instance.fire();
        assertThat(unit.results).hasSize(6);
        assertThat(unit.results).contains("Alice", 20, 40, "Bob", 30, 30);
    });
}
```

- [ ] **Step 2: Add multi-pattern source test**

```java
@Test
void multiPatternSource() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.domain.Order;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                groupBy(and(var p : /persons, var o : /orders[customerId == p.age]),
                        p.name,
                        var total = sum(o.amount)),
                do { results.add(total); }
            }
            """;

    withMyUnitInstance(rule, (instance, unit, listener) -> {
        unit.persons.add(new Person("Alice", 1));
        unit.persons.add(new Person("Bob", 2));
        unit.orders.add(new Order("O1", 1, 100));
        unit.orders.add(new Order("O2", 1, 200));
        unit.orders.add(new Order("O3", 2, 50));
        instance.fire();
        // Alice group: sum(100+200)=300, Bob group: sum(50)=50
        assertThat(unit.results).hasSize(2);
        assertThat(unit.results).containsExactlyInAnyOrder(300.0, 50.0);
    });
}
```

- [ ] **Step 3: Add custom accumulator groupBy test**

```java
@Test
void customAccumulator3Param() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                groupBy(var p : /persons,
                        var g = p.name,
                        int s = 0;,
                        s = s + p.age,
                        int total = s),
                do { results.add(g); results.add(total); }
            }
            """;

    withMyUnitInstance(rule, (instance, unit, listener) -> {
        unit.persons.add(new Person("Alice", 20));
        unit.persons.add(new Person("Alice", 40));
        unit.persons.add(new Person("Bob", 30));
        instance.fire();
        assertThat(unit.results).hasSize(4);
        assertThat(unit.results).contains("Alice", 60, "Bob", 30);
    });
}
```

- [ ] **Step 4: Add custom accumulator with reverse test**

```java
@Test
void customAccumulator5ParamWithRetraction() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                groupBy(var p : /persons,
                        var g = p.name,
                        int s = 0;,
                        (s = s + p.age, s = s - p.age),
                        int total = s),
                do { results.add(g); results.add(total); }
            }
            """;

    withMyUnitInstance(rule, (instance, unit, listener) -> {
        org.drools.ruleunits.api.DataHandle h1 = unit.persons.add(new Person("Alice", 20));
        unit.persons.add(new Person("Alice", 40));
        unit.persons.add(new Person("Bob", 30));
        instance.fire();

        unit.results.clear();
        unit.persons.remove(h1);
        instance.fire();
        // Alice group: only age=40 remains → total=40
        // Bob group: unchanged → total=30
        assertThat(unit.results).hasSize(4);
        assertThat(unit.results).contains("Alice", 40, "Bob", 30);
    });
}
```

- [ ] **Step 5: Run all GroupByTest tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=GroupByTest -q`

Expected: All PASS

- [ ] **Step 6: Commit**

```
test(#40): multi-function, multi-pattern, and custom groupBy tests

Refs #40
```

---

### Task 9: AST structure tests and full test suite

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/GroupByTest.java`

- [ ] **Step 1: Add AST structure test for single-function groupBy**

```java
@Test
void singleFunctionEmitsGroupByAccumulate() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            rule R {
                groupBy(var p : /persons,
                        var g = p.name,
                        var avgAge = avg(p.age)),
                do {}
            }
            """;
    final org.kie.api.KieBase kieBase = new org.drools.drlx.builder.DrlxRuleBuilder().build(rule);
    org.drools.base.definitions.rule.impl.RuleImpl impl =
            (org.drools.base.definitions.rule.impl.RuleImpl) kieBase
                    .getKiePackage("org.drools.drlx.parser")
                    .getRules().stream()
                    .filter(r -> r.getName().equals("R"))
                    .findFirst().orElseThrow();

    org.drools.base.rule.Pattern wrap = impl.getLhs().getChildren().stream()
            .filter(org.drools.base.rule.Pattern.class::isInstance)
            .map(org.drools.base.rule.Pattern.class::cast)
            .filter(p -> p.getSource() instanceof org.drools.base.rule.Accumulate)
            .findFirst().orElseThrow();

    assertThat(wrap.getSource()).isInstanceOf(DrlxGroupByAccumulate.class);
    assertThat(wrap.getSource().isGroupBy()).isTrue();
    assertThat(((org.drools.base.base.ClassObjectType) wrap.getObjectType()).getClassType())
            .isEqualTo(Object[].class);
    assertThat(wrap.getDeclarations()).containsKeys("avgAge", "g");

    org.drools.base.rule.Declaration avgDecl = wrap.getDeclarations().get("avgAge");
    assertThat(avgDecl.getExtractor())
            .isInstanceOf(org.drools.base.base.extractors.ArrayElementReader.class);

    org.drools.base.rule.Declaration gDecl = wrap.getDeclarations().get("g");
    assertThat(gDecl.getExtractor())
            .isInstanceOf(org.drools.base.base.extractors.ArrayElementReader.class);
}
```

- [ ] **Step 2: Run the test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=GroupByTest#singleFunctionEmitsGroupByAccumulate -q`

Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -q`

Expected: All tests PASS — no regressions in existing accumulate, acc keyword, or other tests.

- [ ] **Step 4: Commit**

```
test(#40): AST structure test and full suite verification

Refs #40
```

---

### Task 10: Install and final verification

- [ ] **Step 1: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -q`

Expected: BUILD SUCCESS

- [ ] **Step 2: Run full project tests (including benchmark module if applicable)**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -q`

Expected: All tests PASS

- [ ] **Step 3: Commit final state if any adjustments were needed**

If no changes needed, this step is a no-op.
