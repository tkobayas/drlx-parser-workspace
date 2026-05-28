# #43 Temporal Operators (CEP) — Design Spec

**Issue:** #43 — Pluggable operators (temporal)
**Epic:** #26 — DrlxCompiler enhancement round 2
**Date:** 2026-05-28

## Overview

Add temporal operator support for CEP (Complex Event Processing) in DRLX
constraints. Temporal operators express Allen interval algebra relationships
between two events — e.g., "event B happened after event A."

## DRLX Syntax

Temporal constraints appear inside pattern brackets, alongside regular
constraints:

```java
rule R1 {
    var a : /as,
    var b : /bs[this after a],
    do { ... }
}
```

### Supported Forms

```java
// Default range [1ms, Long.MAX_VALUE]
var b : /bs[this after a],

// Custom range — parsed as duration strings (same as windows)
var b : /bs[this after[3m, 4m] a],

// Single parameter — range [value, Long.MAX_VALUE]
var b : /bs[this after[3m] a],

// Negated
var b : /bs[this not after a],

// Mixed with regular constraints
var b : /bs[this after a, name == "x"],
```

### Operators

All 13 Allen interval algebra temporal operators:

| Operator | Meaning |
|----------|---------|
| `after` | this started after other ended |
| `before` | this ended before other started |
| `coincides` | events overlap in time |
| `during` | this occurs entirely during other |
| `finishes` | same end time, this starts later |
| `finishedby` | same end time, other starts later |
| `includes` | other occurs entirely during this |
| `meets` | this ends exactly when other starts |
| `metby` | other ends exactly when this starts |
| `overlaps` | events overlap, this starts first |
| `overlappedby` | events overlap, other starts first |
| `starts` | same start time, this ends first |
| `startedby` | same start time, other ends first |

### Constraints

- Both patterns must be event types (`@Role(Role.Type.EVENT)`)
- Left side is always `this` (whole-event form)
- Right side is a bound variable name referencing a previous pattern
- Parameters are duration strings parsed by `TimeUtils.parseTimeString()`

### Out of Scope (v1)

- Field-level temporal comparisons (e.g., `timestamp before now-1m`)
- Fuzzy operators (`~is`) — tracked in #66
- 4-parameter form for `during`/`includes` operators

## Grammar Changes

### DrlxParser.g4

Add a generic `customConstraint` rule that is extensible for future operator
types (fuzzy, etc.):

```antlr
drlxExpression
    : identifier ':' expression
    | customConstraint          // NEW — before 'expression'
    | expression
    ;

customConstraint
    : (THIS | identifier) NOT? customOp expression
    ;

customOp
    : identifier ('[' customOpParams ']')?
    ;

customOpParams
    : ~']'+
    ;
```

**Ambiguity analysis:** `(THIS | identifier)` on the left side is completely
unambiguous — `this after a` and `identifier not before b` are never valid
Java/MVEL3 expressions (no infix word-operators exist). ANTLR4 picks
`customConstraint` over `expression` via ordered alternatives.

The operator name is an `identifier` validated at visitor level. This avoids
adding 13 lexer keywords that would conflict with user variable names.

## IR Changes

### DrlxRuleAstModel.java

New record:

```java
public record TemporalConditionIR(
    String operator,           // "after", "before", etc.
    boolean negated,           // true when "not" prefix
    List<String> parameters,   // ["3m", "4m"] or empty list
    String rightBinding        // bound variable name, e.g. "a"
) {}
```

Extend `PatternIR` with one new field:

```java
public record PatternIR(
    String typeName,
    String bindName,
    String entryPoint,
    List<String> conditions,
    List<TemporalConditionIR> temporalConditions,  // NEW
    String castTypeName,
    List<String> positionalArgs,
    boolean passive,
    List<String> watchedProperties,
    String windowType,
    String windowParameter
) implements LhsItemIR {}
```

## Visitor Changes (DrlxToRuleAstVisitor)

When extracting conditions from a pattern's constraint brackets:

1. Check if the `drlxExpression` parse tree node is a `customConstraint`
2. If yes:
   - Validate the operator identifier is one of the 13 temporal operators
   - Extract negation (presence of `NOT` token)
   - Parse parameters from `customOpParams` (split by `,`, trim whitespace)
   - Extract right expression text as the binding name
   - Add to `temporalConditions` list
