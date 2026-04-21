# DRLX GroupElement Infrastructure + `not /pattern` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generalize `RuleIR` to a tree shape (`LhsItemIR` sealed, split `lhs`/`rhs`) and land `not /pattern` as the first real consumer.

**Architecture:** Tree-shape IR enables nested group elements. `GroupElementIR.Kind` currently exposes only `NOT`; `EXISTS`, `AND`, `OR` reserved for future issues. Canonical pipeline (`DrlxToRuleAstVisitor` → `DrlxRuleAstRuntimeBuilder`) emits Drools `GroupElement(NOT)` via `GroupElementFactory.newNotInstance()`. Frozen `DrlxToJavaParserVisitor` throws on `notElement`; `TolerantDrlxToJavaParserVisitor` silent-drops to preserve LSP completion. Proto schema is rebuilt with a recursive `LhsItem`/`GroupElement` message pair; old `.pb` cache is intentionally invalidated.

**Tech Stack:** Java 17, ANTLR4, protobuf (bundled `protoc-25.5`), JUnit 5, AssertJ, Drools `GroupElement` / `GroupElementFactory`.

**Spec:** `/home/tkobayas/claude/public/drlx-parser/specs/2026-04-21-group-element-and-not-design.md`
**Issues:** parent epic #4; this plan covers sub-issues #7 (infra) and #8 (`not`).

---

## File Structure

| Path (relative to `/home/tkobayas/usr/work/mvel3-development/drlx-parser/`) | Role in this plan |
|-----------------------------------------------------------------------------|-------------------|
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` | Remove `RuleItemIR`; add `LhsItemIR` sealed interface, `GroupElementIR` record, `Kind` enum. Split `RuleIR` → `(name, annotations, lhs, rhs)`. |
| `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` | Retire `items` (reserve field 2 on `Rule`); add `lhs` + `rhs`; introduce `LhsItem` oneof, `GroupElement` recursive message, `GroupElementKind` enum. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` | Rewrite proto ↔ IR translation for tree shape; add recursive `toProtoLhs` / `fromProtoLhs` helpers. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | `buildRule` sets `lhs` + `rhs` directly. New `visitNotElement` produces `GroupElementIR(NOT, [PatternIR])`; rejects inner-binding forms. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` | Split: recursive `buildLhs(List<LhsItemIR>, GroupElement parent, ...)`; single `setConsequence` after LHS. Map `GroupElementIR(NOT)` → `GroupElementFactory.newNotInstance()`. |
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4` | Add `NOT : 'not';` keyword token. |
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` | Add `notElement : NOT oopathExpression ;` production and wire into `ruleItem`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java` | New override: `visitNotElement` throws — frozen-visitor symmetry with positional/annotations. |
| `drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java` | New override: `visitNotElement` silent-drops; returns inner pattern's JavaParser node. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java` (CREATE) | `notSuppressesMatch`, `notWithOuterBinding`, `protoRoundTrip_withNot`, `notWithInnerBinding_failsLoud`, `notMultiElementForm_failsParse`. |
| `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxToJavaParserVisitorTest.java` | Add `javaParserVisitor_throwsOnNot`. |
| `drlx-parser-core/src/test/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitorTest.java` | Add `incompleteRule_WithNot` (LSP tolerance). |
| `drlx-parser-core/docs/DESIGN.md` | Update Parser Package table (lines ~180) to mention `LhsItemIR` / `GroupElementIR`. |

**Precedent file paths the plan references verbatim:**

- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/DrlxBuilderTestSupport.java` — test base class; provides `withSession(rule, action)`.
- `drlx-parser-core/src/main/java/org/drools/drlx/domain/Person.java` — test domain class with `name: String`, `age: int`.
- `drlx-parser-core/src/main/java/org/drools/drlx/domain/Order.java` — create or verify; needs `customerId: int`. (Check in Task 6; add if missing.)

**Maven rule (project-wide):** After every modification of `drlx-parser-core`, run `mvn -pl drlx-parser-core -am install` from `/home/tkobayas/usr/work/mvel3-development/drlx-parser/` before running dependent tests. Per the user's CLAUDE.md.

**Commit convention:** prior work uses short imperative subjects with scope (`feat(ir):`, `feat(grammar):`, `feat(visitor):`, `test(…)`, `refactor(ir):`, etc.) and `Refs #N` trailer. All commits in this plan end with `Refs #7 #8` except the grammar-only task which ends with `Refs #7 #8` too (same landing).

---

## Task 1: Refactor IR + proto + translation + visitor + runtime to tree shape (atomic, no behaviour change)

**Why atomic:** `RuleIR.items` is read by visitor, runtime builder, and proto translator simultaneously. Splitting the refactor across commits would leave the build red between them. After this task, **all 50 existing tests still pass** — the external behaviour is identical; only internal IR shape has changed.

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` (full rewrite)
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Update `DrlxRuleAstModel.java`**

Replace the entire file contents with:

```java
package org.drools.drlx.builder;

import java.util.List;

/**
 * In-memory intermediate representation produced by {@link DrlxToRuleAstVisitor}
 * and consumed by {@link DrlxRuleAstRuntimeBuilder}. Also the payload that
 * {@link DrlxRuleAstParseResult} serialises to protobuf for build caching.
 */
public final class DrlxRuleAstModel {

    private DrlxRuleAstModel() {
    }

    public record CompilationUnitIR(String packageName, List<String> imports, List<RuleIR> rules) {
    }

    public record RuleIR(String name,
                         List<RuleAnnotationIR> annotations,
                         List<LhsItemIR> lhs,
                         ConsequenceIR rhs) {
    }

    public record RuleAnnotationIR(Kind kind, String rawValue) {
        public enum Kind { SALIENCE, DESCRIPTION }
    }

    /** LHS tree node — either a pattern leaf or a nested group element. */
    public sealed interface LhsItemIR permits PatternIR, GroupElementIR {
    }

    public record PatternIR(String typeName,
                            String bindName,
                            String entryPoint,
                            List<String> conditions,
                            String castTypeName,
                            List<String> positionalArgs) implements LhsItemIR {
    }

    public record GroupElementIR(Kind kind, List<LhsItemIR> children) implements LhsItemIR {
        public enum Kind {
            NOT
            // EXISTS, AND, OR — reserved for follow-up issues #9, #11.
        }
    }

    public record ConsequenceIR(String block) {
    }
}
```

- [ ] **Step 2: Update `drlx_rule_ast.proto`**

Replace the entire file contents with:

```proto
syntax = "proto3";

package org.drools.drlx.builder.proto;

option java_package = "org.drools.drlx.builder.proto";
option java_outer_classname = "DrlxRuleAstProto";

message CompilationUnitParseResult {
  string source_hash = 1;
  string package_name = 2;
  repeated string imports = 3;
  repeated RuleParseResult rules = 4;
}

message RuleParseResult {
  string name = 1;
  reserved 2;                                      // retired flat `items` field
  repeated RuleAnnotationParseResult annotations = 3;
  repeated LhsItemParseResult lhs = 4;             // NEW — tree-shape LHS
  ConsequenceParseResult rhs = 5;                  // NEW — consequence out of items
}

