# Rule-Level Annotations Batch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 7 new rule-level annotations (`@NoLoop`, `@LockOnActive`, `@AutoFocus`, `@Disabled`, `@AgendaGroup`, `@ActivationGroup`, `@RuleFlowGroup`) to the drlx-parser annotation pipeline.

**Architecture:** Extend the existing 6-layer pipeline (annotation class → IR Kind → proto enum → resolver → proto conversion → runtime application). Refactor `Kind` from a plain enum to an `ArgShape`-bearing enum so argument validation is data-driven.

**Tech Stack:** Java 21, protobuf, JUnit 5, AssertJ, drools `RuleImpl` API

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `drlx-parser-core/src/main/java/org/drools/drlx/annotations/NoLoop.java` | Create | Marker annotation |
| `drlx-parser-core/src/main/java/org/drools/drlx/annotations/LockOnActive.java` | Create | Marker annotation |
| `drlx-parser-core/src/main/java/org/drools/drlx/annotations/AutoFocus.java` | Create | Marker annotation |
| `drlx-parser-core/src/main/java/org/drools/drlx/annotations/Disabled.java` | Create | Marker annotation |
| `drlx-parser-core/src/main/java/org/drools/drlx/annotations/AgendaGroup.java` | Create | String-arg annotation |
| `drlx-parser-core/src/main/java/org/drools/drlx/annotations/ActivationGroup.java` | Create | String-arg annotation |
| `drlx-parser-core/src/main/java/org/drools/drlx/annotations/RuleFlowGroup.java` | Create | String-arg annotation |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:30-32` | Modify | Add ArgShape to Kind enum |
| `drlx-parser-core/src/main/proto/drlx_rule_ast.proto:80-85` | Modify | Add 7 proto enum entries |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:37-44,231-304` | Modify | FQN registry + refactored extractAnnotationLiteral |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java:286-301` | Modify | Proto conversion switch arms |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java:1460-1468` | Modify | Runtime application switch arms |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java` | Modify | 15 new tests |

---

### Task 1: Annotation Classes + IR Kind Refactor

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/NoLoop.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/LockOnActive.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/AutoFocus.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/Disabled.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/AgendaGroup.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/ActivationGroup.java`
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/annotations/RuleFlowGroup.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`

- [ ] **Step 1: Create 4 marker annotation classes**

All in `drlx-parser-core/src/main/java/org/drools/drlx/annotations/`. Each follows the same pattern — no `value()` method.

`NoLoop.java`:
```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface NoLoop {
}
```

`LockOnActive.java`:
```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface LockOnActive {
}
```

`AutoFocus.java`:
```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AutoFocus {
}
```

`Disabled.java`:
```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Disabled {
}
```

- [ ] **Step 2: Create 3 string-argument annotation classes**

`AgendaGroup.java`:
```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AgendaGroup {
    String value();
}
```

`ActivationGroup.java`:
```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ActivationGroup {
    String value();
}
```

`RuleFlowGroup.java`:
```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface RuleFlowGroup {
    String value();
}
```

- [ ] **Step 3: Refactor Kind enum with ArgShape**

In `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`, replace lines 30-32:

```java
public record RuleAnnotationIR(Kind kind, String rawValue) {
    public enum Kind { SALIENCE, DESCRIPTION, DATASOURCE }
}
```

with:

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

- [ ] **Step 4: Compile check**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS (existing code that switches on `Kind` will warn about missing cases but compiles)

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/annotations/NoLoop.java \
  drlx-parser-core/src/main/java/org/drools/drlx/annotations/LockOnActive.java \
  drlx-parser-core/src/main/java/org/drools/drlx/annotations/AutoFocus.java \
  drlx-parser-core/src/main/java/org/drools/drlx/annotations/Disabled.java \
  drlx-parser-core/src/main/java/org/drools/drlx/annotations/AgendaGroup.java \
  drlx-parser-core/src/main/java/org/drools/drlx/annotations/ActivationGroup.java \
  drlx-parser-core/src/main/java/org/drools/drlx/annotations/RuleFlowGroup.java \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#6): add 7 annotation classes and ArgShape Kind enum"
```

