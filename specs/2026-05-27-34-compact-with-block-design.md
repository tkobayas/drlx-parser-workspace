# Compact Setter Block (`t{...}`) — Issue #34

**Date:** 2026-05-27
**Issue:** https://github.com/tkobayas/drlx-parser/issues/34
**Epic:** #26
**Scope:** MVEL3 (grammar, AST, transpiler) + javaparser-mvel (AST node)

## Summary

Add compact setter block syntax `t{status = RECEIVED, timestamp = new Date()}` to
MVEL3 as an expression-level construct. The block applies property assignments to
the target and returns the target, enabling inline use such as
`alerts.update(t{status = RECEIVED, timestamp = new Date()})`.

## DRLXXXX Reference

Lines 339–349 (§ 'with' style blocks):

```
t{status = RECEIVED, timestamp = new Date()};
```

Lines 475–478 (§ Add/Remove/Update, compact update):

```
alerts.update(t{status = RECEIVED, timestamp = new Date()});
```

## Relationship to Existing `with`

MVEL3 already has `with (t) { stmt; stmt; }` as a **statement** (`WithStatement`).
The compact form differs in three ways:

1. **No keyword** — `t{...}` instead of `with (t) {...}`
2. **Comma-separated assignments** — not semicolon-separated statements
3. **Expression semantics** — returns the target, so it can appear as a method
   argument, variable initializer, etc.

## Design

### 1. Grammar (Mvel3Parser.g4)

New expression alternative and supporting rule:

```antlr
// In the expression rule, new alternative:
| identifier '{' compactWithAssignment (',' compactWithAssignment)* '}'  #CompactWithExpression

// New rule:
compactWithAssignment
    : identifier '=' expression
    ;
```

**Ambiguity analysis:** `identifier '{'` does not conflict with any existing Java
or MVEL3 construct. Labeled statements use `:`, standalone blocks follow keywords
or appear at statement level (not after an identifier in expression position).

**Target restriction:** Only simple identifiers, not chained expressions like
`person.address{...}`. Keeps the grammar unambiguous and covers all DRLXXXX
examples.

**Assignment LHS:** Bare property name (identifier). No dotted paths, no array
access.

### 2. AST Node (javaparser-mvel)

New class `CompactWithExpression` in `org.mvel3.parser.ast.expr`, extending
`Expression`:

```java
public class CompactWithExpression extends Expression {
    private NameExpr target;                    // e.g. "t"
    private NodeList<AssignExpr> assignments;   // e.g. [status = RECEIVED, ...]
}
```

Each assignment is a standard `AssignExpr` where the target (LHS) is a `NameExpr`
(bare property name) and the value (RHS) is any `Expression`.

Supporting changes in javaparser-mvel:
- `GenericVisitor` / `VoidVisitor`: add `visit(CompactWithExpression, A)`
- All visitor adapters/defaults: add default implementation
- `CloneVisitor`: add clone support
- `JavaParserMetaModel`: register `CompactWithExpressionMetaModel`
- `CompactWithExpressionMetaModel`: new metamodel class

### 3. Visitor (Mvel3ToJavaParserVisitor)

`visitCompactWithExpression` converts the parse tree to the AST:

```
CompactWithExpression(
    target = NameExpr("t"),
    assignments = [
        AssignExpr(NameExpr("status"), NameExpr("RECEIVED"), ASSIGN),
        AssignExpr(NameExpr("timestamp"), MethodCallExpr("new Date()"), ASSIGN)
    ]
)
```

### 4. Transpiler (MVELToJavaRewriter)

#### Shared helper

Extract the common logic from the existing `WithStatement` / `ModifyStatement`
case into a reusable method:

```java
private void expandContextBlock(
        NameExpr contextName, List<? extends Statement> statements,
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

The existing `ModifyStatement` case calls `expandContextBlock(..., true)`, the
existing `WithStatement` case calls `expandContextBlock(..., false)`, and the new
`CompactWithExpression` case converts its assignments to statements and calls the
same helper.

#### Case 1: Standalone — `t{status = RECEIVED, timestamp = new Date()};`

The `CompactWithExpression` appears as the expression of an `ExpressionStmt`.
Convert each compact assignment to a full property-access assignment, then replace
the statement with a block:

```java
// Input:
t{status = RECEIVED, timestamp = new Date()};

// Output:
t.status = RECEIVED;
t.timestamp = new Date();
```

#### Case 2: Inline (method argument) — `alerts.update(t{...})`

The `CompactWithExpression` appears as a method call argument. Hoist the expanded
assignments before the enclosing statement, then replace the expression with just
the target:

```java
// Input:
alerts.update(t{status = RECEIVED, timestamp = new Date()});

// Output:
t.status = RECEIVED;
t.timestamp = new Date();
alerts.update(t);
```

The hoisting walks up the AST from the `CompactWithExpression` to find the nearest
enclosing `ExpressionStmt`, inserts the expanded assignments before it, and
replaces the compact expression with a `NameExpr` of the target.

### 5. Print Visitor (MVELPrintVisitor)

Add `visit(CompactWithExpression)` to emit:

```
t{status = RECEIVED, timestamp = new Date()}
```

Comma-separated, no trailing comma, braces immediately after identifier.

### 6. Type Resolution

The `CompactWithExpression` resolves to the type of its target. Each assignment's
LHS is resolved as a property of the target's type. This is handled by the
existing property-access rewriting in the transpiler (the `withContextName` /
`withContextType` mechanism).

## Scope Boundaries

**In scope:**
- Grammar rule for `identifier '{' assignments '}'`
- `CompactWithExpression` AST node
- Transpiler rewriting (standalone and inline forms)
- Print visitor
- MVEL3 unit tests (parse, transpile, evaluate)

**Out of scope:**
- Test block `t[...]` — separate issue #65
- Chained expression targets (`person.address{...}`)
- Nested compact-with (`t{address = a{city = "NYC"}}`)
- DRLX integration tests (will be a follow-up once MVEL3 is released)

## Test Plan

1. **Parse tests:** Verify `t{status = RECEIVED}` and
   `t{status = RECEIVED, timestamp = new Date()}` parse to correct AST
2. **Transpile tests:** Verify standalone form expands to property-access
   assignments
3. **Transpile tests:** Verify inline form hoists assignments and replaces
   expression with target
4. **Evaluate tests:** End-to-end execution of standalone and inline forms against
   domain objects with getters/setters
5. **Print tests:** Round-trip: parse → print → parse produces identical AST
6. **Shared logic:** Verify `with(t){...}` still works after refactoring to use
   shared helper
