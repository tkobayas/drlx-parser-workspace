---
layout: post
title: "#86 — date gates land quietly"
date: 2026-06-10
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [annotations, epic-78, date-effective, date-expires]
---

# #86 — date gates land quietly

`@DateEffective` and `@DateExpires` — calendar-based rule activation windows. A rule annotated with `@DateEffective("2025-06-01")` only fires after that date; `@DateExpires("2025-12-31")` only fires before it. Both together create a window. I picked this up as the next #78 sub-issue after `@Timer`/`@Duration` shipped yesterday.

## ISO-8601, not the legacy format

Traditional DRL uses `d-MMM-yyyy` — `"01-Jan-2025"` — parsed by `DateUtils` with locale-dependent month names. I wanted something cleaner for DRLX. Claude proposed three options: reuse `DateUtils`, go ISO-8601 (`yyyy-MM-dd`), or support both. ISO-8601 was the obvious choice — unambiguous, locale-independent, and `LocalDate.parse()` validates it natively with no custom formatter needed.

The conversion to Drools' `Calendar`-based API is one line:

```java
case DATE_EFFECTIVE -> {
    LocalDate date = LocalDate.parse(ann.rawValue());
    Calendar cal = GregorianCalendar.from(
            date.atStartOfDay(ZoneId.systemDefault()));
    rule.setDateEffective(cal);
}
```

Malformed strings throw `DateTimeParseException` at build time. The legacy format `"01-Jan-2025"` fails too — intentionally.

## The proto file the plan forgot

Adding `Kind` enum entries looks like a one-file change. It isn't. The `DrlxRuleAstParseResult` class has exhaustive switch expressions for protobuf serialization — one mapping `Kind → AnnotationKind`, the other mapping back. The proto file itself needs new enum values. The compiler caught it instantly (`the switch expression does not cover all possible input values`), but the plan had listed three files to change and the actual count was five. Future annotation work: model, proto, parse result, visitor, runtime builder.

## Pseudo clock for deterministic date tests

The first cut of runtime tests used far-past and far-future dates (`"2020-01-01"` and `"2099-01-01"`) to avoid clock dependence. They worked, but I wanted deterministic control. `PseudoClockScheduler.setStartupTime()` sets the session clock to an absolute epoch millis — combine it with a small helper and the tests read naturally:

```java
PseudoClockScheduler clock = (PseudoClockScheduler) kieSession.getSessionClock();
clock.setStartupTime(toEpochMillis("2025-07-01"));
```

Seven runtime tests cover: after/before effective date, after/before expiry date, inside a window, before a window, after a window. All use fixed dates with no system clock dependency.

## What landed

| Item | Detail |
|------|--------|
| Annotations | `@DateEffective`, `@DateExpires` |
| Format | ISO-8601 date-only (`yyyy-MM-dd`) |
| Files | 8 changed (+501 lines) |
| Tests | 16 new (399 total), 0 failures |
| Commits | 2 on `main` |
