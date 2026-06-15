# Form B `if`/`else` with per-branch consequences — Implementation Plan (#22)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Form B of DRLX `if`/`else` — each branch contains its own consequence (`do` or bare expression), and the compiler decomposes the single rule into N synthetic `RuleImpl` objects, one per branch.

**Architecture:** Rule decomposition at the visitor level. Grammar adds `branchConsequence` to `branchItem`. When any branch contains a consequence, the visitor detects Form B and calls `buildConditionalBranchFormB`, which produces N `RuleIR` objects (one per branch) with shared prefix, cumulative guards, branch-specific patterns, and branch-specific consequences. No IR model changes. No runtime builder changes — each synthetic rule flows through the existing `buildRule` pipeline.

**Tech Stack:** Java 21 (records, sealed interfaces), ANTLR 4, drools-core 10.1.0, JUnit 5, AssertJ.

**Spec:** `specs/2026-06-11-if-else-form-b-design.md`

---

## File map

**Modify:**
- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` — add `branchConsequence` rule; add `branchConsequence` + `oopathExpression` alternatives to `branchItem`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` — `isFormB`, `buildConditionalBranchFormB`, `extractBranchConsequence`; change `buildRule` → `List<RuleIR>`; update `visitDrlxCompilationUnit` to `addAll`; update `buildBranchItem` for `oopathExpression`

**No changes (spec-confirmed):**
- `DrlxRuleAstModel.java` — no new IR types
- `DrlxRuleAstRuntimeBuilder.java` — synthetic rules use existing pipeline

**Create:**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBParseTest.java` — parse-level IR assertions
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBTest.java` — runtime tests (extends `DrlxBuilderTestSupport`)

---

## Build & test commands

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml clean install -Dmvel3.compiler.lambda.persistence=true
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseFormBParseTest
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseFormBTest
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true
```

All `git` commands use `git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser <cmd>`.

Baseline: 415 tests pass (with `-Dmvel3.compiler.lambda.persistence=true`).

---

## Task 1: Grammar — `branchConsequence` rule + `branchItem` extensions

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBParseTest.java`

- [ ] **Step 1: Write failing parse test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBParseTest.java
package org.drools.drlx.builder.syntax;

import java.util.List;

import org.antlr.v4.runtime.CharStreams;
import org.antlr.v4.runtime.CommonTokenStream;
import org.drools.drlx.builder.DrlxRuleAstModel.CompilationUnitIR;
import org.drools.drlx.builder.DrlxRuleAstModel.ConsequenceIR;
import org.drools.drlx.builder.DrlxRuleAstModel.EvalIR;
import org.drools.drlx.builder.DrlxRuleAstModel.LhsItemIR;
import org.drools.drlx.builder.DrlxRuleAstModel.PatternIR;
import org.drools.drlx.builder.DrlxRuleAstModel.RuleIR;
import org.drools.drlx.builder.DrlxToRuleAstVisitor;
import org.drools.drlx.parser.DrlxLexer;
import org.drools.drlx.parser.DrlxParser;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class IfElseFormBParseTest {

    @Test
    void parsesFormB_twoRulesGenerated() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println(c + " " + p); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println(c + " " + p); }
                    }
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules).hasSize(2);
        assertThat(rules.get(0).name()).isEqualTo("R$0");
        assertThat(rules.get(1).name()).isEqualTo("R$1");
    }

    private static List<RuleIR> parseRules(String source) {
        DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(source));
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        DrlxParser parser = new DrlxParser(tokens);
        DrlxParser.DrlxCompilationUnitContext ctx = parser.drlxCompilationUnit();
        assertThat(parser.getNumberOfSyntaxErrors()).isZero();
        DrlxToRuleAstVisitor visitor = new DrlxToRuleAstVisitor(tokens);
        CompilationUnitIR unit = visitor.visitDrlxCompilationUnit(ctx);
        return unit.rules();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseFormBParseTest#parsesFormB_twoRulesGenerated`
Expected: FAIL — `branchConsequence` is not a valid grammar rule yet, so `do { ... }` inside a branch body causes a parse error.

- [ ] **Step 3: Add `branchConsequence` rule and update `branchItem` in grammar**

In `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`, add the `branchConsequence` rule after the `branchItem` rule (after line 154), and add `oopathExpression` + `branchConsequence` alternatives to `branchItem`.

