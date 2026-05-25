# #63 — MVEL3 `==` Transpilation for Reference Types — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make MVEL3's transpiler generate `java.util.Objects.equals()` for `==`/`!=` on reference types instead of Java `==` (reference equality).

**Architecture:** Add a fallback method `rewriteReferenceEquality()` in `MVELToJavaRewriter` that is called when `OverloadRewriter.overload()` returns null. The method checks if the operator is `==`/`!=` on non-primitive, non-enum types and rewrites to `Objects.equals()`.

**Tech Stack:** Java, JavaParser AST, JUnit 5, MVEL3 transpiler

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `src/main/java/org/mvel3/transpiler/MVELToJavaRewriter.java` | Modify | Add `rewriteReferenceEquality()` method, call from `rewrite()` |
| `src/test/java/org/mvel3/ConstraintTranspilerTest.java` | Modify | Add test cases for String/Object/enum/null/primitive `==`/`!=` |

---

### Task 1: Write failing tests for String `==` and `!=`

**Files:**
- Modify: `src/test/java/org/mvel3/ConstraintTranspilerTest.java`

- [ ] **Step 1: Add String equality test**

Add after the existing `testBigDecimalStringNonEquality` test (~line 108):

```java
@Test
void testStringEquality() {
    testExpression(c -> c.withDeclaration(Declaration.of("_this", Person.class)), "{var x = name == \"John\";}",
                   "{var x = java.util.Objects.equals(_this.getName(), \"John\");}");
}
```

- [ ] **Step 2: Add String non-equality test**

```java
@Test
void testStringNonEquality() {
    testExpression(c -> c.withDeclaration(Declaration.of("_this", Person.class)), "{var x = name != \"John\";}",
                   "{var x = !java.util.Objects.equals(_this.getName(), \"John\");}");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=ConstraintTranspilerTest#testStringEquality+testStringNonEquality -DfailIfNoTests=false`

Expected: FAIL — both tests should fail because MVEL3 currently generates `_this.getName() == "John"` instead of `java.util.Objects.equals(...)`.

- [ ] **Step 4: Commit failing tests**

```
Refs tkobayas/drlx-parser#63
```

---

### Task 2: Write failing tests for bound variable, nested property, and Object equality

**Files:**
- Modify: `src/test/java/org/mvel3/ConstraintTranspilerTest.java`

- [ ] **Step 1: Add String == bound variable test**

```java
@Test
void testStringEqualityWithBoundVariable() {
    testExpression(c -> {
        c.withDeclaration(Declaration.of("_this", Person.class));
        c.addDeclaration("$n", String.class);
    }, "{var x = name == $n;}",
       "{var x = java.util.Objects.equals(_this.getName(), $n);}");
}
```

- [ ] **Step 2: Add nested property String == test**

```java
@Test
void testNestedPropertyStringEquality() {
    testExpression(c -> c.withDeclaration(Declaration.of("_this", Person.class)), "{var x = parent.name == \"John\";}",
                   "{var x = java.util.Objects.equals(_this.getParent().getName(), \"John\");}");
}
```

- [ ] **Step 3: Add Object == Object test (Person == bound variable)**

```java
@Test
void testObjectEquality() {
    testExpression(c -> {
        c.withDeclaration(Declaration.of("_this", Person.class));
        c.addDeclaration("$p", Person.class);
    }, "{var x = parent == $p;}",
       "{var x = java.util.Objects.equals(_this.getParent(), $p);}");
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=ConstraintTranspilerTest#testStringEqualityWithBoundVariable+testNestedPropertyStringEquality+testObjectEquality -DfailIfNoTests=false`

Expected: FAIL

- [ ] **Step 5: Commit failing tests**

```
Refs tkobayas/drlx-parser#63
```

---

### Task 3: Write failing tests for exclusions (enum, null, primitive)

**Files:**
- Modify: `src/test/java/org/mvel3/ConstraintTranspilerTest.java`

- [ ] **Step 1: Add enum == test (should stay as Java ==)**

```java
@Test
void testEnumEqualityUnchanged() {
    testExpression(c -> c.withDeclaration(Declaration.of("_this", Person.class)), "{var x = gender == Gender.MALE;}",
                   "{var x = _this.getGender() == Gender.MALE;}");
}
```

- [ ] **Step 2: Add null comparison test (should stay as Java ==)**

```java
@Test
void testNullComparisonUnchanged() {
    testExpression(c -> c.withDeclaration(Declaration.of("_this", Person.class)), "{var x = name == null;}",
                   "{var x = _this.getName() == null;}");
}
```

