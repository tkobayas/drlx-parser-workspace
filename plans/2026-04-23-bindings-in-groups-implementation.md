# Bound Patterns as Group Children — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let bound patterns (`var l : /foo` and `Type l = /foo`) appear as children of any CE paren form (`and`, `or`, `not`, `exists`), so DRLX can express joins inside explicit groups. Example: `and(var l : /locations, /persons[name == l.city])`.

**Architecture:** Factor `boundOopath` out of `rulePattern` as a reusable non-terminal. Widen `groupChild` to include it. The visitor's existing `PatternIR(typeName, bindName, ...)` record already models the result; `buildPatternFromBoundOopath` becomes the single source of truth for bound-pattern extraction. IR, runtime, and proto are unchanged. The LSP tolerant visitor gets one `visitBoundOopath` override.

**Tech Stack:** ANTLR4 grammar, Java 21, Drools 10.1, JUnit 5, AssertJ, Maven.

**Spec:** `specs/2026-04-23-bindings-in-groups-design.md` (issue #17).

**Working directory:** The project code is at `/home/tkobayas/usr/work/mvel3-development/drlx-parser` (the workspace at `/home/tkobayas/claude/public/drlx-parser` holds this plan and the spec). All `mvn` and source-tree paths below are relative to the project directory. Plan/spec commits happen in the workspace.

**Maven rule (from CLAUDE.md):** After modifying any Maven module, run `mvn -pl drlx-parser-core -am install` before running tests. Every task that modifies code includes this step explicitly.

**Test baseline:** 84 passing before this plan starts. Final: **94 passing** (4 + 3 + 2 + 1 new tests across four files).

---

## File Structure

### Modified (4 files)
| File | Responsibility |
|------|----------------|
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` | Grammar — factor `boundOopath`, widen `groupChild` |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Visitor — new helper, collapse `buildPattern`, extend `buildGroupChild` |
| `drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java` | LSP tolerant visitor — new `visitBoundOopath` override |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NestedGroupTest.java` | Extend with one nested-binding test |
| `docs/DRLXXXX.md` | Promote bindings-in-groups examples from "not yet" to documented; add scope subsection |

### Created (3 files)
| File | Contents |
|------|----------|
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java` | 4 tests — AC driver for bindings inside `and(...)` |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrBindingScopeTest.java` | 3 tests — branch-local OR semantics |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotExistsBindingTest.java` | 2 tests — within-group binding under NOT/EXISTS |

### Not touched
IR (`DrlxRuleAstModel.java`), runtime (`DrlxRuleAstRuntimeBuilder.java`), proto (`drlx_rule_ast.proto`), `DrlxToJavaParserVisitor` (frozen core LSP parent).

### Domain model used by tests (already exists — no new fixtures)
- `Person(name, age)` — `org.drools.drlx.domain.Person`
- `Location(city, district)` — `org.drools.drlx.domain.Location`
- DataStores: `persons`, `persons1`, `persons2`, `locations` — declared in `org.drools.drlx.ruleunit.MyUnit`

Tests use `Person`-to-`Person` joins (via `persons1`/`persons2` and `name`) for clear semantics, and `Location`-to-`Person` with `name == l.city` for the explicit-type test — this keeps the domain unchanged.

---

## Task 1: Baseline sanity — confirm 84 green

- [ ] **Step 1: Confirm baseline**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core -am install -DskipTests -q
mvn -pl drlx-parser-core test
```

Expected: `Tests run: 84, Failures: 0, Errors: 0, Skipped: 0` (aggregate across default profile — there are 79 default + 5 no-persist; the default profile shows 79 here). Either number is acceptable so long as **0 failures**.

If failures appear, stop and investigate before touching anything. The plan assumes a clean starting state.

---

## Task 2: Write the first failing test (TDD lead)

The single most expressive test — a binding visible to a right sibling via join — goes in first. It will fail at parse time until Task 3 lands. This anchors the plan in observable behaviour.

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java`

- [ ] **Step 1: Create the test file with one failing test**

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

class BindingInGroupTest extends DrlxBuilderTestSupport {

    @Test
    void andBindingVisibleToRightSibling() {
        // `and(var p : /persons1, /persons2[name == p.name])` — joins two
        // person streams by name. Binding `p` from the first child must be
        // visible to the second child's condition. Primary AC for #17.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule JoinByName {
                    and(var p : /persons1, /persons2[ name == p.name ]),
                    do { System.out.println("joined"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            persons1.insert(new Person("Alice", 30));
            persons1.insert(new Person("Bob", 40));
            persons2.insert(new Person("Alice", 25));
            persons2.insert(new Person("Carol", 50));

            // Exactly one pair joins: (Alice/persons1, Alice/persons2).
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).hasSize(1);
        });
    }
}
```

- [ ] **Step 2: Run it — confirm parse failure**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core test -Dtest=BindingInGroupTest
```

Expected: **FAIL** — parser rejects `var p : /persons1` as a groupChild (no `boundOopath` arm yet). Error text will mention an ANTLR parse error at the `var` or `:` token.

- [ ] **Step 3: Do not commit yet.** Move to Task 3 — the test exists to drive the grammar change; it will stay failing until Task 3 & 4 are done.

---

## Task 3: Grammar — factor `boundOopath`, widen `groupChild`

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4:61-102`

- [ ] **Step 1: Edit the grammar**

Replace the existing block (current lines 61-102):

```antlr
// Unified child for any group element. No trailing `,` — the parent
// group (paren form) uses `,` as a list separator, and the top-level
// `ruleItem` owns the CE terminator `,`.
groupChild
    : oopathExpression
    | notElement
    | existsElement
    | andElement
    | orElement
    ;
```

with the widened form (bound arm listed first — ANTLR first-match wins, and `boundOopath` starts with two identifiers + `:`/`=`, never with `/`, so there's no ambiguity):

```antlr
// Unified child for any group element. No trailing `,` — the parent
// group (paren form) uses `,` as a list separator, and the top-level
// `ruleItem` owns the CE terminator `,`.
// `boundOopath` MUST come before `oopathExpression`: ANTLR picks first
// match, and a bound form (identifier identifier ':' ...) has a strictly
// more constrained prefix than a bare oopath (which starts with '/').
groupChild
    : boundOopath
    | oopathExpression
    | notElement
    | existsElement
    | andElement
    | orElement
    ;
```

Then replace the existing `rulePattern` rule (current lines 98-102):

```antlr
// Pattern: type bind : oopathExpression ,
// Aligns with: RulePattern(SimpleName type, SimpleName bind, OOPathExpr expr)
rulePattern
    : identifier identifier (':' | '=') oopathExpression ','
    ;
```

with the factored form, and insert the new `boundOopath` non-terminal immediately above it:

```antlr
// Bound pattern body without trailing `,`. Reused by `rulePattern`
// (top-level, with terminator `,`) and by `groupChild` (inside a CE
// paren form, where `,` is the sibling separator).
boundOopath
    : identifier identifier (':' | '=') oopathExpression
    ;

// Top-level pattern: `boundOopath ,`. CE terminator owned by ruleItem.
rulePattern
    : boundOopath ','
    ;
```

- [ ] **Step 2: Rebuild + run existing tests — expect no regression**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core -am install -DskipTests -q
mvn -pl drlx-parser-core test -Dtest='!BindingInGroupTest'
```

Expected: **all existing tests pass** (the grammar refactor is behaviour-preserving; `rulePattern` still matches the same inputs).

If tests fail, the cause is almost always visitor code that reads `ctx.identifier(0)` / `ctx.identifier(1)` directly off `RulePatternContext`. Task 4 fixes that — but this intermediate state should still produce valid tree walks because the generated `RulePatternContext` now exposes `boundOopath()` instead of `identifier()` / `oopathExpression()` on itself. If there are compile errors in `DrlxToRuleAstVisitor.java` referencing the old accessors, the whole suite will fail to build — proceed to Task 4 immediately rather than trying to patch mid-flight.

- [ ] **Step 3: Run the failing test again — confirm it still fails**

```bash
mvn -pl drlx-parser-core test -Dtest=BindingInGroupTest
```

Expected: **FAIL** — test still fails. It now reaches the visitor rather than the parser, but the visitor's `buildGroupChild` does not yet handle `boundOopath`. This is the correct mid-point state.

- [ ] **Step 4: Do not commit yet** — grammar + visitor are one coherent change; commit once they land together in Task 4.

---

## Task 4: Visitor — factor out `buildPatternFromBoundOopath`, extend `buildGroupChild`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java:244-270`

- [ ] **Step 1: Add new helper `buildPatternFromBoundOopath` just before the existing `buildPatternFromOopath`**

Insert above line 253 (`private PatternIR buildPatternFromOopath(...)`):

```java
    private PatternIR buildPatternFromBoundOopath(DrlxParser.BoundOopathContext ctx) {
        String typeName = ctx.identifier(0).getText();
        String bindName = ctx.identifier(1).getText();
        DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
        String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
        String castTypeName = extractCastType(oopathCtx);
        List<String> conditions = extractConditions(oopathCtx);
        List<String> positionalArgs = extractPositionalArgs(oopathCtx);
        return new PatternIR(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs);
    }
```

- [ ] **Step 2: Collapse `buildPattern` to delegate** — replace the body of the current `buildPattern(RulePatternContext)` (lines 261-270) with a one-line delegator:

```java
    private PatternIR buildPattern(DrlxParser.RulePatternContext ctx) {
        return buildPatternFromBoundOopath(ctx.boundOopath());
    }
```

- [ ] **Step 3: Extend `buildGroupChild`** — replace the existing method (lines 244-251) with:

```java
    private LhsItemIR buildGroupChild(DrlxParser.GroupChildContext c) {
        if (c.boundOopath() != null)      return buildPatternFromBoundOopath(c.boundOopath());
        if (c.oopathExpression() != null) return buildPatternFromOopath(c.oopathExpression());
        if (c.notElement() != null)       return buildNotElement(c.notElement());
        if (c.existsElement() != null)    return buildExistsElement(c.existsElement());
        if (c.andElement() != null)       return buildAndElement(c.andElement());
        if (c.orElement() != null)        return buildOrElement(c.orElement());
        throw new IllegalArgumentException("Unsupported group child: " + c.getText());
    }
```

Note: `boundOopath()` appears as the first check, matching grammar alternative order.

- [ ] **Step 4: Rebuild + run all tests**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core -am install -DskipTests -q
mvn -pl drlx-parser-core test
```

Expected: **all tests pass, including `BindingInGroupTest#andBindingVisibleToRightSibling`**. Count should be 85 (was 84; +1 from the new test).

- [ ] **Step 5: Commit — grammar + visitor as one atomic change**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
git add drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 \
        drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
        drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java
git commit -m "$(cat <<'EOF'
feat: bindings as group CE children — grammar + visitor + first test

Factors `boundOopath` out of rulePattern as a reusable non-terminal.
Widens `groupChild` to accept it as the first alternative. Visitor
helper `buildPatternFromBoundOopath` becomes the single source of
truth for bound-pattern extraction; `buildPattern(RulePatternContext)`
collapses to a delegator.

Refs #17
EOF
)"
```

---

## Task 5: LSP tolerant visitor — `visitBoundOopath` override

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java`

- [ ] **Step 1: Add `visitBoundOopath` override**

Insert after the existing `visitOrElement` method (currently around line 83) and before the existing `buildRulePatternStandIn` helper:

```java
    @Override
    public Node visitBoundOopath(DrlxParser.BoundOopathContext ctx) {
        // Tolerant stand-in: build a RulePattern with the actual type + bind names
        // from the bound form. Preserves completion token population for the
        // inner oopath while the user is typing inside a `var l : /foo` inside a group.
        SimpleName type = new SimpleName(ctx.identifier(0).getText());
        SimpleName bind = new SimpleName(ctx.identifier(1).getText());
        OOPathExpr expr = (OOPathExpr) visit(ctx.oopathExpression());
        RulePattern pattern = new RulePattern(null, type, bind, expr);
        type.setParentNode(pattern);
        bind.setParentNode(pattern);
        expr.setParentNode(pattern);
        return pattern;
    }
```

- [ ] **Step 2: Update `visitFirstGroupChild` to route bound children**

Replace the existing method (around lines 96-107):

```java
    private Node visitFirstGroupChild(List<DrlxParser.GroupChildContext> children) {
        if (children == null || children.isEmpty()) {
            return buildRulePatternStandIn(null);
        }
        DrlxParser.GroupChildContext first = children.get(0);
        if (first.oopathExpression() != null) return buildRulePatternStandIn(first.oopathExpression());
        if (first.notElement() != null)       return visit(first.notElement());
        if (first.existsElement() != null)    return visit(first.existsElement());
        if (first.andElement() != null)       return visit(first.andElement());
        if (first.orElement() != null)        return visit(first.orElement());
        return buildRulePatternStandIn(null);
    }
```

with (the `boundOopath` arm added first, matching grammar order):

```java
    private Node visitFirstGroupChild(List<DrlxParser.GroupChildContext> children) {
        if (children == null || children.isEmpty()) {
            return buildRulePatternStandIn(null);
        }
        DrlxParser.GroupChildContext first = children.get(0);
        if (first.boundOopath() != null)      return visit(first.boundOopath());
        if (first.oopathExpression() != null) return buildRulePatternStandIn(first.oopathExpression());
        if (first.notElement() != null)       return visit(first.notElement());
        if (first.existsElement() != null)    return visit(first.existsElement());
        if (first.andElement() != null)       return visit(first.andElement());
        if (first.orElement() != null)        return visit(first.orElement());
        return buildRulePatternStandIn(null);
    }
```

- [ ] **Step 3: Rebuild + run all tests**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core -am install -DskipTests -q
mvn -pl drlx-parser-core test
```

Expected: **all 85 tests pass**. The LSP test suite (`TolerantDrlxToJavaParserVisitorTest`) should remain green — the new override is only reachable from a new grammar path.

- [ ] **Step 4: Commit**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
git add drlx-parser-core/src/main/java/org/drools/drlx/parser/TolerantDrlxToJavaParserVisitor.java
git commit -m "$(cat <<'EOF'
feat: tolerant LSP visitor handles boundOopath in groups

Adds visitBoundOopath override that produces a RulePattern stand-in
with real type/bind names, preserving the completion-token invariant
for in-progress typing inside `and(var l : /foo, ...)`. Updates
visitFirstGroupChild to route bound children through visit().

Refs #17
EOF
)"
```

---

## Task 6: `BindingInGroupTest` — add 3 more tests

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java`

- [ ] **Step 1: Append three tests**

After the existing `andBindingVisibleToRightSibling` test, add:

```java
    @Test
    void andBindingExplicitType() {
        // Explicit-type form: `Person p = /persons1`. Same join semantics,
        // different surface syntax. Confirms `boundOopath` handles both
        // `:` and `=` operators.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule JoinByNameExplicitType {
                    and(Person p = /persons1, /persons2[ name == p.name ]),
                    do { System.out.println("joined"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Alice", 25));

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).hasSize(1);
        });
    }

    @Test
    void andBothChildrenBound() {
        // Both children carry bindings. Binding `p1` visible to second
        // child's condition; second child also binds `p2`. Confirms no
        // cross-binding leakage or ordering bugs.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule BothBound {
                    and(var p1 : /persons1, var p2 : /persons2[ name == p1.name ]),
                    do { System.out.println("bound pair"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Alice", 25));
            persons2.insert(new Person("Bob", 40));

            // One matching pair: (Alice, Alice).
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).hasSize(1);
        });
    }

    @Test
    void bareOopathStillWorks() {
        // Regression: the grammar refactor and the visitor bound-arm must
        // not break existing bare-oopath group children.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule BareOopathsInAnd {
                    and(/persons1[ age > 18 ], /persons2[ age > 18 ]),
                    do { System.out.println("both adult"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Bob", 40));

            // No join constraint — fires once per Cartesian match (1×1 = 1).
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).hasSize(1);
        });
    }