---

### Task 2: Proto + Serialization

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`

- [ ] **Step 1: Add proto enum entries**

In `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`, replace lines 80-85:

```protobuf
enum AnnotationKind {
  ANNOTATION_KIND_UNSPECIFIED = 0;
  ANNOTATION_KIND_SALIENCE = 1;
  ANNOTATION_KIND_DESCRIPTION = 2;
  ANNOTATION_KIND_DATASOURCE = 3;
}
```

with:

```protobuf
enum AnnotationKind {
  ANNOTATION_KIND_UNSPECIFIED = 0;
  ANNOTATION_KIND_SALIENCE = 1;
  ANNOTATION_KIND_DESCRIPTION = 2;
  ANNOTATION_KIND_DATASOURCE = 3;
  ANNOTATION_KIND_NO_LOOP = 4;
  ANNOTATION_KIND_LOCK_ON_ACTIVE = 5;
  ANNOTATION_KIND_AUTO_FOCUS = 6;
  ANNOTATION_KIND_DISABLED = 7;
  ANNOTATION_KIND_AGENDA_GROUP = 8;
  ANNOTATION_KIND_ACTIVATION_GROUP = 9;
  ANNOTATION_KIND_RULEFLOW_GROUP = 10;
}
```

- [ ] **Step 2: Update fromProtoKind**

In `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`, replace the `fromProtoKind` method (lines 286-294):

```java
private static RuleAnnotationIR.Kind fromProtoKind(DrlxRuleAstProto.AnnotationKind k) {
    return switch (k) {
        case ANNOTATION_KIND_SALIENCE -> RuleAnnotationIR.Kind.SALIENCE;
        case ANNOTATION_KIND_DESCRIPTION -> RuleAnnotationIR.Kind.DESCRIPTION;
        case ANNOTATION_KIND_DATASOURCE -> RuleAnnotationIR.Kind.DATASOURCE;
        case ANNOTATION_KIND_UNSPECIFIED, UNRECOGNIZED ->
                throw new IllegalStateException("Unknown proto annotation kind: " + k);
    };
}
```

with:

```java
private static RuleAnnotationIR.Kind fromProtoKind(DrlxRuleAstProto.AnnotationKind k) {
    return switch (k) {
        case ANNOTATION_KIND_SALIENCE -> RuleAnnotationIR.Kind.SALIENCE;
        case ANNOTATION_KIND_DESCRIPTION -> RuleAnnotationIR.Kind.DESCRIPTION;
        case ANNOTATION_KIND_DATASOURCE -> RuleAnnotationIR.Kind.DATASOURCE;
        case ANNOTATION_KIND_NO_LOOP -> RuleAnnotationIR.Kind.NO_LOOP;
        case ANNOTATION_KIND_LOCK_ON_ACTIVE -> RuleAnnotationIR.Kind.LOCK_ON_ACTIVE;
        case ANNOTATION_KIND_AUTO_FOCUS -> RuleAnnotationIR.Kind.AUTO_FOCUS;
        case ANNOTATION_KIND_DISABLED -> RuleAnnotationIR.Kind.DISABLED;
        case ANNOTATION_KIND_AGENDA_GROUP -> RuleAnnotationIR.Kind.AGENDA_GROUP;
        case ANNOTATION_KIND_ACTIVATION_GROUP -> RuleAnnotationIR.Kind.ACTIVATION_GROUP;
        case ANNOTATION_KIND_RULEFLOW_GROUP -> RuleAnnotationIR.Kind.RULEFLOW_GROUP;
        case ANNOTATION_KIND_UNSPECIFIED, UNRECOGNIZED ->
                throw new IllegalStateException("Unknown proto annotation kind: " + k);
    };
}
```

- [ ] **Step 3: Update toProtoKind**

Replace the `toProtoKind` method (lines 296-301):

```java
private static DrlxRuleAstProto.AnnotationKind toProtoKind(RuleAnnotationIR.Kind k) {
    return switch (k) {
        case SALIENCE -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_SALIENCE;
        case DESCRIPTION -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_DESCRIPTION;
        case DATASOURCE -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_DATASOURCE;
    };
}
```

with:

```java
private static DrlxRuleAstProto.AnnotationKind toProtoKind(RuleAnnotationIR.Kind k) {
    return switch (k) {
        case SALIENCE -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_SALIENCE;
        case DESCRIPTION -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_DESCRIPTION;
        case DATASOURCE -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_DATASOURCE;
        case NO_LOOP -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_NO_LOOP;
        case LOCK_ON_ACTIVE -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_LOCK_ON_ACTIVE;
        case AUTO_FOCUS -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_AUTO_FOCUS;
        case DISABLED -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_DISABLED;
        case AGENDA_GROUP -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_AGENDA_GROUP;
        case ACTIVATION_GROUP -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_ACTIVATION_GROUP;
        case RULEFLOW_GROUP -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_RULEFLOW_GROUP;
    };
}
```

- [ ] **Step 4: Compile check**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/proto/drlx_rule_ast.proto \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#6): proto enum and serialization for 7 new annotations"
```

