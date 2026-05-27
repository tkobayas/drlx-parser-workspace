# Compact Setter Block (`t{...}`) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add compact setter block syntax `t{prop = val, ...}` to MVEL3 as an expression that applies property assignments and returns the target.

**Architecture:** New grammar rule in Mvel3Parser.g4, new `CompactWithExpression` AST node in javaparser-mvel extending `Expression`, transpiler rewriting in `MVELToJavaRewriter` (sharing logic with existing `WithStatement`/`ModifyStatement` handling), and print visitor support.

**Tech Stack:** ANTLR4 (grammar), javaparser-mvel (AST), MVEL3 (transpiler), JUnit 5 + AssertJ (tests)

**Repositories:**
- javaparser-mvel: `/home/tkobayas/usr/work/mvel3-development/javaparser-mvel` (branch `compact-with`)
- MVEL3: `/home/tkobayas/usr/work/mvel3-development/mvel` (branch `compact-with`)

**Build commands:**
- javaparser-mvel: `mvn -f /home/tkobayas/usr/work/mvel3-development/javaparser-mvel/pom.xml -pl javaparser-core install -DskipTests`
- MVEL3: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml install -DskipTests`
- MVEL3 tests: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test`

---

### Task 1: Create `CompactWithExpression` AST node in javaparser-mvel

**Files:**
- Create: `javaparser-mvel/javaparser-core/src/main/java/org/mvel3/parser/ast/expr/CompactWithExpression.java`
- Create: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/metamodel/CompactWithExpressionMetaModel.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/GenericVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/VoidVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/GenericVisitorAdapter.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/VoidVisitorAdapter.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/GenericVisitorWithDefaults.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/VoidVisitorWithDefaults.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/GenericListVisitorAdapter.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/CloneVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/ModifierVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/HashCodeVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/EqualsVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/NoCommentEqualsVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/NoCommentHashCodeVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/ObjectIdentityEqualsVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/ast/visitor/ObjectIdentityHashCodeVisitor.java`
- Modify: `javaparser-mvel/javaparser-core/src/main/java/com/github/javaparser/metamodel/JavaParserMetaModel.java`

All paths below are relative to `/home/tkobayas/usr/work/mvel3-development/javaparser-mvel`.

- [ ] **Step 1: Create `CompactWithExpression.java`**

Create `javaparser-core/src/main/java/org/mvel3/parser/ast/expr/CompactWithExpression.java`:

```java
package org.mvel3.parser.ast.expr;

import com.github.javaparser.TokenRange;
import com.github.javaparser.ast.AllFieldsConstructor;
import com.github.javaparser.ast.Generated;
import com.github.javaparser.ast.Node;
import com.github.javaparser.ast.NodeList;
import com.github.javaparser.ast.expr.AssignExpr;
import com.github.javaparser.ast.expr.Expression;
import com.github.javaparser.ast.expr.NameExpr;
import com.github.javaparser.ast.observer.ObservableProperty;
import com.github.javaparser.ast.visitor.CloneVisitor;
import com.github.javaparser.ast.visitor.GenericVisitor;
import com.github.javaparser.ast.visitor.VoidVisitor;
import com.github.javaparser.metamodel.CompactWithExpressionMetaModel;
import com.github.javaparser.metamodel.JavaParserMetaModel;

import java.util.Optional;
import java.util.function.Consumer;

import static com.github.javaparser.utils.Utils.assertNotNull;

public class CompactWithExpression extends Expression {

    private NameExpr target;

    private NodeList<AssignExpr> assignments;

    @AllFieldsConstructor
    public CompactWithExpression(NameExpr target, NodeList<AssignExpr> assignments) {
        this(null, target, assignments);
    }