```

- [ ] **Step 2: Rebuild + run tests**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core -am install -DskipTests -q
mvn -pl drlx-parser-core test -Dtest=BindingInGroupTest
```

Expected: **4 tests, 0 failures**.

- [ ] **Step 3: Full-suite run**

```bash
mvn -pl drlx-parser-core test
```

Expected: **88 total, 0 failures** (84 baseline + 4 new).

- [ ] **Step 4: Commit**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/BindingInGroupTest.java
git commit -m "$(cat <<'EOF'
test: add BindingInGroupTest explicit-type, both-bound, bare regression

Completes the BindingInGroupTest file with coverage for the `Type p =`
explicit-type form, both-children-bound, and bare-oopath regression.

Refs #17
EOF
)"
```

---

## Task 7: `OrBindingScopeTest` — branch-local semantics (Decision #2)

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrBindingScopeTest.java`

- [ ] **Step 1: Create the test file**

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
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class OrBindingScopeTest extends DrlxBuilderTestSupport {

    @Test
    void orBranchBindingLocal() {
        // `or(and(var p : /persons1, /persons2[name == p.name]),
        //    and(var p : /persons2, /persons3[name == p.name]))`
        // Each branch has its own `p`; each branch is an independent join.
        // Top-level fires once per satisfied branch (Drools LogicTransformer
        // expands top-level OR into separate rule instances).
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule BranchLocalJoin {
                    or(and(var p : /persons1, /persons2[ name == p.name ]),
                       and(var p : /persons2, /persons3[ name == p.name ])),
                    do { System.out.println("branch fired"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");
            final EntryPoint persons3 = kieSession.getEntryPoint("persons3");

            // First branch satisfied: Alice in persons1 and persons2.
            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Alice", 25));
            // Second branch satisfied: Bob in persons2 and persons3.
            persons2.insert(new Person("Bob", 40));
            persons3.insert(new Person("Bob", 50));

            // Each branch fires once (Drools OR-expansion).
            assertThat(kieSession.fireAllRules()).isEqualTo(2);
            assertThat(fired).hasSize(2);
        });
    }

    @Test
    void orDirectBindingBranchLocal() {
        // `or(var p : /persons1, var p : /persons2)` — same bind name on
        // both branches, different streams. Each `p` is distinct; no
        // unification across branches. Fires once per matching branch.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule DirectBranchLocal {
                    or(var p : /persons1, var p : /persons2),
                    do { System.out.println("branch fired"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Bob", 40));

            // Each branch fires once — both matched.
            assertThat(kieSession.fireAllRules()).isEqualTo(2);
            assertThat(fired).hasSize(2);
        });
    }

    @Test
    void orBindingNotVisibleAfterGroup() {
        // `or(var p : /persons1), /persons2[ name == p.name ]` — `p`
        // is branch-local, NOT visible to the sibling after `or(...)`.
        // Drools surfaces this at KieBase build time as an "unresolved"
        // error on `p`. Assert the build throws. This documents the
        // branch-local stance at the system boundary (Decision #2).
        //
        // If Drools ever changes to produce a silent zero-fire instead,
        // update the assertion here and document the observed behaviour.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule UnresolvedAfterOr {
                    or(var p : /persons1),
                    /persons2[ name == p.name ],
                    do { System.out.println("should not build"); }
                }
                """;

        assertThatThrownBy(() -> withSession(rule, ks -> { /* never runs */ }))
                .isInstanceOf(RuntimeException.class);
        // Minimal assertion: the build or KieSession-creation fails. We do
        // not pin the exact error text — Drools may reword it across
        // versions. Any RuntimeException is sufficient evidence that `p`
        // does not escape the OR group (branch-local Decision #2).
    }
}
```

- [ ] **Step 2: Rebuild + run tests**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core -am install -DskipTests -q
mvn -pl drlx-parser-core test -Dtest=OrBindingScopeTest
```

Expected: **3 tests, 0 failures**.

**Possible mid-plan surprise:** the `orBindingNotVisibleAfterGroup` test asserts that the build throws. If Drools instead produces a zero-fire session (no throw), the test will fail with "no exception thrown." In that case:
1. Delete the `assertThatThrownBy(...)` block.
2. Replace with a `withSession(rule, kieSession -> { ... assertThat(kieSession.fireAllRules()).isZero(); })`.
3. Update the test comment to record the observed-to-be-silent behaviour.
4. The test still validates the branch-local stance, just via zero-fire instead of build-error.

Do this fix-forward; don't let an unexpected build behaviour block the plan. Capture the observed behaviour in a comment for the next maintainer.

- [ ] **Step 3: Full-suite run**

```bash
mvn -pl drlx-parser-core test
```

Expected: **91 total, 0 failures** (88 + 3 new).

- [ ] **Step 4: Commit**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/OrBindingScopeTest.java
git commit -m "$(cat <<'EOF'
test: OrBindingScopeTest — branch-local binding semantics

Covers Decision #2: bindings inside or(...) are visible only within
their own branch and never escape the group. Three tests exercise the
branch-local join case, same-name-on-both-branches distinct scopes,
and the build-time error when a sibling tries to reference a binding
introduced inside or(...).

Refs #17
EOF
)"
```

---

## Task 8: `NotExistsBindingTest` — within-group binding (Decision #5)

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotExistsBindingTest.java`

- [ ] **Step 1: Create the test file**

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

class NotExistsBindingTest extends DrlxBuilderTestSupport {

    @Test
    void notBindingWithinGroup() {
        // `not(var p : /persons1, /persons2[name == p.name])` — fires
        // when NO (p, matching-person-in-persons2) pair joins. Binding
        // `p` is a within-group join; never visible outside the `not`.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule NoJoinExists {
                    not(var p : /persons1, /persons2[ name == p.name ]),
                    do { System.out.println("no join"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            // Alice in persons1, no Alice in persons2 → no join exists → fires.
            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Bob", 40));

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).hasSize(1);
        });
    }

    @Test
    void existsBindingWithinGroup() {
        // `exists(var p : /persons1, /persons2[name == p.name])` — fires
        // when at least one joined pair exists. Binding is within-group.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule JoinExists {
                    exists(var p : /persons1, /persons2[ name == p.name ]),
                    do { System.out.println("join exists"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            // Alice in both → join exists → fires.
            persons1.insert(new Person("Alice", 30));
            persons2.insert(new Person("Alice", 25));

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).hasSize(1);
        });
    }
}
```

- [ ] **Step 2: Rebuild + run tests**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core -am install -DskipTests -q
mvn -pl drlx-parser-core test -Dtest=NotExistsBindingTest
```

Expected: **2 tests, 0 failures**.

- [ ] **Step 3: Full-suite run**

```bash
mvn -pl drlx-parser-core test
```

Expected: **93 total, 0 failures** (91 + 2 new).

- [ ] **Step 4: Commit**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotExistsBindingTest.java
git commit -m "$(cat <<'EOF'
test: NotExistsBindingTest — within-group bindings in not/exists

Covers Decision #5: bindings inside not(...) and exists(...) parse,
emit correctly, and are within-group joins with no outside scope.

Refs #17
EOF
)"
```

---

## Task 9: Extend `NestedGroupTest` with nested-binding case

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NestedGroupTest.java` (append test at end of class, before closing brace)

- [ ] **Step 1: Append the new test** at the end of the class (after `notContainingOr`, before the closing `}`):

```java
    @Test
    void andWithNestedNotCarryingBinding() {
        // `and(var p : /persons1, not(/persons2[name == p.name]))` —
        // binding `p` from outer AND is visible to the nested NOT's
        // oopath condition. Fires when some p in persons1 has no
        // same-named match in persons2. Confirms bindings survive
        // nesting across groupChild boundaries.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule NoSameNameInOther {
                    and(var p : /persons1, not(/persons2[ name == p.name ])),
                    do { System.out.println("unique to persons1"); }
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

            final EntryPoint persons1 = kieSession.getEntryPoint("persons1");
            final EntryPoint persons2 = kieSession.getEntryPoint("persons2");

            // Alice and Bob in persons1; only Alice in persons2.
            // → For Bob, no match in persons2 → fires.
            // → For Alice, match in persons2 → does not fire.
            persons1.insert(new Person("Alice", 30));
            persons1.insert(new Person("Bob", 40));
            persons2.insert(new Person("Alice", 25));

            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).hasSize(1);
        });
    }
```

- [ ] **Step 2: Rebuild + run tests**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn -pl drlx-parser-core -am install -DskipTests -q
mvn -pl drlx-parser-core test -Dtest=NestedGroupTest
```

Expected: **4 tests, 0 failures** (3 existing + 1 new).

- [ ] **Step 3: Full-suite run**

```bash
mvn -pl drlx-parser-core test
```

Expected: **94 total, 0 failures** (93 + 1).

**If this test fails** with a Drools binding-not-found error when `p` crosses into the nested NOT, the culprit is `GroupElement.pack()` rewriting the tree in a way that drops the binding (documented in spec §9 as a possible gotcha). Fallback: change the nested form to `not(var unused : /persons2[name == p.name])` — an explicit binding inside the NOT — which may route through `pack()` differently. If that also fails, document the limitation in the test comment and in HANDOFF.md rather than expanding scope here.

- [ ] **Step 4: Commit**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
git add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NestedGroupTest.java
git commit -m "$(cat <<'EOF'
test: NestedGroupTest — binding survives and→not nesting

Confirms outer AND binding is visible inside a nested NOT's condition,
covering the `pack()` regression surface flagged in the spec §9.

Refs #17
EOF
)"
```

---

## Task 10: Update `docs/DRLXXXX.md` §"'and' / 'or' structures"

The DRLXXXX doc already shows binding examples in §573-588, so the main task is to add a new subsection on **scope**, and add a note that bindings inside groups are now supported.

**Files:**
- Modify: `docs/DRLXXXX.md`

- [ ] **Step 1: Add a scope subsection** after line 588 (just before the "Passive '?' declared" block at line 590), insert:

```markdown

**Binding scope in group CEs**

Bindings introduced inside `or(...)` are visible only within their own branch. DRLX does not unify same-named bindings across OR branches; use `and(...)` to make a binding visible across an OR. Bindings inside `not(...)` and `exists(...)` are visible only within the negated group.

```

The exact placement: after the "Nested 'or', top level 'and' is implicit" example table (line 587-588) and before "Passive '?' declared" (line 590).

- [ ] **Step 2: Verify by eyeball**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
sed -n '585,600p' docs/DRLXXXX.md
```

Confirm the new paragraph sits between the nested-or example and the passive `?` example.

- [ ] **Step 3: Commit**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
git add docs/DRLXXXX.md
git commit -m "$(cat <<'EOF'
docs: DRLXXXX §16 — binding scope in group CEs

Adds an explicit scope paragraph after the nested-or example: OR bindings
are branch-local, NOT/EXISTS bindings are within-group. Reflects the
newly-implemented behaviour from #17.

Refs #17
EOF
)"
```

---

## Task 11: Final full-build sanity

- [ ] **Step 1: Full clean build + test**

```bash
cd /home/tkobayas/usr/work/mvel3-development/drlx-parser
mvn clean install
```

Expected: green across the whole tree. Total new tests: 10 (4 + 3 + 2 + 1). Total tests after: **94 passing** (baseline 84 + 10 new).

- [ ] **Step 2: Confirm all commits landed**

```bash
git log --oneline origin/main..HEAD
```

Expected: six commits from this plan —

```
docs: DRLXXXX §16 — binding scope in group CEs
test: NestedGroupTest — binding survives and→not nesting
test: NotExistsBindingTest — within-group bindings in not/exists
test: OrBindingScopeTest — branch-local binding semantics
test: add BindingInGroupTest explicit-type, both-bound, bare regression
feat: tolerant LSP visitor handles boundOopath in groups
feat: bindings as group CE children — grammar + visitor + first test
```

If the push target is main, push now (only with user approval):

```bash
git push origin main
```

---

## Self-Review

**Spec coverage check:**

| Spec section / Decision | Task(s) |
|---|---|
| Decision #1 — all four CE forms accept bindings | Tasks 3 (grammar), 4 (visitor), 6-9 (tests for and/or/not/exists) |
| Decision #2 — branch-local OR | Task 7 (`OrBindingScopeTest`) |
| Decision #3 — factor `boundOopath` | Task 3 (grammar), Task 4 (visitor) |
| Decision #4 — no binding resolution at DRLX | Implicit: no code for resolution appears in any task |
| Decision #5 — NOT/EXISTS bindings within-group | Task 8 (`NotExistsBindingTest`) |
| Decision #6 — LSP tolerant visitor override | Task 5 |
| Decision #7 — non-goals | Respected: no consensus rule, shadowing detection, `?/`, etc. |
| §1 Grammar | Task 3 |
| §2 Visitor | Task 4 |
| §3 No-change layers | Respected: no edits to IR/runtime/proto |
| §4 LSP | Task 5 |
| §5 Tests (4 new files / extensions) | Tasks 6-9 |
| §6 DRLXXXX updates | Task 10 |
| §7 Implementation order | Task numbering matches |
| §8 Gotchas | ANTLR alternative order in Task 3 comment; `$`-prefixed names out of scope; Drools error-shape handled in Task 7 fallback |

All spec sections map to at least one task. No gaps.

**Placeholder scan:** No `TBD`, no `// TODO`, no "implement later". Every code block is complete.

**Type consistency:**
- `buildPatternFromBoundOopath` defined in Task 4 Step 1; referenced in Task 4 Step 2, Step 3. ✓
- `visitBoundOopath` defined in Task 5 Step 1; referenced implicitly via `visit(first.boundOopath())` in Task 5 Step 2. ✓
- `BoundOopathContext` — ANTLR-generated from the grammar edit in Task 3; available to visitor in Task 4. ✓
- `DrlxParser.GroupChildContext#boundOopath()` accessor — ANTLR-generated from Task 3's added alternative; used in Task 4 Step 3 and Task 5 Step 2. ✓
- `PatternIR` record fields — unchanged; existing constructor signature `(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs)` reused verbatim. ✓

Plan is internally consistent. Ready to execute.