---

### Task 3: Resolver — FQN Registry + extractAnnotationLiteral Refactor

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

- [ ] **Step 1: Add FQN constants and expand SUPPORTED_ANNOTATION_KINDS**

Replace lines 37-44:

```java
private static final String SALIENCE_FQN = "org.drools.drlx.annotations.Salience";
private static final String DESCRIPTION_FQN = "org.drools.drlx.annotations.Description";
private static final String DATASOURCE_FQN = "org.drools.drlx.annotations.DataSource";

private static final Map<String, Kind> SUPPORTED_ANNOTATION_KINDS = Map.of(
        SALIENCE_FQN, Kind.SALIENCE,
        DESCRIPTION_FQN, Kind.DESCRIPTION,
        DATASOURCE_FQN, Kind.DATASOURCE);
```

with:

```java
private static final String SALIENCE_FQN = "org.drools.drlx.annotations.Salience";
private static final String DESCRIPTION_FQN = "org.drools.drlx.annotations.Description";
private static final String DATASOURCE_FQN = "org.drools.drlx.annotations.DataSource";
private static final String NO_LOOP_FQN = "org.drools.drlx.annotations.NoLoop";
private static final String LOCK_ON_ACTIVE_FQN = "org.drools.drlx.annotations.LockOnActive";
private static final String AUTO_FOCUS_FQN = "org.drools.drlx.annotations.AutoFocus";
private static final String DISABLED_FQN = "org.drools.drlx.annotations.Disabled";
private static final String AGENDA_GROUP_FQN = "org.drools.drlx.annotations.AgendaGroup";
private static final String ACTIVATION_GROUP_FQN = "org.drools.drlx.annotations.ActivationGroup";
private static final String RULEFLOW_GROUP_FQN = "org.drools.drlx.annotations.RuleFlowGroup";

private static final Map<String, Kind> SUPPORTED_ANNOTATION_KINDS = Map.ofEntries(
        Map.entry(SALIENCE_FQN, Kind.SALIENCE),
        Map.entry(DESCRIPTION_FQN, Kind.DESCRIPTION),
        Map.entry(DATASOURCE_FQN, Kind.DATASOURCE),
        Map.entry(NO_LOOP_FQN, Kind.NO_LOOP),
        Map.entry(LOCK_ON_ACTIVE_FQN, Kind.LOCK_ON_ACTIVE),
        Map.entry(AUTO_FOCUS_FQN, Kind.AUTO_FOCUS),
        Map.entry(DISABLED_FQN, Kind.DISABLED),
        Map.entry(AGENDA_GROUP_FQN, Kind.AGENDA_GROUP),
        Map.entry(ACTIVATION_GROUP_FQN, Kind.ACTIVATION_GROUP),
        Map.entry(RULEFLOW_GROUP_FQN, Kind.RULEFLOW_GROUP));
```

