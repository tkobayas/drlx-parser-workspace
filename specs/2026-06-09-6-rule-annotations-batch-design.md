# Rule-Level Annotations Batch — Design Spec

**Issue:** #6  
**Epic:** #78  
**Date:** 2026-06-09

## Problem

The annotation pipeline supports only `@Salience(int)`, `@Description(String)`, and `@DataSource(String)`. Typical DRL rules use additional attributes (`no-loop`, `agenda-group`, `lock-on-active`, etc.) that have no DRLX annotation equivalents yet. All target attributes have existing `RuleImpl` setters — no drools-side changes are needed.

## Scope

Seven new rule-level annotations, split into two argument shapes:

| Annotation | Style | RuleImpl call |
|------------|-------|---------------|
| `@NoLoop` | marker (no args) | `setNoLoop(true)` |
| `@LockOnActive` | marker | `setLockOnActive(true)` |
| `@AutoFocus` | marker | `setAutoFocus(true)` |
| `@Disabled` | marker | `setEnabled(EnabledBoolean.ENABLED_FALSE)` |
| `@AgendaGroup(String)` | string arg | `setAgendaGroup(...)` |
| `@ActivationGroup(String)` | string arg | `setActivationGroup(...)` |
| `@RuleFlowGroup(String)` | string arg | `setRuleFlowGroup(...)` |