message LhsItemParseResult {
  oneof kind {
    PatternParseResult pattern = 1;
    GroupElementParseResult group = 2;
  }
}

message GroupElementParseResult {
  GroupElementKind kind = 1;
  repeated LhsItemParseResult children = 2;        // recursive
}

message PatternParseResult {
  string type_name = 1;
  string bind_name = 2;
  string entry_point = 3;
  repeated string conditions = 4;
  string cast_type_name = 5;
  repeated string positional_args = 6;
}

message ConsequenceParseResult {
  string block = 1;
}

message RuleAnnotationParseResult {
  AnnotationKind kind = 1;
  string raw_value = 2;
}

enum AnnotationKind {
  ANNOTATION_KIND_UNSPECIFIED = 0;
  ANNOTATION_KIND_SALIENCE = 1;
  ANNOTATION_KIND_DESCRIPTION = 2;
}

enum GroupElementKind {
  GROUP_ELEMENT_KIND_UNSPECIFIED = 0;
  GROUP_ELEMENT_KIND_NOT = 1;
  // 2, 3, 4 reserved for EXISTS, AND, OR
}
```

- [ ] **Step 3: Regenerate the proto Java class**

Run from `/home/tkobayas/usr/work/mvel3-development/drlx-parser/`:

```bash
/tmp/protoc-25.5/bin/protoc --java_out=drlx-parser-core/src/main/java \
    --proto_path=drlx-parser-core/src/main/proto \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto
```

Expected: no output on success. The generated file `drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java` is rewritten.

(If `/tmp/protoc-25.5` is not present, check HANDOFF.md / prior commits — the bundled protoc path is what prior sessions used.)

- [ ] **Step 4: Rewrite `DrlxRuleAstParseResult.java`**

Replace the entire file contents with:

```java
package org.drools.drlx.builder;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.ArrayList;
import java.util.HexFormat;
import java.util.List;

import org.drools.drlx.builder.DrlxRuleAstModel.CompilationUnitIR;
import org.drools.drlx.builder.DrlxRuleAstModel.ConsequenceIR;
import org.drools.drlx.builder.DrlxRuleAstModel.GroupElementIR;
import org.drools.drlx.builder.DrlxRuleAstModel.LhsItemIR;
import org.drools.drlx.builder.DrlxRuleAstModel.PatternIR;
import org.drools.drlx.builder.DrlxRuleAstModel.RuleAnnotationIR;
import org.drools.drlx.builder.DrlxRuleAstModel.RuleIR;
import org.drools.drlx.builder.proto.DrlxRuleAstProto;

/**
 * Persists a compact DRLX-specific rule AST for runtime rebuilds.
 * All protobuf translation works against {@link DrlxRuleAstModel}; there is
 * no direct ANTLR dependency here.
 */
public final class DrlxRuleAstParseResult {

    private static final String FILE_NAME = "drlx-rule-ast.pb";

    private DrlxRuleAstParseResult() {
    }

    public static Path parseResultFilePath(Path dir) {
        return dir.resolve(FILE_NAME);
    }

    public static void save(String drlxSource, CompilationUnitIR data, Path outputDir) throws IOException {
        DrlxRuleAstProto.CompilationUnitParseResult.Builder builder =
                DrlxRuleAstProto.CompilationUnitParseResult.newBuilder()
                        .setSourceHash(hashSource(drlxSource))
                        .setPackageName(data.packageName());

        data.imports().forEach(builder::addImports);
        data.rules().forEach(rule -> builder.addRules(toProtoRule(rule)));

        Files.createDirectories(outputDir);
        try (OutputStream out = Files.newOutputStream(parseResultFilePath(outputDir))) {
            builder.build().writeTo(out);
        }
    }

    public static CompilationUnitIR load(String drlxSource, Path parseResultFile) throws IOException {
        if (!Files.exists(parseResultFile)) {
            return null;
        }

        DrlxRuleAstProto.CompilationUnitParseResult parseResult;
        try (InputStream in = Files.newInputStream(parseResultFile)) {
            parseResult = DrlxRuleAstProto.CompilationUnitParseResult.parseFrom(in);
        }

        if (!parseResult.getSourceHash().equals(hashSource(drlxSource))) {
            return null;
        }

        List<RuleIR> rules = new ArrayList<>(parseResult.getRulesCount());
        for (DrlxRuleAstProto.RuleParseResult ruleParseResult : parseResult.getRulesList()) {
            List<LhsItemIR> lhs = new ArrayList<>(ruleParseResult.getLhsCount());
            for (DrlxRuleAstProto.LhsItemParseResult itemParseResult : ruleParseResult.getLhsList()) {
                lhs.add(fromProtoLhs(itemParseResult, parseResultFile));
            }
            ConsequenceIR rhs = ruleParseResult.hasRhs()
                    ? new ConsequenceIR(ruleParseResult.getRhs().getBlock())
                    : null;

            List<RuleAnnotationIR> ruleAnnotations = new ArrayList<>(ruleParseResult.getAnnotationsCount());
            for (DrlxRuleAstProto.RuleAnnotationParseResult annPR : ruleParseResult.getAnnotationsList()) {
                ruleAnnotations.add(new RuleAnnotationIR(fromProtoKind(annPR.getKind()), annPR.getRawValue()));
            }

            rules.add(new RuleIR(
                    ruleParseResult.getName(),
                    List.copyOf(ruleAnnotations),
                    List.copyOf(lhs),
                    rhs));
        }

        return new CompilationUnitIR(parseResult.getPackageName(),
                List.copyOf(parseResult.getImportsList()),
                List.copyOf(rules));
    }

    private static LhsItemIR fromProtoLhs(DrlxRuleAstProto.LhsItemParseResult item, Path file) {
        return switch (item.getKindCase()) {
            case PATTERN -> {
                DrlxRuleAstProto.PatternParseResult pattern = item.getPattern();
                String castTypeName = pattern.getCastTypeName().isEmpty() ? null : pattern.getCastTypeName();
                yield new PatternIR(
                        pattern.getTypeName(),
                        pattern.getBindName(),
                        pattern.getEntryPoint(),
                        List.copyOf(pattern.getConditionsList()),
                        castTypeName,
                        List.copyOf(pattern.getPositionalArgsList()));
            }
            case GROUP -> {
                DrlxRuleAstProto.GroupElementParseResult group = item.getGroup();
                List<LhsItemIR> children = new ArrayList<>(group.getChildrenCount());
                for (DrlxRuleAstProto.LhsItemParseResult child : group.getChildrenList()) {
                    children.add(fromProtoLhs(child, file));
                }
                yield new GroupElementIR(fromProtoGroupKind(group.getKind()), List.copyOf(children));
            }
            case KIND_NOT_SET -> throw new IllegalStateException("LHS item without payload in " + file);
        };
    }

