# #85 — @Duration/@Timer Rule Annotations

**Issue:** #85 (parent epic: #78)
**Date:** 2026-06-10
**Status:** Design

## Context

DRLX already supports seven rule-level annotations (`@Salience`, `@Description`,
`@DataSource`, `@NoLoop`, `@LockOnActive`, `@Disabled`, `@ActivationGroup`) via a
well-established pipeline: annotation class → visitor resolution → IR model →
protobuf serialisation → runtime application on `RuleImpl`.

DRL exposes two timer-related rule attributes:

- **`timer`** — schedules rule activation on a recurring or delayed basis.
  Supports `int:` (interval) and `cron:` protocols. Maps to `IntervalTimer` or
  `CronTimer` via `RuleBuilder.buildTimer()`.
- **`duration`** — CEP one-shot delayed activation. Maps to
  `DurationTimer(long ms)`.

Both call `RuleImpl.setTimer(Timer)` internally.

RuleUnit's runtime fully supports timer-based rules: `RuleUnitExecutorImpl`
creates a `TimerService`, and the RETE network builds `TimerNode` when
`rule.getTimer() != null`.

## Decisions

1. **Two separate annotations** — `@Timer(String)` for interval/cron scheduling,
   `@Duration(String)` for CEP delayed activation. Matches DRL's two separate
   attributes.
2. **`int:` and `cron:` protocols only** — `expr:` (runtime expression) is out of
   scope; it requires `Declaration` resolution wiring that adds substantial
   complexity. Can be added as a follow-up.
3. **Runtime validation** — the visitor stores the raw string via
   `ArgShape.STRING`. Validation and `Timer` construction happen in
   `DrlxRuleAstRuntimeBuilder`, where we have access to drools'
   `TimeUtils`, `CronExpression`, and `RuleBuilder.buildTimer()`.
4. **Reuse `RuleBuilder.buildTimer()`** — `drools-compiler` is already a
   transitive dependency. The lambda-based overload
   `buildTimer(String, RuleBuildContext, Function<String,TimerExpression>, Consumer<String>)`
   accepts `null` context and custom callbacks, avoiding DRL-specific coupling.
5. **`@Duration` accepts time-string only** — e.g. `@Duration("5s")`,
   `@Duration("1h30m")`. No integer literal form.
6. **`@Timer` + `@Duration` on the same rule is an error** — both map to
   `rule.setTimer()`, so having both is nonsensical. Rejected at visitor level.

## Design

### 1. Annotation Classes

Two new classes in `org.drools.drlx.annotations`, following existing pattern:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Timer {
    String value();
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Duration {
    String value();
}
```

### 2. IR Model

Add to `DrlxRuleAstModel.RuleAnnotationIR.Kind`:

```java
TIMER(ArgShape.STRING),
DURATION(ArgShape.STRING);
```

Add to `DrlxToRuleAstVisitor.SUPPORTED_ANNOTATION_KINDS`:

```java
Map.entry("org.drools.drlx.annotations.Timer", Kind.TIMER),
Map.entry("org.drools.drlx.annotations.Duration", Kind.DURATION),
```

### 3. Protobuf

Add to `AnnotationKind` enum in `drlx_rule_ast.proto`:

```protobuf
ANNOTATION_KIND_TIMER = 11;
ANNOTATION_KIND_DURATION = 12;
```

Add corresponding entries in the proto ↔ Kind mapping code
(`DrlxRuleAstParseResult`).

### 4. Visitor — Mutual Exclusion Check

Extend the existing duplicate-detection logic in
`DrlxToRuleAstVisitor.buildRuleAnnotations()`. The current `EnumSet<Kind> seen`
check already prevents two `@Timer` annotations on the same rule. Add an
additional check: if `seen` contains `TIMER` when processing `DURATION` (or vice
versa), throw an error.

### 5. Runtime Builder

In `DrlxRuleAstRuntimeBuilder.applyAnnotations()`:

**`@Timer` case:**

```java
case TIMER -> {
    Timer timer = RuleBuilder.buildTimer(
        ann.rawValue(),
        null,
        s -> s == null ? null : new ConstantTimerExpression(TimeUtils.parseTimeString(s)),
        err -> { throw new RuntimeException(err); }
    );
    if (timer == null) {
        throw new RuntimeException("Failed to build timer from: " + ann.rawValue());
    }
    rule.setTimer(timer);
}
```

**`@Duration` case:**

```java
case DURATION -> {
    long ms = TimeUtils.parseTimeString(ann.rawValue());
    rule.setTimer(new DurationTimer(ms));
}
```

### 6. Syntax Examples

```java
import org.drools.drlx.annotations.Timer;
import org.drools.drlx.annotations.Duration;

// Interval timer: fires after 1s delay, then every 2s
@Timer("int: 1s 2s")
rule CheckStock {
    Stock s : /stocks[ price > 100 ],
    do { alert(s); }
}

// Cron timer: fires every 5 seconds
@Timer("cron: 0/5 * * * * ?")
rule PeriodicCheck {
    Sensor s : /sensors[ active ],
    do { log(s); }
}

// Interval timer with default protocol (int): delay only
@Timer("500ms")
rule DelayedAction {
    Event e : /events[ type == "alert" ],
    do { notify(e); }
}

// Duration: CEP one-shot delayed activation
@Duration("5s")
rule ExpireEvent {
    Event e : /events[ type == "temp" ],
    do { archive(e); }
}
```

### 7. Testing

Add to `RuleAnnotationsTest`. Use Awaitility (new test-scope dependency:
`org.awaitility:awaitility`) instead of `Thread.sleep`.

**Functional tests:**

- `testIntervalTimerFiresRepeatedly()` — `@Timer("int: 100ms 100ms")`, insert a
  fact, use `await().atMost(2, SECONDS).until(() -> fireCount.get() >= 3)`.
- `testCronTimerFiresOnSchedule()` — `@Timer("cron: 0/1 * * * * ?")`, verify at
  least one delayed fire within a generous window.
- `testDurationDelayedFire()` — `@Duration("200ms")`, verify rule doesn't fire
  immediately (count == 0 right after insert+fireAllRules), then
  `await().atMost(2, SECONDS).until(() -> fireCount.get() >= 1)`.

**Validation tests:**

- `testTimerInvalidProtocolFails()` — `@Timer("xyz: 1s")` → RuntimeException
- `testTimerMissingColonFails()` — `@Timer("int 1s")` → RuntimeException
- `testTimerEmptyStringFails()` — `@Timer("")` → RuntimeException
- `testDurationInvalidTimeStringFails()` — `@Duration("abc")` → RuntimeException
- `testDurationEmptyStringFails()` — `@Duration("")` → RuntimeException

**Argument shape tests:**

- `testTimerWithoutArgumentFails()` — bare `@Timer` → rejected
- `testDurationWithoutArgumentFails()` — bare `@Duration` → rejected

**Mutual exclusion test:**

- `testTimerAndDurationTogetherFails()` — both on same rule → RuntimeException

### 8. Dependencies

Add to `drlx-parser-core/pom.xml`:

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <scope>test</scope>
</dependency>
```

Version managed by the drools BOM or set explicitly if not in the BOM.

## Out of Scope

- `expr:` protocol (runtime expression-based timer) — requires Declaration
  resolution wiring. Separate issue if needed.
- `start`/`end`/`repeat-limit` optional timer parameters — supported by
  `buildTimer()` automatically, but not explicitly tested or documented in v1.
  They work if users include them in the string.
- Calendar-based filtering (`calendars` attribute) — separate feature.
