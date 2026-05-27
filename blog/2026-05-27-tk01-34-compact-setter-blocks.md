---
layout: post
title: "#34 — compact setter blocks land in MVEL3"
date: 2026-05-27
type: phase-update
entry_type: note
subtype: diary
projects: [mvel3, drlx-parser]
tags: [compact-with, mvel3, javaparser-mvel, grammar, transpiler]
---

# #34 — compact setter blocks land in MVEL3

The DRLXXXX spec has a compact setter syntax: `t{status = RECEIVED, timestamp = new Date()}`. No keyword, comma-separated assignments, and it returns the target — so it nests inside method calls like `alerts.update(t{...})`. MVEL3 already has `with (t) { ... }` as a statement, but this compact form is fundamentally different: it's an expression.

I decided to implement it as a native MVEL3 feature rather than DRLX-level desugaring. That way the grammar, AST, and transpiler all live in one place, and DRLX consequences get the syntax for free through the grammar import chain.

## Twenty files for one AST node

The new `CompactWithExpression` extends `Expression` in javaparser-mvel. Two fields: a `NameExpr` target and a `NodeList<AssignExpr>` for the assignments. Simple enough — but javaparser's visitor pattern means touching every visitor interface and adapter.

I brought Claude in for the implementation. We hit 15 visitor files in the plan — `GenericVisitor`, `VoidVisitor`, all the adapters, `CloneVisitor`, `ModifierVisitor`, the hash/equals visitors. The first build failed on three more: `DefaultPrettyPrinterVisitor`, `PrettyPrintVisitor` (both implement `VoidVisitor`), and a type-casting `@Override` issue on `CompactWithExpression` itself (those methods don't exist on the parent `Expression` to override). Fixed, rebuilt, installed — and moved to MVEL3.

Later, `mvn clean install` on the full javaparser-mvel project surfaced two more in the symbol-solver module: `DefaultVisitorAdapter` and `TypeExtractor`. Then `ConcreteSyntaxModel` — the lexical preservation registry that maps every AST class to its concrete syntax. Five files the plan hadn't anticipated, each surfacing as a separate compile error.

## Grammar and transpiler

The grammar addition is compact:

```antlr
| identifier '{' compactWithAssignment (',' compactWithAssignment)* '}'  #CompactWithExpression

compactWithAssignment
    : identifier '=' expression
    ;
```

No ambiguity with Java blocks — labeled statements use `:`, and standalone blocks only appear after keywords or at statement level, never after an identifier in expression position.

The transpiler needed a shared helper. The existing `WithStatement` and `ModifyStatement` cases had identical logic for setting `withContextName`/`withContextType`, expanding statements, rewriting bare names to `target.property`, and clearing context. We extracted `expandContextBlock()` and had all three cases call it — `ModifyStatement` with `addUpdateCall=true`, the others with `false`.

Two rewrite forms for compact-with:

**Standalone** — `t{status = RECEIVED, timestamp = new Date()};` becomes `t.status = RECEIVED; t.timestamp = new Date();`

**Inline** — `list.add(t{name = "Luca", age = 35});` hoists the assignments before the enclosing statement:

```java
// becomes:
t.setName("Luca");
t.setAge(35);
list.add(t);
```

One gotcha in the transpiler: after `parent.replace(blockStmt)`, the target `NameExpr` loses its parent chain to the `CompilationUnit`. `calculateResolvedType()` then throws `IllegalStateException: The node is not inserted in a CompilationUnit`. The fix is the same one the existing `WithStatement` case already uses: `targetName.setParentNode(blockStmt)` before calling `expandContextBlock`.

## The version dependency

MVEL3's `pom.xml` pointed at `javaparser.version = 3.25.5-mvel3-1` — a released version. The new AST node lives in `3.25.5-mvel3-SNAPSHOT`. Switching the dependency was straightforward but worth noting: the `compact-with` branches in both repos need to be merged together, and the javaparser-mvel version will need a release before MVEL3 can go back to a released dependency.

## What landed

| Repo | Branch | Commits |
|------|--------|---------|
| javaparser-mvel | `compact-with` | 3 (AST node, symbol-solver fix, ConcreteSyntaxModel fix) |
| MVEL3 | `compact-with` | 4 (grammar+visitor, refactor, transpiler, eval test) |

762 MVEL3 tests green, including 3 new transpile tests and 1 end-to-end evaluation test. Not merged yet — both branches ahead of `main`.
