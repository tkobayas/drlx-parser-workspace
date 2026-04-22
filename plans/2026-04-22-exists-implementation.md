# `exists` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `exists /pattern,` and `exists(/a[, /b, ...]),` in a single landing (#9). Mirrors the NOT pipeline at every layer: lexer, grammar, IR, visitor, runtime, proto, frozen/tolerant visitors.

**Architecture:** Adds an `existsElement` grammar production parallel to `notElement`. The visitor extracts a shared helper (`buildGroupElementFromOopaths`) so both NOT and EXISTS are one-line wrappers. Runtime extends the `GroupElementIR.Kind` switch and generalises the multi-child AND-wrap condition from NOT-only to NOT|EXISTS (Drools `newExistsInstance()` enforces the same single-child constraint as NOT). Proto gets a new additive enum value `GROUP_ELEMENT_KIND_EXISTS = 2`. One pre-requisite task: migrate the proto build from checked-in generated code to `protobuf-maven-plugin` regeneration, so that every proto schema change from now on is a pure `.proto` edit.

**Tech Stack:** Java 17, Maven, ANTLR4 (existing `antlr4-maven-plugin`), `protobuf-maven-plugin` (xolstice, new), `kr.motd.maven:os-maven-plugin` (build extension for OS classifier detection), JUnit 5, AssertJ, Drools `GroupElementFactory.newExistsInstance()`.

**Spec:** `/home/tkobayas/claude/public/drlx-parser/specs/2026-04-22-exists-design.md` (commit `798fb9e`).

**Issue:** `#9`.

---

## File Structure

All paths relative to `/home/tkobayas/usr/work/mvel3-development/drlx-parser/`.