**Out of scope:** `@Duration`/`@Timer` (#85), `@DateEffective`/`@DateExpires` (#86).

## Design Decisions

- **Marker-style for booleans:** Presence means "on", absence means default. No `@NoLoop(true)` form — passing an argument to a marker annotation is an error.
- **`@Disabled` instead of `@Enabled`:** A marker `@Enabled` is redundant (rules are enabled by default). `@Disabled` clearly communicates intent.
- **Empty-string rejection for all STRING annotations:** `@AgendaGroup("")`, `@ActivationGroup("")`, `@RuleFlowGroup("")` are errors, consistent with `@DataSource("")`.

## Layer 1: Annotation Classes

Seven new classes in `org.drools.drlx.annotations`, following the existing pattern (`@Retention(RUNTIME)`, `@Target(TYPE)`).

Marker annotations have no `value()` method:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface NoLoop {
}
```

String-argument annotations have `String value()`:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AgendaGroup {
    String value();
}
```

Full list: `NoLoop`, `LockOnActive`, `AutoFocus`, `Disabled`, `AgendaGroup`, `ActivationGroup`, `RuleFlowGroup`.

## Layer 2: IR Kind Enum with ArgShape

The `Kind` enum in `RuleAnnotationIR` gains an `ArgShape` field, making argument validation data-driven:

```java
public record RuleAnnotationIR(Kind kind, String rawValue) {
    public enum Kind {
        SALIENCE(ArgShape.INT),
        DESCRIPTION(ArgShape.STRING),
        DATASOURCE(ArgShape.STRING),
        NO_LOOP(ArgShape.NONE),
        LOCK_ON_ACTIVE(ArgShape.NONE),
        AUTO_FOCUS(ArgShape.NONE),
        DISABLED(ArgShape.NONE),
        AGENDA_GROUP(ArgShape.STRING),
        ACTIVATION_GROUP(ArgShape.STRING),
        RULEFLOW_GROUP(ArgShape.STRING);

        enum ArgShape { NONE, INT, STRING }
        final ArgShape argShape;
        Kind(ArgShape argShape) { this.argShape = argShape; }
    }
}
```

Marker annotations store `""` as `rawValue`.

## Layer 3: Proto Changes

Seven new entries in the `AnnotationKind` enum:

```protobuf
ANNOTATION_KIND_NO_LOOP = 4;
ANNOTATION_KIND_LOCK_ON_ACTIVE = 5;
ANNOTATION_KIND_AUTO_FOCUS = 6;
ANNOTATION_KIND_DISABLED = 7;
ANNOTATION_KIND_AGENDA_GROUP = 8;
ANNOTATION_KIND_ACTIVATION_GROUP = 9;
ANNOTATION_KIND_RULEFLOW_GROUP = 10;
```

`fromProtoKind` and `toProtoKind` in `DrlxRuleAstParseResult` gain 7 new case arms each, following the existing pattern.

## Layer 4: Resolver Changes

### FQN registry

Seven new entries in `SUPPORTED_ANNOTATION_KINDS`:

```java
"org.drools.drlx.annotations.NoLoop" → Kind.NO_LOOP
"org.drools.drlx.annotations.LockOnActive" → Kind.LOCK_ON_ACTIVE
"org.drools.drlx.annotations.AutoFocus" → Kind.AUTO_FOCUS
"org.drools.drlx.annotations.Disabled" → Kind.DISABLED
"org.drools.drlx.annotations.AgendaGroup" → Kind.AGENDA_GROUP
"org.drools.drlx.annotations.ActivationGroup" → Kind.ACTIVATION_GROUP
"org.drools.drlx.annotations.RuleFlowGroup" → Kind.RULEFLOW_GROUP
```

### extractAnnotationLiteral refactored

Switch on `kind.argShape` instead of individual kinds:

```java
private static String extractAnnotationLiteral(AnnotationContext annCtx,
                                                Kind kind, int line, int col) {
    switch (kind.argShape) {
        case NONE -> {
            if (annCtx.elementValue() != null) {
                throw new RuntimeException(
                    "@" + kindDisplayName(kind) + " takes no arguments at " + line + ":" + col);
            }
            return "";
        }
        case INT -> {
            if (annCtx.elementValue() == null) {
                throw new RuntimeException(
                    "@" + kindDisplayName(kind) + " expects one argument at " + line + ":" + col);
            }
            String text = annCtx.elementValue().getText();
            try {
                return String.valueOf(Integer.parseInt(text));
            } catch (NumberFormatException e) {
                throw new RuntimeException(
                    "@" + kindDisplayName(kind) + " expects int literal, got '" + text + "' at " + line + ":" + col);
            }
        }
        case STRING -> {
            if (annCtx.elementValue() == null) {
                throw new RuntimeException(
                    "@" + kindDisplayName(kind) + " expects one argument at " + line + ":" + col);
            }
            String text = annCtx.elementValue().getText();
            if (text.length() >= 2 && text.startsWith("\"") && text.endsWith("\"")) {
                String value = text.substring(1, text.length() - 1);
                if (value.isEmpty()) {
                    throw new RuntimeException(
                        "@" + kindDisplayName(kind) + " expects non-empty string literal at " + line + ":" + col);
                }
                return value;
            }
            throw new RuntimeException(
                "@" + kindDisplayName(kind) + " expects string literal, got '" + text + "' at " + line + ":" + col);
        }
    }
}
```

### kindDisplayName

Updated to handle multi-word enum names. Split on `_`, capitalize each segment, join: `NO_LOOP` → `NoLoop`, `LOCK_ON_ACTIVE` → `LockOnActive`, `SALIENCE` → `Salience` (single segment — existing behavior preserved).

### Error message for unsupported annotations

Updated to list all 10 supported annotations.

## Layer 5: Runtime Application

Seven new case arms in `applyAnnotations` in `DrlxRuleAstRuntimeBuilder`:

```java
case NO_LOOP -> rule.setNoLoop(true);
case LOCK_ON_ACTIVE -> rule.setLockOnActive(true);
case AUTO_FOCUS -> rule.setAutoFocus(true);
case DISABLED -> rule.setEnabled(EnabledBoolean.ENABLED_FALSE);
case AGENDA_GROUP -> rule.setAgendaGroup(ann.rawValue());
case ACTIVATION_GROUP -> rule.setActivationGroup(ann.rawValue());
case RULEFLOW_GROUP -> rule.setRuleFlowGroup(ann.rawValue());
```

New import: `org.drools.base.base.EnabledBoolean`.

## Layer 6: Tests

All tests in `RuleAnnotationsTest.java`.

### Happy path (7 tests)

| Test | Verifies |
|------|----------|
| `testNoLoopPreventsRefire` | Rule that modifies its own fact fires only once |
| `testLockOnActivePreventsRefire` | Rule in agenda group with lock-on-active fires once |
| `testAutoFocusActivatesGroup` | Rule in non-default agenda group fires without explicit focus |
| `testDisabledRuleDoesNotFire` | `@Disabled` rule with matching facts produces zero firings |
| `testAgendaGroupAssignment` | Rule fires after `setFocus("g1")` |
| `testActivationGroupOnlyOneFiresPerGroup` | Two rules in same activation group — only one fires |
| `testRuleFlowGroupAssignment` | Rule is assigned to the correct ruleflow group |

### Negative (7 tests)

| Test | Input | Expected error |
|------|-------|----------------|
| `testNoLoopRejectsArgument` | `@NoLoop(true)` | "takes no arguments" |
| `testLockOnActiveRejectsArgument` | `@LockOnActive(true)` | "takes no arguments" |
| `testAutoFocusRejectsArgument` | `@AutoFocus(true)` | "takes no arguments" |
| `testDisabledRejectsArgument` | `@Disabled(false)` | "takes no arguments" |
| `testAgendaGroupRejectsEmptyString` | `@AgendaGroup("")` | "expects non-empty string literal" |
| `testActivationGroupRejectsEmptyString` | `@ActivationGroup("")` | "expects non-empty string literal" |
| `testRuleFlowGroupRejectsEmptyString` | `@RuleFlowGroup("")` | "expects non-empty string literal" |

### Combination (1 test)

| Test | Input | Verifies |
|------|-------|----------|
| `testMultipleAnnotationsCombined` | `@Salience(5)`, `@NoLoop`, `@AgendaGroup("g1")` | All three effects apply independently |
