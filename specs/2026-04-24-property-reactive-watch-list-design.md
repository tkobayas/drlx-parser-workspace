# Property-reactive watch list — Design

**Issue:** #13 · **Parent epic:** #4 · **Candidate:** #10 in `specs/IMPLEMENT_SYNTAX_CANDIDATES.md`

## Goal

Add a DRLX pattern syntax for the property-reactive watch list — a second `[...]` block on the OOPath root that names the properties whose modification should re-fire the rule. Match Drools DRL7 semantics (the `@Watch` annotation) minus the `@PropertyReactive` class-level check.

## Syntax

```
oopathRoot
    : identifier (HASH identifier)?
      ('(' expression (',' expression)* ')')?                           // positional
      ('[' (conds+=drlxExpression (',' conds+=drlxExpression)*)? ']')?  // conditions — now accepts []
      ('[' watches+=watchItem (',' watches+=watchItem)* ']')?           // watch list — NEW
    ;

watchItem
    : '*'
    | '!' '*'
    | '!'? identifier
    ;
```

Two grammar edits:

1. **First `[]` block accepts empty** (`(drlxExpression ...)?`) so `[][basePay]` (watch-only) parses.
2. **New second `[]` block** holds the watch list.

Applies uniformly to every `oopathRoot` — top-level bound patterns and patterns nested inside `not` / `exists` / `and` / `or`.

### Accepted forms

```
/employees[basePay]                          // conditions, no watch
/employees[salary > 0][basePay, bonusPay]    // conditions + watch
/employees[][basePay, !bonusPay]             // watch only
/employees[][*]                              // watch all
/employees[][!*]                             // watch none
Employee e: /employees[salary > 0][basePay]  // bound, top-level
not /employees[status == "x"][status]        // inside NOT
```

## Architecture

Five layers are touched, each change local to a single file:

| Layer | File | Change |
|-------|------|--------|
| Grammar | `DrlxParser.g4` | Allow empty first block; add second block with `watchItem` |
| IR | `DrlxRuleAstModel.java` | `PatternIR` gains `List<String> watchedProperties` |
| Proto | `drlx_rule_ast.proto` | `PatternParseResult` gains `repeated string watched_properties = 8` |
| Visitor | `DrlxToRuleAstVisitor.java` | Extract watch-list strings from `root.watches` |
| Runtime | `DrlxRuleAstRuntimeBuilder.java` | Validate + call `pattern.addWatchedProperties` |
| Parse result | `DrlxRuleAstParseResult.java` | Read/write `watched_properties` |

Data-flow: grammar tokens → visitor emits raw `List<String>` into `PatternIR` → runtime builder validates against accessible properties and calls Drools' `Pattern.addWatchedProperties`.

### IR

```java
public record PatternIR(String typeName,
                        String bindName,
                        String entryPoint,
                        List<String> conditions,
                        String castTypeName,
                        List<String> positionalArgs,
                        boolean passive,
                        List<String> watchedProperties) implements LhsItemIR {
}
```

Watch entries stored as raw strings (`"basePay"`, `"!bonusPay"`, `"*"`, `"!*"`) — matches what `Pattern.addWatchedProperties` expects. Empty list = no watch list in source.

### Proto

```proto
message PatternParseResult {
  string type_name = 1;
  string bind_name = 2;
  string entry_point = 3;
  repeated string conditions = 4;
  string cast_type_name = 5;
  repeated string positional_args = 6;
  bool passive = 7;
  repeated string watched_properties = 8;   // NEW
}
```

Proto3 field 8, non-breaking.

### Visitor

Grammar-level labels (`conds+=`, `watches+=`) make the two `[]` blocks unambiguous:

```java
private static List<String> extractWatchedProperties(DrlxParser.OopathExpressionContext ctx) {
    DrlxParser.OopathRootContext root = ctx.oopathRoot();
    if (root == null || root.watches == null || root.watches.isEmpty()) {
        return List.of();
    }
    return root.watches.stream()
            .map(ParserRuleContext::getText)
            .toList();
}
```

The existing `extractConditions` switches its root path from `root.drlxExpression()` to `root.conds` (same nodes, relabelled).

No validation in the visitor — raw strings flow through as-is.

### Runtime builder

`buildPattern` adds one block after constraint construction, before `return pattern`:

```java
if (!parseResult.watchedProperties().isEmpty()) {
    List<String> validated = validateWatchedProperties(
            parseResult.watchedProperties(), patternClass, parseResult.typeName());
    pattern.addWatchedProperties(validated);
}
```

### Validation

Mirrors `PatternBuilder.processListenedPropertiesAnnotation` (drools-compiler) minus the `@PropertyReactive` class check:

