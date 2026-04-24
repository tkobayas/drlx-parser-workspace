# Property-reactive watch list — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Spec:** `specs/2026-04-24-property-reactive-watch-list-design.md`
**Issue:** #13

**Goal:** Add a DRLX pattern syntax for the property-reactive watch list (second `[...]` block on `oopathRoot`) with DRL7-parity validation (wildcards, exclusions, unknown/duplicate detection — minus the class-level `@PropertyReactive` check).

**Architecture:** Grammar adds an optional second `[...]` block on `oopathRoot` plus a `watchItem` rule covering `*`, `!*`, `name`, `!name`. The visitor extracts raw strings into a new `PatternIR.watchedProperties` field; the runtime builder validates each entry against `ClassUtils.getAccessibleProperties` and calls `pattern.addWatchedProperties`. The proto schema gains one repeated string field. Error mode: `RuntimeException` at build time.

**Tech Stack:** ANTLR4 (grammar), protobuf-java (IR cache), Drools `Pattern` (runtime), AssertJ + JUnit 5 + `TrackingAgendaEventListener` (tests).

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` | Modify | Add second `[...]` block + `watchItem` rule; make first block empty-OK |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` | Modify | Extend `PatternIR` record with `watchedProperties` |
| `drlx-parser-core/src/main/proto/drlx_rule_ast.proto` | Modify | Add `repeated string watched_properties = 8` |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` | Modify | Serialize / deserialize watch list |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Modify | Extract watch list, pass to `PatternIR` |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java` | Modify | Validate + `pattern.addWatchedProperties` |
| `drlx-parser-core/src/test/java/org/drools/drlx/domain/ReactiveEmployee.java` | Create | `@PropertyReactive` fixture |
| `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java` | Modify | Add `DataStore<ReactiveEmployee> reactiveEmployees` |
| `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java` | Modify | Parse-only sanity test for new grammar |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java` | Create | End-to-end happy-path + error tests |

---

## Task 1: Add `ReactiveEmployee` fixture + wire into MyUnit

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/domain/ReactiveEmployee.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`

- [ ] **Step 1: Create `ReactiveEmployee`**

Write `drlx-parser-core/src/test/java/org/drools/drlx/domain/ReactiveEmployee.java`:

```java
package org.drools.drlx.domain;

import org.kie.api.definition.type.PropertyReactive;

@PropertyReactive
public class ReactiveEmployee {

    private int salary;
    private int basePay;
    private int bonusPay;

    public ReactiveEmployee() {
    }

    public ReactiveEmployee(int salary, int basePay, int bonusPay) {
        this.salary = salary;
        this.basePay = basePay;
        this.bonusPay = bonusPay;
    }

    public int getSalary() {
        return salary;
    }

    public void setSalary(int salary) {
        this.salary = salary;
    }

    public int getBasePay() {
        return basePay;
    }

    public void setBasePay(int basePay) {
        this.basePay = basePay;
    }

    public int getBonusPay() {
        return bonusPay;
    }

    public void setBonusPay(int bonusPay) {
        this.bonusPay = bonusPay;
    }
}
```

- [ ] **Step 2: Add `reactiveEmployees` DataStore to MyUnit**

In `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java`, add import and field:

```java
import org.drools.drlx.domain.ReactiveEmployee;
```

Inside the class body, append:

```java
    public DataStore<ReactiveEmployee> reactiveEmployees;
```

- [ ] **Step 3: Build to confirm the fixture compiles**

Run: `mvn -pl drlx-parser-core -am test-compile`
Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/domain/ReactiveEmployee.java drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/MyUnit.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: add ReactiveEmployee fixture for property-reactive tests

Refs #13"
```

---

## Task 2: Grammar — allow empty first `[]`, add watch-list block

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:131-141`

- [ ] **Step 1: Write a failing parse-only test**

Append to `drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java`, inside the class body:

```java
    @Test
    void testParseWatchListOnOopathRoot() {
        String rule = """
                class Foo {
                    rule R1 {
                       var e : /reactiveEmployees[salary > 0][basePay, !bonusPay],
                       do { }
                    }
                }
                """;

        DrlxParser.CompilationUnitContext compilationUnitContext = parseCompilationUnitAsAntlrAST(rule);

        assertThat(compilationUnitContext).isNotNull();
    }

    @Test
    void testParseWatchListWithEmptyConditions() {
        String rule = """
                class Foo {
                    rule R1 {
                       var e : /reactiveEmployees[][basePay],
                       do { }
                    }
                }
                """;

        DrlxParser.CompilationUnitContext compilationUnitContext = parseCompilationUnitAsAntlrAST(rule);

        assertThat(compilationUnitContext).isNotNull();
    }

    @Test
    void testParseWatchListWildcard() {
        String rule = """
                class Foo {
                    rule R1 {
                       var e : /reactiveEmployees[][*],
                       do { }
                    }
                }
                """;

        DrlxParser.CompilationUnitContext compilationUnitContext = parseCompilationUnitAsAntlrAST(rule);

        assertThat(compilationUnitContext).isNotNull();
    }
```

- [ ] **Step 2: Run the tests — confirm they fail**

Run: `mvn -pl drlx-parser-core test -Dtest='DrlxParserTest#testParseWatchList*'`
Expected: FAIL with parser errors like `mismatched input '['` or `no viable alternative`.

- [ ] **Step 3: Modify the grammar**

Edit `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`. Replace the `oopathRoot` rule (around lines 131-136) and append a new `watchItem` rule after `oopathChunk`:

```antlr4
// OOPath root chunk - the only place positional (...) is grammatically valid.
// Now accepts an optional second [...] block for the property-reactive watch
// list. The first (conditions) block may be empty to allow `[][watch]` form.
oopathRoot
    : identifier (HASH identifier)?
      ('(' expression (',' expression)* ')')?
      ('[' (drlxExpression (',' drlxExpression)*)? ']')?
      ('[' watchItem (',' watchItem)* ']')?
    ;

// OOPath chunk - navigation segments after the root; no positional
oopathChunk
    : identifier (HASH identifier)? ('[' drlxExpression (',' drlxExpression)* ']')?
    ;

// DRLX expression used inside oopathChunk conditions
// Allows optional binding (name: expression) or a plain expression
drlxExpression
    : identifier ':' expression
    | expression
    ;

// Watch-list item on property-reactive pattern:
//   '*'         → watch all
//   '!' '*'     → watch none
//   '!'? name   → include / exclude one property
watchItem
    : '!'? ('*' | identifier)
    ;
```

(Three edits: (1) make the first `[...]` inner non-empty `?`; (2) add the second `[...]` block; (3) append `watchItem` rule.)

- [ ] **Step 4: Rebuild (regenerates ANTLR sources)**

Run: `mvn -pl drlx-parser-core -am test-compile`
Expected: BUILD SUCCESS. If compilation fails in the visitor with messages about missing methods, that's fine — next tasks fix the visitor; but the grammar-regenerated classes must compile.

If the visitor doesn't compile, double-check the grammar edit was additive (conditions block structure unchanged except for the inner `?`).

- [ ] **Step 5: Run the parse-only tests — confirm they pass**

Run: `mvn -pl drlx-parser-core test -Dtest='DrlxParserTest#testParseWatchList*'`
Expected: 3 tests PASS.

Note: if `DrlxParserTest#testParseWatchList*` reports all as SKIPPED, that's the known Surefire `-Dtest=<Class>` + `@DisabledIfSystemProperty` interaction (see HANDOFF.md). Run the full suite: `mvn -pl drlx-parser-core test` — the three tests should be visible and passing.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 drlx-parser-core/src/test/java/org/drools/drlx/parser/DrlxParserTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "grammar: add watch-list block on oopath root