Update comment on `conditionalBranch` rule (line 125):
```antlr
// 'if' / 'else if' / 'else' branching — DRLXXXX §"if/else".
// Form A: pattern-only branches with a single trailing `do` at rule level.
// Form B: per-branch consequences — each branch contains `do`/bare actions.
// CE terminator `,` (after the whole construct) owned by ruleItem.
```

Update `branchItem` (replace lines 146–154):
```antlr
// Branch items mirror `ruleItem` minus the trailing `,` and minus
// `ruleConsequence` (Form A). Form B consequences use `branchConsequence`.
// `boundOopath` MUST come before `branchConsequence` / `oopathExpression`:
// ANTLR picks first match, and bound form has a strictly more constrained
// prefix than a bare expression or oopath.
branchItem
    : boundOopath
    | oopathExpression
    | notElement
    | existsElement
    | andElement
    | orElement
    | testElement
    | conditionalBranch
    | branchConsequence
    ;
```

Add new rule after `branchItem` (after the closing `;`):
```antlr
// Per-branch consequence (Form B). `DO statement` for explicit form
// (typically a block: `do { stmt; stmt; }`). Bare `expression` for
// single-expression actions without semicolons.
branchConsequence
    : DO statement
    | expression
    ;
```

**Note:** The bare form uses `expression` (not `statement`) so that `Email.to(...).send()` parses without requiring a semicolon — matching the DRLXXXX examples. The `DO` form uses `statement` to support blocks: `do { stmt1; stmt2; }`.

- [ ] **Step 4: Rebuild grammar and run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml clean install -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseFormBParseTest#parsesFormB_twoRulesGenerated`
Expected: FAIL — parse succeeds (no syntax errors) but the visitor still produces 1 `RuleIR` (not 2). The test asserts `hasSize(2)` and will fail. This confirms the grammar works but the visitor hasn't been updated yet.

- [ ] **Step 5: Verify existing tests still pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true`
Expected: 415 tests pass. Grammar additions don't affect existing rules.

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBParseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "grammar: add branchConsequence rule for Form B if/else (#22)"
```

---

## Task 2: `buildRule` → `List<RuleIR>` + Form detection

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

- [ ] **Step 1: Add `isFormB` helper method**

Add after `buildConditionalBranch` (after line 869):

```java
private boolean isFormB(DrlxParser.ConditionalBranchContext ctx) {
    for (DrlxParser.BranchBodyContext body : ctx.branchBody()) {
        for (DrlxParser.BranchItemContext item : body.branchItem()) {
            if (item.branchConsequence() != null) {
                return true;
            }
        }
    }
    return false;
}
```

- [ ] **Step 2: Change `buildRule` return type to `List<RuleIR>`**

Change the method signature (line 118):
```java
private List<RuleIR> buildRule(DrlxParser.RuleDeclarationContext ctx,
                               Map<String, String> annotationImports) {
```

Change the loop from for-each to indexed for (line 137). Replace:
```java
for (DrlxParser.RuleItemContext itemCtx : ctx.ruleBody().ruleItem()) {
```
with:
```java
List<DrlxParser.RuleItemContext> ruleItems = ctx.ruleBody().ruleItem();
for (int idx = 0; idx < ruleItems.size(); idx++) {
    DrlxParser.RuleItemContext itemCtx = ruleItems.get(idx);
```

In the conditional branch handling (line 217-218), add Form B detection. Replace:
```java
} else if (itemCtx.conditionalBranch() != null) {
    lhs.add(buildConditionalBranch(itemCtx.conditionalBranch()));
}
```
with:
```java
} else if (itemCtx.conditionalBranch() != null) {
    if (isFormB(itemCtx.conditionalBranch())) {
        flushPending(lhs, pendingPattern, pendingAccs);
        if (rhs != null) {
            throw new RuntimeException(
                    "Rule with per-branch consequences cannot also have a trailing `do` consequence");
        }
        if (idx < ruleItems.size() - 1) {
            if (ruleItems.get(idx + 1).ruleConsequence() != null) {
                throw new RuntimeException(
                        "Rule with per-branch consequences cannot also have a trailing `do` consequence");
            }
            throw new RuntimeException(
                    "Items after a per-branch if/else are not supported");
        }
        return buildConditionalBranchFormB(
                itemCtx.conditionalBranch(),
                List.copyOf(lhs), annotations, parameters, name);
    }
    lhs.add(buildConditionalBranch(itemCtx.conditionalBranch()));
}
```

