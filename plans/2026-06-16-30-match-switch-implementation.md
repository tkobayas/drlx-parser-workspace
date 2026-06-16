# #30 Match (switch) Conditional Element — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `match` as a switch-like conditional element that desugars to synthetic rules (Form B style), supporting value match, type match (`#Type`), and type match with constraints (`#Type[...]`).

**Architecture:** New `MATCH` lexer token + grammar rules (`matchBranch`, `matchCase`, `matchDefault`, `matchPattern`, `matchCaseBody`). Visitor method `buildMatchBranch` generates `List<RuleIR>` — one synthetic rule per case — paralleling the existing `buildConditionalBranchFormB`. Conditions are `EvalIR` nodes: equality for value match, `instanceof` (+ cast + constraints) for type match.

**Tech Stack:** ANTLR4 grammar, Java 17, JUnit 5 + AssertJ, Drools RuleUnit runtime

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `DrlxLexer.g4` | Modify | Add `MATCH` token |
| `DrlxParser.g4` | Modify | Add `matchBranch`, `matchCase`, `matchDefault`, `matchPattern`, `matchCaseBody` rules; integrate into `ruleItem` and `branchItem` |
| `DrlxToRuleAstVisitor.java` | Modify | Add `buildMatchBranch`, `buildMatchCondition`, integrate into `buildRule` and `buildBranchItem` |
| `MatchParseTest.java` | Create | Parse-level + IR tests for match |
| `MatchTest.java` | Create | Runtime tests for match |

---

### Task 1: Grammar — MATCH token and matchBranch rules

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4`
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`

- [ ] **Step 1: Add MATCH token to DrlxLexer.g4**

Add after the `ELSE` line in DrlxLexer.g4:

```
MATCH    : 'match';
```

Full file after change:
```
// DRLX Lexer - minimal extension of MVEL3 lexer

lexer grammar DrlxLexer;

import Mvel3Lexer;

// DRLX-specific keywords
UNIT   : 'unit';
RULE   : 'rule';
NOT      : 'not';
EXISTS   : 'exists';
DRLX_AND : 'and';
DRLX_OR  : 'or';
TEST     : 'test';
IF       : 'if';
ELSE     : 'else';
MATCH    : 'match';
```

- [ ] **Step 2: Add matchBranch grammar rules to DrlxParser.g4**

Add after the `branchConsequence` rule (after line 166) in DrlxParser.g4:

```antlr
// 'match' conditional element — DRLXXXX §"match".
// Form B only (per-case consequences). Desugars to one synthetic rule per case.
// CE terminator `,` (after the whole construct) owned by ruleItem.
matchBranch
    : MATCH '(' expression ')' matchCase+ matchDefault?
    ;

matchCase
    : CASE matchPattern matchCaseBody
    ;

matchDefault
    : DEFAULT matchCaseBody
    ;

// Match pattern: type match `#Type` with optional constraints `[expr, ...]`,
// or value match (any expression).
matchPattern
    : HASH identifier ('[' drlxExpression (',' drlxExpression)* ']')?
    | expression
    ;

// Match case body: block form, `do statement` form, or bare expression form.
matchCaseBody
    : '{' (branchItem (',' branchItem)*)? '}'
    | DO statement
    | expression
    ;
```

- [ ] **Step 3: Integrate matchBranch into ruleItem**

In the `ruleItem` rule, add `matchBranch ','?` as a new alternative. Add it after the `conditionalBranch` alternative:

```antlr
ruleItem
    : rulePattern
    | oopathExpression ','
    | accumulateItem ','
    | accKeywordItem ','
    | groupByKeywordItem ','
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','
    | conditionalBranch ','?
    | matchBranch ','?
    | ruleConsequence
    ;
```

- [ ] **Step 4: Integrate matchBranch into branchItem**

In the `branchItem` rule, add `matchBranch` as a new alternative. Add it after `conditionalBranch`:

```antlr
branchItem
    : boundOopath
    | oopathExpression
    | notElement
    | existsElement
    | andElement
    | orElement
    | testElement
    | conditionalBranch
    | matchBranch
    | branchConsequence
    ;
