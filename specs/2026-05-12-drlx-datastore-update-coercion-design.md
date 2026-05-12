## Design — DataStore.update(T) coercion in DRLX consequences (issue #45)

**Issue:** #45 (split from #37, second iteration of DataStore CRUD)
**Epic:** #26
**Date:** 2026-05-12
**Scope:** Compile-time consequence-text rewrite in `drlx-parser-core`. No runtime API change, no grammar change, no upstream Drools change.

## Motivation

The unit-field globals work landed in `cab2862` makes `alerts` resolve as a consequence symbol, so `alerts.add(t)` and `alerts.remove(t)` work directly — `DataStore<T>` exposes both. The third CRUD operation, `alerts.update(t)`, fails: `DataStore<T>` only exposes `update(DataHandle, T)`. The single-arg form has to be translated.

Upstream Drools handles this in `drools-model-codegen/Consequence.java` (lines 345–380) by rewriting the call's scope at AST-codegen time to wrap the DataStore in `ConsequenceDataStoreImpl`, a runtime wrapper that needs `RuleContext` (the `drools` variable) for `getMatch()` / `getKieBase()` and adds a `BitMask` argument for property reactivity.

DRLX has neither `RuleContext` infrastructure in consequences nor BitMask / property reactivity yet. So porting the upstream wrapper is heavier than the problem warrants. The lighter equivalent — same effective semantics — is a direct AST rewrite to the handle-aware form, leaning on `lookup(Object)` which `ListDataStore` already exposes (the same method `remove(T)` uses internally).

## Scope

**In scope**
- New helper class `DataStoreUpdateRewriter` in `drlx-parser-core/src/main/java/org/drools/drlx/builder/`. Stateless apart from a reusable `JavaParser` instance. Pure: source string + DataStore-global names → rewritten source string.
- Wire `DrlxRuleAstRuntimeBuilder.buildRule()` to compute the set of DataStore-typed global names from the existing `globalTypes` map and run the rewrite before constructing `DrlxLambdaConsequence`.
- New tests in `DataStoreCrudTest`: `updateByObjectViaDataStore` (happy path), `updateOfMissingFactThrows` (negative path), `updateWithComplexArgIsLeftAlone` (rewriter declines complex args).
- Focused unit test of `DataStoreUpdateRewriter` in isolation (no MVEL3, no Drools).

**Out of scope**
- `with`-block compact update syntax (`alerts.update(t{prop = val})`). Tracked separately as #34, blocked on this.
- Property-reactive update with bitmask. DRLX has no property reactivity yet.
- Porting `ConsequenceDataStoreImpl` from upstream. Requires `RuleContext` infrastructure that DRLX doesn't have.
- Generalised symbol-solver-driven type resolution for arbitrary DataStore-typed expressions (e.g. `getStore().update(t)`). YAGNI for the current need.
- Any change to `DataStore` API, drools-ruleunits, or drools-core.

## Architecture

The consequence body is parsed at three points across the pipeline. The DRLX file is parsed by ANTLR once, which extracts the consequence body as raw text. At runtime-build time the new rewriter parses that text with JavaParser to recognise and rewrite DataStore method calls. MVEL3 then parses the rewritten text for actual compilation. Each pass does a distinct job; cost is build-time only.

```
                   DRLX source
                        │
                        ▼
            DrlxToRuleAstVisitor (ANTLR)
                        │
                  ConsequenceIR
                  (raw text body)
                        │
                        ▼
       DrlxRuleAstRuntimeBuilder.build(parseResult)
                        │
                  buildGlobalTypeMap → globalTypes (Map<String, Type>)
                        │
                  dataStoreGlobalNames =
                    globalTypes filtered by DataStore.class.isAssignableFrom(...)
                        │
                        ▼
            for each rule: buildRule
                        │
                        ▼
        DataStoreUpdateRewriter.rewrite(body, dataStoreGlobalNames)   ← NEW
                        │
                  rewritten body
                        │
                        ▼
   lambdaCompiler.createLambdaConsequence(rewrittenBody, types, globalNames)
                        │
                        ▼
                 DrlxLambdaConsequence
                  (compiles via MVEL3)
```

### Cost-control: cheap guards before parse

JavaParser parsing a small block costs ~0.5–2ms warm. With hundreds of rules this matters. Two cheap pre-checks gate the parse:

1. If `dataStoreGlobalNames` is empty → return input unchanged (no DataStore globals at all).
2. For each name, check `body.contains(name + ".update(")`. If none match → return input unchanged.

The `JavaParser` instance is created once per `DrlxRuleAstRuntimeBuilder` (i.e., once per build), not per rule. Cold-start (~20–50ms) is paid once.

Net cost is proportional to *rules with a DataStore-typed `update(` call*, not total rule count.

### Rewrite shape

```
<global>.update(<arg>)
```

becomes

```
<global>.update(
    java.util.Objects.requireNonNull(
        <global>.lookup(<arg>),
        "DataStore '<global>' has no DataHandle for the given fact"),
    <arg>)
```

The inline `Objects.requireNonNull` form keeps the rewrite as a single `MethodCallExpr` substitution — drop-in for any context (statement, lambda, ternary). The error message names the global so the user can identify which store didn't contain the fact.