```java
private static List<String> validateWatchedProperties(List<String> raw,
                                                      Class<?> patternClass,
                                                      String typeLabel) {
    List<String> accessible = ClassUtils.getAccessibleProperties(patternClass);
    List<String> result = new ArrayList<>();
    for (String item : raw) {
        if (item.equals("*") || item.equals("!*")) {
            if (result.contains("*") || result.contains("!*")) {
                throw new RuntimeException("Duplicate usage of wildcard " + item
                        + " in watch list on " + typeLabel);
            }
            result.add(item);
            continue;
        }
        boolean neg = item.startsWith("!");
        String name = neg ? item.substring(1) : item;
        if (!accessible.contains(name)) {
            throw new RuntimeException("Unknown property '" + name
                    + "' in watch list on " + typeLabel);
        }
        if (result.contains(name) || result.contains("!" + name)) {
            throw new RuntimeException("Duplicate property '" + name
                    + "' in watch list on " + typeLabel);
        }
        result.add(item);
    }
    return result;
}
```

Accessible-properties source: `org.drools.util.ClassUtils.getAccessibleProperties(Class)`, the helper Drools itself uses (getter/setter reflection).

Error class `RuntimeException` matches existing validation style in `DrlxRuleAstRuntimeBuilder` (e.g. `resolvePositionalField`).

### Class-level `@PropertyReactive` check

Deliberately **not** implemented. DRL requires the pattern class to be `@PropertyReactive` (otherwise the watch list is a no-op and the user probably made a mistake). DRLX has no `TypeDeclaration` concept yet; adding this check would need one. MVP permits the watch list on any class — slightly more permissive than DRL7, not a safety issue. Reconsider if/when DRLX gains type-declaration support.

## Error behaviour (DRL parity minus class check)

| Case | Behaviour |
|------|-----------|
| Unknown property | `RuntimeException("Unknown property 'X' in watch list on <type>")` at build |
| Duplicate property | `RuntimeException("Duplicate property 'X' in watch list on <type>")` at build |
| Duplicate wildcard | `RuntimeException("Duplicate usage of wildcard * in watch list on <type>")` at build |
| Property on non-`@PropertyReactive` class | Accepted at build; runtime ignores watch list (Drools default when class is not property-reactive) |

Defense in depth: `PropertySpecificUtil.setPropertyOnMask` in Drools throws `RuntimeException("Unknown property ...")` if anything slips past our validation — a backstop, not the primary error path.

## Testing

New test file: `PropertyReactiveWatchListTest.java` in `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/`, using `TrackingAgendaEventListener` for strict rule assertions (per session 2026-04-24 convention).

### Happy-path tests

| Case | Syntax fragment | Behaviour verified |
|------|-----------------|--------------------|
| Plain watch list | `/employees[salary > 0][basePay, bonusPay]` | Modify unlisted field → no fire; listed → fires |
| Watch-only, empty conditions | `/employees[][basePay]` | `[][...]` form parses and wires |
| Wildcard all | `/employees[][*]` | Any property modification fires |
| Wildcard none | `/employees[][!*]` | No property modification re-fires |
| Negative exclusion | `/employees[][!bonusPay]` | Modify `bonusPay` → no fire; others → fires |
| Nested inside NOT | `not /employees[status == "X"][status]` | Watch list applied to nested pattern |
| Serialization round-trip | any above | `writeToProto` / `parseFrom` preserves watch list |

### Error-path tests

Expect `RuntimeException` at `DrlxRuleBuilder.build()`:

| Case | Message fragment |
|------|------------------|
| Unknown property | `"Unknown property 'basePya'"` |
| Duplicate property | `"Duplicate property 'basePay'"` |
| Duplicate wildcard | `"Duplicate usage of wildcard *"` |

### Parse-only coverage

One `DrlxRuleParserTest` case confirms the grammar accepts the new syntax (sanity check independent of runtime build).

### Fixture

Reuse or add a simple `@org.kie.api.definition.type.PropertyReactive`-annotated bean (`basePay`, `bonusPay`, `salary` getters/setters) so the bitmask actually gates modification notifications; otherwise Drools falls back to notifying on every modify and the tests would pass regardless.

## Regression surface

Existing `PatternIR` construction sites (four: two in `DrlxToRuleAstVisitor`, one in `DrlxRuleAstParseResult`, one in the visitor helper) gain a `List.of()` for the new field. All 96 existing tests continue to pass.

## Out of scope

- Auto-adding constraint-referenced fields to the watch list (DRL7's `PatternBuilder` does this). Can be added in a follow-up; current MVP is explicit-only.
- Class-level `@PropertyReactive` requirement (needs DRLX type-declaration support).
- Watch lists on non-root chunks (`/a[...][...]/b[...][...]`). Drools' `Pattern` is a root-level concept; navigation chunks don't have their own pattern.