Note: `Map.of()` supports up to 10 entries, but `Map.ofEntries()` is cleaner for this many.

- [ ] **Step 2: Refactor kindDisplayName**

Replace lines 231-233:

```java
private static String kindDisplayName(Kind kind) {
    return kind.name().charAt(0) + kind.name().substring(1).toLowerCase();
}
```

with:

```java
private static String kindDisplayName(Kind kind) {
    StringBuilder sb = new StringBuilder();
    for (String part : kind.name().split("_")) {
        sb.append(part.charAt(0)).append(part.substring(1).toLowerCase());
    }
    return sb.toString();
}
```

This converts `NO_LOOP` → `NoLoop`, `LOCK_ON_ACTIVE` → `LockOnActive`, and preserves existing behavior for single-word names like `SALIENCE` → `Salience`.

- [ ] **Step 3: Refactor extractAnnotationLiteral to switch on ArgShape**

Replace lines 268-304:

```java
private static String extractAnnotationLiteral(DrlxParser.AnnotationContext annCtx,
                                               Kind kind, int line, int col) {
    if (annCtx.elementValue() == null) {
        throw new RuntimeException("@" + kindDisplayName(kind) + " expects one argument at " + line + ":" + col);
    }
    String text = annCtx.elementValue().getText();
    switch (kind) {
        case SALIENCE -> {
            try {
                return String.valueOf(Integer.parseInt(text));
            } catch (NumberFormatException e) {
                throw new RuntimeException(
                        "@Salience expects int literal, got '" + text + "' at " + line + ":" + col);
            }
        }
        case DESCRIPTION -> {
            if (text.length() >= 2 && text.startsWith("\"") && text.endsWith("\"")) {
                return text.substring(1, text.length() - 1);
            }
            throw new RuntimeException(
                    "@Description expects string literal, got '" + text + "' at " + line + ":" + col);
        }
        case DATASOURCE -> {
            if (text.length() >= 2 && text.startsWith("\"") && text.endsWith("\"")) {
                String name = text.substring(1, text.length() - 1);
                if (name.isEmpty()) {
                    throw new RuntimeException(
                            "@Datasource expects non-empty string literal at " + line + ":" + col);
                }
                return name;
            }
            throw new RuntimeException(
                    "@Datasource expects string literal, got '" + text + "' at " + line + ":" + col);
        }
        default -> throw new IllegalStateException("Unhandled annotation kind: " + kind);
    }
}
```

with:

```java
private static String extractAnnotationLiteral(DrlxParser.AnnotationContext annCtx,
                                               Kind kind, int line, int col) {
    String displayName = kindDisplayName(kind);
    switch (kind.argShape) {
        case NONE -> {
            if (annCtx.elementValue() != null) {
                throw new RuntimeException(
                        "@" + displayName + " takes no arguments at " + line + ":" + col);
            }
            return "";
        }
        case INT -> {
            if (annCtx.elementValue() == null) {
                throw new RuntimeException(
                        "@" + displayName + " expects one argument at " + line + ":" + col);
            }
            String text = annCtx.elementValue().getText();
            try {
                return String.valueOf(Integer.parseInt(text));
            } catch (NumberFormatException e) {
                throw new RuntimeException(
                        "@" + displayName + " expects int literal, got '" + text + "' at " + line + ":" + col);
            }
        }
        case STRING -> {
            if (annCtx.elementValue() == null) {
                throw new RuntimeException(
                        "@" + displayName + " expects one argument at " + line + ":" + col);
            }
            String text = annCtx.elementValue().getText();
            if (text.length() >= 2 && text.startsWith("\"") && text.endsWith("\"")) {
                String value = text.substring(1, text.length() - 1);
                if (value.isEmpty()) {
                    throw new RuntimeException(
                            "@" + displayName + " expects non-empty string literal at " + line + ":" + col);
                }
                return value;
            }
            throw new RuntimeException(
                    "@" + displayName + " expects string literal, got '" + text + "' at " + line + ":" + col);
        }
        default -> throw new IllegalStateException("Unhandled arg shape: " + kind.argShape);
    }
}
```