    private static DrlxRuleAstProto.RuleParseResult toProtoRule(RuleIR rule) {
        DrlxRuleAstProto.RuleParseResult.Builder builder = DrlxRuleAstProto.RuleParseResult.newBuilder()
                .setName(rule.name());
        rule.lhs().forEach(item -> builder.addLhs(toProtoLhs(item)));
        if (rule.rhs() != null) {
            builder.setRhs(DrlxRuleAstProto.ConsequenceParseResult.newBuilder()
                    .setBlock(rule.rhs().block()));
        }
        for (RuleAnnotationIR ann : rule.annotations()) {
            builder.addAnnotations(DrlxRuleAstProto.RuleAnnotationParseResult.newBuilder()
                    .setKind(toProtoKind(ann.kind()))
                    .setRawValue(ann.rawValue())
                    .build());
        }
        return builder.build();
    }

    private static DrlxRuleAstProto.LhsItemParseResult toProtoLhs(LhsItemIR item) {
        DrlxRuleAstProto.LhsItemParseResult.Builder builder = DrlxRuleAstProto.LhsItemParseResult.newBuilder();
        if (item instanceof PatternIR p) {
            DrlxRuleAstProto.PatternParseResult.Builder pb = DrlxRuleAstProto.PatternParseResult.newBuilder()
                    .setTypeName(p.typeName())
                    .setBindName(p.bindName())
                    .setEntryPoint(p.entryPoint());
            if (p.castTypeName() != null) {
                pb.setCastTypeName(p.castTypeName());
            }
            p.conditions().forEach(pb::addConditions);
            p.positionalArgs().forEach(pb::addPositionalArgs);
            builder.setPattern(pb);
        } else if (item instanceof GroupElementIR g) {
            DrlxRuleAstProto.GroupElementParseResult.Builder gb = DrlxRuleAstProto.GroupElementParseResult.newBuilder()
                    .setKind(toProtoGroupKind(g.kind()));
            g.children().forEach(child -> gb.addChildren(toProtoLhs(child)));
            builder.setGroup(gb);
        } else {
            throw new IllegalArgumentException("Unsupported LHS item: " + item);
        }
        return builder.build();
    }

    private static RuleAnnotationIR.Kind fromProtoKind(DrlxRuleAstProto.AnnotationKind k) {
        return switch (k) {
            case ANNOTATION_KIND_SALIENCE -> RuleAnnotationIR.Kind.SALIENCE;
            case ANNOTATION_KIND_DESCRIPTION -> RuleAnnotationIR.Kind.DESCRIPTION;
            case ANNOTATION_KIND_UNSPECIFIED, UNRECOGNIZED ->
                    throw new IllegalStateException("Unknown proto annotation kind: " + k);
        };
    }

    private static DrlxRuleAstProto.AnnotationKind toProtoKind(RuleAnnotationIR.Kind k) {
        return switch (k) {
            case SALIENCE -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_SALIENCE;
            case DESCRIPTION -> DrlxRuleAstProto.AnnotationKind.ANNOTATION_KIND_DESCRIPTION;
        };
    }

    private static GroupElementIR.Kind fromProtoGroupKind(DrlxRuleAstProto.GroupElementKind k) {
        return switch (k) {
            case GROUP_ELEMENT_KIND_NOT -> GroupElementIR.Kind.NOT;
            case GROUP_ELEMENT_KIND_UNSPECIFIED, UNRECOGNIZED ->
                    throw new IllegalStateException("Unknown proto group-element kind: " + k);
        };
    }

    private static DrlxRuleAstProto.GroupElementKind toProtoGroupKind(GroupElementIR.Kind k) {
        return switch (k) {
            case NOT -> DrlxRuleAstProto.GroupElementKind.GROUP_ELEMENT_KIND_NOT;
        };
    }

    private static String hashSource(String drlxSource) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            return HexFormat.of().formatHex(digest.digest(drlxSource.getBytes(StandardCharsets.UTF_8)));
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 not available", e);
        }
    }
}
```

- [ ] **Step 5: Update `DrlxToRuleAstVisitor.buildRule` to emit the new shape**

In `DrlxToRuleAstVisitor.java`:

Replace the `RuleItemIR` import line:
```java
import org.drools.drlx.builder.DrlxRuleAstModel.RuleItemIR;
```
with:
```java
import org.drools.drlx.builder.DrlxRuleAstModel.LhsItemIR;
```

Replace the `buildRule` method:

```java
    private RuleIR buildRule(DrlxParser.RuleDeclarationContext ctx,
                             Map<String, String> annotationImports) {
        String name = ctx.identifier().getText();
        List<RuleAnnotationIR> annotations = buildRuleAnnotations(ctx, annotationImports);
        List<LhsItemIR> lhs = new ArrayList<>();
        ConsequenceIR rhs = null;
        if (ctx.ruleBody() != null) {
            for (DrlxParser.RuleItemContext itemCtx : ctx.ruleBody().ruleItem()) {
                if (itemCtx.ruleConsequence() != null) {
                    if (rhs != null) {
                        throw new RuntimeException(
                                "rule '" + name + "' has more than one consequence block");
                    }
                    rhs = new ConsequenceIR(extractConsequence(itemCtx.ruleConsequence()));
                } else if (itemCtx.rulePattern() != null) {
                    lhs.add(buildPattern(itemCtx.rulePattern()));
                } else {
                    throw new IllegalArgumentException("Unsupported rule item: " + itemCtx.getText());
                }
            }
        }
        return new RuleIR(name, annotations, List.copyOf(lhs), rhs);
    }
```

Delete the old `buildItem` method (no longer referenced). Leave `buildPattern`, `extractConsequence`, and helpers below it as-is.

- [ ] **Step 6: Refactor `DrlxRuleAstRuntimeBuilder` to the new shape**

In `DrlxRuleAstRuntimeBuilder.java`:

Replace the `RuleItemIR` import line:
```java
import org.drools.drlx.builder.DrlxRuleAstModel.RuleItemIR;
```
with:
```java
import org.drools.drlx.builder.DrlxRuleAstModel.LhsItemIR;
import org.drools.drlx.builder.DrlxRuleAstModel.GroupElementIR;
```

Replace the `buildRule` method (currently iterates `items`):

```java
    private RuleImpl buildRule(RuleIR parseResult, TypeResolver typeResolver) {
        lambdaCompiler.beginRule(parseResult.name());

        RuleImpl rule = new RuleImpl(parseResult.name());
        rule.setResource(rule.getResource());
        applyAnnotations(rule, parseResult.annotations());

        GroupElement root = GroupElementFactory.newAndInstance();
        Map<String, BoundVariable> boundVariables = new LinkedHashMap<>();

        buildLhs(parseResult.lhs(), root, typeResolver, boundVariables);

        if (parseResult.rhs() != null) {
            Map<String, Type<?>> types = lambdaCompiler.getTypeMap(root);
            rule.setConsequence(lambdaCompiler.createLambdaConsequence(parseResult.rhs().block(), types));
        }

        rule.setLhs(root);
        return rule;
    }

    private void buildLhs(List<LhsItemIR> items, GroupElement parent,
                          TypeResolver typeResolver, Map<String, BoundVariable> boundVariables) {
        for (LhsItemIR item : items) {
            if (item instanceof PatternIR patternIr) {
                Pattern pattern = buildPattern(patternIr, typeResolver, boundVariables);
                parent.addChild(pattern);
                Declaration declaration = pattern.getDeclaration();
                if (declaration != null) {
                    Class<?> patternClass = ((ClassObjectType) pattern.getObjectType()).getClassType();
                    boundVariables.put(declaration.getIdentifier(),
                            new BoundVariable(declaration.getIdentifier(), patternClass, pattern));
                }
            } else if (item instanceof GroupElementIR group) {
                GroupElement ge = switch (group.kind()) {
                    case NOT -> GroupElementFactory.newNotInstance();
                };
                buildLhs(group.children(), ge, typeResolver, boundVariables);
                parent.addChild(ge);
            } else {
                throw new IllegalArgumentException("Unsupported LHS item: " + item.getClass().getName());
            }
        }
    }
