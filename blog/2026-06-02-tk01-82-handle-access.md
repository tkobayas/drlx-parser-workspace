---
layout: post
title: "#82 — handle access and the null that came from three layers down"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [queries, fact-handle, mvel3, AbstractMap, ValueResolver, epic-77]
---

# #82 — handle access and the null that came from three layers down

The last query feature from epic #77's backlog: `t.handles()[1].getObject()` and `t.handles().result.getObject()`. The design spec was done last session — lazy FactHandle resolution via entry point search, zero Drools changes. I expected this to be mechanical wiring. Two things disagreed.

## The generic type that MVEL3 can't see through

The plan called for `QueryResultHandleRow extends AbstractMap<String, Object>`, mirroring `QueryResultRow`'s shape from #57. I brought Claude in and we built it exactly that way — `get(Object key)` dispatches on String vs Integer, `findFactHandle()` iterates entry points. Clean, compiles, matches the spec.

The first E2E test — `t.handles()[1].getObject()` — failed immediately:

```
Method 'getObject' cannot be resolved in context
  t.handles().get(1).getObject()
```

MVEL3 transpiles `t.handles()[1]` to `t.handles().get(1)`. The return type of `AbstractMap<String, Object>.get()` is `Object`. MVEL3 resolves chained methods from the *declared* generic parameter, not the runtime type. So `.getObject()` on `Object` is unresolvable — `InternalFactHandle` is invisible to the type system at compile time.

The fix: change `AbstractMap<String, Object>` to `AbstractMap<String, InternalFactHandle>`. Now `get()` returns `InternalFactHandle` and MVEL3 can resolve `.getObject()`. The #57 entry mentioned how MVEL3's `maybeRewriteToGetter()` rewrites map access — the same mechanism, and the same sensitivity to declared types.

## The ValueResolver that vanished

Test two. The type resolution was fixed but now a runtime error:

```
UnsupportedOperationException: Handle access requires a ValueResolver
  — not available in this context
```

`QueryResultRow` was being created with `null` as its `ValueResolver`. The `QueryResultRowReader.getValue(ValueResolver, Object)` method passes it through — so somewhere upstream, `null` was being supplied.

Claude traced the call chain. `DrlxLambdaConsequence.evaluate()` receives a perfectly good `ValueResolver` parameter but calls `knowledgeHelper.getMatch().getDeclarationValue(declarationId)` to extract variables. That call goes to `RuleTerminalNodeLeftTuple.getDeclarationValue()`, which calls `decl.getValue(this)` — a single-arg call. Three layers down, `TupleValueExtractor` has a default method:

```java
default Object getValue(BaseTuple tuple) {
    return getValue(null, tuple);  // null ValueResolver
}
```

The `ValueResolver` available in the consequence gets dropped by a convenience API. The fix: bypass `Match.getDeclarationValue()` entirely and call `Declaration.getValue(valueResolver, tuple)` directly in `DrlxLambdaConsequence`:

```java
InternalMatch match = knowledgeHelper.getMatch();
Map<String, Declaration> declarations =
    match.getTerminalNode().getSubRule().getOuterDeclarations();
for (String declarationId : match.getDeclarationIds()) {
    Declaration decl = declarations.get(declarationId);
    vars.put(declarationId, decl.getValue(valueResolver, match.getTuple()));
}
```

This is how Drools' own `InternalMatch` does it — the existing `getDeclarationValue()` was always a shortcut that happened to work because nothing previously needed the `ValueResolver`.

## What landed

| Item | Detail |
|------|--------|
| Commit | `480067e` on `main` |
| New file | `QueryResultHandleRow.java` — `AbstractMap<String, InternalFactHandle>`, lazy entry-point search |
| Modified | `QueryResultRow.java`, `QueryResultRowReader.java`, `DrlxLambdaConsequence.java`, `QueryResultRowTest.java`, `QueryTest.java` |
| Tests | 3 new E2E tests (indexed, named, identity); 340 total, all green |
| Issues | Closes #82 |