3. If no: process as regular string condition (existing path)

### Temporal Operator Set

```java
private static final Set<String> TEMPORAL_OPERATORS = Set.of(
    "after", "before", "coincides", "during",
    "finishes", "finishedby", "includes",
    "meets", "metby", "overlaps", "overlappedby",
    "starts", "startedby"
);
```

Unknown operators in `customConstraint` position throw a parse error with a
clear message. This set can grow when fuzzy operators are added (#66).

### Visitor Validation for Temporal Operators

For temporal operators specifically (the only custom operators supported in v1):

- **Left side must be `this`**: The grammar allows `identifier` on the left for
  future extensibility, but the visitor rejects non-`this` left-hand sides for
  temporal operators. When fuzzy operators are added, `identifier` on the left
  will be valid for those.
- **Right side must be a bound variable name**: The grammar uses `expression`
  for generality, but the visitor validates that the right side is a simple
  identifier that resolves to a previously bound variable.

## Runtime Constraint — DrlxTemporalConstraint

New class, structurally similar to `DrlxLambdaBetaConstraint`:

```java
public class DrlxTemporalConstraint
        extends MutableTypeConstraint<ContextEntry> {

    private final TemporalPredicate temporalPredicate;
    private final Declaration[] requiredDeclarations;

    // Constructor, getters ...

    @Override
    public boolean isTemporal() { return true; }

    @Override
    public ConstraintType getType() { return ConstraintType.BETA; }

    @Override
    public boolean isAllowedCachedLeft(ContextEntry context, FactHandle handle) {
        DrlxBetaContextEntry ctx = (DrlxBetaContextEntry) context;
        EventHandle thisEvent = (EventHandle) handle;
        EventHandle otherEvent = (EventHandle) ctx.tuple.get(requiredDeclarations[0]);

        return temporalPredicate.evaluate(
            thisEvent.getStartTimestamp(),
            thisEvent.getDuration(),
            thisEvent.getEndTimestamp(),
            otherEvent.getStartTimestamp(),
            otherEvent.getDuration(),
            otherEvent.getEndTimestamp()
        );
    }

    @Override
    public boolean isAllowedCachedRight(BaseTuple tuple, ContextEntry context) {
        DrlxBetaContextEntry ctx = (DrlxBetaContextEntry) context;
        EventHandle thisEvent = (EventHandle) ctx.handle;
        EventHandle otherEvent = (EventHandle) tuple.get(requiredDeclarations[0]);

        return temporalPredicate.evaluate(
            thisEvent.getStartTimestamp(),
            thisEvent.getDuration(),
            thisEvent.getEndTimestamp(),
            otherEvent.getStartTimestamp(),
            otherEvent.getDuration(),
            otherEvent.getEndTimestamp()
        );
    }

    @Override
    public ContextEntry createContext() {
        return new DrlxLambdaBetaConstraint.DrlxBetaContextEntry();
    }

    // clone(), replaceDeclaration(), serialization stubs ...
}
```

**Key design points:**
- Returns `true` from `isTemporal()` — this is critical for the RETE network
  to handle temporal windows, event expiration, and negative CE delay correctly
- Reuses `DrlxBetaContextEntry` (no new context entry needed)
- Casts `FactHandle` to `EventHandle` (kie-api interface) for timestamp access
- Position 1 in `evaluate()` = `this` (current pattern), position 2 = other
  binding

## Temporal Predicate Factory

Static factory that maps operator name + parameters to the correct drools
`TemporalPredicate` implementation:

```java
static TemporalPredicate create(String operator, boolean negated,
                                 List<String> params) {
    long min = 1, max = Long.MAX_VALUE;
    if (params.size() == 1) {
        min = TimeUtils.parseTimeString(params.get(0));
    } else if (params.size() >= 2) {
        min = TimeUtils.parseTimeString(params.get(0));
        max = TimeUtils.parseTimeString(params.get(1));
    }

    TemporalPredicate pred = switch (operator) {
        case "after"        -> createAfterOrBefore(AfterPredicate.class, min, max, params);
        case "before"       -> createAfterOrBefore(BeforePredicate.class, min, max, params);
        // ... all 13 operators mapped to their predicate classes
        default -> throw new IllegalArgumentException("Unknown temporal operator: " + operator);
    };

    return negated ? pred.negate() : pred;
}
```

Each predicate class has public constructors for 0, 1, and 2 duration
parameters using `TimeUnit.MILLISECONDS`. The exact constructor signatures
vary slightly between predicate types (e.g., `MeetsPredicate` only accepts a
single threshold). The factory validates parameter count per operator and
throws a clear error for unsupported parameter counts.

## Runtime Builder Integration

In `DrlxRuleAstRuntimeBuilder.buildPattern()`, after processing regular
conditions:

```java
for (TemporalConditionIR tc : parseResult.temporalConditions()) {
    BoundVariable ref = boundVariables.get(tc.rightBinding());
    if (ref == null) {
        throw new RuntimeException(
            "Temporal constraint references unknown binding '" + tc.rightBinding() + "'");
    }
    // Validate both patterns are events
    validateEventType(pattern, tc.operator());
    validateEventType(ref.pattern(), tc.operator());

    TemporalPredicate predicate = TemporalPredicateFactory.create(
        tc.operator(), tc.negated(), tc.parameters());
    pattern.addConstraint(
        new DrlxTemporalConstraint(predicate, new Declaration[] { ref.declaration() }));
}
```

**Event type validation:** Checks that both patterns' `ObjectType.isEvent()`
returns `true`. This catches misuse at build time with a clear error:
"Temporal operator 'after' requires event types (@Role(Type.EVENT))".

## Validation & Error Handling

| Check | When | Error |
|-------|------|-------|
| Unknown operator name | Visitor (parse time) | "Unknown custom operator 'xyz'. Supported temporal operators: after, before, ..." |
| Non-event pattern type | Runtime builder | "Temporal operator 'after' requires event types (@Role(Type.EVENT)) but 'Person' is not an event" |
| Unknown right-hand binding | Runtime builder | "Temporal constraint references unknown binding 'a'" |
| Right-hand binding not an event | Runtime builder | Same event-type error |

## Testing Strategy

### Grammar/Visitor Tests

- Parse `this after a` → verify IR: operator="after", negated=false, params=[], rightBinding="a"
- Parse `this after[3m, 4m] a` → verify params=["3m", "4m"]
- Parse `this not before b` → verify negated=true
- Parse mixed: `this after a, name == "x"` → 1 temporal + 1 regular condition
- Error: unknown operator → clear error message

### Runtime Integration Tests

Using pseudo-clock and `@Role(Role.Type.EVENT)` test types (same infrastructure
as `WindowTest`):

- `this after a`: insert a, advance clock, insert b → rule fires
- `this after a`: insert a, insert b immediately → rule does not fire
- `this after[3m, 4m] a`: insert a, advance 3.5m, insert b → fires
- `this after[3m, 4m] a`: insert a, advance 1m, insert b → does not fire
- `this not after a`: insert a, insert b immediately → fires
- `this before a`: reverse ordering → fires
- At least one test per operator to verify correct wiring

### Edge Cases

- Multiple temporal constraints on same pattern: `[this after a, this before c]`
- Temporal + regular constraints combined
- Temporal constraint with window: `var b : /bs[this after a] | time[5s]`

## Dependencies

All required classes are already in the classpath (transitive via
`drools-ruleunits-impl`):

- `org.kie.api.runtime.rule.EventHandle` — kie-api
- `org.drools.core.common.DefaultEventHandle` — drools-core
- `org.drools.model.functions.temporal.*Predicate` — drools-canonical-model
- `org.drools.base.time.TimeUtils` — drools-base

No new Maven dependencies needed.

## Files to Modify

| File | Change |
|------|--------|
| `DrlxParser.g4` | Add `customConstraint`, `customOp`, `customOpParams` rules |
| `DrlxRuleAstModel.java` | Add `TemporalConditionIR` record; extend `PatternIR` |
| `DrlxToRuleAstVisitor.java` | Handle `customConstraint` in condition extraction |
| `DrlxRuleAstRuntimeBuilder.java` | Process temporal conditions, create `DrlxTemporalConstraint` |

## New Files

| File | Purpose |
|------|---------|
| `DrlxTemporalConstraint.java` | Runtime temporal constraint (extends `MutableTypeConstraint`) |
| `TemporalPredicateFactory.java` | Maps operator name + params → drools `TemporalPredicate` |
