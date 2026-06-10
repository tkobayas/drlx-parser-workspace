# @DateEffective / @DateExpires Rule Annotations

**Issue:** #86 (parent epic #78)
**Status:** Approved, ready for implementation
**Date:** 2026-06-10

## Summary

Add `@DateEffective` and `@DateExpires` annotations to control calendar-based rule activation windows. A rule annotated with `@DateEffective` only fires after the specified date; one with `@DateExpires` only fires before the specified date. Both can be combined to create a date window.

## Syntax

```java
import org.drools.drlx.annotations.DateEffective;
import org.drools.drlx.annotations.DateExpires;

@DateEffective("2025-06-01")
rule SummerStart {
    // fires only on or after 2025-06-01
}

@DateExpires("2025-12-31")
rule YearEnd {
    // fires only before 2025-12-31
}

@DateEffective("2025-06-01")
@DateExpires("2025-12-31")
rule SeasonalDiscount {
    // fires only within the date window
}
```

**Date format:** ISO-8601 date-only (`yyyy-MM-dd`). No time-of-day component — activation starts at midnight (start of day) in the system default time zone.

## Design decisions

### ISO-8601 over legacy DRL format

Traditional DRL uses `d-MMM-yyyy` (e.g. `"01-Jan-2025"`) via `DateUtils.parseDate()`, which is locale-dependent. DRLX uses ISO-8601 (`yyyy-MM-dd`) because it is unambiguous, locale-independent, and what modern Java developers expect. We parse via `java.time.LocalDate.parse()` which validates ISO-8601 format natively.

### No mutual exclusion

Unlike `@Timer`/`@Duration` (which are mutually exclusive), `@DateEffective` and `@DateExpires` are independent and complementary. No constraint is enforced between them. If a user sets effective > expires, the rule simply never fires — same as Drools' existing behavior.

### No compile-time date validation

We validate the date string format at parse time but do not check whether the date is in the past/future or whether effective < expires. These are legitimate use cases (e.g. a rule that has already expired, or testing with pseudo clock).

## Components

### 1. Annotation classes

Two new annotations in `org.drools.drlx.annotations`:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DateEffective {
    String value();
}
```

Same pattern for `DateExpires`. Follows the exact same template as `@Timer`, `@Duration`, etc.

### 2. IR model — Kind enum

Add to `RuleAnnotationIR.Kind` in `DrlxRuleAstModel.java`:

```java
DATE_EFFECTIVE(ArgShape.STRING),
DATE_EXPIRES(ArgShape.STRING),
```

### 3. Visitor wiring — DrlxToRuleAstVisitor

- Add FQN constants: `DATE_EFFECTIVE_FQN`, `DATE_EXPIRES_FQN`
- Add entries to `SUPPORTED_ANNOTATION_KINDS` map
- Update the error message in `resolveKind()` to list the new annotations
- No mutual exclusion constraint needed

### 4. Runtime application — DrlxRuleAstRuntimeBuilder

Add cases to `applyAnnotations()`:

```java
case DATE_EFFECTIVE -> {
    LocalDate date = LocalDate.parse(ann.rawValue());
    Calendar cal = GregorianCalendar.from(
        date.atStartOfDay(ZoneId.systemDefault()));
    rule.setDateEffective(cal);
}
case DATE_EXPIRES -> {
    LocalDate date = LocalDate.parse(ann.rawValue());
    Calendar cal = GregorianCalendar.from(
        date.atStartOfDay(ZoneId.systemDefault()));
    rule.setDateExpires(cal);
}
```

`LocalDate.parse()` throws `DateTimeParseException` on malformed input, which surfaces at rule build time.

The Drools runtime (`RuleImpl.isEffective()`) already checks `dateEffective`/`dateExpires` using `Calendar.getInstance()` with `valueResolver.getCurrentTime()` — no runtime changes needed.

## Testing strategy

### Parse-level tests
- `@DateEffective("2025-01-01")` extracts `Kind.DATE_EFFECTIVE` with raw value `"2025-01-01"`
- `@DateExpires("2025-12-31")` extracts `Kind.DATE_EXPIRES` with raw value `"2025-12-31"`
- Both annotations together on one rule
- FQN usage without import

### Error tests
- Missing argument: `@DateEffective` without `("...")` fails
- Empty string: `@DateEffective("")` fails
- Malformed date: `@DateEffective("not-a-date")` fails at build time
- Invalid format: `@DateEffective("01-Jan-2025")` fails (legacy format rejected)
- Duplicate: two `@DateEffective` on same rule fails

### Runtime tests (pseudo clock)
- **Before effective date:** rule does not fire
- **After effective date:** rule fires
- **Before expiry date:** rule fires
- **After expiry date:** rule does not fire
- **Within date window:** both annotations, rule fires
- **Outside date window:** both annotations, rule does not fire

All runtime tests use `PseudoClockScheduler` with `ClockTypeOption.get(ClockType.PSEUDO_CLOCK)` to set deterministic time, following the pattern established in `@Timer`/`@Duration` tests.

## Files changed

| File | Change |
|------|--------|
| `annotations/DateEffective.java` | New — annotation definition |
| `annotations/DateExpires.java` | New — annotation definition |
| `DrlxRuleAstModel.java` | Add `DATE_EFFECTIVE`, `DATE_EXPIRES` to `Kind` enum |
| `DrlxToRuleAstVisitor.java` | Add FQN constants, map entries, update error message |
| `DrlxRuleAstRuntimeBuilder.java` | Add switch cases in `applyAnnotations()` |
| `RuleAnnotationsTest.java` | Add parse, error, and runtime tests |

## Scope

**In scope:** `@DateEffective` and `@DateExpires` with ISO-8601 date-only strings.

**Out of scope:** Date-time support (`yyyy-MM-ddTHH:mm`), `@Calendars` annotation, `expr:` protocol for dynamic dates.
