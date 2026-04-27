# Constraint property-reactive mask (#19)

**Date:** 2026-04-27
**Issue:** [tkobayas/drlx-parser#19](https://github.com/tkobayas/drlx-parser/issues/19)
**Follow-up to:** #13 (property-reactive watch list)
**Spans repos:** `mvel3-development/mvel` (feature branch `mvel3-read-props`), `mvel3-development/drlx-parser`

## Problem

`DrlxLambdaConstraint` does not override `Constraint.getListenedPropertyMask`. The default implementation returns `AllSetButLastBitMask.get()` ("watch every property"). At the alpha node, this defeats the watch list added in #13: any pattern with an alpha constraint re-fires on modifications to constraint-unrelated properties, regardless of the watch list.

End-user effect: `[][basePay, bonusPay]` works correctly, but `[salary > 0][basePay, bonusPay]` doesn't — modifying `salary` re-fires (correct), but modifying any *other* unrelated property also re-fires (incorrect — should be limited to constraint reads ∪ watch list).

## Reference implementation

`drools-model-compiler/.../LambdaConstraint.java:166-188`. Returns a `BitMask` derived from `evaluator.getReactiveProps()` (compile-time-extracted property names). Drools-model performs the AST walk during its own type-checking pass (`ExpressionTyper.addReactOnPropertyForArgument`) and stores the result on the generated `ConstraintEvaluator`.

DRLX cannot ride along with that pass — DRLX's visitor produces a `PatternIR` carrying constraint expressions as strings, and MVEL3 owns parsing+compilation internally.

## Approach

Two-repo change. **MVEL3 first, DRLX second.**

### MVEL3: surface read properties on `Evaluator`

`MVELTranspiler` already runs `VariableAnalyser` over the parsed AST at line 120-121, *before* the rewrite phase (so bare property names like `salary` are still in their pre-rewrite form). This is the natural extension point.

**Changes:**

1. **`Evaluator` interface** — add a default accessor:
   ```java
   default String[] getReadProperties() { return new String[0]; }
   ```

2. **`VariableAnalyser`** — extend (or add a parallel `ReadPropertyAnalyser`) to collect:
   - `NameExpr` names *not* in `available` declarations — these are free identifiers, which in POJO context represent implicit reads on the context object (e.g. `salary` → property `salary`).
   - `MethodCallExpr` with no scope, no args, JavaBeans getter shape (`getX`/`isX`) — decoded to property name (`getSalary` → `salary`, `isActive` → `active`).
   
   The analyser returns *candidate names* — it does not introspect the POJO type to validate them. Filtering against actual settable properties is the consumer's job.

3. **`MVELTranspiler.transpileContent`** — after running the analyser, generate an override on the generated class:
   ```java
   public String[] getReadProperties() { return new String[]{"salary", "basePay"}; }
   ```

**Why "read properties" not "reactive properties":** "Reactive" is rules-engine vocabulary. MVEL is a general-purpose expression compiler — what it actually exposes is *names the expression reads from its context*. `getReadProperties()` is truthful, leaves `getWrittenProperties()` available for future symmetry (e.g. assignments in block expressions), and decouples from any specific consumer's semantics.

**Tests (MVEL3):** unit tests under `mvel/src/test/java/...` covering:
- Bare property: `salary > 0` → `["salary"]`
- Getter call: `getBasePay() == 5000` → `["basePay"]`
- Boolean getter: `isActive()` → `["active"]`
- Multiple props: `salary > 0 && basePay == 1000` → `["salary", "basePay"]`
- No properties: `1 == 1` → `[]`
- Declaration use only: `(c) -> c > 0` (where `c` is declared) → `[]`

### DRLX: consume it in `DrlxLambdaConstraint`

Once MVEL3 is installed locally, override `getListenedPropertyMask` mirroring `LambdaConstraint:170-188`:

```java
@Override
public BitMask getListenedPropertyMask(Optional<Pattern> pattern,
                                       ObjectType objectType,
                                       List<String> settableProperties) {
    String[] reads = evaluator.getReadProperties();
    if (reads.length == 0) {
        return super.getListenedPropertyMask(pattern, objectType, settableProperties);
    }
    BitMask mask = getEmptyPropertyReactiveMask(settableProperties.size());
    for (String prop : reads) {
        int pos = settableProperties.indexOf(prop);
        if (pos >= 0) { // ignore unsettable props
            mask = mask.set(pos + PropertySpecificUtil.CUSTOM_BITS_OFFSET);
        }
    }
    return mask;
}
```

No changes to grammar, IR, proto, or visitor.

**Tests (DRLX):** three additions to `PropertyReactiveWatchListTest`:

1. `conditionPlusWatchList_firesOnConstraintProp` — `[salary > 0][basePay]`, modify `salary` → re-fires (constraint mask bit for `salary` is set).
2. `conditionPlusWatchList_firesOnWatchProp` — `[salary > 0][basePay]`, modify `basePay` → re-fires (watch list bit for `basePay` is set).
3. `conditionPlusWatchList_doesNotFireOnUnrelated` — `[salary > 0][basePay]`, modify `bonusPay` → does NOT re-fire. **This is the regression case for #19.** Currently fails (alpha mask is AllSet, so `bonusPay` triggers re-eval); with the fix, mask = `{salary, basePay}`, `bonusPay` is excluded.

Plus remove the `// MVP scope:` comment block at lines 12–15 of `PropertyReactiveWatchListTest` — the limitation is gone.

## Components

```
MVEL3 (feature branch: mvel3-read-props)
├── Evaluator                        (interface — add default getReadProperties())
├── VariableAnalyser                 (extend — collect free NameExpr + getter MethodCallExpr)
└── MVELTranspiler.transpileContent  (extend — emit getReadProperties override)

DRLX (#19)
└── DrlxLambdaConstraint
    └── getListenedPropertyMask      (NEW — read evaluator.getReadProperties(), build BitMask)
```

## Data flow

```
Build time:
  DRLX rule source
    → visitor produces PatternIR with constraint expression string
    → DrlxLambdaConstraint(expression, patternType, declarations)
    → MVEL.compilePojoEvaluator
      → MVELTranspiler.transpileContent
        → VariableAnalyser (NEW: collect free NameExpr + getter MethodCallExpr)
        → emit getReadProperties() override in generated class
      → Evaluator instance with populated getReadProperties()

Runtime alpha-mask plumbing:
  AlphaNode constructor
    → constraint.getListenedPropertyMask(pattern, type, settableProps)
      → evaluator.getReadProperties()    (was: default returning [])
      → build property-specific BitMask  (was: AllSetButLastBitMask)
    → AlphaNode.declaredMask shrinks
    → AbstractTerminalNode.inferredMask = α.declared ∪ rtn.declared
      now bounded by actual property reads ∪ watch list
```

## Scope

**In scope (v1):**
- Bare property reads (`salary`)
- No-arg getter calls on context (`getSalary()`, `isActive()`)
- Multi-property expressions (`a > 0 && b == 1`)

**Out of scope (deferred to follow-ups):**
- Nested property access (`address.city` — would yield only `address`, mirroring drools-model)
- Method calls with arguments (e.g. `getName().contains(x)` — the `getName()` part should count, but recursing into method args is more complex)
- Explicit `@Watch`-style override / `getReactivityBitMask()` (drools-model has this; DRLX has no syntactic counterpart yet)
- Aliased property access via the pattern variable (`e.salary` where `e` is the binding) — see Open question

## Error handling

- **MVEL3 analyser** — if the AST is malformed, it would have failed parsing already; no defensive paths needed in the analyser itself.
- **DRLX `getListenedPropertyMask`** — if `getReadProperties()` returns `[]` (no reads found, or evaluator predates the MVEL3 change), fall back to `super.getListenedPropertyMask(...)` (preserves current AllSetButLastBitMask behavior — important for safety: a constraint like `1 == 1` shouldn't suddenly listen to nothing).
- **Properties referenced but not in `settableProperties`** — silently filtered (matches `LambdaConstraint:182-184`).

## Open questions

1. **Aliased property access in DRLX.** Can a user write `[e.salary > 0]` where `e` is the pattern binding? If yes, the MVEL3 analyser would see this as a `FieldAccessExpr`, not a `NameExpr`, and miss it under v1 scope. Need to confirm by inspecting existing DRLX tests / grammar before locking the analyser scope. If aliased form is supported, add `FieldAccessExpr` where the scope NameExpr is the binding to the v1 collection.

2. **Proto round-trip path.** `DrlxLambdaConstraint`'s second constructor (line 39) takes a pre-compiled evaluator. After proto deserialization, does the runtime use this constructor, or does it re-compile from the expression? If pre-compiled, the new `getReadProperties()` must round-trip through whatever serialization MVEL3 uses for the evaluator class. Need to confirm during plan stage.

3. **Issue tracking for the MVEL3 side.** Should the MVEL3 work be tracked as its own issue (in some MVEL3 tracker, or as a referenced sub-task under #19), or rolled into #19 commits with `Refs #19`? Defer to user during plan stage.

## Non-goals

- Full `ExpressionTyper` parity (out of scope — would require porting drools-model's type-resolution logic into MVEL3, which is a much larger investment than #19 needs).
- Touching the visitor or IR. The MVEL3-side extraction makes IR-level reactive-prop plumbing unnecessary.

## Effort estimate

- **MVEL3 changes:** ~half day. ~80-100 LOC including tests.
- **DRLX changes (#19):** ~half day. ~30 LOC change (override + 3 tests + remove MVP comment).

## Success criteria

- All existing tests in `drlx-parser-core` pass (108 currently).
- Three new tests in `PropertyReactiveWatchListTest` pass: condition + watch list combinations correctly bound the alpha mask.
- New MVEL3 unit tests for `getReadProperties()` pass.
- The `// MVP scope:` limitation comment in `PropertyReactiveWatchListTest` can be removed.
