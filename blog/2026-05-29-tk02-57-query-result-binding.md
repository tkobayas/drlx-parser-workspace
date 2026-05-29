---
layout: post
title: "#57 — query result binding: the Map trick that saved a code generator"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [queries, result-binding, mvel3, AbstractMap, epic-77]
---

# #57 — query result binding: the Map trick that saved a code generator

The DRLXXXX spec says `var t : /trusts(a, var b)` should bind the entire query result row to `t`, with named access (`t.a`), indexed access (`t[0]`), and method calls (`t.objects()`). The spec says "generated POJO class." I was bracing for bytecode generation — ASM or ByteBuddy, a custom classloader, runtime type creation. None of that happened.

## MVEL3 already knows how to do this

The question was how `t.a` would compile. MVEL3 transpiles expressions to Java — `t.a` becomes a `FieldAccessExpr` in the JavaParser AST, then gets rewritten to a getter call. Claude traced the rewriter and found `maybeRewriteToGetter()` at line 1475 of `MVELToJavaRewriter.java`:

```java
if (isAssignableBy(mapType, type)) {
    getter = mapGetMethod;
    arg    = new StringLiteralExpr(n.getNameAsString());
}
```

If the scope resolves to a `Map` type, MVEL3 rewrites `map.key` to `map.get("key")`. Similarly, `rewriteArrayAccessExpr()` rewrites `map[0]` to `map.get(0)` for non-array types. Both paths already exist. Nobody wrote them for this feature — they're part of MVEL3's general type-aware transpilation.

The design fell out of that: make `QueryResultRow` extend `AbstractMap<String, Object>`, override `get(Object key)` to dispatch on `String` (named lookup) vs `Integer` (indexed lookup), and let MVEL3 do the rest. Zero MVEL3 changes. Zero bytecode generation.

## Twelve lines in the builder

The runtime builder already creates a `QueryElement` and binds individual output variables like `b` from `var b`. The result binding adds one block after that loop — build a name-to-index map from the query's parameter declarations, create a `QueryResultRowReader` that wraps the `Object[]` into a `QueryResultRow`, and register the binding:

```java
if (patternIr.bindName() != null) {
    Map<String, Integer> nameToIndex = new LinkedHashMap<>();
    Declaration[] qParams = targetQuery.getParameters();
    for (int i = 0; i < qParams.length; i++) {
        nameToIndex.put(qParams[i].getIdentifier(), i);
    }
    QueryResultRowReader rowReader = new QueryResultRowReader(nameToIndex);
    Pattern resultPattern = queryElement.getResultPattern();
    Declaration rowDecl = new Declaration(patternIr.bindName(), rowReader, resultPattern);
    resultPattern.addDeclaration(rowDecl);
    boundVariables.put(patternIr.bindName(),
            new BoundVariable(patternIr.bindName(), QueryResultRow.class,
                              resultPattern, rowDecl));
}
```

The generated lambda for `do { results.add(t.result); }` confirms it works — MVEL3 emits `results.add(t.get("result"))` without any special handling on our side.

## Two issues surfaced

Handle access (`t.handles[0]`) is deferred to #82. The `ReadAccessor` interface receives `(ValueResolver, Object)` — it doesn't get the tuple, so there's no clean way to expose `FactHandle` chains without Drools runtime changes.

A real bug turned up in `DrlxEvalExpression`: its `evaluate()` method uses `fh.getObject()` to extract values, bypassing the declaration's `ReadAccessor` entirely. The beta constraint does it correctly with `decl.getValue(null, fh.getObject())`. This means any `test` expression referencing a query result binding — or even an individual query output variable — would hit a `ClassCastException`. Filed as #83.

## What landed

| Item | Detail |
|------|--------|
| Commits | `c04369b`..`f0d7894` on `main` (5 commits) |
| New files | `QueryResultRow.java`, `QueryResultRowReader.java`, `QueryResultRowTest.java` |
| Modified | `DrlxRuleAstRuntimeBuilder.java` (+12 lines), `QueryTest.java` (+5 tests) |
| Issues | Closes #57; filed #82 (handles), #83 (eval ReadAccessor bypass) |
