# Design — Adopt `TrackingAgendaEventListener` for strict rule assertion

**Issue:** #18
**Date:** 2026-04-24
**Scope:** Test infrastructure only. No production code, no grammar, no IR, no proto changes.

## Motivation

Current test assertion style has two problems:

1. **`assertThat(kieSession.fireAllRules()).isEqualTo(N)` asserts a count, not identity.** A test can silently fire the wrong rule and still pass, as long as the count matches. For rules that differ only in salience, annotation, or guard condition, this is a real gap.
2. **Boilerplate duplication.** Ten test files currently attach an anonymous `DefaultAgendaEventListener` to capture rule names into a local `List<String>`. The same 6-line pattern is copy-pasted 12+ times.

Drools 10.2.0 (unreleased at time of writing) ships `org.drools.core.event.TrackingAgendaEventListener` — a reusable listener that records rule names across all 10 agenda events. Until 10.2.0 ships and drlx-parser picks it up, this spec vendors a verbatim copy of the class into `src/test` and migrates the qualifying tests to use it.

## Scope

**In scope**
- Copy `TrackingAgendaEventListener.java` from Drools 10.2.0-rc2 into drlx-parser `src/test`.
- Modify `DrlxBuilderTestSupport.withSession` to auto-attach the listener and expose it to the test lambda.
- Migrate 12 test files that either (a) already use a custom anon listener, or (b) exercise multiple rules end-to-end.
- Record a policy statement in this spec: bare `fireAllRules() == N` is allowed only for single-rule tests.

**Out of scope**
- No new assertion helpers on `TrackingAgendaEventListener` — kept verbatim to minimise diff against upstream and make the 10.2.0 swap a single `rm`.
- No wrapping helper method on `DrlxBuilderTestSupport` beyond the `withSession` signature change.
- Four single-rule smoke-test files (`InlineCastTest`, `TypeInferenceTest`, `PositionalTest`, `DrlxCompilerNoPersistTest`) keep their bare `fireAllRules() == N` assertions.
- No production-code changes.

## Vendored class

Path:

```
drlx-parser-core/src/test/java/org/drools/core/event/TrackingAgendaEventListener.java
```

**Why the upstream package path is preserved.** When drlx-parser upgrades to Drools 10.2.0, the cleanup is a single `rm` — the FQCN `org.drools.core.event.TrackingAgendaEventListener` resolves from `drools-core` with zero import changes in any test. A test-util package (`org.drools.drlx.testutil`) would have required mass find-replace on upgrade.