| Path | Role |
|------|------|
| `drlx-parser-core/pom.xml` | Add `protobuf-maven-plugin` + register generated proto dir with `build-helper-maven-plugin`. |
| `pom.xml` (parent) | Add `os-maven-plugin` as a build extension + pin plugin versions (`version.protobuf-maven-plugin`, `version.os-maven-plugin`). |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java` | **Delete.** Regenerated at build time from now on. |
| `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` | Add `GROUP_ELEMENT_KIND_EXISTS = 2` enum value; delete the "reserved" comment. |
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4` | Add `EXISTS : 'exists';` token. |
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` | Add `existsElement` to `ruleItem`; add the `existsElement` production. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` | Expose `GroupElementIR.Kind.EXISTS`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Extract `buildGroupElementFromOopaths` helper; refactor `buildNotElement`; add `buildExistsElement`; extend `buildRule` dispatch. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` | Extend `buildLhs` switch + generalise AND-wrap condition to NOT\|EXISTS. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` | Extend `fromProtoGroupKind` / `toProtoGroupKind` to handle `EXISTS`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java` | Add `visitExistsElement` that throws (frozen-visitor convention). |
| `drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java` | Add `visitExistsElement` delegating to `ctx.oopathExpression(0)` (LSP tolerance). |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java` | **Create.** 5 new tests (spec Decision #7 — reduced mirror). |

**Maven rule:** after modifying `drlx-parser-core`, run `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install`. ANTLR + proto regen are automatic on `compile`. Do NOT `cd` — use `-f` / `-C` flags.

**Commit convention:** scoped imperative subject (`build(proto):`, `feat(grammar):`, `test(exists):`, etc.), HEREDOC body, `Refs #9` trailer for in-progress commits and `Closes #9` on the last. No `Co-Authored-By`. No emoji. Commit directly on `main`.

**Current test count:** 68 passing (63 default + 5 no-persist). **Expected after this landing: 73** (68 + 5 new EXISTS tests).

---

## Task 1: Migrate proto build to `protobuf-maven-plugin`

**Files:**
- Modify: `pom.xml` (parent — add plugin/extension versions + os-maven-plugin build extension)
- Modify: `drlx-parser-core/pom.xml` (add protobuf-maven-plugin + extend build-helper-maven-plugin source list)
- Delete: `drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java`

Rationale: the current `DrlxRuleAstProto.java` is a checked-in `protoc` output (marked `DO NOT EDIT`). Without `protoc` on `PATH`, schema evolution is blocked. Migrating to `protobuf-maven-plugin` (which downloads the pinned `protoc` binary from Maven Central) makes schema changes a pure `.proto` edit — exactly the pattern `antlr4-maven-plugin` already uses in this module. This task is an independent refactor; all subsequent proto edits (including this landing's EXISTS value) just change `.proto`.

- [ ] **Step 1: Parent `pom.xml` — add plugin version properties and os-maven-plugin extension**

Read `pom.xml` at the workspace root first so you can locate the right `<properties>` block and the existing `<build>` block. Then:

In `<properties>` (near `<version.protobuf>3.25.5</version.protobuf>` at line ~29), add:

```xml
        <version.protobuf-maven-plugin>0.6.1</version.protobuf-maven-plugin>
        <version.os-maven-plugin>1.7.1</version.os-maven-plugin>
```

In `<build>`, add (or extend) an `<extensions>` block:

```xml
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>${version.os-maven-plugin}</version>
            </extension>
        </extensions>
```

`os-maven-plugin` populates the `${os.detected.classifier}` Maven property (e.g. `linux-x86_64`) used to pick the correct `protoc` binary.

Also add plugin management for the protobuf plugin so version pinning is centralised. Inside `<build><pluginManagement><plugins>` (create these wrappers if they don't already exist):

```xml
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>${version.protobuf-maven-plugin}</version>
            </plugin>
```

- [ ] **Step 2: `drlx-parser-core/pom.xml` — add protobuf-maven-plugin execution and extend build-helper source list**

Immediately after the `antlr4-maven-plugin` `<plugin>` block (ends at line ~94), add:

```xml
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:${version.protobuf}:exe:${os.detected.classifier}</protocArtifact>
                    <!-- defaults: protoSourceRoot = src/main/proto; outputDirectory = target/generated-sources/protobuf/java -->
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

Then extend the existing `build-helper-maven-plugin` execution at line ~95-114 to also add the proto generated-sources dir. Change the `<sources>` block:

**Before:**

```xml
                            <sources>
                                <source>target/generated-sources/antlr4</source>
                            </sources>
```

**After:**

```xml
                            <sources>
                                <source>target/generated-sources/antlr4</source>
                                <source>target/generated-sources/protobuf/java</source>
                            </sources>
```

- [ ] **Step 3: Delete the checked-in generated file**

```bash
rm /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java
```

The regenerated version will live at `target/generated-sources/protobuf/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java`. Callers import from the same package — no code changes needed.

- [ ] **Step 4: Full build — prove regen produces byte-equivalent behaviour**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am install 2>&1 | grep -E "Tests run:|BUILD|protobuf-maven-plugin" | tail -20
```

Expected:
- `protobuf-maven-plugin` logs lines like `[INFO] --- protobuf:0.6.1:compile (default) @ drlx-parser-core ---` and `[INFO] Compiling 1 proto file(s) to …/target/generated-sources/protobuf/java`.
- `BUILD SUCCESS`.
- All 68 existing tests green (including `protoRoundTrip_withNot`, which proves the regenerated file is functionally equivalent).

If `protoc` download fails (network or classifier issue), the error is `Could not resolve dependencies for com.google.protobuf:protoc:exe:…`. Check that `os-maven-plugin` is registered as a `<extension>` (not a plain plugin) — the classifier is only populated when it runs as an extension.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    pom.xml drlx-parser-core/pom.xml

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser rm \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
build(proto): regenerate DrlxRuleAstProto via protobuf-maven-plugin

The checked-in DrlxRuleAstProto.java (marked DO NOT EDIT) made
schema evolution require a system protoc install. Switch to the
xolstice protobuf-maven-plugin, which downloads the pinned protoc
3.25.5 artifact from Maven Central and regenerates at compile time
— the same pattern antlr4-maven-plugin already uses in this
module. os-maven-plugin is added as a build extension to populate
${os.detected.classifier}.

No behaviour change: 68 tests still pass (incl. protoRoundTrip_withNot,
which exercises the regenerated class).

Refs #9
EOF
)"
```

---

## Task 2: RED — failing happy-path test `existsAllowsMatch`

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java`

Fails today because the lexer doesn't recognise `exists` as a keyword — the parser then tries to match `exists /persons[…]` as a `rulePattern` (`identifier identifier (':'|'=') oopathExpression ','`) and fails. Will flip GREEN after Task 3.

- [ ] **Step 1: Create the file with the first test**

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

class ExistsTest extends DrlxBuilderTestSupport {

    @Test
    void existsAllowsMatch() {
        // `exists /persons[age>=18]` — rule fires ONLY when at least one adult
        // person is inserted. Inverse of notSuppressesMatch.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;

                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;

                rule HasAdult {
                    exists /persons[ age >= 18 ],
                    do { System.out.println("adult exists"); }
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

            // No facts → EXISTS unsatisfied, rule does not fire.
            assertThat(kieSession.fireAllRules()).isZero();
            assertThat(fired).isEmpty();

            // Insert a child → still no adult, EXISTS unsatisfied.
            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isZero();

            // Insert an adult → EXISTS satisfied, rule fires once.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).containsExactly("HasAdult");
        });
    }
}
```

- [ ] **Step 2: Run the target test — expect RED**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='ExistsTest#existsAllowsMatch' 2>&1 | tail -20
```

Expected: FAILURE. Surefire shows a parse error — probably `DRLX parse error at line N:M - …` pointing at the `exists` token or the following `/`. Either form is fine; the point is the parser rejects `exists` in the current grammar.

- [ ] **Step 3: Commit (RED)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(exists): failing happy-path for bare exists (RED)

Adds existsAllowsMatch — `exists /persons[age>=18]` fires only
when an adult exists. Fails today because the grammar does not
recognise `exists`; flips GREEN once the lexer/grammar/visitor/
runtime/proto atomic landing arrives.

Refs #9
EOF
)"
```

---

## Task 3: Atomic plumbing — lexer + grammar + IR + visitor + runtime + proto + frozen/tolerant

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4`
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java`

All these layers are interdependent — partial state leaves the build red. One atomic commit. Flips `existsAllowsMatch` GREEN.

- [ ] **Step 1: Lexer — add `EXISTS` token**

In `DrlxLexer.g4`, immediately after `NOT : 'not';`:

**Before:**

```antlr
NOT  : 'not';
```

**After:**

```antlr
NOT    : 'not';
EXISTS : 'exists';
```

- [ ] **Step 2: Parser — add `existsElement` to `ruleItem`; add `existsElement` production**

In `DrlxParser.g4`:

**Replace `ruleItem` (currently at lines ~51-55):**

```antlr
ruleItem
    : rulePattern
    | notElement
    | ruleConsequence
    ;