Change the return statement at the end of `buildRule` (line 225):
```java
return List.of(new RuleIR(name, annotations, parameters, List.copyOf(lhs), rhs));
```

- [ ] **Step 3: Add stub `buildConditionalBranchFormB`**

Add after `isFormB`:

```java
private List<RuleIR> buildConditionalBranchFormB(
        DrlxParser.ConditionalBranchContext ctx,
        List<LhsItemIR> commonPrefix,
        List<RuleAnnotationIR> annotations,
        List<RuleParameterIR> parameters,
        String ruleName) {
    throw new UnsupportedOperationException("Form B not implemented yet");
}
```

- [ ] **Step 4: Update `visitDrlxCompilationUnit` caller**

Change line 107 from:
```java
ctx.ruleDeclaration().forEach(ruleCtx -> rules.add(buildRule(ruleCtx, annotationImports)));
```
to:
```java
ctx.ruleDeclaration().forEach(ruleCtx -> rules.addAll(buildRule(ruleCtx, annotationImports)));
```

- [ ] **Step 5: Update `buildBranchItem` for new alternatives**

In `buildBranchItem` (line 871), add handling for `oopathExpression` and a safety-net for `branchConsequence`. Add after the `boundOopath` check (line 872):

```java
if (ctx.oopathExpression() != null)   return buildPatternFromOopath(ctx.oopathExpression());
```

And before the final `throw` (line 879):
```java
if (ctx.branchConsequence() != null) {
    throw new IllegalArgumentException(
            "branchConsequence in buildBranchItem — should be handled by buildConditionalBranchFormB");
}
```

- [ ] **Step 6: Run existing tests to verify Form A still works**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true`
Expected: 415 tests pass. Form A returns singleton lists, `addAll` collects them identically to `add`.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "refactor: buildRule returns List<RuleIR>, add Form B detection stub (#22)"
```

---

## Task 3: `buildConditionalBranchFormB` — core decomposition

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBParseTest.java`

- [ ] **Step 1: Implement `extractBranchConsequence`**

Add after `extractConsequence` (after line 971):

```java
private String extractBranchConsequence(DrlxParser.BranchConsequenceContext ctx) {
    if (ctx.DO() != null) {
        return trimBraces(getText(ctx.statement()));
    }
    return getText(ctx.expression());
}
```

- [ ] **Step 2: Implement `buildConditionalBranchFormB`**

Replace the stub from Task 2 with the full implementation:

```java
private List<RuleIR> buildConditionalBranchFormB(
        DrlxParser.ConditionalBranchContext ctx,
        List<LhsItemIR> commonPrefix,
        List<RuleAnnotationIR> annotations,
        List<RuleParameterIR> parameters,
        String ruleName) {

    int conditionCount = ctx.expression().size();
    int bodyCount = ctx.branchBody().size();
    boolean hasFinalElse = bodyCount > conditionCount;

    List<RuleIR> syntheticRules = new ArrayList<>();
    List<String> priorConditions = new ArrayList<>();

    for (int i = 0; i < bodyCount; i++) {
        boolean isElse = (i == bodyCount - 1 && hasFinalElse);
        String condition = isElse ? null : getText(ctx.expression(i));
        DrlxParser.BranchBodyContext body = ctx.branchBody(i);

        if (body.branchItem().isEmpty()) {
            throw new RuntimeException("empty branch body");
        }

        List<LhsItemIR> branchLhs = new ArrayList<>();
        List<String> consequenceTexts = new ArrayList<>();
        boolean seenConsequence = false;

        for (DrlxParser.BranchItemContext bi : body.branchItem()) {
            if (bi.branchConsequence() != null) {
                seenConsequence = true;
                consequenceTexts.add(extractBranchConsequence(bi.branchConsequence()));
            } else {
                if (seenConsequence) {
                    throw new RuntimeException(
                            "Action statements must follow all patterns and group elements in a branch body");
                }
                if (bi.conditionalBranch() != null && isFormB(bi.conditionalBranch())) {
                    throw new RuntimeException(
                            "Nested per-branch consequences are not supported; "
                            + "extract the inner if/else into a separate rule");
                }
                branchLhs.add(buildBranchItem(bi));
            }
        }

        if (consequenceTexts.isEmpty()) {
            throw new RuntimeException(
                    "Form B branch must contain at least one action statement; "
                    + "use Form A with a trailing `do` for pattern-only branches");
        }

        List<LhsItemIR> fullLhs = new ArrayList<>(commonPrefix);
        for (String prior : priorConditions) {
            String negated = "!(" + prior + ")";
            fullLhs.add(new EvalIR(negated, extractIdentifiers(negated)));
        }
        if (condition != null) {
            fullLhs.add(new EvalIR(condition, extractIdentifiers(condition)));
        }
        fullLhs.addAll(branchLhs);

        StringBuilder combined = new StringBuilder();
        for (String text : consequenceTexts) {
            String trimmed = text.trim();
            if (!trimmed.endsWith(";")) {
                trimmed += ";";
            }
            if (combined.length() > 0) {
                combined.append(" ");
            }
            combined.append(trimmed);
        }

        syntheticRules.add(new RuleIR(
                ruleName + "$" + i,
                annotations, parameters,
                List.copyOf(fullLhs),
                new ConsequenceIR(combined.toString())));

        if (condition != null) {
            priorConditions.add(condition);
        }
    }

    return syntheticRules;
}
```

- [ ] **Step 3: Run the two-branch parse test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseFormBParseTest#parsesFormB_twoRulesGenerated`
Expected: PASS — 2 rules with names `R$0` and `R$1`.