    @Generated("com.github.javaparser.generator.core.node.MainConstructorGenerator")
    public CompactWithExpression(TokenRange tokenRange, NameExpr target, NodeList<AssignExpr> assignments) {
        super(tokenRange);
        setTarget(target);
        setAssignments(assignments);
        customInitialization();
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.AcceptGenerator")
    public <R, A> R accept(final GenericVisitor<R, A> v, final A arg) {
        return v.visit(this, arg);
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.AcceptGenerator")
    public <A> void accept(final VoidVisitor<A> v, final A arg) {
        v.visit(this, arg);
    }

    @Generated("com.github.javaparser.generator.core.node.PropertyGenerator")
    public NameExpr getTarget() {
        return target;
    }

    @Generated("com.github.javaparser.generator.core.node.PropertyGenerator")
    public CompactWithExpression setTarget(final NameExpr target) {
        assertNotNull(target);
        if (target == this.target) {
            return this;
        }
        notifyPropertyChange(ObservableProperty.TARGET, this.target, target);
        if (this.target != null)
            this.target.setParentNode(null);
        this.target = target;
        setAsParentNodeOf(target);
        return this;
    }

    @Generated("com.github.javaparser.generator.core.node.PropertyGenerator")
    public NodeList<AssignExpr> getAssignments() {
        return assignments;
    }

    @Generated("com.github.javaparser.generator.core.node.PropertyGenerator")
    public CompactWithExpression setAssignments(final NodeList<AssignExpr> assignments) {
        assertNotNull(assignments);
        if (assignments == this.assignments) {
            return this;
        }
        notifyPropertyChange(ObservableProperty.EXPRESSIONS, this.assignments, assignments);
        if (this.assignments != null)
            this.assignments.setParentNode(null);
        this.assignments = assignments;
        setAsParentNodeOf(assignments);
        return this;
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.TypeCastingGenerator")
    public boolean isCompactWithExpression() {
        return true;
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.TypeCastingGenerator")
    public CompactWithExpression asCompactWithExpression() {
        return this;
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.TypeCastingGenerator")
    public Optional<CompactWithExpression> toCompactWithExpression() {
        return Optional.of(this);
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.TypeCastingGenerator")
    public void ifCompactWithExpression(Consumer<CompactWithExpression> action) {
        action.accept(this);
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.RemoveMethodGenerator")
    public boolean remove(Node node) {
        if (node == null) {
            return false;
        }
        for (int i = 0; i < assignments.size(); i++) {
            if (assignments.get(i) == node) {
                assignments.remove(i);
                return true;
            }
        }
        return super.remove(node);
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.ReplaceMethodGenerator")
    public boolean replace(Node node, Node replacementNode) {
        if (node == null) {
            return false;
        }
        for (int i = 0; i < assignments.size(); i++) {
            if (assignments.get(i) == node) {
                assignments.set(i, (AssignExpr) replacementNode);
                return true;
            }
        }
        if (node == target) {
            setTarget((NameExpr) replacementNode);
            return true;
        }
        return super.replace(node, replacementNode);
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.CloneGenerator")
    public CompactWithExpression clone() {
        return (CompactWithExpression) accept(new CloneVisitor(), null);
    }

    @Override
    @Generated("com.github.javaparser.generator.core.node.GetMetaModelGenerator")
    public CompactWithExpressionMetaModel getMetaModel() {
        return JavaParserMetaModel.compactWithExpressionMetaModel;
    }
}
```

- [ ] **Step 2: Create `CompactWithExpressionMetaModel.java`**

Create `javaparser-core/src/main/java/com/github/javaparser/metamodel/CompactWithExpressionMetaModel.java`:

```java
package com.github.javaparser.metamodel;

import java.util.Optional;
import org.mvel3.parser.ast.expr.CompactWithExpression;
import com.github.javaparser.ast.Generated;
import com.github.javaparser.ast.Node;

@Generated("com.github.javaparser.generator.metamodel.NodeMetaModelGenerator")
public class CompactWithExpressionMetaModel extends ExpressionMetaModel {

    @Generated("com.github.javaparser.generator.metamodel.NodeMetaModelGenerator")
    CompactWithExpressionMetaModel(Optional<BaseNodeMetaModel> superBaseNodeMetaModel) {
        super(superBaseNodeMetaModel, CompactWithExpression.class, "CompactWithExpression", "org.mvel3.parser.ast.expr", false, false);
    }

    @Generated("com.github.javaparser.generator.metamodel.NodeMetaModelGenerator")
    protected CompactWithExpressionMetaModel(Optional<BaseNodeMetaModel> superNodeMetaModel, Class<? extends Node> type, String name, String packageName, boolean isAbstract, boolean hasWildcard) {
        super(superNodeMetaModel, type, name, packageName, isAbstract, hasWildcard);
    }

    public PropertyMetaModel targetPropertyMetaModel;

    public PropertyMetaModel assignmentsPropertyMetaModel;
}
```

- [ ] **Step 3: Add visitor methods to all visitor interfaces and adapters**

Add to `GenericVisitor.java` (after the `visit(WithStatement)` line):
```java
R visit(CompactWithExpression n, A arg);
```
Add import: `import org.mvel3.parser.ast.expr.CompactWithExpression;`

Add to `VoidVisitor.java` (after the `visit(WithStatement)` line):
```java
void visit(CompactWithExpression n, A arg);
```
Add import: `import org.mvel3.parser.ast.expr.CompactWithExpression;`

Add to `GenericVisitorAdapter.java`:
```java
@Override
public R visit(final CompactWithExpression n, final A arg) {
    R result;
    {
        result = n.getAssignments().accept(this, arg);
        if (result != null)
            return result;
    }
    {
        result = n.getTarget().accept(this, arg);
        if (result != null)
            return result;
    }
    if (n.getComment().isPresent()) {
        result = n.getComment().get().accept(this, arg);
        if (result != null)
            return result;
    }
    return null;
}
```

Add to `VoidVisitorAdapter.java`:
```java
@Override
public void visit(final CompactWithExpression n, final A arg) {
    n.getAssignments().forEach(p -> p.accept(this, arg));
    n.getTarget().accept(this, arg);
    n.getComment().ifPresent(l -> l.accept(this, arg));
}
```

Add to `GenericVisitorWithDefaults.java`:
```java
@Override
public R visit(final CompactWithExpression n, final A arg) {
    return defaultAction(n, arg);
}
```

Add to `VoidVisitorWithDefaults.java`:
```java
@Override
public void visit(final CompactWithExpression n, final A arg) {
    defaultAction(n, arg);
}
```

Add to `GenericListVisitorAdapter.java`:
```java
public List<R> visit(final CompactWithExpression n, final A arg) {
    List<R> result = new ArrayList<>();
    List<R> tmp;
    {
        tmp = n.getAssignments().accept(this, arg);
        if (tmp != null)
            result.addAll(tmp);
    }
    {
        tmp = n.getTarget().accept(this, arg);
        if (tmp != null)
            result.addAll(tmp);
    }
    {
        tmp = n.getComment().accept(this, arg);
        if (tmp != null)
            result.addAll(tmp);
    }
    return result;
}
```

Add to `CloneVisitor.java`:
```java
public Visitable visit(final CompactWithExpression n, final Object arg) {
    NodeList<AssignExpr> assignments = cloneList(n.getAssignments(), arg);
    NameExpr target = cloneNode(n.getTarget(), arg);
    Comment comment = cloneNode(n.getComment(), arg);
    CompactWithExpression r = new CompactWithExpression(n.getTokenRange().orElse(null), target, assignments);
    r.setComment(comment);
    n.getOrphanComments().stream().map(Comment::clone).forEach(r::addOrphanComment);
    copyData(n, r);
    return r;
}
```

Add to `ModifierVisitor.java`:
```java
public Visitable visit(final CompactWithExpression n, final A arg) {
    NodeList<AssignExpr> assignments = modifyList(n.getAssignments(), arg);
    NameExpr target = (NameExpr) n.getTarget().accept(this, arg);
    Comment comment = n.getComment().map(s -> (Comment) s.accept(this, arg)).orElse(null);
    if (target == null)
        return null;
    n.setAssignments(assignments);
    n.setTarget(target);
    n.setComment(comment);
    return n;
}
```

Add to `HashCodeVisitor.java`:
```java
public Integer visit(final CompactWithExpression n, final Void arg) {
    return (n.getAssignments().accept(this, arg)) * 31 + (n.getTarget().accept(this, arg)) * 31 + (n.getComment().isPresent() ? n.getComment().get().accept(this, arg) : 0);
}
```

Add to `EqualsVisitor.java`:
```java
public Boolean visit(final CompactWithExpression n, final Visitable arg) {
    final CompactWithExpression n2 = (CompactWithExpression) arg;
    if (!nodesEquals(n.getAssignments(), n2.getAssignments()))
        return false;
    if (!nodeEquals(n.getTarget(), n2.getTarget()))
        return false;
    if (!nodeEquals(n.getComment(), n2.getComment()))
        return false;
    return true;
}
```

Add to `NoCommentEqualsVisitor.java`:
```java
public Boolean visit(final CompactWithExpression n, final Visitable arg) {
    final CompactWithExpression n2 = (CompactWithExpression) arg;
    if (!nodesEquals(n.getAssignments(), n2.getAssignments()))
        return false;
    if (!nodeEquals(n.getTarget(), n2.getTarget()))
        return false;
    return true;
}
```

Add to `NoCommentHashCodeVisitor.java`:
```java
public Integer visit(final CompactWithExpression n, final Void arg) {
    return (n.getAssignments().accept(this, arg)) * 31 + (n.getTarget().accept(this, arg));
}
```

Add to `ObjectIdentityEqualsVisitor.java`:
```java
public Boolean visit(final CompactWithExpression n, final Visitable arg) {
    return n == arg;
}
```

Add to `ObjectIdentityHashCodeVisitor.java`:
```java
public Integer visit(final CompactWithExpression n, final Void arg) {
    return n.hashCode();
}
```

Add import `import org.mvel3.parser.ast.expr.CompactWithExpression;` to each file.
For `CloneVisitor.java`, also add `import com.github.javaparser.ast.expr.AssignExpr;` and `import com.github.javaparser.ast.expr.NameExpr;`.

- [ ] **Step 4: Register metamodel in `JavaParserMetaModel.java`**

In `javaparser-core/src/main/java/com/github/javaparser/metamodel/JavaParserMetaModel.java`:

Add static field declaration (near the other MVEL metamodel fields, around line 1436):
```java
public static final CompactWithExpressionMetaModel compactWithExpressionMetaModel = new CompactWithExpressionMetaModel(Optional.of(expressionMetaModel));
```

Add to the `nodeMetaModels` list (near line 429):
```java
nodeMetaModels.add(compactWithExpressionMetaModel);
```

Add constructor parameter registration (near line 320):
```java
compactWithExpressionMetaModel.getConstructorParameters().add(compactWithExpressionMetaModel.targetPropertyMetaModel);
compactWithExpressionMetaModel.getConstructorParameters().add(compactWithExpressionMetaModel.assignmentsPropertyMetaModel);
```

Add property meta model initialization (near line 991):
```java
compactWithExpressionMetaModel.targetPropertyMetaModel = new PropertyMetaModel(compactWithExpressionMetaModel, "target", com.github.javaparser.ast.expr.NameExpr.class, Optional.of(nameExprMetaModel), false, false, false, false);
compactWithExpressionMetaModel.getDeclaredPropertyMetaModels().add(compactWithExpressionMetaModel.targetPropertyMetaModel);
compactWithExpressionMetaModel.assignmentsPropertyMetaModel = new PropertyMetaModel(compactWithExpressionMetaModel, "assignments", com.github.javaparser.ast.expr.AssignExpr.class, Optional.of(assignExprMetaModel), false, false, true, false);
compactWithExpressionMetaModel.getDeclaredPropertyMetaModels().add(compactWithExpressionMetaModel.assignmentsPropertyMetaModel);
```

- [ ] **Step 5: Build and install javaparser-mvel**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/javaparser-mvel/pom.xml -pl javaparser-core install -DskipTests`

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit javaparser-mvel changes**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/javaparser-mvel add -A
git -C /home/tkobayas/usr/work/mvel3-development/javaparser-mvel commit -m "feat: add CompactWithExpression AST node for compact setter blocks"
```

---

### Task 2: Add grammar rule and parser visitor in MVEL3

**Files:**
- Modify: `mvel/src/main/antlr4/org/mvel3/parser/antlr4/Mvel3Parser.g4`
- Modify: `mvel/src/main/java/org/mvel3/parser/antlr4/Mvel3ToJavaParserVisitor.java`
- Test: `mvel/src/test/java/org/mvel3/parser/MvelParserTest.java`

All paths below are relative to `/home/tkobayas/usr/work/mvel3-development/mvel`.

- [ ] **Step 1: Write parse test**

Add to `src/test/java/org/mvel3/parser/MvelParserTest.java` (after the `testWithStatementSemicolon` test):

```java
@Test
void testCompactWithExpression() {
    String expr = "{ t{status = RECEIVED, timestamp = new Date()}; }";

    BlockStmt expression = StaticMvelParser.parseBlock(expr);
    verifyBodyWithBetterDiff(printNode(expression),
            "{" + newLine() +
            "    t{status = RECEIVED, timestamp = new Date()};" + newLine() +
            "}");
}

@Test
void testCompactWithExpressionSingleAssignment() {
    String expr = "{ t{status = RECEIVED}; }";

    BlockStmt expression = StaticMvelParser.parseBlock(expr);
    verifyBodyWithBetterDiff(printNode(expression),
            "{" + newLine() +
            "    t{status = RECEIVED};" + newLine() +
            "}");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MvelParserTest#testCompactWithExpression+testCompactWithExpressionSingleAssignment`

Expected: FAIL — grammar does not yet support `t{...}` syntax.

- [ ] **Step 3: Add grammar rule to `Mvel3Parser.g4`**

In `src/main/antlr4/org/mvel3/parser/antlr4/Mvel3Parser.g4`, add the new expression alternative. Insert it in the `expression` rule after the `SquareBracketExpression` line (line 77) and before the `InlineCastExpression` line:

```antlr
    | identifier '{' compactWithAssignment (',' compactWithAssignment)* '}'  #CompactWithExpression
```

Add the new rule after the `withStatement` rule (after line 189):

```antlr
// MVEL-specific compact with expression: t{prop = val, ...}
compactWithAssignment
    : identifier '=' expression
    ;
```

- [ ] **Step 4: Add visitor method `visitCompactWithExpression` in `Mvel3ToJavaParserVisitor.java`**

Add import:
```java
import org.mvel3.parser.ast.expr.CompactWithExpression;
```

Add method (after the `visitWithStatement` method):
```java
public Node visitCompactWithExpression(Mvel3Parser.CompactWithExpressionContext ctx) {
    String targetName = ctx.identifier().getText();
    NameExpr target = new NameExpr(targetName);
    target.setTokenRange(TokenRangeConverter.createTokenRange(ctx));

    NodeList<AssignExpr> assignments = new NodeList<>();
    for (Mvel3Parser.CompactWithAssignmentContext assignCtx : ctx.compactWithAssignment()) {
        String propName = assignCtx.identifier().getText();
        NameExpr propTarget = new NameExpr(propName);
        propTarget.setTokenRange(TokenRangeConverter.createTokenRange(assignCtx));
        Expression value = (Expression) visit(assignCtx.expression());
        assignments.add(new AssignExpr(propTarget, value, AssignExpr.Operator.ASSIGN));
    }

    return new CompactWithExpression(TokenRangeConverter.createTokenRange(ctx), target, assignments);
}
```

Add import for `AssignExpr`:
```java
import com.github.javaparser.ast.expr.AssignExpr;
```

- [ ] **Step 5: Add print visitor support in `MVELPrintVisitor.java`**

In `src/main/java/org/mvel3/parser/printer/MVELPrintVisitor.java`, add import:
```java
import org.mvel3.parser.ast.expr.CompactWithExpression;
```

Add method (after the `visit(WithStatement)` method):
```java
@Override
public void visit(CompactWithExpression compactWith, Void arg) {
    compactWith.getTarget().accept(this, arg);
    printer.print("{");
    NodeList<AssignExpr> assignments = compactWith.getAssignments();
    for (int i = 0; i < assignments.size(); i++) {
        AssignExpr a = assignments.get(i);
        printer.print(PrintUtil.printNode(a.getTarget()));
        printer.print(" = ");
        printer.print(PrintUtil.printNode(a.getValue()));
        if (i < assignments.size() - 1) {
            printer.print(", ");
        }
    }
    printer.print("}");
}
```

Add import for `AssignExpr`:
```java
import com.github.javaparser.ast.expr.AssignExpr;
```

- [ ] **Step 6: Run parse tests to verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MvelParserTest#testCompactWithExpression+testCompactWithExpressionSingleAssignment`

Expected: PASS

- [ ] **Step 7: Run all parser tests to check for regressions**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MvelParserTest`

Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add -A
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "feat: add compact-with grammar rule and parser visitor

Refs tkobayas/drlx-parser#34"
```

---

### Task 3: Refactor transpiler to extract shared context-block logic

**Files:**
- Modify: `mvel/src/main/java/org/mvel3/transpiler/MVELToJavaRewriter.java`
- Test: `mvel/src/test/java/org/mvel3/MVELTranspilerTest.java`

All paths below are relative to `/home/tkobayas/usr/work/mvel3-development/mvel`.

- [ ] **Step 1: Run existing `with` and `modify` transpile tests to establish baseline**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest#tesWith+testModify`

Expected: PASS (baseline)

- [ ] **Step 2: Extract shared helper method**

In `src/main/java/org/mvel3/transpiler/MVELToJavaRewriter.java`, add a new private method:

```java
private void expandContextBlock(NameExpr contextName, List<? extends Statement> statements,
                                BlockStmt outputBlock, boolean addUpdateCall) {
    withContextName = contextName;
    withContextType = contextName.calculateResolvedType();
    statements.forEach(s -> outputBlock.addStatement(s));
    outputBlock.getStatements().forEach(this::rewriteNode);
    if (addUpdateCall) {
        outputBlock.addStatement(
                new MethodCallExpr("update", new NameExpr(contextName.getNameAsString())));
    }
    withContextName = null;
    withContextType = null;
}
```

- [ ] **Step 3: Refactor `WithStatement`/`ModifyStatement` case to use the helper**

Replace the existing `case "ModifyStatement"` / `case "WithStatement"` block (lines 428-448) with:

```java
            case "ModifyStatement" :
                isModifyWithContext = true;
            case "WithStatement" : {
                AbstractContextStatement modifyStmt = (AbstractContextStatement) node;
                NameExpr nameExpr = (NameExpr) modifyStmt.getTarget();
                BlockStmt blockStmt = new BlockStmt();
                modifyStmt.replace(blockStmt);
                modifyStmt.getTarget().setParentNode(blockStmt);
                expandContextBlock(nameExpr,
                        modifyStmt.getExpressions().stream().map(n -> (Statement) n).toList(),
                        blockStmt, isModifyWithContext);
                break;
            }
```

- [ ] **Step 4: Run existing tests to verify refactoring is safe**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest#tesWith+testModify`

Expected: PASS (same as baseline)

- [ ] **Step 5: Run full transpiler test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest`

Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add -A
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "refactor: extract expandContextBlock helper from with/modify transpilation

Refs tkobayas/drlx-parser#34"
```

---

### Task 4: Add transpiler support for compact-with (standalone form)

**Files:**
- Modify: `mvel/src/main/java/org/mvel3/transpiler/MVELToJavaRewriter.java`
- Test: `mvel/src/test/java/org/mvel3/MVELTranspilerTest.java`

All paths below are relative to `/home/tkobayas/usr/work/mvel3-development/mvel`.

- [ ] **Step 1: Write transpile test for standalone form**

Add to `src/test/java/org/mvel3/MVELTranspilerTest.java`:

```java
@Test
void testCompactWithStandalone() {
    test(ctx -> ctx.addDeclaration("foo", Foo.class),
         "foo{countTest = 5};",
         "{ foo.setCountTest(5); }",
         result -> assertThat(allUsedBindings(result)).containsExactlyInAnyOrder("foo"));
}

@Test
void testCompactWithStandaloneMultipleAssignments() {
    test(ctx -> ctx.addDeclaration("$p", Person.class),
         "$p{name = \"Luca\", age = 35};",
         "{ $p.setName(\"Luca\"); $p.setAge(35); }",
         result -> assertThat(allUsedBindings(result)).containsExactlyInAnyOrder("$p"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest#testCompactWithStandalone+testCompactWithStandaloneMultipleAssignments`

Expected: FAIL — transpiler does not handle `CompactWithExpression` yet.

- [ ] **Step 3: Add `CompactWithExpression` case to `MVELToJavaRewriter`**

In `src/main/java/org/mvel3/transpiler/MVELToJavaRewriter.java`, add import:
```java
import org.mvel3.parser.ast.expr.CompactWithExpression;
```

In the `rewriteNode` method's switch statement, add a new case (after the `WithStatement` case):

```java
            case "CompactWithExpression" : {
                CompactWithExpression compactWith = (CompactWithExpression) node;
                NameExpr targetName = compactWith.getTarget();

                // Convert compact assignments to ExpressionStmts
                List<Statement> expandedStmts = new ArrayList<>();
                for (AssignExpr assignment : compactWith.getAssignments()) {
                    expandedStmts.add(new ExpressionStmt(assignment));
                }

                // Find the parent ExpressionStmt that wraps this compact-with
                Node parent = compactWith.getParentNode().orElse(null);
                if (parent instanceof ExpressionStmt) {
                    // Standalone form: replace ExpressionStmt with expanded block
                    BlockStmt blockStmt = new BlockStmt();
                    parent.replace(blockStmt);
                    expandContextBlock(targetName, expandedStmts, blockStmt, false);
                }
                break;
            }
```

Add import:
```java
import com.github.javaparser.ast.stmt.ExpressionStmt;
import java.util.ArrayList;
```

(Check if `ArrayList` and `ExpressionStmt` are already imported before adding.)

- [ ] **Step 4: Run transpile tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest#testCompactWithStandalone+testCompactWithStandaloneMultipleAssignments`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add -A
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "feat: transpiler support for standalone compact-with expression

Refs tkobayas/drlx-parser#34"
```

---

### Task 5: Add transpiler support for compact-with (inline form)

**Files:**
- Modify: `mvel/src/main/java/org/mvel3/transpiler/MVELToJavaRewriter.java`
- Test: `mvel/src/test/java/org/mvel3/MVELTranspilerTest.java`

All paths below are relative to `/home/tkobayas/usr/work/mvel3-development/mvel`.

- [ ] **Step 1: Write transpile test for inline form**

Add to `src/test/java/org/mvel3/MVELTranspilerTest.java`:

```java
@Test
void testCompactWithInlineMethodArg() {
    test(ctx -> {
             ctx.addDeclaration("$p", Person.class);
             ctx.addDeclaration("list", java.util.List.class);
         },
         "list.add($p{name = \"Luca\", age = 35});",
         "{ $p.setName(\"Luca\"); $p.setAge(35); } list.add($p);",
         result -> assertThat(allUsedBindings(result)).containsExactlyInAnyOrder("$p", "list"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest#testCompactWithInlineMethodArg`

Expected: FAIL

- [ ] **Step 3: Extend the `CompactWithExpression` case for inline form**

In `src/main/java/org/mvel3/transpiler/MVELToJavaRewriter.java`, extend the `CompactWithExpression` case to handle the inline form. Replace the entire case block from Task 4 Step 3 with:

```java
            case "CompactWithExpression" : {
                CompactWithExpression compactWith = (CompactWithExpression) node;
                NameExpr targetName = compactWith.getTarget();

                // Convert compact assignments to ExpressionStmts
                List<Statement> expandedStmts = new ArrayList<>();
                for (AssignExpr assignment : compactWith.getAssignments()) {
                    expandedStmts.add(new ExpressionStmt(assignment));
                }

                // Find the parent ExpressionStmt that wraps this compact-with
                Node parent = compactWith.getParentNode().orElse(null);
                if (parent instanceof ExpressionStmt) {
                    // Standalone form: replace ExpressionStmt with expanded block
                    BlockStmt blockStmt = new BlockStmt();
                    parent.replace(blockStmt);
                    expandContextBlock(targetName, expandedStmts, blockStmt, false);
                } else {
                    // Inline form: hoist assignments before the enclosing statement
                    // Walk up to find the nearest enclosing ExpressionStmt
                    Node current = parent;
                    while (current != null && !(current instanceof ExpressionStmt)) {
                        current = current.getParentNode().orElse(null);
                    }
                    if (current instanceof ExpressionStmt enclosingStmt) {
                        // Create a block with expanded assignments before the enclosing statement
                        BlockStmt blockStmt = new BlockStmt();
                        enclosingStmt.replace(blockStmt);
                        expandContextBlock(targetName, expandedStmts, blockStmt, false);
                        // Replace compact-with expression with just the target name
                        compactWith.replace(new NameExpr(targetName.getNameAsString()));
                        // Add the original statement (now with target substituted) to the block
                        blockStmt.addStatement(enclosingStmt);
                        // Rewrite the original statement too (for any remaining transpilation)
                        rewriteNode(enclosingStmt);
                    }
                }
                break;
            }
```

- [ ] **Step 4: Run inline test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest#testCompactWithInlineMethodArg`

Expected: PASS

- [ ] **Step 5: Run all compact-with and existing with/modify tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest#testCompactWithStandalone+testCompactWithStandaloneMultipleAssignments+testCompactWithInlineMethodArg+tesWith+testModify`

Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add -A
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "feat: transpiler support for inline compact-with (hoist before call)

Refs tkobayas/drlx-parser#34"
```

---

### Task 6: Add end-to-end evaluation tests

**Files:**
- Test: `mvel/src/test/java/org/mvel3/MVELTranspilerTest.java`

All paths below are relative to `/home/tkobayas/usr/work/mvel3-development/mvel`.

- [ ] **Step 1: Write evaluation test for standalone form**

Add to `src/test/java/org/mvel3/MVELTranspilerTest.java`:

```java
@Test
void testCompactWithStandaloneEval() {
    Person person = new Person("John");
    person.setAge(20);
    MVEL mvel = new MVEL();
    mvel.addImport(Person.class.getCanonicalName());
    Evaluator<Map<String, Object>, Void, String> evaluator =
            mvel.compileMapBlock("$p{name = \"Luca\", age = 35};\n return null;",
                    String.class, new HashSet<>(),
                    Map.of("$p", Type.type(Person.class)));
    Map<String, Object> vars = new HashMap<>();
    vars.put("$p", person);
    evaluator.eval(vars);
    assertThat(person.getName()).isEqualTo("Luca");
    assertThat(person.getAge()).isEqualTo(35);
}
```

- [ ] **Step 2: Run evaluation test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=MVELTranspilerTest#testCompactWithStandaloneEval`

Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test`

Expected: All tests PASS

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add -A
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "test: end-to-end evaluation tests for compact-with expression

Refs tkobayas/drlx-parser#34"
```

---

### Task 7: Install MVEL3 snapshot for downstream use

- [ ] **Step 1: Install MVEL3 with tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml install`

Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 2: Verify MVEL3 snapshot is available**

Run: `ls ~/.m2/repository/org/mvel/mvel3/3.0.0-SNAPSHOT/mvel3-3.0.0-SNAPSHOT.jar`

Expected: File exists