Second [...] block declares the property-reactive watch list. First
block now accepts empty so [][watch] parses. New watchItem rule covers
'*', '!*', 'name', '!name'.

Refs #13"
```

---

## Task 3: Extend `PatternIR` with `watchedProperties`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java:35-42`

- [ ] **Step 1: Add the field to the record**

Replace the `PatternIR` record declaration:

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

- [ ] **Step 2: Update construction sites (compile stubs)**

Run: `mvn -pl drlx-parser-core -am test-compile`
Expected: FAIL with "constructor PatternIR cannot be applied" at the existing call sites.

Each failing call site gets `, List.of()` appended as the last argument — minimal change to keep things compiling. Sites are:

- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` (two `new PatternIR(...)` calls, around lines 263 and 272): append `, List.of()` before `)`.
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` (one `new PatternIR(...)` call, around line 103): append `, List.copyOf(pattern.getWatchedPropertiesList())`. Since the proto field doesn't exist yet (Task 4), temporarily use `, List.of()` to make it compile. Task 4's commit will replace this placeholder.

Example for `DrlxToRuleAstVisitor.java:263`:

```java
        return new PatternIR(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs, passive, List.of());
```

- [ ] **Step 3: Verify compilation succeeds**

Run: `mvn -pl drlx-parser-core -am test-compile`
Expected: BUILD SUCCESS.

- [ ] **Step 4: Verify all existing tests still pass**

Run: `mvn -pl drlx-parser-core test`
Expected: 96 tests passing (same baseline; a behavior-neutral field addition).

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "ir: add watchedProperties to PatternIR

Placeholder List.of() at all construction sites; visitor extraction
and proto wiring follow.

Refs #13"
```

---

## Task 4: Extend proto schema + serialization

**Files:**
- Modify: `drlx-parser-core/src/main/proto/drlx_rule_ast.proto:36-44`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java` (two sites: `fromProtoLhs` and `toProtoLhs`)

- [ ] **Step 1: Add the proto field**

Edit `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`. In the `PatternParseResult` message, add field 8:

```proto
message PatternParseResult {
  string type_name = 1;
  string bind_name = 2;
  string entry_point = 3;
  repeated string conditions = 4;
  string cast_type_name = 5;
  repeated string positional_args = 6;
  bool passive = 7;
  repeated string watched_properties = 8;
}
```

- [ ] **Step 2: Regenerate proto classes**

Run: `mvn -pl drlx-parser-core -am test-compile`
Expected: BUILD SUCCESS — protoc regenerates `DrlxRuleAstProto`, making `getWatchedPropertiesList()` / `addWatchedProperties(...)` available.

- [ ] **Step 3: Wire serialization (`toProtoLhs`)**

In `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`, inside `toProtoLhs` (around line 143), after the existing `p.positionalArgs().forEach(pb::addPositionalArgs);` line, add:

```java
            p.watchedProperties().forEach(pb::addWatchedProperties);
```

- [ ] **Step 4: Wire deserialization (`fromProtoLhs`)**

In the same file, in `fromProtoLhs` (around line 103), replace the `List.of()` placeholder introduced in Task 3 with the real call:

```java
                yield new PatternIR(
                        pattern.getTypeName(),
                        pattern.getBindName(),
                        pattern.getEntryPoint(),
                        List.copyOf(pattern.getConditionsList()),
                        castTypeName,
                        List.copyOf(pattern.getPositionalArgsList()),
                        pattern.getPassive(),
                        List.copyOf(pattern.getWatchedPropertiesList()));
```

- [ ] **Step 5: Run all existing tests**

Run: `mvn -pl drlx-parser-core test`
Expected: 96 tests passing. Proto changes are backwards-compatible; empty watch list round-trips cleanly as empty list.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/proto/drlx_rule_ast.proto drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "proto: serialize PatternIR.watchedProperties