```

Note: the existing `buildRule` also calls `rule.setLhs(root)` somewhere after the loop — preserve that ordering. If your current `buildRule` doesn't explicitly call `setLhs` (and `GroupElementFactory.newAndInstance()` is already wired elsewhere), leave that bit as it is. Check the current file layout before blindly replacing.

- [ ] **Step 7: Remove the now-obsolete `ConsequenceIR` import from `DrlxRuleAstRuntimeBuilder` if still present, and any `items()` / `RuleItemIR` references. Compile-check.**

Run from `/home/tkobayas/usr/work/mvel3-development/drlx-parser/`:

```bash
mvn -pl drlx-parser-core -am compile 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`. If the compile fails with missing-symbol errors (e.g. `RuleItemIR`, `items()`), find the remaining reference and update it.

- [ ] **Step 8: Run full test suite — prove no behaviour change**

```bash
mvn -pl drlx-parser-core -am install 2>&1 | tail -25
```

Expected: `BUILD SUCCESS`. All ~50 tests pass (no new tests yet).

- [ ] **Step 9: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
refactor(ir): generalize RuleIR to tree-shape LHS

Replace the flat List<RuleItemIR> (sealed: PatternIR | ConsequenceIR)
with a split layout — sealed LhsItemIR (permits PatternIR,
GroupElementIR) holds the LHS tree; ConsequenceIR becomes a direct
field (rhs) on RuleIR. Proto schema mirrored (old `items` field
retired via `reserved`; new `lhs` + `rhs` added; LhsItem oneof and
recursive GroupElement message introduced). Runtime builder splits
into a recursive buildLhs pass plus a single setConsequence call.

GroupElementIR.Kind currently exposes only NOT; EXISTS, AND, OR
enum slots reserved for follow-up issues (#9, #11).

No user-visible behaviour change — all existing tests pass against
the new shape, exercising the refactor as its own regression guard.

Refs #7 #8
EOF
)"
```

---

## Task 2: Add `NOT` lexer token

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4`

- [ ] **Step 1: Add the token to `DrlxLexer.g4`**

Append after the `RULE` line (the file currently lists DRLX-specific keywords):

```antlr
NOT  : 'not';
```

Final file after edit:

```antlr
// DRLX Lexer - minimal extension of MVEL3 lexer

lexer grammar DrlxLexer;

import Mvel3Lexer;

// DRLX-specific keywords
UNIT : 'unit';
RULE : 'rule';
NOT  : 'not';
```

- [ ] **Step 2: Regenerate parser sources and run the suite**

```bash
mvn -pl drlx-parser-core -am install 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`. Generated `DrlxLexer.java` in `target/generated-sources/antlr4/` now has a `NOT` token type. No tests change; the token is not yet referenced in the parser grammar.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(lexer): add NOT keyword token

Reserves 'not' as a keyword at the lexer layer. Parser rules for
notElement land in the next commit.

Refs #7 #8
EOF
)"
```

---

## Task 3: Add `notElement` grammar production

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`

- [ ] **Step 1: Add the `notElement` production and wire into `ruleItem`**

In `DrlxParser.g4`, replace:

```antlr
// Rule item can be a pattern or consequence
ruleItem
    : rulePattern
    | ruleConsequence
    ;
```

with:

```antlr
// Rule item can be a pattern, a `not` group element, or a consequence
ruleItem
    : rulePattern
    | notElement
    | ruleConsequence
    ;

// 'not' group element — single-element form only in this landing.
// DRLX spec §"'not' / 'exists'" line 599. Multi-element `not(/a, /b)`
// deferred to a follow-up issue.
notElement
    : NOT oopathExpression
    ;
```

- [ ] **Step 2: Regenerate + compile**

```bash
mvn -pl drlx-parser-core -am install 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`. All existing tests still pass (no parser changes exercise the new production). The generated `DrlxParser.java` now has `NotElementContext` and `visitNotElement` inherited no-op in `DrlxParserBaseVisitor`.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(grammar): add notElement production for single-element `not`

ruleItem now accepts 'not /pattern'. Multi-element form 'not(/a, /b)'
is deliberately out of scope for this landing — grammar reports
'no viable alternative' on the paren form (DRLX spec §'not'/'exists'
line 599 allows parens to be omitted for single child).

Visitor and runtime wiring follow in subsequent commits.

Refs #7 #8
EOF
)"
```

---

## Task 4: RED — failing happy-path test `notSuppressesMatch`

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

- [ ] **Step 1: Create `NotTest.java` with the first failing test**

```java
package org.drools.drlx.builder.syntax;

import java.util.ArrayList;
import java.util.List;

import org.drools.drlx.domain.Person;
import org.junit.jupiter.api.Test;
import org.kie.api.event.rule.AfterMatchFiredEvent;
import org.kie.api.event.rule.DefaultAgendaEventListener;
import org.kie.api.runtime.rule.EntryPoint;

import static org.assertj.core.api.Assertions.assertThat;

class NotTest extends DrlxBuilderTestSupport {

    @Test
    void notSuppressesMatch() {
        // `not /persons[age < 18]` → rule fires ONLY when no under-18 person exists.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                unit MyUnit;

                rule OnlyAdults {
                    not /persons[ age < 18 ],
                    do { System.out.println("only adults"); }
                }
                """;

        withSession(rule, kieSession -> {
            final List<String> fired = new ArrayList<>();
            kieSession.addEventListener(new DefaultAgendaEventListener() {
                @Override
                public void afterMatchFired(AfterMatchFiredEvent event) {
                    fired.add(event.getMatch().getRule().getName());
                }
            });

            final EntryPoint persons = kieSession.getEntryPoint("persons");

            // No under-18 → rule fires once.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).containsExactly("OnlyAdults");

            // Insert an under-18 → the NOT becomes unsatisfied; no further firings.
            fired.clear();
            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isZero();
            assertThat(fired).isEmpty();
        });
    }
}
```

- [ ] **Step 2: Run the test — expect it to FAIL**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest=NotTest#notSuppressesMatch 2>&1 | tail -30
```

Expected: test fails. Most likely error: `IllegalArgumentException: Unsupported rule item: not /persons[...]` thrown from `DrlxToRuleAstVisitor.buildRule` — the visitor doesn't handle `notElement` yet, so the `else` branch in the item walk fires. That's the target RED state. If the error text is different (e.g. a parser error instead), investigate before proceeding — the grammar from Task 3 should already accept the input.

- [ ] **Step 3: Commit the failing test**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): add failing happy-path notSuppressesMatch (RED)

Core semantic proof of NOT — rule should fire when no under-18 exists
and stop firing once one is inserted. Currently fails because the
visitor rejects the new notElement production; wiring lands in the
next commit.

Refs #7 #8
EOF
)"
```