```

**With:**

```antlr
ruleItem
    : rulePattern
    | notElement
    | existsElement
    | ruleConsequence
    ;
```

**Append a new `existsElement` production immediately after the `notElement` production (after line ~64):**

```antlr
// 'exists' group element. Structurally identical to notElement —
// bare `exists /a,` for single child; paren `exists(/a[, /b, ...]),`
// for single-in-parens or multi-element. Both NOT and EXISTS are
// scope-delimiter ALWAYS in Drools. DRLX spec §"'not' / 'exists'"
// line 597.
existsElement
    : EXISTS oopathExpression ','
    | EXISTS '(' oopathExpression (',' oopathExpression)* ')' ','
    ;
```

- [ ] **Step 3: Proto — add `GROUP_ELEMENT_KIND_EXISTS = 2`**

In `drlx_rule_ast.proto`:

**Before:**

```proto
enum GroupElementKind {
  GROUP_ELEMENT_KIND_UNSPECIFIED = 0;
  GROUP_ELEMENT_KIND_NOT = 1;
  // 2, 3, 4 reserved for EXISTS, AND, OR
}
```

**After:**

```proto
enum GroupElementKind {
  GROUP_ELEMENT_KIND_UNSPECIFIED = 0;
  GROUP_ELEMENT_KIND_NOT = 1;
  GROUP_ELEMENT_KIND_EXISTS = 2;
  // 3, 4 reserved for AND, OR
}
```

- [ ] **Step 4: IR — expose `Kind.EXISTS`**

In `DrlxRuleAstModel.java`:

**Before:**

```java
    public record GroupElementIR(Kind kind, List<LhsItemIR> children) implements LhsItemIR {
        public enum Kind {
            NOT
            // EXISTS, AND, OR — reserved for follow-up issues #9, #11.
        }
    }