Field 8 on PatternParseResult. Empty list round-trips cleanly.

Refs #13"
```

---

## Task 5: Visitor extraction

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

- [ ] **Step 1: Add `extractWatchedProperties` helper**

In `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`, after the existing `extractPositionalArgs` method (around line 315), add:

```java
    private static List<String> extractWatchedProperties(DrlxParser.OopathExpressionContext ctx) {
        DrlxParser.OopathRootContext root = ctx.oopathRoot();
        if (root == null || root.watchItem() == null || root.watchItem().isEmpty()) {
            return List.of();
        }
        return root.watchItem().stream()
                .map(org.antlr.v4.runtime.ParserRuleContext::getText)
                .toList();
    }
```

(Use `ParserRuleContext.getText()`, not the visitor's `getText(ctx)` helper — we want whitespace-stripped token text, matching how `extractPositionalArgs` handles its own extraction.)

- [ ] **Step 2: Route watch list into both `PatternIR` factories**

In the same file, update `buildPatternFromBoundOopath` (around line 254). Replace the line `return new PatternIR(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs, passive, List.of());` with:

```java
        List<String> watchedProperties = extractWatchedProperties(ctx.oopathExpression());
        return new PatternIR(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs, passive, watchedProperties);
```

Update `buildPatternFromOopath` (around line 266). Replace the `return new PatternIR("", "", entryPoint, conditions, castTypeName, positionalArgs, passive, List.of());` line with:

```java
        List<String> watchedProperties = extractWatchedProperties(oopathCtx);
        return new PatternIR("", "", entryPoint, conditions, castTypeName, positionalArgs, passive, watchedProperties);
```

- [ ] **Step 3: Run existing tests — no regressions**

Run: `mvn -pl drlx-parser-core test`
Expected: 96 tests passing. Watch list is now extracted into `PatternIR` but not yet applied at runtime — behavior unchanged.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "visitor: extract watch-list items into PatternIR

extractWatchedProperties reads root.watchItem() and emits raw strings
('*', '!*', 'name', '!name'). Runtime builder wiring follows.

Refs #13"
```

---

## Task 6: Runtime builder — validate + apply watch list

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

- [ ] **Step 1: Add imports**

At the top of `DrlxRuleAstRuntimeBuilder.java`, among existing imports, add:

```java
import org.drools.base.util.PropertyReactivityUtil;
```

- [ ] **Step 2: Add `validateWatchedProperties` helper**

In the same file, add a private static method at the end of the class (just before the closing `}`):

```java
    private static List<String> validateWatchedProperties(List<String> raw,
                                                          Class<?> patternClass,
                                                          String typeLabel) {
        List<String> accessible = PropertyReactivityUtil.getAccessibleProperties(patternClass);
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

Ensure `java.util.ArrayList` is imported (it very likely already is; check the existing imports).

- [ ] **Step 3: Call the validator + apply to the pattern**

In `buildPattern` (around line 297, after the constraints-for loop, before `return pattern;`), insert:

```java
        if (!parseResult.watchedProperties().isEmpty()) {
            List<String> validated = validateWatchedProperties(
                    parseResult.watchedProperties(), patternClass, parseResult.typeName());
            pattern.addWatchedProperties(validated);
        }
```

- [ ] **Step 4: Run existing tests**

Run: `mvn -pl drlx-parser-core test`
Expected: 96 tests passing. No existing test uses a watch list, so the new branch is dormant.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "runtime: validate watch list + call Pattern.addWatchedProperties

Mirrors PatternBuilder.processListenedPropertiesAnnotation (DRL7) minus
the @PropertyReactive class check. Throws RuntimeException on unknown
property, duplicate property, or duplicate wildcard.

Refs #13"
```

---