---

## Task 5: GREEN — implement `visitNotElement` + runtime NOT handling

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

*(Runtime builder already handles `GroupElementIR(NOT)` from Task 1 — the `switch` over `Kind.NOT` was added there. This task only wires the visitor.)*

- [ ] **Step 1: Import `GroupElementIR` in the visitor**

Add import in `DrlxToRuleAstVisitor.java` (with the other model imports):

```java
import org.drools.drlx.builder.DrlxRuleAstModel.GroupElementIR;
```

- [ ] **Step 2: Handle the `notElement` branch in `buildRule`**

In `DrlxToRuleAstVisitor.buildRule`, extend the item-walk to recognise `notElement`:

Replace:

```java
                } else if (itemCtx.rulePattern() != null) {
                    lhs.add(buildPattern(itemCtx.rulePattern()));
                } else {
                    throw new IllegalArgumentException("Unsupported rule item: " + itemCtx.getText());
                }
```

with:

```java
                } else if (itemCtx.rulePattern() != null) {
                    lhs.add(buildPattern(itemCtx.rulePattern()));
                } else if (itemCtx.notElement() != null) {
                    lhs.add(buildNotElement(itemCtx.notElement(), name));
                } else {
                    throw new IllegalArgumentException("Unsupported rule item: " + itemCtx.getText());
                }
```

- [ ] **Step 3: Add the `buildNotElement` helper**

Add a new private method in `DrlxToRuleAstVisitor` (place it directly above `buildPattern`):

```java
    private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx, String ruleName) {
        // The grammar gives us a DrlxParser.OopathExpressionContext directly.
        // Build a PatternIR-shaped leaf from it without a bindName — `not` does
        // not expose a binding slot in the grammar, so bindName is always empty here.
        // If the grammar is later extended to allow `not var p : ...`, reject that
        // in this method with a clear message.
        DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
        String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
        String castTypeName = extractCastType(oopathCtx);
        List<String> conditions = extractConditions(oopathCtx);
        List<String> positionalArgs = extractPositionalArgs(oopathCtx);

        // typeName defaults to the entry-point identifier when the grammar-level
        // rulePattern `Type bind :` prefix is absent — the runtime builder resolves
        // the object type from it downstream. For single-element `not`, this matches
        // the spec's "not /persons[...]" form where no Type prefix appears.
        PatternIR inner = new PatternIR(entryPoint, "", entryPoint, conditions, castTypeName, positionalArgs);

        return new GroupElementIR(GroupElementIR.Kind.NOT, List.of(inner));
    }
```

**Note on `typeName`:** the current `PatternIR` distinguishes `typeName` (Java type used for Drools `ClassObjectType`) from `entryPoint` (the entry-point/data-source name from the OOPath root). In the `Type bind : /entry[...]` form they often differ; for `not /persons[...]` there is no `Type bind :` prefix and the entry-point name doubles as the type name. If `withSession` / `DrlxBuilderTestSupport` resolves this against a declared entry point with a known type (as it does for `Location`, `Person` in existing tests), the runtime builder will infer the class. If runtime resolution fails, reopen this step and resolve the type explicitly from the unit declaration (cross-check how `PositionalTest` / `InlineCastTest` handle this — they follow the same pattern).

- [ ] **Step 4: Run the target test — expect it to PASS**

```bash
mvn -pl drlx-parser-core -am install -DskipTests 2>&1 | tail -5 && \
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest=NotTest#notSuppressesMatch 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`, test passes (1 test run, 0 failures). If it fails with a Drools-side error (e.g. "unable to resolve type for NOT pattern"), check the `typeName` note above and trace through `DrlxRuleAstRuntimeBuilder.buildPattern` to confirm the type resolves.

- [ ] **Step 5: Run the full test suite — prove no regressions**

```bash
mvn -pl drlx-parser-core -am install 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`. 50 existing + 1 new = 51 tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(visitor): emit GroupElementIR(NOT) for notElement

DrlxToRuleAstVisitor recognises DrlxParser.NotElementContext in the
item walk and produces a GroupElementIR(NOT, [PatternIR]) for the
inner OOPath. Runtime builder already maps NOT → newNotInstance()
from the Task-1 refactor, so notSuppressesMatch goes green.

Refs #7 #8
EOF
)"
```

---

## Task 6: Positive test — `notWithOuterBinding`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`
- (Possibly) Create: `drlx-parser-core/src/main/java/org/drools/drlx/domain/Order.java`

- [ ] **Step 1: Check whether `Order` (with `customerId`) exists as a test domain class**

```bash
find /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java/org/drools/drlx/domain -name "Order.java"
```

If it exists, open it and confirm it has an `int` (or `Integer`) field `customerId`. If it doesn't, create `drlx-parser-core/src/main/java/org/drools/drlx/domain/Order.java`:

```java
package org.drools.drlx.domain;

public class Order {

    private final String id;
    private final int customerId;
    private final int amount;

    public Order(String id, int customerId, int amount) {
        this.id = id;
        this.customerId = customerId;
        this.amount = amount;
    }

    public String getId() {
        return id;
    }

    public int getCustomerId() {
        return customerId;
    }

    public int getAmount() {
        return amount;
    }
}
```

- [ ] **Step 2: Also check whether `Person` exposes an `id` or equivalent**

Existing `org.drools.drlx.domain.Person` has `name: String` and `age: int` based on the rule-annotations test. Add an integer identity field if missing — prefer reusing a pre-existing field to avoid churn. If there's no integer identity field on `Person`, tweak the test in Step 3 to use a different correlation (e.g. `Person.age == Order.customerId`) rather than adding fields.

Verify with:

```bash
grep -A 20 "class Person" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java/org/drools/drlx/domain/Person.java
```

**Do not add new fields to `Person` just for this test.** Reuse whatever it has.

- [ ] **Step 3: Add `notWithOuterBinding` to `NotTest.java`**

Append after `notSuppressesMatch`:

```java
    @Test
    void notWithOuterBinding() {
        // `var p : /persons[...], not /orders[customerId == p.age]` — outer binding 'p'
        // is referenced inside the NOT's constraint, proving beta-join from the outer
        // pattern into the NOT group element.
        //
        // Domain note: this test correlates Order.customerId with Person.age only to
        // avoid adding a new integer field to Person. The semantic — "fire when a
        // person exists who has no matching order" — is what's being tested, not the
        // field choice.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.domain.Order;

                unit MyUnit;

                rule OrphanedPerson {
                    Person p : /persons[ age > 0 ],
                    not /orders[ customerId == p.age ],
                    do { System.out.println("orphan: " + p); }
                }
                """;

        withSession(rule, kieSession -> {
            final List<String> fired = new ArrayList<>();
            kieSession.addEventListener(new DefaultAgendaEventListener() {
                @Override
                public void afterMatchFired(AfterMatchFiredEvent event) {
                    fired.add(event.getMatch().getRule().getName());
                }
            });

            final EntryPoint persons = kieSession.getEntryPoint("persons");
            final EntryPoint orders = kieSession.getEntryPoint("orders");

            // Person with no matching order → rule fires.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).containsExactly("OrphanedPerson");

            // Insert matching order → NOT becomes unsatisfied; no further firing.
            fired.clear();
            orders.insert(new org.drools.drlx.domain.Order("O1", 30, 100));
            assertThat(kieSession.fireAllRules()).isZero();
        });
    }
```