```

**After:**

```java
    public record GroupElementIR(Kind kind, List<LhsItemIR> children) implements LhsItemIR {
        public enum Kind {
            NOT,
            EXISTS
            // AND, OR — reserved for follow-up issue #11.
        }
    }
```

- [ ] **Step 5: Visitor — extract helper, add `buildExistsElement`, extend dispatch**

In `DrlxToRuleAstVisitor.java`:

**Replace the existing `buildNotElement` method (at lines ~202-208):**

```java
    private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx) {
        List<LhsItemIR> children = new ArrayList<>();
        for (DrlxParser.OopathExpressionContext oopathCtx : ctx.oopathExpression()) {
            children.add(buildPatternFromOopath(oopathCtx));
        }
        return new GroupElementIR(GroupElementIR.Kind.NOT, List.copyOf(children));
    }
```

**With (adds `buildExistsElement` and extracts the shared helper):**

```java
    private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx) {
        return buildGroupElementFromOopaths(ctx.oopathExpression(), GroupElementIR.Kind.NOT);
    }

    private GroupElementIR buildExistsElement(DrlxParser.ExistsElementContext ctx) {
        return buildGroupElementFromOopaths(ctx.oopathExpression(), GroupElementIR.Kind.EXISTS);
    }

    private GroupElementIR buildGroupElementFromOopaths(
            List<DrlxParser.OopathExpressionContext> oopathCtxs,
            GroupElementIR.Kind kind) {
        List<LhsItemIR> children = new ArrayList<>();
        for (DrlxParser.OopathExpressionContext oopathCtx : oopathCtxs) {
            children.add(buildPatternFromOopath(oopathCtx));
        }
        return new GroupElementIR(kind, List.copyOf(children));
    }
```

No new imports — `ArrayList`, `List`, `DrlxParser`, `LhsItemIR`, `GroupElementIR` are all already imported.

**Extend the dispatch in `buildRule` at lines ~99-105:**

**Before:**

```java
                } else if (itemCtx.rulePattern() != null) {
                    lhs.add(buildPattern(itemCtx.rulePattern()));
                } else if (itemCtx.notElement() != null) {
                    lhs.add(buildNotElement(itemCtx.notElement()));
                } else {
                    throw new IllegalArgumentException("Unsupported rule item: " + itemCtx.getText());
                }
```

**After:**

```java
                } else if (itemCtx.rulePattern() != null) {
                    lhs.add(buildPattern(itemCtx.rulePattern()));
                } else if (itemCtx.notElement() != null) {
                    lhs.add(buildNotElement(itemCtx.notElement()));
                } else if (itemCtx.existsElement() != null) {
                    lhs.add(buildExistsElement(itemCtx.existsElement()));
                } else {
                    throw new IllegalArgumentException("Unsupported rule item: " + itemCtx.getText());
                }