```

- [ ] **Step 5: Rebuild parser and verify no ANTLR errors**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml generate-sources -q
```
Expected: BUILD SUCCESS, no ANTLR warnings or errors.

- [ ] **Step 6: Install the module**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml install -DskipTests -q
```
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4 drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#30): add MATCH token and matchBranch grammar rules"
```

---

### Task 2: Parse tests — verify grammar accepts match syntax

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/MatchParseTest.java`

- [ ] **Step 1: Write parse tests**

Create `MatchParseTest.java` with a `parseRules` helper (same pattern as `IfElseFormBParseTest`) and the following tests:

```java
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

class MatchParseTest {

    private static final String PREAMBLE = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Customer;
            import org.drools.drlx.domain.Product;
            import org.drools.drlx.domain.Rating;
            import org.drools.drlx.domain.Rates;
            import org.drools.drlx.ruleunit.CreditUnit;
            unit CreditUnit;
            """;

    @Test
    void valueMatch_blockBody_threeRules() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW {
                            var p : /products[ rate == Rates.HIGH ],
                            do { System.out.println("low"); }
                        }
                        case Rating.MEDIUM {
                            var p : /products[ rate == Rates.MEDIUM ],
                            do { System.out.println("medium"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules).hasSize(3);
        assertThat(rules.get(0).name()).isEqualTo("R$0");
        assertThat(rules.get(1).name()).isEqualTo("R$1");
        assertThat(rules.get(2).name()).isEqualTo("R$2");
    }

    @Test
    void valueMatch_correctConditionsAndGuards() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW {
                            var p : /products[ rate == Rates.HIGH ],
                            do { System.out.println("low"); }
                        }
                        case Rating.MEDIUM {
                            var p : /products[ rate == Rates.MEDIUM ],
                            do { System.out.println("medium"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        List<RuleIR> rules = parseRules(rule);

        // R$0: commonPrefix + EvalIR(equality) + branch pattern
        RuleIR r0 = rules.get(0);
        assertThat(r0.lhs().get(0)).isInstanceOf(PatternIR.class);
        assertThat(r0.lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
                e -> assertThat(e.expression()).isEqualTo("c.creditRating == Rating.LOW"));
        assertThat(r0.lhs().get(2)).isInstanceOf(PatternIR.class);

        // R$1: commonPrefix + negated-guard + EvalIR(equality) + branch pattern
        RuleIR r1 = rules.get(1);
        assertThat(r1.lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
                e -> assertThat(e.expression()).isEqualTo("!(c.creditRating == Rating.LOW)"));
        assertThat(r1.lhs().get(2)).isInstanceOfSatisfying(EvalIR.class,
                e -> assertThat(e.expression()).isEqualTo("c.creditRating == Rating.MEDIUM"));

        // R$2 (default): commonPrefix + two negated guards, no positive condition
        RuleIR r2 = rules.get(2);
        assertThat(r2.lhs()).hasSize(3);
        assertThat(r2.lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
                e -> assertThat(e.expression()).startsWith("!("));
        assertThat(r2.lhs().get(2)).isInstanceOfSatisfying(EvalIR.class,
                e -> assertThat(e.expression()).startsWith("!("));
    }

    @Test
    void typeMatch_instanceofCondition() {
        String rule = PREAMBLE.replace("import org.drools.drlx.ruleunit.CreditUnit;", "import org.drools.drlx.ruleunit.MyUnit;")
                .replace("unit CreditUnit;", "unit MyUnit;") + """
                rule R {
                    var o : /objects,
                    match (o)
                        case #Car {
                            do { System.out.println("car"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules).hasSize(2);
        assertThat(rules.get(0).lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
                e -> assertThat(e.expression()).isEqualTo("o instanceof Car"));
    }

    @Test
    void typeMatchWithConstraints_instanceofAndCast() {
        String rule = PREAMBLE.replace("import org.drools.drlx.ruleunit.CreditUnit;", "import org.drools.drlx.ruleunit.MyUnit;")
                .replace("unit CreditUnit;", "unit MyUnit;") + """
                rule R {
                    var o : /objects,
                    match (o)
                        case #Car[speed > 80] {
                            do { System.out.println("fast car"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules).hasSize(2);
        assertThat(rules.get(0).lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
                e -> assertThat(e.expression()).isEqualTo("o instanceof Car && ((Car)o).speed > 80"));
    }

    @Test
    void typeMatchWithMultipleConstraints() {
        String rule = PREAMBLE.replace("import org.drools.drlx.ruleunit.CreditUnit;", "import org.drools.drlx.ruleunit.MyUnit;")
                .replace("unit CreditUnit;", "unit MyUnit;") + """
                rule R {
                    var o : /objects,
                    match (o)
                        case #Car[speed > 80, vin == "ABC"] {
                            do { System.out.println("fast ABC car"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules.get(0).lhs().get(1)).isInstanceOfSatisfying(EvalIR.class,
                e -> assertThat(e.expression()).isEqualTo(
                        "o instanceof Car && ((Car)o).speed > 80 && ((Car)o).vin == \"ABC\""));
    }

    @Test
    void noDefault_twoRules() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW {
                            do { System.out.println("low"); }
                        }
                        case Rating.MEDIUM {
                            do { System.out.println("medium"); }
                        }
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules).hasSize(2);
    }

    @Test
    void doStatementBody() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW do { System.out.println("low"); }
                        default do { System.out.println("other"); }
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules).hasSize(2);
        assertThat(rules.get(0).rhs().block()).contains("low");
    }

    @Test
    void bareExpressionBody() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW System.out.println("low")
                        default System.out.println("other")
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules).hasSize(2);
        assertThat(rules.get(0).rhs().block()).contains("low");
    }

    @Test
    void trailingComma_accepted() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW {
                            do { System.out.println("low"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        },
                }
                """;
        List<RuleIR> rules = parseRules(rule);
        assertThat(rules).hasSize(2);
    }

    @Test
    void error_emptyCaseBody() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW {}
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        assertThatThrownBy(() -> parseRules(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("at least one action");
    }

    @Test
    void error_trailingDoConflict() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW {
                            do { System.out.println("low"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        },
                    do { System.out.println("trailing"); }
                }
                """;
        assertThatThrownBy(() -> parseRules(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("trailing");
    }

    @Test
    void error_itemsAfterMatch() {
        String rule = PREAMBLE + """
                rule R {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW {
                            do { System.out.println("low"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        },
                    var p : /products,
                }
                """;
        assertThatThrownBy(() -> parseRules(rule))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("not supported");
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

- [ ] **Step 2: Run the tests to verify they fail (visitor not yet implemented)**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=MatchParseTest -pl . -q
```
Expected: Tests fail — visitor does not handle `matchBranch` yet.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/MatchParseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#30): add parse and IR tests for match conditional element"
```

---

### Task 3: Visitor — buildMatchBranch and integration

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

- [ ] **Step 1: Add buildMatchCondition helper method**

Add this private method to `DrlxToRuleAstVisitor` (after `buildConditionalBranchFormB`). It generates the condition string from a match pattern and subject expression:

```java
private String buildMatchCondition(DrlxParser.MatchPatternContext patternCtx,
                                    String subjectText) {
    if (patternCtx.HASH() != null) {
        // Type match: #Type or #Type[constraints]
        String typeName = patternCtx.identifier().getText();
        StringBuilder sb = new StringBuilder();
        sb.append(subjectText).append(" instanceof ").append(typeName);
        if (patternCtx.drlxExpression() != null && !patternCtx.drlxExpression().isEmpty()) {
            for (DrlxParser.DrlxExpressionContext drlxExpr : patternCtx.drlxExpression()) {
                String constraintText = getText(drlxExpr);
                sb.append(" && ((").append(typeName).append(")").append(subjectText).append(").").append(constraintText);
            }
        }
        return sb.toString();
    }
    // Value match: subject == expression
    return subjectText + " == " + getText(patternCtx.expression());
}
```

- [ ] **Step 2: Add buildMatchBranch method**

Add this private method to `DrlxToRuleAstVisitor` (after `buildMatchCondition`). It generates synthetic rules from a matchBranch — parallel to `buildConditionalBranchFormB`:

```java
private List<RuleIR> buildMatchBranch(
        DrlxParser.MatchBranchContext ctx,
        List<LhsItemIR> commonPrefix,
        List<RuleAnnotationIR> annotations,
        List<RuleParameterIR> parameters,
        String ruleName) {

    String subjectText = getText(ctx.expression());

    List<RuleIR> syntheticRules = new ArrayList<>();
    List<String> priorConditions = new ArrayList<>();

    // Process each case
    for (int i = 0; i < ctx.matchCase().size(); i++) {
        DrlxParser.MatchCaseContext caseCtx = ctx.matchCase(i);
        String condition = buildMatchCondition(caseCtx.matchPattern(), subjectText);
        DrlxParser.MatchCaseBodyContext body = caseCtx.matchCaseBody();

        List<LhsItemIR> branchLhs = new ArrayList<>();
        List<String> consequenceTexts = new ArrayList<>();

        processCaseBody(body, branchLhs, consequenceTexts);

        List<LhsItemIR> fullLhs = new ArrayList<>(commonPrefix);
        for (String prior : priorConditions) {
            String negated = "!(" + prior + ")";
            fullLhs.add(new EvalIR(negated, extractIdentifiers(negated)));
        }
        fullLhs.add(new EvalIR(condition, extractIdentifiers(condition)));
        fullLhs.addAll(branchLhs);

        syntheticRules.add(new RuleIR(
                ruleName + "$" + syntheticRules.size(),
                annotations, parameters,
                List.copyOf(fullLhs),
                new ConsequenceIR(combineConsequences(consequenceTexts))));

        priorConditions.add(condition);
    }

    // Process default (if present)
    if (ctx.matchDefault() != null) {
        DrlxParser.MatchCaseBodyContext body = ctx.matchDefault().matchCaseBody();

        List<LhsItemIR> branchLhs = new ArrayList<>();
        List<String> consequenceTexts = new ArrayList<>();

        processCaseBody(body, branchLhs, consequenceTexts);

        List<LhsItemIR> fullLhs = new ArrayList<>(commonPrefix);
        for (String prior : priorConditions) {
            String negated = "!(" + prior + ")";
            fullLhs.add(new EvalIR(negated, extractIdentifiers(negated)));
        }
        fullLhs.addAll(branchLhs);

        syntheticRules.add(new RuleIR(
                ruleName + "$" + syntheticRules.size(),
                annotations, parameters,
                List.copyOf(fullLhs),
                new ConsequenceIR(combineConsequences(consequenceTexts))));
    }

    return syntheticRules;
}
```

- [ ] **Step 3: Add processCaseBody helper method**

Add this private method. It handles the three forms of `matchCaseBody` — block, `do statement`, and bare expression:

```java
private void processCaseBody(DrlxParser.MatchCaseBodyContext body,
                              List<LhsItemIR> branchLhs,
                              List<String> consequenceTexts) {
    if (body.branchItem() != null && !body.branchItem().isEmpty()) {
        // Block form: { branchItem, branchItem, ... }
        boolean seenConsequence = false;
        for (DrlxParser.BranchItemContext bi : body.branchItem()) {
            if (bi.branchConsequence() != null) {
                seenConsequence = true;
                consequenceTexts.add(extractBranchConsequence(bi.branchConsequence()));
            } else {
                if (seenConsequence) {
                    throw new RuntimeException(
                            "Action statements must follow all patterns and group elements in a match case body");
                }
                branchLhs.add(buildBranchItem(bi));
            }
        }
        if (consequenceTexts.isEmpty()) {
            throw new RuntimeException(
                    "Match case must contain at least one action statement");
        }
    } else if (body.DO() != null) {
        // do statement form
        consequenceTexts.add(trimBraces(getText(body.statement())));
    } else if (body.expression() != null) {
        // bare expression form
        consequenceTexts.add(getText(body.expression()));
    }
    if (consequenceTexts.isEmpty()) {
        throw new RuntimeException(
                "Match case must contain at least one action statement");
    }
}
```

- [ ] **Step 4: Add combineConsequences helper method**

Add this private method. It combines multiple consequence texts into a single block string (same logic as in `buildConditionalBranchFormB`, extracted for reuse):

```java
private static String combineConsequences(List<String> consequenceTexts) {
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
    return combined.toString();
}
```

- [ ] **Step 5: Make trimBraces accessible to processCaseBody**

The existing `trimBraces` method is `private static` and already accessible within the class. No change needed — just confirm it is accessible for the `processCaseBody` call to `trimBraces(getText(body.statement()))`.

- [ ] **Step 6: Integrate matchBranch into buildRule**

In the `buildRule` method, add a `matchBranch` branch alongside the existing `conditionalBranch` handling. Insert this block after the `conditionalBranch` else-if block (after the closing `}` of the `itemCtx.conditionalBranch() != null` block, before the `else` throw):

```java
} else if (itemCtx.matchBranch() != null) {
    flushPending(lhs, pendingPattern, pendingAccs);
    if (rhs != null) {
        throw new RuntimeException(
                "Rule with per-case consequences cannot also have a trailing `do` consequence");
    }
    if (idx < ruleItems.size() - 1) {
        if (ruleItems.get(idx + 1).ruleConsequence() != null) {
            throw new RuntimeException(
                    "Rule with per-case consequences cannot also have a trailing `do` consequence");
        }
        throw new RuntimeException(
                "Items after a match element are not supported");
    }
    return buildMatchBranch(
            itemCtx.matchBranch(),
            List.copyOf(lhs), annotations, parameters, name);
}
```

- [ ] **Step 7: Integrate matchBranch into buildBranchItem**

In the `buildBranchItem` method, add a `matchBranch` case. Insert before the `branchConsequence` check:

```java
if (ctx.matchBranch() != null) {
    throw new IllegalArgumentException(
            "matchBranch in buildBranchItem — match inside branch bodies should be handled at the ruleItem level");
}
```

Note: The spec says match can be nested inside if/else branches (`branchItem`), but a nested match with per-case consequences would be a Form B nesting issue. For now, reject it with a clear error. The grammar allows parsing it, but the visitor rejects nesting — same pattern as nested Form B if/else.

- [ ] **Step 8: Run parse tests**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=MatchParseTest -pl . -q
```
Expected: All tests pass.

- [ ] **Step 9: Run full test suite to check for regressions**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -q
```
Expected: All existing tests still pass.

- [ ] **Step 10: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "feat(#30): implement buildMatchBranch visitor for match conditional element"
```

---

### Task 4: Runtime tests

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/MatchTest.java`

- [ ] **Step 1: Write runtime tests**

Create `MatchTest.java` extending `DrlxBuilderTestSupport`:

```java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.domain.Car;
import org.drools.drlx.domain.Customer;
import org.drools.drlx.domain.Product;
import org.drools.drlx.domain.Rating;
import org.drools.drlx.domain.Rates;
import org.drools.drlx.domain.Vehicle;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MatchTest extends DrlxBuilderTestSupport {

    @Test
    void valueMatch_lowRating_firstCaseFires() {
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
                    match (c.creditRating)
                        case Rating.LOW {
                            var p : /products[ rate == Rates.HIGH ],
                            do { System.out.println("low: " + c.name + " " + p.name); }
                        }
                        case Rating.MEDIUM {
                            var p : /products[ rate == Rates.MEDIUM ],
                            do { System.out.println("medium: " + c.name); }
                        }
                        default {
                            do { System.out.println("other: " + c.name); }
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
    void valueMatch_mediumRating_secondCaseFires() {
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
                    match (c.creditRating)
                        case Rating.LOW {
                            var p : /products[ rate == Rates.HIGH ],
                            do { System.out.println("low"); }
                        }
                        case Rating.MEDIUM {
                            var p : /products[ rate == Rates.MEDIUM ],
                            do { System.out.println("medium"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Bob", Rating.MEDIUM));
            unit.products.add(new Product("luxury", Rates.HIGH));
            unit.products.add(new Product("standard", Rates.MEDIUM));
            unit.products.add(new Product("budget", Rates.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$1");
        });
    }

    @Test
    void valueMatch_noMatch_defaultFires() {
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
                    match (c.creditRating)
                        case Rating.LOW {
                            do { System.out.println("low"); }
                        }
                        case Rating.MEDIUM {
                            do { System.out.println("medium"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Carol", Rating.HIGH));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$2");
        });
    }

    @Test
    void valueMatch_noDefault_noMatchDoesNotFire() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW {
                            do { System.out.println("low"); }
                        }
                        case Rating.MEDIUM {
                            do { System.out.println("medium"); }
                        }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Carol", Rating.HIGH));
            assertThat(instance.fire()).isZero();
            assertThat(listener.getAfterMatchFired()).isEmpty();
        });
    }

    @Test
    void typeMatch_carInstanceofFires() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Vehicle;
                import org.drools.drlx.domain.Car;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var o : /objects,
                    match (o)
                        case #Car {
                            do { System.out.println("car: " + o.vin); }
                        }
                        default {
                            do { System.out.println("vehicle: " + o.vin); }
                        }
                }
                """;
        withMyUnitInstance(rule, (instance, unit, listener) -> {
            unit.objects.add(new Car("CAR1", 100));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void typeMatch_nonCar_defaultFires() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Vehicle;
                import org.drools.drlx.domain.Car;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var o : /objects,
                    match (o)
                        case #Car {
                            do { System.out.println("car"); }
                        }
                        default {
                            do { System.out.println("vehicle: " + o.vin); }
                        }
                }
                """;
        withMyUnitInstance(rule, (instance, unit, listener) -> {
            unit.objects.add(new Vehicle("VEH1"));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$1");
        });
    }

    @Test
    void typeMatchWithConstraints_fastCarFires() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Vehicle;
                import org.drools.drlx.domain.Car;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var o : /objects,
                    match (o)
                        case #Car[speed > 80] {
                            do { System.out.println("fast car"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        withMyUnitInstance(rule, (instance, unit, listener) -> {
            unit.objects.add(new Car("FAST1", 100));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void typeMatchWithConstraints_slowCarFallsToDefault() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Vehicle;
                import org.drools.drlx.domain.Car;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R1 {
                    var o : /objects,
                    match (o)
                        case #Car[speed > 80] {
                            do { System.out.println("fast car"); }
                        }
                        default {
                            do { System.out.println("other"); }
                        }
                }
                """;
        withMyUnitInstance(rule, (instance, unit, listener) -> {
            unit.objects.add(new Car("SLOW1", 50));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$1");
        });
    }

    @Test
    void doStatementBody_fires() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW do { System.out.println("low"); }
                        default do { System.out.println("other"); }
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }

    @Test
    void bareExpressionBody_fires() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Customer;
                import org.drools.drlx.domain.Rating;
                import org.drools.drlx.ruleunit.CreditUnit;
                unit CreditUnit;
                rule R1 {
                    var c : /customers,
                    match (c.creditRating)
                        case Rating.LOW System.out.println("low")
                        default System.out.println("other")
                }
                """;
        withCreditUnitInstance(rule, (instance, unit, listener) -> {
            unit.customers.add(new Customer("Alice", Rating.LOW));
            assertThat(instance.fire()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1$0");
        });
    }
}
```

- [ ] **Step 2: Run runtime tests**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=MatchTest -pl . -q
```
Expected: All tests pass.

- [ ] **Step 3: Run full test suite**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -q
```
Expected: All tests pass (no regressions).

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/MatchTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "test(#30): add runtime tests for match conditional element"
```