Import `org.drools.drlx.domain.Order` at the top of the file:

```java
import org.drools.drlx.domain.Order;
```

If the `DrlxBuilderTestSupport.withSession` setup does not automatically wire an `orders` entry point, check how `persons` is wired and replicate. Reading `DrlxBuilderTestSupport.java` before proceeding is worthwhile.

- [ ] **Step 4: Run — expect pass**

```bash
mvn -pl drlx-parser-core -am install -DskipTests 2>&1 | tail -3 && \
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest=NotTest#notWithOuterBinding 2>&1 | tail -10
```

Expected: test passes. If it fails with an "unknown entry point" error, extend `DrlxBuilderTestSupport` to declare `orders` alongside `persons` — same mechanism used for `persons`.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java \
    drlx-parser-core/src/main/java/org/drools/drlx/domain/Order.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): beta-joined NOT via outer binding

Proves outer `var p : /persons[...]` binding resolves inside the
NOT's constraint — Drools builds the NOT GroupElement with a left
input from p. Adds Order domain class if it did not already exist.

Refs #7 #8
EOF
)"
```

---

## Task 7: Positive test — `protoRoundTrip_withNot`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

Proto round-trip is also exercised implicitly by the persistence-mode run of `notSuppressesMatch`, but an explicit unit-level test documents the IR tree shape and is independent of the KieSession machinery.

- [ ] **Step 1: Add `protoRoundTrip_withNot` to `NotTest.java`**

Append after `notWithOuterBinding`:

```java
    @Test
    void protoRoundTrip_withNot() throws Exception {
        // Build a RuleIR containing GroupElementIR(NOT, [PatternIR]) by hand,
        // serialize via DrlxRuleAstParseResult.save + load, assert structural
        // equality. This is a unit-level proto test — no KieSession involved.
        org.drools.drlx.builder.DrlxRuleAstModel.PatternIR inner =
                new org.drools.drlx.builder.DrlxRuleAstModel.PatternIR(
                        "persons", "", "persons",
                        java.util.List.of("age < 18"),
                        null, java.util.List.of());

        org.drools.drlx.builder.DrlxRuleAstModel.GroupElementIR notGroup =
                new org.drools.drlx.builder.DrlxRuleAstModel.GroupElementIR(
                        org.drools.drlx.builder.DrlxRuleAstModel.GroupElementIR.Kind.NOT,
                        java.util.List.of(inner));

        org.drools.drlx.builder.DrlxRuleAstModel.RuleIR ruleIR =
                new org.drools.drlx.builder.DrlxRuleAstModel.RuleIR(
                        "OnlyAdults",
                        java.util.List.of(),
                        java.util.List.of(notGroup),
                        new org.drools.drlx.builder.DrlxRuleAstModel.ConsequenceIR(
                                "System.out.println(\"only adults\");"));

        org.drools.drlx.builder.DrlxRuleAstModel.CompilationUnitIR original =
                new org.drools.drlx.builder.DrlxRuleAstModel.CompilationUnitIR(
                        "org.drools.drlx.parser",
                        java.util.List.of("org.drools.drlx.domain.Person"),
                        java.util.List.of(ruleIR));

        java.nio.file.Path tmpDir = java.nio.file.Files.createTempDirectory("drlx-proto-roundtrip");
        try {
            String source = "// round-trip source";
            org.drools.drlx.builder.DrlxRuleAstParseResult.save(source, original, tmpDir);

            org.drools.drlx.builder.DrlxRuleAstModel.CompilationUnitIR reloaded =
                    org.drools.drlx.builder.DrlxRuleAstParseResult.load(
                            source,
                            org.drools.drlx.builder.DrlxRuleAstParseResult.parseResultFilePath(tmpDir));

            assertThat(reloaded).isEqualTo(original);
        } finally {
            // Clean up — one .pb file inside tmpDir plus the directory itself.
            java.nio.file.Path pb = org.drools.drlx.builder.DrlxRuleAstParseResult.parseResultFilePath(tmpDir);
            java.nio.file.Files.deleteIfExists(pb);
            java.nio.file.Files.deleteIfExists(tmpDir);
        }
    }
```

(If you prefer, add `import` statements to the top of the file and drop the fully-qualified names. FQN is used here so the code in the plan is unambiguously copy-pasteable.)

- [ ] **Step 2: Run**

```bash
mvn -pl drlx-parser-core -am install -DskipTests 2>&1 | tail -3 && \
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest=NotTest#protoRoundTrip_withNot 2>&1 | tail -10
```

Expected: test passes. `record`-based equality (`assertThat(reloaded).isEqualTo(original)`) works out of the box for the IR graph because records implement `equals` / `hashCode` structurally, and `List.copyOf` preserves element order.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): proto round-trip for RuleIR with GroupElementIR(NOT)

Unit-level test — builds the IR by hand, saves via
DrlxRuleAstParseResult.save, loads it back, and asserts structural
equality. Covers the recursive LhsItem/GroupElement proto messages
added in the Task-1 refactor.

Refs #7 #8
EOF
)"
```

---

## Task 8: Negative test — `notWithInnerBinding_failsParse`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

The grammar is `notElement : NOT oopathExpression` and `oopathExpression : '/' oopathRoot (…)`. `var p :` is part of `rulePattern`, not `oopathExpression`, so `not var p : /persons[...]` cannot match — the parser rejects with "no viable alternative" before the visitor sees anything. Document this behaviour with a parse-layer negative test. A visitor-side belt-and-braces check is YAGNI for (α); reinstate it if multi-element `not(...)` ever wraps a full `rulePattern` and needs enforcement there.

- [ ] **Step 1: Append the test**

```java
    @Test
    void notWithInnerBinding_failsParse() {
        // `not var p : /persons[...]` — bindings inside `not` can never escape
        // to the outer scope (the NOT would either match nothing or require the
        // binding to be undefined). The grammar does not accept the `var p :`
        // prefix inside `notElement`, so the parser rejects at that level.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                unit MyUnit;

                rule IllegalInnerBinding {
                    not var p : /persons[ age < 18 ],
                    do { System.out.println("should not compile"); }
                }
                """;

        org.assertj.core.api.Assertions.assertThatThrownBy(() ->
                withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("no viable alternative");
    }
```

- [ ] **Step 2: Run the test**

```bash
mvn -pl drlx-parser-core -am install -DskipTests 2>&1 | tail -3 && \
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='NotTest#notWithInnerBinding_failsParse' 2>&1 | tail -10
```