```

- [ ] **Step 6: Runtime — extend switch + generalise AND-wrap**

In `DrlxRuleAstRuntimeBuilder.java`, inside `buildLhs` (at lines ~237-252):

**Before:**

```java
            } else if (item instanceof GroupElementIR group) {
                GroupElement ge = switch (group.kind()) {
                    case NOT -> GroupElementFactory.newNotInstance();
                };
                // Drools' NOT requires a single child. For multi-element
                // not(/a, /b) wrap the children in an AND so the NOT has
                // exactly one child that represents the conjunction.
                if (group.kind() == GroupElementIR.Kind.NOT && group.children().size() > 1) {
                    GroupElement andInstance = GroupElementFactory.newAndInstance();
                    buildLhs(group.children(), andInstance, typeResolver, entryPointTypes, unitClass, boundVariables);
                    ge.addChild(andInstance);
                } else {
                    buildLhs(group.children(), ge, typeResolver, entryPointTypes, unitClass, boundVariables);
                }
                parent.addChild(ge);
```

**After:**

```java
            } else if (item instanceof GroupElementIR group) {
                GroupElement ge = switch (group.kind()) {
                    case NOT    -> GroupElementFactory.newNotInstance();
                    case EXISTS -> GroupElementFactory.newExistsInstance();
                };
                // Drools enforces exactly one child on both NOT and EXISTS
                // (GroupElement.addChild). For multi-element forms wrap the
                // children in an AND so the group element has exactly one
                // child that represents the conjunction.
                if ((group.kind() == GroupElementIR.Kind.NOT
                        || group.kind() == GroupElementIR.Kind.EXISTS)
                        && group.children().size() > 1) {
                    GroupElement andInstance = GroupElementFactory.newAndInstance();
                    buildLhs(group.children(), andInstance, typeResolver, entryPointTypes, unitClass, boundVariables);
                    ge.addChild(andInstance);
                } else {
                    buildLhs(group.children(), ge, typeResolver, entryPointTypes, unitClass, boundVariables);
                }
                parent.addChild(ge);
```

- [ ] **Step 7: Proto mapping — extend `fromProtoGroupKind` / `toProtoGroupKind`**

In `DrlxRuleAstParseResult.java` (at lines ~180-192):

**Before:**

```java
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
```

**After:**

```java
    private static GroupElementIR.Kind fromProtoGroupKind(DrlxRuleAstProto.GroupElementKind k) {
        return switch (k) {
            case GROUP_ELEMENT_KIND_NOT    -> GroupElementIR.Kind.NOT;
            case GROUP_ELEMENT_KIND_EXISTS -> GroupElementIR.Kind.EXISTS;
            case GROUP_ELEMENT_KIND_UNSPECIFIED, UNRECOGNIZED ->
                    throw new IllegalStateException("Unknown proto group-element kind: " + k);
        };
    }

    private static DrlxRuleAstProto.GroupElementKind toProtoGroupKind(GroupElementIR.Kind k) {
        return switch (k) {
            case NOT    -> DrlxRuleAstProto.GroupElementKind.GROUP_ELEMENT_KIND_NOT;
            case EXISTS -> DrlxRuleAstProto.GroupElementKind.GROUP_ELEMENT_KIND_EXISTS;
        };
    }
```

- [ ] **Step 8: Frozen visitor — add `visitExistsElement` that throws**

In `DrlxToJavaParserVisitor.java`, immediately after the existing `visitNotElement` method (at lines ~402-408):

```java
    @Override
    public Node visitExistsElement(DrlxParser.ExistsElementContext ctx) {
        throw new UnsupportedOperationException(
                "`exists` is not supported in DrlxToJavaParserVisitor — "
                + "use DrlxToRuleAstVisitor for DRLX→RuleImpl. "
                + "Note: this visitor is frozen for new DRLX syntax.");
    }
```

- [ ] **Step 9: Tolerant visitor — add `visitExistsElement` delegating to first oopath**

In `TolerantDrlxToJavaParserVisitor.java`, immediately after the existing `visitNotElement` method (at lines ~50-64):

```java
    @Override
    public Node visitExistsElement(DrlxParser.ExistsElementContext ctx) {
        // Silent-drop `exists` so LSP completion keeps working while the user
        // is typing inside the inner pattern. Same strategy as visitNotElement:
        // wrap the (first) inner OOPath in a RulePattern stand-in.
        SimpleName type = new SimpleName("var");
        SimpleName bind = new SimpleName("_");
        OOPathExpr expr = (OOPathExpr) visit(ctx.oopathExpression(0));
        RulePattern pattern = new RulePattern(null, type, bind, expr);
        type.setParentNode(pattern);
        bind.setParentNode(pattern);
        expr.setParentNode(pattern);
        return pattern;
    }
```

No new imports — `SimpleName`, `OOPathExpr`, `RulePattern`, `Node`, `DrlxParser` are all already imported at the top of the file.

- [ ] **Step 10: Run the RED test — expect GREEN**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='ExistsTest#existsAllowsMatch' 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`, `Tests run: 1, Failures: 0, Errors: 0`. If it still fails, the most likely causes in order:
1. Proto regen didn't pick up the new enum value — verify `target/generated-sources/protobuf/java/org/drools/drlx/builder/proto/DrlxRuleAstProto.java` contains `GROUP_ELEMENT_KIND_EXISTS`.
2. ANTLR regen didn't produce `ExistsElementContext` — verify the lexer has `EXISTS : 'exists';` above any catch-all IDENTIFIER rule (order matters in ANTLR).
3. Dispatch branch missing — verify `buildRule` calls `buildExistsElement` when `itemCtx.existsElement() != null`.

- [ ] **Step 11: Run the full suite — prove no regressions**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am install 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`, grand total **69 tests passing** (68 baseline + 1 new, 0 failures). NotTest still 7/7 green (unchanged).

- [ ] **Step 12: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4 \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java \
    drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java \
    drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(grammar): accept exists /pattern and exists(/a[, /b, ...])

Adds the 'exists' group-element CE: bare form `exists /a,` and paren
form `exists(/a[, /b, ...]),`. Mirrors notElement structurally —
same trailing-comma policy, same inner-binding restriction (parse
error symmetric with NOT).

Plumbing:
 - Lexer: new EXISTS token.
 - Grammar: new existsElement production; added to ruleItem.
 - IR: GroupElementIR.Kind.EXISTS exposed (slot previously reserved).
 - Visitor: extracted shared buildGroupElementFromOopaths helper;
   buildNotElement and new buildExistsElement both delegate. Dispatch
   in buildRule extended.
 - Runtime: switch extended to newExistsInstance(); AND-wrap condition
   generalised from NOT-only to NOT|EXISTS since Drools enforces the
   same single-child constraint on both.
 - Proto: additive GROUP_ELEMENT_KIND_EXISTS = 2. Old .pb files still
   deserialise (they only contain NOT).
 - Frozen / tolerant visitors: mirror visitNotElement.

Flips existsAllowsMatch GREEN. Total tests now 69 (was 68).

Refs #9
EOF
)"
```

---

## Task 4: Happy test — `existsWithOuterBinding`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java`

Proves beta-join from an outer pattern's binding into the EXISTS constraint — symmetric with `notWithOuterBinding`.

- [ ] **Step 1: Append the test**

Insert inside `ExistsTest` after `existsAllowsMatch`. Add `Order` to the domain imports at the top of the file:

```java
import org.drools.drlx.domain.Order;
```

```java
    @Test
    void existsWithOuterBinding() {
        // `Person p : /persons[...], exists /orders[customerId == p.age]` —
        // outer binding 'p' is referenced inside the EXISTS constraint, proving
        // beta-join from the outer pattern into the EXISTS group element.
        //
        // Domain note: correlates Order.customerId with Person.age only to
        // avoid adding a new integer field to Person. The semantic — "fire
        // when a person exists AND at least one order references them" — is
        // what's tested.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Order;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule ConfirmedPerson {
                    Person p : /persons[ age > 0 ],
                    exists /orders[ customerId == p.age ],
                    do { System.out.println("confirmed: " + p); }
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

            // Person but no order → EXISTS unsatisfied, no firing.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isZero();
            assertThat(fired).isEmpty();

            // Add a matching order (customerId == Alice.age = 30) → fires.
            orders.insert(new Order("O1", 30, 100));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).containsExactly("ConfirmedPerson");
        });
    }
```

- [ ] **Step 2: Run the target test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='ExistsTest#existsWithOuterBinding' 2>&1 | tail -10
```

Expected: passes. If it fails with a "p is not declared" error, the scope-delimiter behaviour for EXISTS isn't wiring the outer declaration through — but since EXISTS is `ScopeDelimiter.ALWAYS` on the inner side only (outer bindings leak IN, inner bindings don't leak OUT), this should work identically to NOT.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(exists): outer binding referenced inside exists constraint

existsWithOuterBinding — `Person p : /persons[...], exists
/orders[customerId == p.age]`. Proves beta-join from the outer
pattern's binding `p` into the EXISTS group element. Symmetric
with notWithOuterBinding.

Refs #9
EOF
)"
```

---

## Task 5: Happy test — `existsMultiElement_crossProduct`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java`

Exercises the generalised AND-wrap path for multi-child EXISTS. Inverse semantics of `notMultiElement_crossProduct`.

- [ ] **Step 1: Append the test**

```java
    @Test
    void existsMultiElement_crossProduct() {
        // `exists(/persons[age>=18], /orders[amount>1000])` — EXISTS-with-AND
        // semantics: fires while at least one `(adult, high-value order)`
        // combination exists. Exercises the AND-wrap path for multi-child
        // EXISTS (Drools' newExistsInstance() enforces single-child).
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Order;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule HasAdultWithHighValueOrder {
                    exists(/persons[ age >= 18 ], /orders[ amount > 1000 ]),
                    do { System.out.println("adult with high-value order exists"); }
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

            // Neither side → EXISTS unsatisfied, no firing.
            assertThat(kieSession.fireAllRules()).isZero();

            // Only adult → EXISTS still unsatisfied (missing order side).
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isZero();

            // Add a high-value order → both sides match → EXISTS satisfied,
            // rule fires once.
            orders.insert(new Order("O1", 99, 5000));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).containsExactly("HasAdultWithHighValueOrder");
        });
    }
```

- [ ] **Step 2: Run the target test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='ExistsTest#existsMultiElement_crossProduct' 2>&1 | tail -10
```

Expected: passes. If it fails with `EXISTS can have only a single child element`, the AND-wrap generalisation in Task 3 Step 6 is incorrect — verify the condition reads `NOT || EXISTS` (not `NOT && EXISTS`).

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(exists): multi-element exists(/a, /b) cross-product

existsMultiElement_crossProduct — `exists(/persons[age>=18],
/orders[amount>1000])` fires when at least one (adult,
high-value order) combination exists. Exercises the generalised
AND-wrap path (Drools' newExistsInstance() enforces single-child,
same as NOT).

Refs #9
EOF
)"
```

---

## Task 6: Negative test — `existsWithInnerBinding_failsParse`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java`

Guards spec Decision #2 — inner bindings inside `exists` are a parse error (symmetric with NOT).

- [ ] **Step 1: Append the test**

Add `assertThatThrownBy` to the static imports at the top of the file:

```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
```

```java
    @Test
    void existsWithInnerBinding_failsParse() {
        // `exists var p : /persons[...]` — bindings inside `exists` can never
        // escape to the outer scope. The grammar does not accept `var p :`
        // inside existsElement, so the parser rejects at that level.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule IllegalInnerExistsBinding {
                    exists var p : /persons[ age >= 18 ],
                    do { System.out.println("should not compile"); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("parse error")
                .hasMessageContaining("var");
    }
```

- [ ] **Step 2: Run the target test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='ExistsTest#existsWithInnerBinding_failsParse' 2>&1 | tail -10
```

Expected: passes — `var` is not part of `oopathExpression`, so the parser fails when it encounters it inside `existsElement`.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(exists): inner binding rejected at parse time

existsWithInnerBinding_failsParse — `exists var p : /persons[...]`
is a parse error. Symmetric with NOT's inner-binding restriction.
Guards spec Decision #2.

Refs #9
EOF
)"
```

---

## Task 7: Negative test — `existsEmpty_failsParse`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java`

Guards spec Decision #4 — `exists()` with zero inner oopaths is a parse error.

- [ ] **Step 1: Append the test**

```java
    @Test
    void existsEmpty_failsParse() {
        // `exists()` — parens with zero inner oopaths. Grammar's first
        // oopathExpression is not optional, so this is a parse error.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule EmptyExists {
                    exists(),
                    do { System.out.println("unreachable"); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, kieSession -> { /* unreachable */ }))
                .hasMessageContaining("parse error");
    }
```

- [ ] **Step 2: Run the target test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='ExistsTest#existsEmpty_failsParse' 2>&1 | tail -10
```

Expected: passes.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/ExistsTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(exists): empty exists() rejected at parse time

existsEmpty_failsParse — `exists()` with zero inner oopaths is a
parse error. Grammar requires at least one oopathExpression inside
the parens. Guards spec Decision #4.

Refs #9
EOF
)"
```

---

## Task 8: Final verification + issue close + HANDOFF update

- [ ] **Step 1: Full install + test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am install 2>&1 | grep -E "Tests run:|BUILD" | tail -15
```

Expected: `BUILD SUCCESS`, grand total **73 tests passing** (default-test 68 + no-persist 5), 0 failures. Per-class:
- `NotTest`: **7** (unchanged).
- `ExistsTest`: **5** — `existsAllowsMatch`, `existsWithOuterBinding`, `existsMultiElement_crossProduct`, `existsWithInnerBinding_failsParse`, `existsEmpty_failsParse`.

- [ ] **Step 2: Git log sanity check**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser log --oneline origin/main..HEAD
```

Expected: 7 new commits since the last push, in order:
1. `build(proto): regenerate DrlxRuleAstProto via protobuf-maven-plugin`
2. `test(exists): failing happy-path for bare exists (RED)`
3. `feat(grammar): accept exists /pattern and exists(/a[, /b, ...])`
4. `test(exists): outer binding referenced inside exists constraint`
5. `test(exists): multi-element exists(/a, /b) cross-product`
6. `test(exists): inner binding rejected at parse time`
7. `test(exists): empty exists() rejected at parse time`

- [ ] **Step 3: Push**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push origin main
```

- [ ] **Step 4: Close #9**

```bash
gh issue comment 9 --repo tkobayas/drlx-parser --body "exists /pattern and exists(/a[, /b, ...]) landed together. Proto build migrated to protobuf-maven-plugin (unblocks future schema evolution). 5 new tests in ExistsTest; full suite 73 passing."
gh issue close 9 --repo tkobayas/drlx-parser
```

- [ ] **Step 5: HANDOFF update**

Update `/home/tkobayas/claude/public/drlx-parser/HANDOFF.md` via the `handover` skill or by hand. Key content:
- Session: #9 `exists` landed — bare + paren + multi-element, 7 commits.
- Test count: 73 (was 68).
- Proto build migrated to `protobuf-maven-plugin`; future schema changes are `.proto`-only edits.
- #9 CLOSED.
- Next candidate: #11 `and(...)` / `or(...)` group CEs — `Kind.AND`, `Kind.OR` slots still reserved in the IR enum and proto. Can reuse `buildGroupElementFromOopaths` (new helper). AND-wrap logic in `buildLhs` may need different generalisation (AND/OR are not single-child).
- Gotchas (carry forward): NOT/EXISTS single-child + AND-wrap; ANTLR labelled-alternative Context split.

Then commit HANDOFF and push the workspace.

---

## Appendix — notes for the implementer

- **Why Task 1 is independent of EXISTS:** removing the checked-in `DrlxRuleAstProto.java` and migrating to `protobuf-maven-plugin` is mechanical — it produces the same bytecode. Committing it separately keeps the #9 feature commit focused and makes it trivial to back out the infra change if it misbehaves in CI.
- **`protobuf-maven-plugin` 0.6.1 quirks:** the plugin expects `${os.detected.classifier}`, which is only populated when `os-maven-plugin` is registered as a `<extension>` (not a plain `<plugin>`). Registering it in the parent `pom.xml` covers all submodules.
- **Why Task 3 is atomic:** the lexer, grammar, visitor, runtime, and proto are all coupled — adding `Kind.EXISTS` to the IR without extending the runtime switch makes the build red; extending the grammar without extending the visitor dispatch makes `buildRule` throw at runtime. Splitting would leave the build broken between commits.
- **AND-wrap condition generalisation:** reading the runtime diff in Task 3 Step 6, the `&& size > 1` only triggers when the user writes the multi-element form. Single-child (bare or single-in-parens) EXISTS/NOT bypass the wrap and attach directly — the existing Drools behaviour.
- **Inner-binding symmetry:** `exists var p : /a` fails at the grammar level because `oopathExpression` has no `var bind :` prefix production. No visitor check needed. Same mechanism as `notWithInnerBinding_failsParse`.
- **Frozen visitor intent:** `DrlxToJavaParserVisitor` is explicitly not updated for new DRLX syntax — it's kept in place for the LSP path. Throwing on `visitExistsElement` preserves that policy; the tolerant subclass overrides to keep LSP working.
- **`#11` reuse hook:** `buildGroupElementFromOopaths` takes a `GroupElementIR.Kind` enum, so adding AND/OR variants is one-line-per-variant. The runtime switch and AND-wrap condition will need more thought for #11 since AND/OR are not single-child CEs.