- [ ] **Step 4: Update unsupported-annotation error message in resolveKind**

In the `resolveKind` method (line 256), update the error message to list all 10 supported annotations:

Replace:
```java
throw new RuntimeException(
        "unsupported DRLX rule annotation '@" + nameText + "' at "
        + line + ":" + col + " — supported: @Salience, @Description, @DataSource");
```

with:
```java
throw new RuntimeException(
        "unsupported DRLX rule annotation '@" + nameText + "' at "
        + line + ":" + col + " — supported: @Salience, @Description, @DataSource, "
        + "@NoLoop, @LockOnActive, @AutoFocus, @Disabled, "
        + "@AgendaGroup, @ActivationGroup, @RuleFlowGroup");
```

- [ ] **Step 5: Compile check**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Run existing annotation tests to verify no regressions**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=RuleAnnotationsTest -q`
Expected: All 13 existing tests pass

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "refactor(#6): ArgShape-based resolver with 7 new annotation FQNs"
```

---

### Task 4: Runtime Application

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Add EnabledBoolean import**

Add this import to `DrlxRuleAstRuntimeBuilder.java` (alongside the existing `SalienceInteger` import at line 21):

```java
import org.drools.base.base.EnabledBoolean;
```

- [ ] **Step 2: Add 7 new case arms to applyAnnotations**

Replace lines 1460-1468:

```java
private static void applyAnnotations(RuleImpl rule, List<RuleAnnotationIR> annotations) {
    for (RuleAnnotationIR ann : annotations) {
        switch (ann.kind()) {
            case SALIENCE -> rule.setSalience(new SalienceInteger(Integer.parseInt(ann.rawValue())));
            case DESCRIPTION -> rule.addMetaAttribute("Description", ann.rawValue());
            case DATASOURCE -> { }
        }
    }
}
```

with:

```java
private static void applyAnnotations(RuleImpl rule, List<RuleAnnotationIR> annotations) {
    for (RuleAnnotationIR ann : annotations) {
        switch (ann.kind()) {
            case SALIENCE -> rule.setSalience(new SalienceInteger(Integer.parseInt(ann.rawValue())));
            case DESCRIPTION -> rule.addMetaAttribute("Description", ann.rawValue());
            case DATASOURCE -> { }
            case NO_LOOP -> rule.setNoLoop(true);
            case LOCK_ON_ACTIVE -> rule.setLockOnActive(true);
            case AUTO_FOCUS -> rule.setAutoFocus(true);
            case DISABLED -> rule.setEnabled(EnabledBoolean.ENABLED_FALSE);
            case AGENDA_GROUP -> rule.setAgendaGroup(ann.rawValue());
            case ACTIVATION_GROUP -> rule.setActivationGroup(ann.rawValue());
            case RULEFLOW_GROUP -> rule.setRuleFlowGroup(ann.rawValue());
        }
    }
}
```

- [ ] **Step 3: Compile check**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#6): runtime application for 7 new annotations"
```

---

### Task 5: Install module before testing

- [ ] **Step 1: Install drlx-parser-core**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -DskipTests -q`
Expected: BUILD SUCCESS

This is required because the test runner may pick up dependent modules via the local repo.

---

### Task 6: Negative Tests — Marker Annotations Reject Arguments

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

- [ ] **Step 1: Write 4 negative tests for marker annotations**

Add the following tests to `RuleAnnotationsTest.java`:

```java
@Test
void testNoLoopRejectsArgument() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.NoLoop;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @NoLoop(true)
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@NoLoop takes no arguments");
}

@Test
void testLockOnActiveRejectsArgument() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.LockOnActive;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @LockOnActive(true)
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@LockOnActive takes no arguments");
}

@Test
void testAutoFocusRejectsArgument() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.AutoFocus;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @AutoFocus(true)
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@AutoFocus takes no arguments");
}

@Test
void testDisabledRejectsArgument() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Disabled;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Disabled(false)
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@Disabled takes no arguments");
}
```