Expected: test passes. If the parser produces a different error string (e.g. "missing ',' at 'var'"), update the `hasMessageContaining` argument to match whatever actually surfaces — the invariant being tested is that the input is rejected, not the exact message wording.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): inner binding inside `not` rejected at parse time

The grammar `notElement : NOT oopathExpression` does not permit a
`var p :` or `Type bind :` prefix — those live under `rulePattern`.
Documents that `not var p : /persons[...]` is a parse error, pending
a visitor-side check if multi-element `not(...)` ever wraps a full
rulePattern in a follow-up.

Refs #7 #8
EOF
)"
```

---

## Task 9: Negative test — `notMultiElementForm_failsParse`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

- [ ] **Step 1: Append the test**

```java
    @Test
    void notMultiElementForm_failsParse() {
        // Multi-element `not(/a, /b)` is spec'd (DRLX §'not'/'exists' line 597)
        // but deferred to a follow-up issue. Grammar in this landing rejects it.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                unit MyUnit;

                rule MultiNotNotYet {
                    not(/persons[ age < 18 ], /persons[ age > 80 ]),
                    do { System.out.println("unreachable"); }
                }
                """;

        org.assertj.core.api.Assertions.assertThatThrownBy(() ->
                withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("no viable alternative");
    }
```

- [ ] **Step 2: Run**

```bash
mvn -pl drlx-parser-core -am install -DskipTests 2>&1 | tail -3 && \
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest=NotTest#notMultiElementForm_failsParse 2>&1 | tail -10
```

Expected: test passes. If the error text is different (e.g. "missing expression"), adjust the `hasMessageContaining` argument to whatever the parser actually reports — the point is that the parser rejects.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): multi-element form rejected at parse time

Documents that `not(/a, /b)` is not yet supported — grammar reports
no-viable-alternative. Multi-element support is tracked separately
under the epic follow-up.

Refs #7 #8
EOF
)"
```

---

## Task 10: Frozen visitor — `DrlxToJavaParserVisitor.visitNotElement` throws

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxToJavaParserVisitorTest.java`

- [ ] **Step 1: Add the override to `DrlxToJavaParserVisitor`**

Find a natural spot near `visitOopathRoot` (which already has the positional-rejection pattern). Add:

```java
    @Override
    public Node visitNotElement(DrlxParser.NotElementContext ctx) {
        throw new UnsupportedOperationException(
                "`not` is not supported in DrlxToJavaParserVisitor — "
                + "use DrlxToRuleAstVisitor for DRLX→RuleImpl. "
                + "Note: this visitor is frozen for new DRLX syntax.");
    }
```

- [ ] **Step 2: Write the failing test**

Open `DrlxToJavaParserVisitorTest.java`. Follow the pattern of the existing positional/annotations rejection tests in this file. Add:

```java
    @Test
    void javaParserVisitor_throwsOnNot() {
        String cu = """
                unit MyUnit;

                rule R1 {
                    not /persons[ age < 18 ],
                    do {}
                }
                """;

        org.assertj.core.api.Assertions.assertThatThrownBy(
                () -> org.drools.drlx.util.DrlxHelper.parseDrlxCompilationUnitAsJavaParserAST(cu))
                .isInstanceOf(UnsupportedOperationException.class)
                .hasMessageContaining("`not` is not supported in DrlxToJavaParserVisitor");
    }
```

Note: if the existing test helper used in this file is named differently (check via grep — the rule-annotations rejection test used a helper like `parseDrlxCompilationUnitAsJavaParserAST`), use the same one. The canonical DRLX entry point is `drlxCompilationUnit`.

- [ ] **Step 3: Run**

```bash
mvn -pl drlx-parser-core -am install -DskipTests 2>&1 | tail -3 && \
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='DrlxToJavaParserVisitorTest#javaParserVisitor_throwsOnNot' 2>&1 | tail -10
```

Expected: test passes.

- [ ] **Step 4: Run the full suite**

```bash
mvn -pl drlx-parser-core -am install 2>&1 | tail -10
```

Expected: 50 + 6 = 56 tests, all pass.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java \
    drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxToJavaParserVisitorTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(parser): DrlxToJavaParserVisitor throws on `not`

Symmetric with its positional rejection (visitOopathRoot) and its
rule-annotations rejection (visitRuleDeclaration). The visitor is
frozen for new DRLX syntax — TolerantDrlxToJavaParserVisitor will
silent-drop `not` in the next commit to preserve LSP completion.

Refs #7 #8
EOF
)"
```

---

## Task 11: Tolerant visitor — `TolerantDrlxToJavaParserVisitor.visitNotElement` silent-drops + test

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitorTest.java`

- [ ] **Step 1: Add the override to `TolerantDrlxToJavaParserVisitor`**

Symmetric with the `visitRuleDeclaration` override this session added for rule annotations. Place near it:

```java
    @Override
    public Node visitNotElement(DrlxParser.NotElementContext ctx) {
        // Silent-drop `not` so LSP completion keeps working while the user is
        // typing inside the inner pattern. Delegate to the inner oopath so
        // tokenIdJPNodeMap stays populated for the inner expression.
        return visit(ctx.oopathExpression());
    }
```

The returned `Node` is whatever the inherited `visitOopathExpression` produces — in the LSP flow this is enough to keep the token→AST map populated for the inner expression.

- [ ] **Step 2: Add the failing tolerant test**

In `TolerantDrlxToJavaParserVisitorTest.java`, near `incompleteRule_WithAnnotation` (the test added this session):

```java
    @Test
    void incompleteRule_WithNot() {
        // LSP scenario: DRLX unit with a `not` pattern and an unfinished consequence.
        // Tolerant visitor must silent-drop the `not` wrapper so completion still works
        // inside the inner OOPath — DrlxToJavaParserVisitor.visitNotElement throws,
        // and the LSP code-completion flow inherits that throw without this override.
        String compilationUnitString = """
                unit MyUnit;

                rule R1 {
                   not /persons[ age > 18 ],
                   do { System.
                """;

        DrlxHelper.TolerantParseResult<CompilationUnit> parseResult =
                parseDrlxCompilationUnitAsJavaParserASTWithTolerance(compilationUnitString);
        CompilationUnit compilationUnit = parseResult.getResultNode();

        assertThat(compilationUnit.getTypes()).hasSize(1);
        assertThat(compilationUnit.getType(0)).isInstanceOf(RuleDeclaration.class);

        RuleDeclaration ruleDecl = (RuleDeclaration) compilationUnit.getType(0);
        assertThat(ruleDecl.getName().asString()).isEqualTo("R1");

        // Rule body still parsed and completion marker still reachable.
        org.mvel3.parser.ast.expr.RuleBody ruleBody = ruleDecl.getRuleBody();
        assertThat(ruleBody).isNotNull();
        RuleConsequence consequence = (RuleConsequence) ruleBody.getItems().get(ruleBody.getItems().size() - 1);
        BlockStmt consequenceBlock = (BlockStmt) consequence.getStatement();
        Statement stmt = consequenceBlock.getStatements().get(0);
        ExpressionStmt exprStmt = (ExpressionStmt) stmt;
        FieldAccessExpr fieldAccess = (FieldAccessExpr) exprStmt.getExpression();
        assertThat(fieldAccess.getName().asString()).isEqualTo("__COMPLETION_FIELD__");
        assertThat(((NameExpr) fieldAccess.getScope()).getName().asString()).isEqualTo("System");
    }
```

- [ ] **Step 3: Run**

```bash
mvn -pl drlx-parser-core -am install -DskipTests 2>&1 | tail -3 && \
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='TolerantDrlxToJavaParserVisitorTest#incompleteRule_WithNot' 2>&1 | tail -10
```

Expected: test passes.

- [ ] **Step 4: Run the full suite**

```bash
mvn -pl drlx-parser-core -am install 2>&1 | tail -10
```

Expected: 50 existing + 7 new = **57 tests**, all passing. Skip count may be >0 in the no-persist run (the tolerant test is annotated `@DisabledIfSystemProperty(matches = "false")`).

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java \
    drlx-parser-core/src/test/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitorTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
fix(parser): tolerate `not` in LSP visitor

DrlxToJavaParserVisitor.visitNotElement throws (frozen-visitor
policy); TolerantDrlxToJavaParserVisitor inherits that throw, so
drlx-lsp code completion would die as soon as the user typed
`not /` inside a rule body. Override silent-drops the `not`
wrapper and delegates to the inner OOPath, keeping the
tokenIdJPNodeMap populated for the inner expression.

Symmetric with the rule-annotations tolerance fix from this
session.

Refs #7 #8
EOF
)"
```

---

## Task 12: Update `DESIGN.md` — mention `LhsItemIR` / `GroupElementIR`

**Files:**
- Modify: `drlx-parser-core/docs/DESIGN.md` (verify path; may be top-level `docs/DESIGN.md` in the project)

- [ ] **Step 1: Locate the IR / Parser table**

```bash
grep -n "LhsItemIR\|RuleItemIR\|PatternIR\|GroupElementIR\|Parser Package" \
    /home/tkobayas/usr/work/mvel3-development/drlx-parser/docs/DESIGN.md | head -20
```

The Parser Package table sits around line 180 (as of this session). Around lines 170-175 there's an IR/type table.

- [ ] **Step 2: Update the IR table to mention the new types**

Add a row for `LhsItemIR` and `GroupElementIR` in the IR/types table. Replace a section like:

```markdown
| `DrlxRuleAstModel` | In-memory IR. Records: `CompilationUnitIR`, `RuleIR`, `RuleAnnotationIR`, `PatternIR`, `ConsequenceIR`. |
```

with:

```markdown
| `DrlxRuleAstModel` | In-memory IR. Records: `CompilationUnitIR`, `RuleIR(name, annotations, lhs, rhs)`, `RuleAnnotationIR`, `PatternIR`, `ConsequenceIR`, `GroupElementIR`. Sealed interface `LhsItemIR` permits `PatternIR | GroupElementIR` for tree-shape LHS. |
```

(Match the exact phrasing of existing rows; the snippet above is an example — update whatever the real row currently says.)

Add a short paragraph or bullet in the "Constraint Types" / "LHS tree" area (if one exists) noting:
> `GroupElementIR.Kind` currently supports only `NOT`. `EXISTS`, `AND`, `OR` enum slots are reserved for follow-up issues (#9, #11); the proto schema likewise reserves their field numbers.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add docs/DESIGN.md

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
docs: note LhsItemIR / GroupElementIR in DESIGN.md

Tree-shape LHS landed in the IR refactor; DESIGN.md now names the
new types. Enum slots for EXISTS/AND/OR flagged as reserved for
follow-up issues.

Refs #7 #8
EOF
)"
```

---

## Final verification

- [ ] **Step 1: Full install + test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am install 2>&1 | tail -25
```

Expected: `BUILD SUCCESS`. Test count per class (exact values, compare against output):
- `InlineCastTest`: 2
- `PositionalTest`: 8
- `RuleAnnotationsTest`: 9
- `NotTest`: **5** (notSuppressesMatch, notWithOuterBinding, protoRoundTrip_withNot, notWithInnerBinding_failsParse, notMultiElementForm_failsParse)
- `DrlxRuleBuilderTest`: 5
- `DrlxCompilerTest`: 6
- `DrlxToJavaParserVisitorTest`: 5 (was 4 + `javaParserVisitor_throwsOnNot`)
- `TolerantDrlxToJavaParserVisitorTest`: 8 (was 7 + `incompleteRule_WithNot`)
- `DrlxParserTest`: 4
- with-persist run total: **52**
- `DrlxCompilerNoPersistTest`: 5
- no-persist run total: 5
- Grand total: **57**

If counts don't match, find the discrepancy before wrapping — usually a dropped test or an unfinished task.

- [ ] **Step 2: Git log sanity check**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser log --oneline main..HEAD
```

Expected: ~12 commits on `main` starting with `refactor(ir)` and ending with `docs`, all trailing `Refs #7 #8`.

- [ ] **Step 3: Close / update GitHub issues**

```bash
gh issue comment 7 --repo tkobayas/drlx-parser --body "Landed. Tree-shape LHS + proto + visitor + runtime all merged; see commits \`refactor(ir)\` through \`docs\` on main."
gh issue close 7 --repo tkobayas/drlx-parser

gh issue comment 8 --repo tkobayas/drlx-parser --body "Single-element \`not /pattern\` landed. Multi-element \`not(/a, /b)\` still open — tracked as follow-up under this issue."
# Don't close #8 — the multi-element form is still open.
```

(Close only #7; leave #8 open for the multi-element follow-up.)

- [ ] **Step 4: HANDOFF update**

Update `/home/tkobayas/claude/public/drlx-parser/HANDOFF.md` via the `handover` skill or by hand. Key content: commit range for this landing, test count (57), issue status (#7 closed, #8 partially shipped), and next candidate feature.

---

## Appendix — notes for the implementer

- **The Task 1 commit is unusually large** (≥200 LOC spanning 6 files). That's deliberate; splitting would leave the build red between commits. Review the diff carefully but commit atomically.
- **The `typeName`/`entryPoint` distinction inside `buildNotElement`** (Task 5 Step 3) may require a follow-up if runtime resolution fails. The existing `PositionalTest.java` and `InlineCastTest.java` handle the same field in their `PatternIR` construction — cross-check before writing new resolution logic.
- **`DrlxBuilderTestSupport.withSession`** may need a new `orders` entry-point wiring for Task 6. If so, prefer extending the support class once (for all future tests needing `orders`) rather than inline-configuring per test.
- **Proto round-trip in persist-mode** — the `notSuppressesMatch` test runs in both persist and no-persist modes; the persist mode implicitly exercises `DrlxRuleAstParseResult.save`/`load`. The explicit `protoRoundTrip_withNot` in Task 7 is additional belt-and-braces coverage.
- **All commits use `Refs #7 #8`**, not `Closes`. Close #7 only after final verification. Do not auto-close #8 — multi-element `not` is still outstanding within that issue's scope.