- [ ] **Step 4: Add IR shape tests**

Add to `IfElseFormBParseTest.java`:

```java
@Test
void twoRules_correctLhsAndRhs() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    do { System.out.println(c + " " + p); }
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    do { System.out.println(c + " " + p); }
                }
            }
            """;
    List<RuleIR> rules = parseRules(rule);

    // R$0: LHS = [Pattern(c), EvalIR(c.creditRating == Rating.LOW), Pattern(p)]
    RuleIR r0 = rules.get(0);
    assertThat(r0.lhs()).hasSize(3);
    assertThat(r0.lhs().get(0)).isInstanceOf(PatternIR.class);
    assertThat(r0.lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).contains("Rating.LOW"));
    assertThat(r0.lhs().get(2)).isInstanceOf(PatternIR.class);
    assertThat(r0.rhs()).isNotNull();
    assertThat(r0.rhs().block()).contains("System.out.println");

    // R$1: LHS = [Pattern(c), EvalIR(!(c.creditRating == Rating.LOW)), Pattern(p)]
    RuleIR r1 = rules.get(1);
    assertThat(r1.lhs()).hasSize(3);
    assertThat(r1.lhs().get(0)).isInstanceOf(PatternIR.class);
    assertThat(r1.lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).startsWith("!(").contains("Rating.LOW"));
    assertThat(r1.lhs().get(2)).isInstanceOf(PatternIR.class);
    assertThat(r1.rhs()).isNotNull();
}

@Test
void elseIfChain_cumulativeGuards() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    do { System.out.println("low"); }
                } else if (c.creditRating == Rating.MEDIUM) {
                    var p : /products[ rate == Rates.MEDIUM ],
                    do { System.out.println("medium"); }
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    do { System.out.println("other"); }
                }
            }
            """;
    List<RuleIR> rules = parseRules(rule);
    assertThat(rules).hasSize(3);
    assertThat(rules.get(0).name()).isEqualTo("R$0");
    assertThat(rules.get(1).name()).isEqualTo("R$1");
    assertThat(rules.get(2).name()).isEqualTo("R$2");

    // R$1: [Pattern(c), EvalIR(!(LOW)), EvalIR(MEDIUM), Pattern(p)]
    RuleIR r1 = rules.get(1);
    assertThat(r1.lhs()).hasSize(4);
    assertThat(r1.lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).startsWith("!(").contains("Rating.LOW"));
    assertThat(r1.lhs().get(2)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).contains("Rating.MEDIUM").doesNotStartWith("!("));

    // R$2 (else): [Pattern(c), EvalIR(!(LOW)), EvalIR(!(MEDIUM)), Pattern(p)]
    RuleIR r2 = rules.get(2);
    assertThat(r2.lhs()).hasSize(4);
    assertThat(r2.lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).startsWith("!(").contains("Rating.LOW"));
    assertThat(r2.lhs().get(2)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).startsWith("!(").contains("Rating.MEDIUM"));
}

@Test
void singleIf_noElse_oneRule() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    do { System.out.println(c); }
                }
            }
            """;
    List<RuleIR> rules = parseRules(rule);
    assertThat(rules).hasSize(1);
    assertThat(rules.get(0).name()).isEqualTo("R$0");
    assertThat(rules.get(0).lhs()).hasSize(3);
}

@Test
void bareExpressionConsequence_parsesCorrectly() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    System.out.println(c + " " + p)
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    System.out.println(c + " " + p)
                }
            }
            """;
    List<RuleIR> rules = parseRules(rule);
    assertThat(rules).hasSize(2);
    assertThat(rules.get(0).rhs().block()).contains("System.out.println");
}

@Test
void multipleConsequencesInBranch_combined() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    do { System.out.println("first"); },
                    do { System.out.println("second"); }
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    do { System.out.println("else"); }
                }
            }
            """;
    List<RuleIR> rules = parseRules(rule);
    assertThat(rules.get(0).rhs().block()).contains("first").contains("second");
}

@Test
void mixedBareAndDoInSameBranch_combined() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    System.out.println("bare"),
                    do { System.out.println("explicit"); }
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    do { System.out.println("else"); }
                }
            }
            """;
    List<RuleIR> rules = parseRules(rule);
    assertThat(rules.get(0).rhs().block()).contains("bare").contains("explicit");
}

@Test
void formAUnchanged_singleRuleWithOr() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { System.out.println(c + " " + p); }
            }
            """;
    List<RuleIR> rules = parseRules(rule);
    assertThat(rules).hasSize(1);
    assertThat(rules.get(0).name()).isEqualTo("R");
}
```