- [ ] **Step 2: Run the 4 new tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="RuleAnnotationsTest#testNoLoopRejectsArgument+testLockOnActiveRejectsArgument+testAutoFocusRejectsArgument+testDisabledRejectsArgument" -q`
Expected: All 4 PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#6): marker annotations reject arguments"
```

---

### Task 7: Negative Tests — String Annotations Reject Empty Strings

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

- [ ] **Step 1: Write 3 negative tests for empty-string rejection**

Add the following tests:

```java
@Test
void testAgendaGroupRejectsEmptyString() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.AgendaGroup;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @AgendaGroup("")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@AgendaGroup expects non-empty string literal");
}

@Test
void testActivationGroupRejectsEmptyString() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.ActivationGroup;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @ActivationGroup("")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@ActivationGroup expects non-empty string literal");
}

@Test
void testRuleFlowGroupRejectsEmptyString() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.RuleFlowGroup;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @RuleFlowGroup("")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    assertThatThrownBy(() -> newBuilder().build(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("@RuleFlowGroup expects non-empty string literal");
}
```

- [ ] **Step 2: Run the 3 new tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="RuleAnnotationsTest#testAgendaGroupRejectsEmptyString+testActivationGroupRejectsEmptyString+testRuleFlowGroupRejectsEmptyString" -q`
Expected: All 3 PASS

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#6): string annotations reject empty strings"
```

---

### Task 8: Happy Path Tests — Marker Annotations

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

- [ ] **Step 1: Write testNoLoopPreventsRefire**

This test verifies that a rule with `@NoLoop` fires only once even when the rule modifies its own fact (which would normally trigger re-evaluation).

```java
@Test
void testNoLoopPreventsRefire() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.NoLoop;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @NoLoop
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { p.setAge(p.getAge() + 1); update(p); }
            }
            """;

    withSession(rule, (kieSession, listener) -> {
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 30));

        final int firedCount = kieSession.fireAllRules();

        assertThat(firedCount).isEqualTo(1);
    });
}
```

- [ ] **Step 2: Write testDisabledRuleDoesNotFire**

```java
@Test
void testDisabledRuleDoesNotFire() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Disabled;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Disabled
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    withSession(rule, (kieSession, listener) -> {
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 30));

        final int firedCount = kieSession.fireAllRules();

        assertThat(firedCount).isEqualTo(0);
    });
}
```

- [ ] **Step 3: Write testAutoFocusActivatesGroup**

A rule in a non-default agenda group with `@AutoFocus` should fire without explicit `setFocus()`.

```java
@Test
void testAutoFocusActivatesGroup() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.AgendaGroup;
            import org.drools.drlx.annotations.AutoFocus;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @AgendaGroup("g1")
            @AutoFocus
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    withSession(rule, (kieSession, listener) -> {
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 30));

        final int firedCount = kieSession.fireAllRules();

        assertThat(firedCount).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("R1");
    });
}
```

- [ ] **Step 4: Write testLockOnActivePreventsRefire**

A rule with `@LockOnActive` in an agenda group fires once even when the fact is updated.

```java
@Test
void testLockOnActivePreventsRefire() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.AgendaGroup;
            import org.drools.drlx.annotations.AutoFocus;
            import org.drools.drlx.annotations.LockOnActive;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @AgendaGroup("g1")
            @AutoFocus
            @LockOnActive
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { p.setAge(p.getAge() + 1); update(p); }
            }
            """;

    withSession(rule, (kieSession, listener) -> {
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 30));

        final int firedCount = kieSession.fireAllRules();

        assertThat(firedCount).isEqualTo(1);
    });
}
```

- [ ] **Step 5: Run the 4 marker happy path tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="RuleAnnotationsTest#testNoLoopPreventsRefire+testDisabledRuleDoesNotFire+testAutoFocusActivatesGroup+testLockOnActivePreventsRefire" -q`
Expected: All 4 PASS

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#6): happy path tests for marker annotations"
```

---

### Task 9: Happy Path Tests — String-Argument Annotations

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