## Task 7: End-to-end tests — happy path (plain watch list + watch-only)

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`

- [ ] **Step 1: Write the failing test file with the first two scenarios**

Create `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`:

```java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.domain.ReactiveEmployee;
import org.junit.jupiter.api.Test;
import org.kie.api.runtime.rule.EntryPoint;
import org.kie.api.runtime.rule.FactHandle;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PropertyReactiveWatchListTest extends DrlxBuilderTestSupport {

    @Test
    void plainWatchList_firesOnWatchedProperty_notOnUnwatched() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[salary > 0][basePay, bonusPay],
                    do { System.out.println("fired"); }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(6000, 4000, 1000);
            FactHandle fh = ep.insert(emp);

            // Initial match fires once.
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
            listener.getAfterMatchFired().clear();

            // Modify the SALARY (not in watch list) — must NOT re-fire.
            emp.setSalary(7000);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(0);
            assertThat(listener.getAfterMatchFired()).isEmpty();

            // Modify basePay (watched) — must re-fire.
            emp.setBasePay(5000);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }

    @Test
    void watchOnly_emptyConditions_firesOnWatchedProperty() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[][basePay],
                    do { }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(1000, 2000, 3000);
            FactHandle fh = ep.insert(emp);

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
            listener.getAfterMatchFired().clear();

            // Unwatched property change → no re-fire.
            emp.setBonusPay(4000);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(0);

            // Watched property change → re-fire.
            emp.setBasePay(5000);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }
}
```

Note: the FactHandle import (`org.kie.api.runtime.rule.FactHandle`) is how Drools exposes handles from `EntryPoint.insert`. Verify it imports cleanly — if IDE flags it, check the `kie-api` dependency in `drlx-parser-core/pom.xml`.

- [ ] **Step 2: Run the new tests**

Run: `mvn -pl drlx-parser-core test -Dtest=PropertyReactiveWatchListTest`
Expected: Tests either PASS (if everything is wired) or give a clear failure pointing to remaining work. If SKIPPED per the Surefire quirk, run `mvn -pl drlx-parser-core test` and grep the output for the test class.

- [ ] **Step 3: Commit once green**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: property-reactive watch list happy-path (plain + watch-only)

Refs #13"
```

---

## Task 8: Wildcard + exclusion tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`

- [ ] **Step 1: Append three scenarios**

Add these test methods inside the `PropertyReactiveWatchListTest` class:

```java
    @Test
    void wildcardAll_firesOnAnyPropertyChange() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[][*],
                    do { }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(1000, 2000, 3000);
            FactHandle fh = ep.insert(emp);

            kieSession.fireAllRules();
            listener.getAfterMatchFired().clear();

            // Any modification re-fires.
            emp.setBonusPay(4000);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
        });
    }

    @Test
    void wildcardNone_suppressesAllReEntries() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[][!*],
                    do { }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(1000, 2000, 3000);
            FactHandle fh = ep.insert(emp);

            kieSession.fireAllRules();
            listener.getAfterMatchFired().clear();

            // No modification re-fires.
            emp.setBasePay(9999);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(0);
        });
    }

    @Test
    void negativeExclusion_firesOnOthersButNotExcluded() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[][*, !bonusPay],
                    do { }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(1000, 2000, 3000);
            FactHandle fh = ep.insert(emp);

            kieSession.fireAllRules();
            listener.getAfterMatchFired().clear();

            // bonusPay excluded: no re-fire.
            emp.setBonusPay(4000);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(0);

            // basePay not excluded: re-fire.
            emp.setBasePay(5000);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
        });
    }
```

- [ ] **Step 2: Run them**

Run: `mvn -pl drlx-parser-core test -Dtest=PropertyReactiveWatchListTest`
Expected: All 5 tests pass. If SKIPPED, run full suite.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: watch-list wildcards (*, !*) and negative exclusion

Refs #13"
```

---

## Task 9: Watch list inside NOT

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`

- [ ] **Step 1: Add a test for a nested pattern**