- [ ] **Step 5: Run all parse tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseFormBParseTest`
Expected: All tests pass.

- [ ] **Step 6: Run full test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true`
Expected: 415 + new tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBParseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat: Form B rule decomposition in buildConditionalBranchFormB (#22)"
```

---

## Task 4: Validation rules

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBParseTest.java`

Validation logic was already added in Tasks 2-3. This task adds the error-path tests.

- [ ] **Step 1: Write error tests**

Add to `IfElseFormBParseTest.java`:

```java
@Test
void error_formBBranchWithNoConsequence() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    do { System.out.println(c); }
                } else {
                    var p : /products[ rate == Rates.LOW ]
                }
            }
            """;
    assertThatThrownBy(() -> parseRules(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("at least one action statement");
}

@Test
void error_formBPlusTrailingDo() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    do { System.out.println(c); }
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    do { System.out.println(c); }
                },
                do { System.out.println("trailing"); }
            }
            """;
    assertThatThrownBy(() -> parseRules(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("trailing");
}

@Test
void error_consequenceBeforePattern() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    do { System.out.println("before pattern"); },
                    var p : /products[ rate == Rates.HIGH ]
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    do { System.out.println(c); }
                }
            }
            """;
    assertThatThrownBy(() -> parseRules(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("must follow all patterns");
}

@Test
void error_itemsAfterFormBConditionalBranch() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    var p : /products[ rate == Rates.HIGH ],
                    do { System.out.println(c); }
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    do { System.out.println(c); }
                },
                var extra : /products[ rate == Rates.MEDIUM ]
            }
            """;
    assertThatThrownBy(() -> parseRules(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("not supported");
}

@Test
void error_nestedFormBInsideFormB() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            rule R {
                var c : /customers,
                if (c.creditRating == Rating.LOW) {
                    if (c.name == "Alice") {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println("nested"); }
                    } else {
                        var p : /products[ rate == Rates.MEDIUM ],
                        do { System.out.println("nested else"); }
                    },
                    do { System.out.println("outer"); }
                } else {
                    var p : /products[ rate == Rates.LOW ],
                    do { System.out.println("else"); }
                }
            }
            """;
    assertThatThrownBy(() -> parseRules(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("Nested per-branch consequences are not supported");
}
```

- [ ] **Step 2: Run validation tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseFormBParseTest`
Expected: All tests pass (including error tests).

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBParseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: Form B validation error paths (#22)"
```

---

## Task 5: Runtime tests

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBTest.java`

- [ ] **Step 1: Create runtime test class with binary if/else test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBTest.java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.domain.Customer;
import org.drools.drlx.domain.Product;
import org.drools.drlx.domain.Rating;
import org.drools.drlx.domain.Rates;
import org.drools.ruleunits.api.DataHandle;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class IfElseFormBTest extends DrlxBuilderTestSupport {

    @Test
    void binaryFormB_lowRating_ifBranchFires() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println(c + " " + p); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println(c + " " + p); }
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.LOW));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void binaryFormB_highRating_elseBranchFires() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println(c + " " + p); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println(c + " " + p); }
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Bob", Rating.HIGH));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$1");
        });
    }

    @Test
    void elseIfChain_middleBranchFires() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println("low"); }
                    } else if (c.creditRating == Rating.MEDIUM) {
                        var p : /products[ rate == Rates.MEDIUM ],
                        do { System.out.println("medium"); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println("other"); }
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.MEDIUM));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("standard", Rates.MEDIUM));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$1");
        });
    }

    @Test
    void noFinalElse_noMatchDoesNotFire() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println("low"); }
                    } else if (c.creditRating == Rating.MEDIUM) {
                        var p : /products[ rate == Rates.MEDIUM ],
                        do { System.out.println("medium"); }
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Carol", Rating.HIGH));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("standard", Rates.MEDIUM));
            assertThat(instance.fire()).isZero();
            assertThat(listener.getAfterMatchFired()).isEmpty();
        });
    }

    @Test
    void branchSpecificBindings_accessibleInConsequence() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println(c.name + " gets " + p.name); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println(c.name + " gets " + p.name); }
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.LOW));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void multipleConsequences_allExecute() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println("first: " + c.name); },
                        do { System.out.println("second: " + p.name); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println("else"); }
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.LOW));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void bareExpressionConsequence_firesCorrectly() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        System.out.println(c + " " + p)
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        System.out.println(c + " " + p)
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.LOW));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void propertyReactivity_guardReEvaluatesOnUpdate() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println("low"); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println("other"); }
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            Customer alice = new Customer("Alice", Rating.HIGH);
            DataHandle handle = unit.customers.add(alice);
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$1");
            listener.getAfterMatchFired().clear();

            alice.setCreditRating(Rating.LOW);
            unit.customers.update(handle, alice);
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void nestedFormAInsideFormB_works() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        if (c.name == "Alice") {
                            var p : /products[ rate == Rates.HIGH ]
                        } else {
                            var p : /products[ rate == Rates.MEDIUM ]
                        },
                        do { System.out.println(c + " " + p); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println(c + " " + p); }
                    }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.LOW));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("standard", Rates.MEDIUM));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void ruleAttributes_salienceHonoredOnSyntheticRules() {
        String ruleHigh = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                import org.kie.api.definition.rule.Salience;
                unit CreditUnit;
                @Salience(10)
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        var p : /products[ rate == Rates.HIGH ],
                        do { System.out.println("R1 low"); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println("R1 other"); }
                    }
                }
                rule R2 {
                    var c : /customers,
                    var p : /products,
                    do { System.out.println("R2"); }
                }
                """;
        withCreditUnitInstance(ruleHigh, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.LOW));
            unit.products.add(new Product("luxury", Rates.HIGH));
            assertThat(instance.fire()).isEqualTo(2);
            assertThat(listener.getAfterMatchFired().get(0)).isEqualTo("R1$0");
        });
    }

    @Test
    void nestedFormBInsideFormB_compilationError() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Product;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.domain.Rates;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    if (c.creditRating == Rating.LOW) {
                        if (c.name == "Alice") {
                            var p : /products[ rate == Rates.HIGH ],
                            do { System.out.println("nested"); }
                        } else {
                            var p : /products[ rate == Rates.MEDIUM ],
                            do { System.out.println("nested else"); }
                        },
                        do { System.out.println("outer"); }
                    } else {
                        var p : /products[ rate == Rates.LOW ],
                        do { System.out.println("else"); }
                    }
                }
                """;
        assertThatThrownBy(() -> withCreditUnitInstance(rule, (instance, unit, listener) -> {}))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("Nested per-branch consequences are not supported");
    }
}
```

- [ ] **Step 2: Run runtime tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseFormBTest`
Expected: All 10 tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseFormBTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test: Form B runtime tests — binary, else-if, no-else, nested, bare expression (#22)"
```

---

## Task 6: Full test suite verification

**Files:** none (verification only)

- [ ] **Step 1: Run the full drlx-parser test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true`
Expected: 415 + ~21 new tests = ~436 total. Zero failures.

- [ ] **Step 2: Verify Form A tests are unchanged**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dmvel3.compiler.lambda.persistence=true -Dtest=IfElseParseTest,IfElseTest`
Expected: All existing Form A tests pass without modification.

- [ ] **Step 3: Install the module**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -Dmvel3.compiler.lambda.persistence=true`
Expected: BUILD SUCCESS.