**Source:** copied verbatim from [10.2.0-rc2](https://github.com/apache/incubator-kie-drools/blob/10.2.0-rc2/drools-core/src/main/java/org/drools/core/event/TrackingAgendaEventListener.java) — tabs, license header, field order all preserved to minimise the diff against upstream.

**Only modification:** a class-level Javadoc explaining why the file lives in `src/test` under a Drools package path. The comment states the temporary-copy rationale, names the target Drools version (10.2.0), and instructs future readers to delete the file on upgrade.

## `DrlxBuilderTestSupport.withSession` signature change

```java
// before
protected static void withSession(String rule, Consumer<KieSession> test) { ... }

// after
protected static void withSession(String rule,
                                  BiConsumer<KieSession, TrackingAgendaEventListener> test) {
    KieBase kieBase = new DrlxRuleBuilder().build(rule);
    KieSession kieSession = kieBase.newKieSession();
    TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
    kieSession.addEventListener(listener);
    try {
        test.accept(kieSession, listener);
    } finally {
        kieSession.dispose();
    }
}
```

The listener is always attached — not opt-in. All 13 current `withSession` callers change their lambda shape from `kieSession -> { ... }` to `(kieSession, listener) -> { ... }`. Tests that do not need rule-name assertions accept the `listener` parameter and ignore it.

## Migration pattern

### The 10 anon-listener files

Files: `AndTest`, `OrTest`, `NestedGroupTest`, `NotExistsBindingTest`, `ExistsTest`, `NotTest`, `BindingInGroupTest`, `PassivePatternTest`, `OrBindingScopeTest`, `RuleAnnotationsTest`.

```java
// before
withSession(rule, kieSession -> {
    final List<String> fired = new ArrayList<>();
    kieSession.addEventListener(new DefaultAgendaEventListener() {
        @Override
        public void afterMatchFired(AfterMatchFiredEvent event) {
            fired.add(event.getMatch().getRule().getName());
        }
    });
    // ... inserts ...
    assertThat(kieSession.fireAllRules()).isEqualTo(1);
    assertThat(fired).hasSize(1);
});

// after
withSession(rule, (kieSession, listener) -> {
    // ... inserts ...
    assertThat(kieSession.fireAllRules()).isEqualTo(1);
    assertThat(listener.getAfterMatchFired()).containsExactly("RuleName");
});
```

Changes per test:
- Drop `import org.kie.api.event.rule.DefaultAgendaEventListener`.
- Drop `import org.kie.api.event.rule.AfterMatchFiredEvent` where it was only used for the anon listener.
- Drop the local `List<String> fired` and the anonymous class.
- **Upgrade** `assertThat(fired).hasSize(N)` to `assertThat(listener.getAfterMatchFired()).containsExactly("Name1", ...)` — the ticket's whole point is asserting *which* rule fired, not just how many.
- **Keep** `assertThat(kieSession.fireAllRules()).isEqualTo(N)` alongside. The `int` return is the cycle boundary for multi-fire tests; the listener's list is the cumulative record.

### The 2 non-`withSession` files

Files: `DrlxRuleBuilderTest` (extends nothing related), `DrlxCompilerTest` (builds `KieSession` directly). Inline pattern:

```java
TrackingAgendaEventListener listener = new TrackingAgendaEventListener();
kieSession.addEventListener(listener);
// ... inserts ...
int fired = kieSession.fireAllRules();
assertThat(fired).isEqualTo(2);
assertThat(listener.getAfterMatchFired()).containsExactly("RuleA", "RuleB");
```

### Multi-fire tests

`NotTest`, `NestedGroupTest`, `AndTest` have fire → insert → fire patterns. The listener accumulates across fires within one session — no `resetAllEvents()` calls. Per-cycle fire count is asserted via `fireAllRules() == N`; the final `listener.getAfterMatchFired()` assertion covers the full ordered sequence.

Example:

```java
assertThat(kieSession.fireAllRules()).isEqualTo(1);  // cycle 1
entryPoint.insert(newFact);
assertThat(kieSession.fireAllRules()).isZero();      // cycle 2: nothing new fires
assertThat(listener.getAfterMatchFired()).containsExactly("OnlyRuleThatEverFired");
```

## Files NOT migrated (policy statement)

Stay as bare `fireAllRules() == N`:

- `InlineCastTest` — single rule per test; tests inline-cast syntax.
- `TypeInferenceTest` — single rule per test; tests type inference from unit class.
- `PositionalTest` — single rule per test; tests positional constructor args.
- `DrlxCompilerNoPersistTest` — no-persist smoke test.

Three of the four (`InlineCastTest`, `TypeInferenceTest`, `PositionalTest`) use `withSession`, so they receive an unused `listener` parameter from the new signature. `DrlxCompilerNoPersistTest` builds its own session.

**Policy for future authors** (record in spec, not in code):

> Bare `fireAllRules() == N` is allowed only for single-rule tests where the rule name adds no signal. Any test that exercises multiple rules, firing order, or rule-specific behaviour must assert on `listener.getAfterMatchFired()`.

## Upgrade path (Drools 10.2.0 cleanup)

When drlx-parser upgrades its Drools dependency to 10.2.0+:

1. `rm drlx-parser-core/src/test/java/org/drools/core/event/TrackingAgendaEventListener.java`.
2. Re-run tests. `TrackingAgendaEventListener` resolves from `drools-core` automatically — same FQCN.
3. No import changes, no test-code changes.

## Test verification

- `mvn -pl drlx-parser-core test` — expect 96 default tests passing (baseline: 96).
- `mvn -pl drlx-parser-core test -Dmvel3.compiler.lambda.persistence=false` — expect 5 no-persist tests passing (baseline: 5).
- Total baseline: 101 passing, 0 failing.
- No new tests are added; this is pure test-infrastructure refactoring. Behavioural coverage is unchanged — same assertions, stricter form.

## Risks

- **`BiConsumer` signature change surfaces any overlooked `withSession` caller as a compile error.** Caught by the compiler, not silent.
- **Upgraded assertions (`hasSize(1)` → `containsExactly("Name")`) may surface latent bugs.** If a test was silently firing the *wrong* rule (same count, different name), it now fails. Treat such findings as real bugs in the production code or test rule, not as migration problems.
- **Unused `listener` parameter in single-rule tests is stylistic noise.** Accepted as a tradeoff for the uniform `withSession` signature; alternative was adding a second overload, which was rejected (YAGNI).

## References

| Topic | Path |
|-------|------|
| Upstream source | [`apache/incubator-kie-drools@10.2.0-rc2/drools-core/src/main/java/org/drools/core/event/TrackingAgendaEventListener.java`](https://github.com/apache/incubator-kie-drools/blob/10.2.0-rc2/drools-core/src/main/java/org/drools/core/event/TrackingAgendaEventListener.java) |
| `withSession` today | `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/DrlxBuilderTestSupport.java:13` |
| Existing anon-listener example | `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java:36-42` |