Inside `PropertyReactiveWatchListTest`, add:

```java
    @Test
    void watchListAppliesInsideNot() {
        // NOT over /reactiveEmployees where basePay > 5000. With watch list
        // `[basePay]`, modifications to other properties must not affect
        // the NOT's evaluation.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    not /reactiveEmployees[basePay > 5000][basePay],
                    do { }
                }
                """;

        withSession(rule, (kieSession, listener) -> {
            EntryPoint ep = kieSession.getEntryPoint("reactiveEmployees");
            ReactiveEmployee emp = new ReactiveEmployee(1000, 2000, 3000);
            FactHandle fh = ep.insert(emp);

            // basePay <= 5000 → NOT matches → fires once.
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
            listener.getAfterMatchFired().clear();

            // Bumping bonusPay should NOT re-evaluate the pattern
            // (bonusPay not in watch list).
            emp.setBonusPay(9999);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(0);

            // Bumping basePay > 5000: pattern matches, NOT fails → no fire.
            emp.setBasePay(6000);
            ep.update(fh, emp);
            assertThat(kieSession.fireAllRules()).isEqualTo(0);
        });
    }
```

- [ ] **Step 2: Run it**

Run: `mvn -pl drlx-parser-core test -Dtest=PropertyReactiveWatchListTest`
Expected: All 6 tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: watch list applies inside NOT group element

Refs #13"
```

---

## Task 10: Error-path tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`

- [ ] **Step 1: Add three error tests**

Inside `PropertyReactiveWatchListTest`, add:

```java
    @Test
    void unknownProperty_throwsAtBuild() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[][basePya],
                    do { }
                }
                """;

        assertThatThrownBy(() -> newBuilder().build(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("Unknown property 'basePya'");
    }

    @Test
    void duplicateProperty_throwsAtBuild() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[][basePay, basePay],
                    do { }
                }
                """;

        assertThatThrownBy(() -> newBuilder().build(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("Duplicate property 'basePay'");
    }

    @Test
    void duplicateWildcard_throwsAtBuild() {
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.ReactiveEmployee;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule R1 {
                    var e : /reactiveEmployees[][*, *],
                    do { }
                }
                """;

        assertThatThrownBy(() -> newBuilder().build(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("Duplicate usage of wildcard *");
    }
```

- [ ] **Step 2: Run them**

Run: `mvn -pl drlx-parser-core test -Dtest=PropertyReactiveWatchListTest`
Expected: All 9 tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: watch list error paths (unknown/duplicate/dup-wildcard)

Refs #13"
```

---

## Task 11: Serialization round-trip test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java`

Build an `IR` by hand, save it via `DrlxRuleAstParseResult.save`, load it back, and assert `watchedProperties` survives. Hand-built IR avoids any dependency on `DrlxRuleBuilder` internals.

- [ ] **Step 1: Add imports**

At the top of `PropertyReactiveWatchListTest.java`, add:

```java
import java.nio.file.Path;
import java.util.List;

import org.drools.drlx.builder.DrlxRuleAstModel.CompilationUnitIR;
import org.drools.drlx.builder.DrlxRuleAstModel.ConsequenceIR;
import org.drools.drlx.builder.DrlxRuleAstModel.LhsItemIR;
import org.drools.drlx.builder.DrlxRuleAstModel.PatternIR;
import org.drools.drlx.builder.DrlxRuleAstModel.RuleIR;
import org.drools.drlx.builder.DrlxRuleAstParseResult;
import org.junit.jupiter.api.io.TempDir;
```

- [ ] **Step 2: Add the round-trip test**

Inside `PropertyReactiveWatchListTest`:

```java
    @Test
    void watchListRoundTripsThroughProto(@TempDir Path tempDir) throws Exception {
        PatternIR pattern = new PatternIR(
                "ReactiveEmployee",
                "e",
                "reactiveEmployees",
                List.of("salary > 0"),
                null,
                List.of(),
                false,
                List.of("basePay", "!bonusPay", "*"));

        RuleIR rule = new RuleIR(
                "R1",
                List.of(),
                List.of(pattern),
                new ConsequenceIR(""));

        CompilationUnitIR ir = new CompilationUnitIR(
                "org.drools.drlx.parser",
                "MyUnit",
                List.of("org.drools.drlx.domain.ReactiveEmployee",
                        "org.drools.drlx.ruleunit.MyUnit"),
                List.of(rule));

        String fakeSource = "placeholder source";
        DrlxRuleAstParseResult.save(fakeSource, ir, tempDir);
        CompilationUnitIR loaded = DrlxRuleAstParseResult.load(
                fakeSource, DrlxRuleAstParseResult.parseResultFilePath(tempDir));

        assertThat(loaded).isNotNull();
        LhsItemIR first = loaded.rules().get(0).lhs().get(0);
        assertThat(first).isInstanceOf(PatternIR.class);
        assertThat(((PatternIR) first).watchedProperties())
                .containsExactly("basePay", "!bonusPay", "*");
    }
```

**Note on record constructor arg order:** match the `PatternIR` declaration in `DrlxRuleAstModel.java` exactly — `(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs, passive, watchedProperties)`. If any site in the codebase has reordered, this test will fail at compile — confirm against the record definition before running.

- [ ] **Step 3: Run it**

Run: `mvn -pl drlx-parser-core test -Dtest=PropertyReactiveWatchListTest`
Expected: 10 tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/PropertyReactiveWatchListTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: watch list proto round-trip via hand-built IR

Refs #13"
```

---

## Task 12: Full-suite regression check

- [ ] **Step 1: Run the complete test suite**

Run: `mvn -pl drlx-parser-core test`
Expected: 96 baseline tests + 9-10 new tests = **105-106 tests passing, 0 failures**.

- [ ] **Step 2: Run the no-persist variant**

Run: `mvn -pl drlx-parser-core test -Dmvel3.compiler.lambda.persistence=false`
Expected: same set minus the tests guarded by `@DisabledIfSystemProperty`. Should match or exceed prior baseline (5 no-persist + 66 persistence-agnostic tests passing, 30+ skipped).

- [ ] **Step 3: No commit for this task — verification only.**

If any regression appears, stop and diagnose. Don't commit a fix until the root cause is understood (don't mask with `-DskipTests` or similar).

---

## Task 13: Push branch + update issue

- [ ] **Step 1: Push**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push origin main
```

(Target branch convention is `main` per the workspace; change if the user prefers a feature branch.)

- [ ] **Step 2: Close #13 with a summary**

```bash
gh issue close 13 --repo tkobayas/drlx-parser --comment "Implemented via commits on main. Second [] block on oopathRoot declares the Drools property-reactive watch list. Wildcards (*, !*) and exclusions (!name) supported. Validation mirrors DRL7 PatternBuilder minus the @PropertyReactive class check. New test file PropertyReactiveWatchListTest covers happy path, all wildcard variants, nested-NOT, and the three error conditions (unknown/duplicate/dup-wildcard)."
```

- [ ] **Step 3: Update HANDOFF (next session)**

Either during this task or at session end, update `/home/tkobayas/claude/public/drlx-parser/HANDOFF.md` to reflect #13 closed and next candidates.

---

## Self-Review Notes

- All `List.of()` placeholder calls in Task 3 are intentional — they exist to keep compilation green across the IR-schema change; Task 4 and Task 5 replace them in order.
- The Surefire `-Dtest=<Class>` / `@DisabledIfSystemProperty` quirk is called out in every test-running step; the fallback is always the full suite.
- Task 11 builds the IR by hand to avoid any dependency on `DrlxRuleBuilder` internals — the proto round-trip is exercised through `DrlxRuleAstParseResult.save` / `.load` alone.