- [ ] **Step 3: Add primitive == test (should stay as Java ==)**

```java
@Test
void testPrimitiveEqualityUnchanged() {
    testExpression(c -> c.withDeclaration(Declaration.of("_this", Person.class)), "{var x = age == 30;}",
                   "{var x = _this.getAge() == 30;}");
}
```

- [ ] **Step 4: Run tests to verify behaviour**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=ConstraintTranspilerTest#testEnumEqualityUnchanged+testNullComparisonUnchanged+testPrimitiveEqualityUnchanged -DfailIfNoTests=false`

Expected: The enum and null tests should PASS (they already generate `==`). The primitive test should also PASS. These are regression guards — they confirm the exclusions work before and after the implementation.

- [ ] **Step 5: Commit exclusion tests**

```
Refs tkobayas/drlx-parser#63
```

---

### Task 4: Implement `rewriteReferenceEquality()` and wire it in

**Files:**
- Modify: `src/main/java/org/mvel3/transpiler/MVELToJavaRewriter.java`

- [ ] **Step 1: Add the `rewriteReferenceEquality()` method**

Add after the existing `rewrite()` method (after line 1334):

```java
private Expression rewriteReferenceEquality(BinaryExprTypes binExprTypes) {
    BinaryExpr.Operator op = binExprTypes.getBinaryExpr().getOperator();
    if (op != BinaryExpr.Operator.EQUALS && op != BinaryExpr.Operator.NOT_EQUALS) {
        return null;
    }

    ResolvedType leftType = binExprTypes.leftType;
    ResolvedType rightType = binExprTypes.rightType;

    if ((leftType == null || leftType.isPrimitive()) && (rightType == null || rightType.isPrimitive())) {
        return null;
    }

    Expression left = binExprTypes.left;
    Expression right = binExprTypes.right;

    if (left instanceof NullLiteralExpr || right instanceof NullLiteralExpr) {
        return null;
    }

    if (isEnum(leftType) || isEnum(rightType)) {
        return null;
    }

    MethodCallExpr equalsCall = new MethodCallExpr(
            new NameExpr("java.util.Objects"), "equals",
            new NodeList<>(left.clone(), right.clone()));

    if (op == BinaryExpr.Operator.NOT_EQUALS) {
        return new UnaryExpr(equalsCall, UnaryExpr.Operator.LOGICAL_COMPLEMENT);
    }
    return equalsCall;
}

private boolean isEnum(ResolvedType type) {
    return type != null && type.isReferenceType()
            && type.asReferenceType().getTypeDeclaration().isPresent()
            && type.asReferenceType().getTypeDeclaration().get().isEnum();
}
```

- [ ] **Step 2: Wire into `rewrite()` — add fallback after overloader**

In the `rewrite()` method, change lines 1332-1333 from:

```java
        Expression overloaded = overloader.overload(leftType, left, right, binExprTypes.binaryExpr.getOperator());
        return overloaded;
```

to:

```java
        Expression overloaded = overloader.overload(leftType, left, right, binExprTypes.binaryExpr.getOperator());
        if (overloaded == null) {
            overloaded = rewriteReferenceEquality(binExprTypes);
        }
        return overloaded;
```

- [ ] **Step 3: Add NodeList import if not already present**

Check if `NodeList` is already imported (it is — line 5: `import com.github.javaparser.ast.NodeList;`). No action needed.

- [ ] **Step 4: Run the previously failing tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . -Dtest=ConstraintTranspilerTest -DfailIfNoTests=false`

Expected: ALL tests PASS — the 8 new tests plus all existing tests.

- [ ] **Step 5: Run full test suite to check for regressions**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test`

Expected: All tests pass. If any existing tests fail, it means the reference equality rewrite affected a case that was relying on `==` — investigate and adjust the exclusion logic.

- [ ] **Step 6: Commit implementation**

```
Refs tkobayas/drlx-parser#63
```

---

### Task 5: Install updated MVEL3 and verify drlx-parser tests still pass

- [ ] **Step 1: Install MVEL3 locally**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml install -DskipTests`

- [ ] **Step 2: Run drlx-parser tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`

Expected: All 279 tests pass. The `.intern()` workaround in drlx-parser is redundant now but harmless — it will be cleaned up in a follow-up issue.

- [ ] **Step 3: Commit (no changes expected — this is a verification step)**

No commit needed unless test failures require fixes.