- [ ] **Step 1: Write testAgendaGroupAssignment**

A rule in `@AgendaGroup("g1")` should only fire after the agenda group is focused.

```java
@Test
void testAgendaGroupAssignment() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.AgendaGroup;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @AgendaGroup("g1")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    withSession(rule, (kieSession, listener) -> {
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 30));

        // Without setFocus, rule should not fire
        int firedCount = kieSession.fireAllRules();
        assertThat(firedCount).isEqualTo(0);

        // After setFocus, rule fires
        kieSession.getAgenda().getAgendaGroup("g1").setFocus();
        firedCount = kieSession.fireAllRules();
        assertThat(firedCount).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("R1");
    });
}
```

- [ ] **Step 2: Write testActivationGroupOnlyOneFiresPerGroup**

Two rules in the same activation group — only the first activated fires.

```java
@Test
void testActivationGroupOnlyOneFiresPerGroup() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.ActivationGroup;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @ActivationGroup("only-one")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println("R1"); }
            }

            @ActivationGroup("only-one")
            rule R2 {
                Person p : /persons[ age > 18 ],
                do { System.out.println("R2"); }
            }
            """;

    withSession(rule, (kieSession, listener) -> {
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 30));

        final int firedCount = kieSession.fireAllRules();

        assertThat(firedCount).isEqualTo(1);
    });
}
```

- [ ] **Step 3: Write testRuleFlowGroupAssignment**

Verify the rule is placed in the correct ruleflow group by checking that it does not fire without activation of that group.

```java
@Test
void testRuleFlowGroupAssignment() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.RuleFlowGroup;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @RuleFlowGroup("rfg1")
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { System.out.println(p); }
            }
            """;

    withSession(rule, (kieSession, listener) -> {
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 30));

        // RuleFlowGroup is not active, so rule should not fire
        final int firedCount = kieSession.fireAllRules();
        assertThat(firedCount).isEqualTo(0);
    });
}
```

- [ ] **Step 4: Run the 3 new tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="RuleAnnotationsTest#testAgendaGroupAssignment+testActivationGroupOnlyOneFiresPerGroup+testRuleFlowGroupAssignment" -q`
Expected: All 3 PASS

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#6): happy path tests for string-argument annotations"
```

---

### Task 10: Combination Test + Full Suite

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`

- [ ] **Step 1: Write testMultipleAnnotationsCombined**

```java
@Test
void testMultipleAnnotationsCombined() {
    final String rule = """
            package org.drools.drlx.parser;

            import org.drools.drlx.domain.Person;
            import org.drools.drlx.annotations.Salience;
            import org.drools.drlx.annotations.NoLoop;
            import org.drools.drlx.annotations.AgendaGroup;
            import org.drools.drlx.annotations.AutoFocus;

            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;

            @Salience(5)
            @NoLoop
            @AgendaGroup("g1")
            @AutoFocus
            rule R1 {
                Person p : /persons[ age > 18 ],
                do { p.setAge(p.getAge() + 1); update(p); }
            }
            """;

    withSession(rule, (kieSession, listener) -> {
        final EntryPoint entryPoint = kieSession.getEntryPoint("persons");
        entryPoint.insert(new Person("Alice", 30));

        final int firedCount = kieSession.fireAllRules();

        // @AutoFocus makes the agenda group active, @NoLoop prevents refire
        assertThat(firedCount).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("R1");
    });
}
```

- [ ] **Step 2: Run the combination test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest="RuleAnnotationsTest#testMultipleAnnotationsCombined" -q`
Expected: PASS

- [ ] **Step 3: Run the full RuleAnnotationsTest suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=RuleAnnotationsTest -q`
Expected: All 28 tests pass (13 existing + 15 new)

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
  drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#6): combination test for multiple annotations"
```

---

### Task 11: Full Module Test Suite

- [ ] **Step 1: Run entire drlx-parser-core test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -q`
Expected: All tests pass, no regressions

- [ ] **Step 2: Install and run full drlx-parser build**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml install -q`
Expected: BUILD SUCCESS across all modules
