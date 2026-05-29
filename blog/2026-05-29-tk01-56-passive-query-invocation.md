---
layout: post
title: "#56 — passive query invocation: one boolean and a test that proved nothing"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [queries, passive, openQuery, epic-77]
---

# #56 — passive query invocation: one boolean and a test that proved nothing

The `?/` prefix already works for regular patterns — `?/persons[age > 18]` sets the pattern as passive so it doesn't wake the rule on its own insertions. Query invocations should work the same way: `?/personsByAge(25, var p)` should create a `QueryElement` with `openQuery=false`. Turns out, the entire pipeline already handled it — except for one hardcoded `true`.

## The pipeline was already there

I asked Claude to trace the `?` prefix through the full stack. The grammar already has `QUESTION? '/'` in `oopathExpression`. The visitor already checks `ctx.oopathExpression().QUESTION() != null` and writes it into `PatternIR.passive`. For regular patterns, the runtime builder already calls `pattern.setPassive(parseResult.passive())`.

But `buildQueryElement()` had this:

```java
return new QueryElement(
        resultPattern,
        targetQuery.getName(),
        arguments,
        varIndexArray,
        requiredDeclarations.toArray(new Declaration[0]),
        true,    // hardcoded — always reactive
        false);
```

The fix: `true` becomes `!patternIr.passive()`. That's it.

## A validation guard that can't fire yet

The DRLXXXX spec says "an exception will be thrown if a passive query is called and an agenda 'do' is found." Queries don't support `do` blocks yet — `buildQuery()` never calls `setConsequence()` on `QueryImpl`. But the guard goes in now so it's already waiting when query consequences land:

```java
if (patternIr.passive() && targetQuery.getConsequence() != null) {
    throw new RuntimeException(
            "Cannot passively invoke query '" + targetQuery.getName()
            + "': the query has an agenda-based 'do' block ...");
}
```

## The test that proved nothing

The first test draft inserted a person matching the query, called `fireAllRules()`, and asserted zero firings. It looked right. But the rule also had a reactive pattern on `/locations` — and no location was inserted. Without the reactive side having data, the rule can't match regardless of whether the query is passive or active. The assertion passed trivially. It proved nothing about passivity.

The fix mirrors what `PassivePatternTest` does: insert the reactive side first, so a complete match *exists* in memory, then insert on the passive side and verify the rule still doesn't fire. That's the proof — the match is there, but the passive insertion doesn't wake it.

```java
// 1. Reactive side first — no query match yet, no fire
locations.insert(new Location("paris", "centre"));
assertThat(kieSession.fireAllRules()).isEqualTo(0);

// 2. Passive query side — complete match exists now,
//    but passive insertion MUST NOT wake R1
persons.insert(new Person("Alice", 30));
assertThat(kieSession.fireAllRules()).isEqualTo(0);
```

The contrast test reverses the order: passive side first (no fire), then reactive side triggers and picks up the pending query results.

## What landed

| Item | Detail |
|------|--------|
| Commit | `d0f1f81` on `main` |
| Files | `DrlxRuleAstRuntimeBuilder.java`, `QueryTest.java` |
| Tests | 2 new (passive doesn't wake, reactive does wake) |
| Issue | Closes #56, first item in epic #77 |
