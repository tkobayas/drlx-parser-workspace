# #82 Query Result Handle Access — `t.handles[0]` / `t.handles().a`

**Issue:** #82 (parent epic: #77)
**Depends on:** #57 (query result binding)
**Date:** 2026-06-01

## Problem

`QueryResultRow` (from #57) wraps the `Object[]` of query result values and supports `t.a` (named) and `t[0]` (indexed) access. The DRLXXXX spec (line 925) also defines handle access:

- `t.handles()[0]` — indexed FactHandle access
- `t.handles().a` — named FactHandle access

FactHandles are not stored in the `Object[]`. The #57 implementation deferred handle access because the `ReadAccessor` interface doesn't expose the tuple chain where handles live.

## Approach: Lazy Handle Resolution via ReteEvaluator

Instead of extracting handles eagerly in `QueryElementNode.rowAdded()` (which would require Drools changes), resolve handles lazily by looking up objects in working memory at access time.

`ReteEvaluator` (which extends `ValueResolver`) provides `getFactHandle(Object)`. The `ValueResolver` is already passed to `QueryResultRowReader.getValue(ValueResolver, Object)` — we just need to thread it through to `QueryResultRow`, then use it when `handles` is accessed.

Zero Drools changes. All work is in drlx-parser.

## Design

### Data Flow

```
QueryResultRowReader.getValue(ValueResolver, Object)
  → QueryResultRow(Object[], nameToIndex, valueResolver)
     ├── t.a          → get("a")         → values[idx]                    (unchanged)
     ├── t[0]         → get(0)           → values[0]                      (unchanged)
     ├── t.handles()[0]→ handles().get(0) → reteEvaluator.getFactHandle(values[0])
     └── t.handles().a → handles().get("a")→ reteEvaluator.getFactHandle(values[idx])
```

Handle access uses method call syntax only (`t.handles()`, not `t.handles`). MVEL3 resolves the return type of `handles()` to `QueryResultHandleRow` (a Map), so `.a` rewrites to `.get("a")` and `[0]` rewrites to `.get(0)` automatically. The map property path (`t.handles[0]`) is not supported because `Map.get()` returns `Object` statically and MVEL3 can't resolve `.get(0)` on `Object`.

### New Class: `QueryResultHandleRow`

`AbstractMap<String, Object>` sibling to `QueryResultRow`. Same `values[]` and `nameToIndex`, but `get()` wraps the resolved value in `reteEvaluator.getFactHandle(value)`.

```java
public final class QueryResultHandleRow extends AbstractMap<String, Object> {
    private final Object[] values;
    private final Map<String, Integer> nameToIndex;
    private final ReteEvaluator reteEvaluator;

    @Override
    public Object get(Object key) {
        Object value;
        if (key instanceof String s) {
            Integer idx = nameToIndex.get(s);
            value = idx != null ? values[idx] : null;
        } else if (key instanceof Integer i) {
            value = values[i];
        } else {
            return null;
        }
        return value != null ? findFactHandle(value) : null;
    }

    // Searches all entry points because objects may be in named entry points,
    // not just the default. Could be improved by passing the entry point name
    // through to avoid the linear search.
    private InternalFactHandle findFactHandle(Object object) { ... }

    // entrySet(), size() — same pattern as QueryResultRow
}
```

### Changes to `QueryResultRow`

1. Add `ValueResolver valueResolver` field
2. Constructor accepts `ValueResolver` as third parameter
3. `handles()` returns `new QueryResultHandleRow(values, nameToIndex, (ReteEvaluator) valueResolver)` instead of throwing

### Changes to `QueryResultRowReader`

1. `getValue(ValueResolver, Object)` passes `valueResolver` to the `QueryResultRow` constructor

### Changes to `QueryResultRowReader` (no-arg overload)

`getValue(Object)` (no `ValueResolver`) passes `null` as the `valueResolver`. `handles()` on a row created this way throws `UnsupportedOperationException`. The runtime always uses the two-arg `getValue(ValueResolver, Object)` form, so this path is not exercised in practice.

### No Drools Changes

The `ValueResolver` passed at runtime is always a `ReteEvaluator`. The cast is safe. `getFactHandle(Object)` looks up by identity in the object store — the objects in `values[]` are the actual working memory objects (extracted via `Declaration.getValue()` → `fh.getObject()`), so the lookup succeeds.

## Tests

1. `t.handles()[0]` — indexed handle access returns an `InternalFactHandle`
2. `t.handles().a` — named handle access returns the correct handle
3. Handle identity — `t.handles()[0].getObject()` returns the same object as `t[0]`
4. Handle identity — `t.handles().a.getObject()` returns the same object as `t.a`
5. Out-of-bounds / missing name returns null

## Files Changed

| File | Change |
|------|--------|
| `QueryResultHandleRow.java` | New — `AbstractMap` wrapper resolving handles lazily |
| `QueryResultRow.java` | Add `valueResolver` field, implement `handles()`, intercept `get("handles")` |
| `QueryResultRowReader.java` | Pass `valueResolver` to `QueryResultRow` constructor |
