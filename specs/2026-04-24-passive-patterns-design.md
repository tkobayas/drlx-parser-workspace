# Design — Passive patterns (`?/persons[...]`)

**Issue:** #10
**Date:** 2026-04-24
**Scope:** Oopath-only (bound + bare). Group-CE passive (`?and(...)`, `?or(...)`) is deferred to a future issue.

## Motivation

DRLX spec §"Passive elements" (`docs/DRLXXXX.md`) defines `?` as a modifier that makes a pattern passive — the rule only propagates when *prior* (reactive) data is pushed into it. Canonical example:

```
rule R1 {
  var l : /locations[city == "paris"],
  var p : ?/persons[p.locationId == l.id],
  do {...}
}
```

Inserting a new `Person` alone does not wake the rule; it only joins against existing `Location` tuples. Inserting a new `Location` does wake the rule, and the passive pattern is evaluated against all stored `Person` facts.

Drools supports this at the runtime level via `Pattern.setPassive(boolean)` (`drools-base/src/main/java/org/drools/base/rule/Pattern.java:237`). The flag is consumed in `BetaNode.java:172,272,274` — passing `!rightInputIsPassive` to `memory.linkNode(...)` suppresses the "node dirty" notification that would otherwise schedule the rule for firing. Data still flows through the join; only the "wake up the rule" signal is suppressed on the passive side.

This design adds the DRLX grammar, AST, and wiring to propagate that flag end-to-end.

## Scope

**In scope**
- `?` as a prefix on any `oopathExpression`: bound (`var p : ?/persons[...]`) and bare (inside `and/or/not/exists` group CEs).
- Runtime behaviour: `Pattern.setPassive(true)` is set on the corresponding Drools `Pattern`.
- Proto serialisation of the new flag.

**Out of scope (future issues)**
- Group-CE passive (`?and(...)`, `?or(...)`) — pushes passivity down to children; deserves its own spec.
- Passive queries (`?/trusts(a, var b)`) — different subsystem.
- Any DRLX-layer rejection of "meaningless" passive contexts (e.g. `not(?/x)`). Grammar accepts it uniformly; runtime semantics are whatever Drools gives us.

## Grammar

One-line change in `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`:

```antlr
oopathExpression
    : QUESTION? '/' oopathRoot ('/' oopathChunk)*
    ;
```

`QUESTION` is already defined in `JavaLexer.g4:160`. No other rule changes. `?` becomes grammatically legal in every oopath position — by design, because bare passive inside groups (e.g. `and(/locations[...], ?/persons[...])`) is a natural consequence of attaching passivity to the oopath itself rather than to the bound pattern.

## AST / IR

Add a `passive` boolean to `PatternIR` in `DrlxRuleAstModel.java:35`:

```java
public record PatternIR(String typeName,
                        String bindName,
                        String entryPoint,
                        List<String> conditions,
                        String castTypeName,
                        List<String> positionalArgs,
                        boolean passive) implements LhsItemIR {
}
```

Default `false`. All construction sites (both pattern-building helpers in `DrlxToRuleAstVisitor`) pass the extracted flag.

## Protobuf

Add field 7 to `PatternParseResult` in `drlx-parser-core/src/main/proto/drlx_rule_ast.proto:36`:

```proto
message PatternParseResult {
  string type_name = 1;
  string bind_name = 2;
  string entry_point = 3;
  repeated string conditions = 4;
  string cast_type_name = 5;
  repeated string positional_args = 6;
  bool passive = 7;
}
```

Proto3 `bool` defaults to `false` — existing cached serialisations deserialise with `passive=false`, so no cache invalidation required. IR ↔ proto converters are updated to copy the flag both ways.

## Visitor wiring

`DrlxToRuleAstVisitor` has two pattern-building helpers (`buildPatternFromBoundOopath`, `buildPatternFromOopath` — `DrlxToRuleAstVisitor.java:265,273`). Both extract the flag from their `OopathExpressionContext`:

```java
boolean passive = oopathCtx.QUESTION() != null;
```

…and forward it to the `PatternIR` constructor.

## Runtime builder wiring

`DrlxRuleAstRuntimeBuilder.buildPattern()` (`DrlxRuleAstRuntimeBuilder.java:264`) constructs the Drools `Pattern`. One line added after construction:

```java
Pattern pattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0, objectType,
                              parseResult.bindName(), false);
pattern.setPassive(parseResult.passive());   // NEW
pattern.setSource(new EntryPointId(parseResult.entryPoint()));
```

## Fanout check (plan-phase)

Last session's factor-out of `boundOopath` fanned out to unexpected readers (`DrlxToJavaParserVisitor` frozen-parent LSP visitor, `DrlxParserTest` parse-tree assertions) — see `HANDOFF.md` gotcha. This change is **additive** (adding an optional terminal, no restructuring), so existing readers of `OopathExpressionContext` remain correct: they ignore the new optional `QUESTION` terminal.

Before implementing, the plan will still grep for every reader of `OopathExpressionContext` and confirm none depend on token count or positions in a way that would be disturbed by the optional prefix. Expected result: zero impact, but verified rather than assumed.

## Tests

**Parse tests** (`DrlxParserTest` — existing harness, add new cases):
- Bound passive: `var p : ?/persons[age > 18]` parses; oopath context carries `QUESTION`.
- Bare passive inside `and`: `and(/locations[...], ?/persons[...])` parses; inner child oopath carries `QUESTION`.
- Negative: `? var p : /persons[...]` (i.e. `?` in the wrong position) is a parse error.

**IR tests** (existing IR-assertion harness):
- Passive bound → `PatternIR.passive() == true`.
- Passive bare inside group → child `PatternIR.passive() == true`.
- Plain pattern → `passive() == false` (regression guard).

**Protobuf round-trip test**:
- Serialise a `PatternIR` with `passive=true`, deserialise, assert flag preserved.
- Deserialise a proto with field 7 absent, assert `passive == false`.

**Runtime behavioural test** — new file `PassivePatternTest` (follows the `BindingInGroupTest` / `OrBindingScopeTest` pattern from recent sessions). Canonical spec example:

```
rule R1 {
  var l : /locations[city == "paris"],
  var p : ?/persons[p.locationId == l.id],
  do { /* record fire */ }
}
```

Scenarios:
1. Insert `Person` first, no `Location` → rule does not fire.
2. Insert `Location("paris")` (matching) → rule fires for matching `Person` from step 1.
3. After 1+2, insert a second matching `Person` → rule does **not** re-fire (passive side insertion alone does not wake the rule). **This is the test that proves `setPassive(true)` actually took effect.**
4. After 1+2, insert a second matching `Location("paris")` → rule fires for matching `Person` (reactive side insertion wakes the rule).

All existing tests (94 at end of last session) should remain green unchanged.

## Acceptance

- Grammar accepts `?` as prefix on any `oopathExpression`.
- `PatternIR.passive()` reflects the presence of `?`.
- `Pattern.isPassive()` on the built Drools `Pattern` is `true` iff `PatternIR.passive()` was `true`.
- Proto round-trip preserves the flag; proto3 default `false` handles older serialisations.
- Runtime behavioural test demonstrates passive-side insertion does not wake the rule, reactive-side does.
- No existing tests change.

## References

- Issue: https://github.com/tkobayas/drlx-parser/issues/10
- DRLXXXX spec §"Passive elements" — `docs/DRLXXXX.md:437`
- Candidate #7 — `specs/IMPLEMENT_SYNTAX_CANDIDATES.md:20`
- Drools runtime flag — `drools-base/src/main/java/org/drools/base/rule/Pattern.java:233-239`
- Drools consumption site — `drools-core/src/main/java/org/drools/core/reteoo/BetaNode.java:172,272-274`
- Current grammar — `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:121-126`
- Current IR — `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:35`
- Current runtime builder — `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:264`