### Argument hygiene

`<arg>` appears twice in the rewrite. To avoid double-evaluating side-effecting expressions, the rewriter only matches when the argument is a `NameExpr` (variable reference) or `FieldAccessExpr` (e.g. `this.t`). Any other shape — method call, binary expression, lambda — passes through un-rewritten. MVEL3 then surfaces its own missing-method error for `update(T)`. A unit test asserts this restriction.

### Parse-failure behaviour

If JavaParser can't parse the consequence body (malformed Java), the rewriter returns the original text unchanged. MVEL3's parser then surfaces the diagnostic. Single source of truth for syntax errors; no double-error.

## Components

### `DataStoreUpdateRewriter` (new)

```java
package org.drools.drlx.builder;

public final class DataStoreUpdateRewriter {
    private final JavaParser javaParser;

    public DataStoreUpdateRewriter(JavaParser javaParser) { ... }

    /**
     * Rewrite single-arg DataStore.update calls into the handle-aware two-arg form.
     * Returns input unchanged when no rewrite applies, when guards exclude the body,
     * or when JavaParser fails to parse it.
     */
    public String rewrite(String consequenceBody, Set<String> dataStoreGlobalNames);
}
```

Stateless apart from the cached parser. Pure. No I/O. No logging.

### `DrlxRuleAstRuntimeBuilder` (modified)

Two additions in `buildRule()`:
- After `globalTypes` is built: derive `dataStoreGlobalNames` once (filter by `DataStore.class.isAssignableFrom(...)`).
- Before constructing `DrlxLambdaConsequence`: pass the consequence body through `rewriter.rewrite(body, dataStoreGlobalNames)`.

A single `JavaParser` instance is held by `DrlxRuleAstRuntimeBuilder` for the lifetime of the build.

### `DrlxLambdaConsequence` (unchanged)

The class never sees the original text. It receives the already-rewritten body. Keeps the rewrite step cleanly visible at the build site rather than buried in the consequence class.

## Error handling

| Condition | Behaviour |
|-----------|-----------|
| `update(t)` on a fact not in the store | `NullPointerException` from `Objects.requireNonNull` with message naming the global. Propagates through the consequence frame. |
| Argument is not `NameExpr` / `FieldAccessExpr` | Rewriter passes it through. MVEL3 reports its own `cannot resolve method update(T)` error. |
| Body fails JavaParser parse | Rewriter returns input unchanged. MVEL3 reports the syntax error. |
| No DataStore globals on the unit | Rewriter early-returns. No JavaParser work. |
| Body contains no `<global>.update(` substring | Rewriter early-returns. No JavaParser work. |

## Testing

### Integration (in `DataStoreCrudTest`)

| Test | What it verifies |
|------|------------------|
| `updateByObjectViaDataStore` | Add a fact, then `alerts.update(t)` in a consequence. `TestDataObserver` sees an `update` event with the right fact. |
| `updateOfMissingFactThrows` | Call `alerts.update(t)` on a fact never added. Runtime exception (NPE from `requireNonNull`) propagates with a message naming the global. |
| `updateWithComplexArgIsLeftAlone` | `alerts.update(getThing())` (or similar non-NameExpr) → MVEL3 reports its own missing-`update(T)` error. Confirms we don't silently double-evaluate. |

### Unit (new `DataStoreUpdateRewriterTest`)

Pure text-in / text-out. No MVEL3, no Drools.

| Case | Expected |
|------|----------|
| Empty `dataStoreGlobalNames` | input returned unchanged |
| Body with no `update(` | input returned unchanged |
| Simple `alerts.update(t)` | rewritten to two-arg form with `requireNonNull` |
| Body with `this.alerts.update(t)` (FieldAccessExpr) | also rewritten |
| Body with `getStore().update(t)` (chained scope) | left untouched |
| Body with `alerts.update(getThing())` (complex arg) | left untouched |
| Body with multiple matches | all rewritten |
| Malformed Java | input returned unchanged |
| Body where `<global>` is shadowed by a local | rewritten anyway (basic `NameExpr` match, no scope analysis). Test documents the limitation; see Risks. |

## Risks & mitigations

- **Local shadows the global.** A user could write `var alerts = otherStore; alerts.update(t);`. The String pre-check fires; the AST walk would see a `NameExpr "alerts"` regardless of binding. Mitigation: scope analysis in the rewriter to skip if the name is locally re-bound. Optional in this iteration — a test will document behaviour.
- **`lookup(Object)` is on `ListDataStore` impl, not the `DataStore` interface.** All current `DataStore` impls in drools-ruleunits expose `lookup`, but the call resolves via reflection. If a future impl lacks it, the consequence will fail at runtime with a clear `NoSuchMethodError`. Acceptable for now; flagged for future when DRLX gains property reactivity (which would prompt revisiting the wrapper approach anyway).
- **Multi-line / commented `update(` calls evade the cheap String guard.** False negative is fine — they'd be rare and the cost is just MVEL3 reporting the missing-method error. False positive (the substring appears in a comment but not in code) costs one JavaParser parse, which is negligible.

## Migration / compatibility

- No DRLX grammar change.
- No upstream API change.
- No DataStore interface change.
- All existing tests continue to pass (rewriter is a no-op when nothing matches).
