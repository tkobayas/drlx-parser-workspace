# `if` / `else` branching (Form A) — Implementation Plan (#12)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Prerequisite: #23 must be shipped first.** This plan depends on `EvalIR`, `DrlxLambdaCompiler.createEvalExpression`, the runtime `EvalIR → EvalCondition` mapping, and the user-facing `test` grammar all existing in the codebase. If #23 has not landed, switch to `plans/2026-04-27-test-construct-implementation.md` first.

**Goal:** Implement Form A of DRLXXXX `if` / `else if` / `else` in DRLX rule body — `if (cond) { ruleItems… } …` with a single trailing `do` consequence — by desugaring in the visitor to `or(and(EvalIR(guard), …, body), …)` over the existing `or` / `and` infrastructure plus the `EvalIR` shipped in #23.

**Architecture:** Visitor-level desugar. Grammar parses `if/else if/else`; `DrlxToRuleAstVisitor.buildConditionalBranch` immediately rewrites to `GroupElementIR(OR, [ GroupElementIR(AND, [ EvalIR(c1), …recursive branch body ]), GroupElementIR(AND, [ EvalIR(!(c1)), EvalIR(c2), …branch body ]), … ])`. Cumulative guards split into multiple sequential `EvalIR` nodes (one per prior-condition negation + own condition). No new IR type for `if/else` itself; downstream sees only the desugared `or`-tree.

**Tech Stack:** Java 21 (records, sealed interfaces, switch expressions), ANTLR 4, drools-core 10.1.0, drools-base 10.1.0, EvalIR + EvalCondition pipeline (from #23), JUnit 5, AssertJ.

---

## File map

**Modify:**
- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4` — add `IF`, `ELSE` tokens
- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` — add `conditionalBranch` + `branchBody` productions; add to `ruleItem` alternatives
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` — add `buildConditionalBranch` method; dispatch in `buildRule`

**Create:**
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java` — pure parse-tree assertions + IR-shape verification
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java` — runtime semantics (extends `DrlxBuilderTestSupport`)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Customer.java` — test fact (credit rating)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Product.java` — test fact (rate)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Rating.java` — enum (LOW, MEDIUM, HIGH)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Rates.java` — enum (LOW, MEDIUM, HIGH)
- `drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/CreditUnit.java` — `DataStore<Customer> customers; DataStore<Product> products;`

**Reuse from #23 (no changes expected):**
- `EvalIR`, `DrlxLambdaCompiler.createEvalExpression`, `DrlxEvalExpression`, `EvalCondition` runtime path.

---

## Build & test commands

- `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml clean install`
- `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest`
- `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`

All `git` commands use `git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser <cmd>`.

Baseline assumption: #23 has shipped, so test count starts at ~123 (111 + #23's ~12).

---

## Task 1: Grammar — `IF` / `ELSE` tokens + `conditionalBranch` production

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4`
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java` (new)

- [ ] **Step 1: Write failing parse-only test**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java
package org.drools.drlx.builder.syntax;

import org.antlr.v4.runtime.CharStreams;
import org.antlr.v4.runtime.CommonTokenStream;
import org.drools.drlx.parser.DrlxLexer;
import org.drools.drlx.parser.DrlxParser;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class IfElseParseTest {

    @Test
    void parsesBinaryIfElse() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    if (p.age > 60) {
                        var s : /seniors[ age == p.age ]
                    } else {
                        var j : /juniors[ age == p.age ]
                    },
                    do { System.out.println(p); }
                }
                """;
        DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(rule));
        DrlxParser parser = new DrlxParser(new CommonTokenStream(lexer));
        DrlxParser.CompilationUnitContext ctx = parser.compilationUnit();
        assertThat(parser.getNumberOfSyntaxErrors()).isZero();
        DrlxParser.RuleDeclarationContext rule0 = ctx.ruleDeclaration(0);
        assertThat(rule0.ruleBody().ruleItem()).anyMatch(
                ri -> ri.conditionalBranch() != null);
    }

    @Test
    void parsesElseIfChain() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    if (p.age > 60) { var s : /seniors[ age == p.age ] }
                    else if (p.age > 30) { var a : /adults[ age == p.age ] }
                    else { var j : /juniors[ age == p.age ] },
                    do { System.out.println(p); }
                }
                """;
        DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(rule));
        DrlxParser parser = new DrlxParser(new CommonTokenStream(lexer));
        parser.compilationUnit();
        assertThat(parser.getNumberOfSyntaxErrors()).isZero();
    }

    @Test
    void parsesIfWithoutElse() {
        String rule = """
                package org.drools.drlx.parser;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;
                unit MyUnit;
                rule R {
                    var p : /persons,
                    if (p.age > 60) { var s : /seniors[ age == p.age ] },
                    do { System.out.println(p); }
                }
                """;
        DrlxLexer lexer = new DrlxLexer(CharStreams.fromString(rule));
        DrlxParser parser = new DrlxParser(new CommonTokenStream(lexer));
        parser.compilationUnit();
        assertThat(parser.getNumberOfSyntaxErrors()).isZero();
    }
}
```

- [ ] **Step 2: Run test — verify failure**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest`

Expected: FAIL — `RuleItemContext.conditionalBranch()` doesn't exist; `IF`/`ELSE` tokens unknown.

- [ ] **Step 3: Add `IF` / `ELSE` tokens**

In `DrlxLexer.g4`, alongside `NOT`, `EXISTS`, `DRLX_AND`, `DRLX_OR`, `TEST`:

```antlr
IF       : 'if';
ELSE     : 'else';
```

- [ ] **Step 4: Add `conditionalBranch` + `branchBody` productions**

In `DrlxParser.g4`, modify `ruleItem`:

```antlr
ruleItem
    : rulePattern
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','
    | conditionalBranch ','
    | ruleConsequence
    ;
```

Add new productions (after `testElement` or wherever logical):

```antlr
conditionalBranch
    : IF '(' expression ')' branchBody
      ( ELSE IF '(' expression ')' branchBody )*
      ( ELSE branchBody )?
    ;

branchBody
    : '{' ruleItem* '}'
    ;
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest`

Expected: PASS (all three).

- [ ] **Step 6: Verify no regression**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test`

Expected: full suite passes.

- [ ] **Step 7: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxLexer.g4 \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
grammar: add IF / ELSE tokens and conditionalBranch production

Form A. branchBody reuses ruleItem* so nested if/else, group
elements, patterns, and 'test' inside branches all work.
Visitor desugar follows.

Refs #12
EOF
)"
```

---

## Task 2: Visitor — `buildConditionalBranch` desugar

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java`

**Goal:** Translate

```
if (c1) { B1 } else if (c2) { B2 } else { B3 }
```

to:

```
GroupElementIR(OR, [
  GroupElementIR(AND, [ EvalIR(c1),                B1 expanded… ]),
  GroupElementIR(AND, [ EvalIR(!(c1)), EvalIR(c2), B2 expanded… ]),
  GroupElementIR(AND, [ EvalIR(!(c1)), EvalIR(!(c2)), B3 expanded… ]),
])
```

Cumulative guards split into multiple sequential `EvalIR`s (one per prior-condition negation + own condition); semantically equivalent to a compound `&&` but simpler to construct.

- [ ] **Step 1: Append failing IR-shape test**

In `IfElseParseTest.java`:

```java
import java.util.List;
import org.drools.drlx.builder.DrlxRuleAstParseResult;
import org.drools.drlx.builder.EvalIR;
import org.drools.drlx.builder.GroupElementIR;
import org.drools.drlx.builder.LhsItemIR;
import org.drools.drlx.builder.PatternIR;
import org.drools.drlx.builder.RuleIR;

@Test
void desugarsBinaryIfElseToOrAnd() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                var p : /persons,
                if (p.age > 30) { var s : /seniors[ age == p.age ] }
                else { var j : /juniors[ age == p.age ] },
                do { System.out.println(p); }
            }
            """;
    RuleIR ruleIr = DrlxRuleAstParseResult.parseSingleRule(rule);
    List<LhsItemIR> lhs = ruleIr.lhs();
    assertThat(lhs).hasSize(2);                                    // [Pattern(p), OR(...)]
    assertThat(lhs.get(1)).isInstanceOf(GroupElementIR.class);
    GroupElementIR or = (GroupElementIR) lhs.get(1);
    assertThat(or.kind()).isEqualTo(GroupElementIR.Kind.OR);
    assertThat(or.children()).hasSize(2);

    // Branch 1: AND of [EvalIR("p.age>30"), Pattern(s)]
    GroupElementIR b1 = (GroupElementIR) or.children().get(0);
    assertThat(b1.kind()).isEqualTo(GroupElementIR.Kind.AND);
    assertThat(b1.children().get(0)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).isEqualTo("p.age>30"));
    assertThat(b1.children().get(1)).isInstanceOf(PatternIR.class);

    // Branch 2 (final else): AND of [EvalIR("!(p.age>30)"), Pattern(j)]
    GroupElementIR b2 = (GroupElementIR) or.children().get(1);
    assertThat(b2.kind()).isEqualTo(GroupElementIR.Kind.AND);
    assertThat(b2.children().get(0)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).isEqualTo("!(p.age>30)"));
    assertThat(b2.children().get(1)).isInstanceOf(PatternIR.class);
}

@Test
void desugarsElseIfChainWithCumulativeGuards() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                var p : /persons,
                if (p.age > 60) { var s : /seniors[ age == p.age ] }
                else if (p.age > 30) { var a : /adults[ age == p.age ] }
                else { var j : /juniors[ age == p.age ] },
                do { System.out.println(p); }
            }
            """;
    RuleIR ruleIr = DrlxRuleAstParseResult.parseSingleRule(rule);
    GroupElementIR or = (GroupElementIR) ruleIr.lhs().get(1);
    assertThat(or.children()).hasSize(3);

    // Branch 2: [EvalIR("!(p.age>60)"), EvalIR("p.age>30"), Pattern(a)]
    GroupElementIR b2 = (GroupElementIR) or.children().get(1);
    assertThat(b2.children().get(0)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).isEqualTo("!(p.age>60)"));
    assertThat(b2.children().get(1)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).isEqualTo("p.age>30"));
    assertThat(b2.children().get(2)).isInstanceOf(PatternIR.class);

    // Branch 3 (final else): [EvalIR("!(p.age>60)"), EvalIR("!(p.age>30)"), Pattern(j)]
    GroupElementIR b3 = (GroupElementIR) or.children().get(2);
    assertThat(b3.children().get(0)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).isEqualTo("!(p.age>60)"));
    assertThat(b3.children().get(1)).isInstanceOfSatisfying(EvalIR.class,
            e -> assertThat(e.expression()).isEqualTo("!(p.age>30)"));
    assertThat(b3.children().get(2)).isInstanceOf(PatternIR.class);
}
```

- [ ] **Step 2: Run tests — verify failure**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest`

Expected: FAIL — `buildRule` throws `"Unsupported rule item: …"`.

- [ ] **Step 3: Add dispatch in `buildRule`**

In `DrlxToRuleAstVisitor.java`:

```java
} else if (itemCtx.conditionalBranch() != null) {
    lhs.add(buildConditionalBranch(itemCtx.conditionalBranch()));
}
```

(Place after the `testElement` branch from #23.)

- [ ] **Step 4: Implement `buildConditionalBranch`**

```java
private GroupElementIR buildConditionalBranch(DrlxParser.ConditionalBranchContext ctx) {
    record BranchInput(String condition, DrlxParser.BranchBodyContext body) {}
    java.util.List<BranchInput> branches = new java.util.ArrayList<>();

    int conditionCount = ctx.expression().size();
    int bodyCount = ctx.branchBody().size();

    // Collect (condition, body) for every condition (primary `if` + each `else if`).
    for (int i = 0; i < conditionCount; i++) {
        branches.add(new BranchInput(ctx.expression(i).getText(), ctx.branchBody(i)));
    }

    // Final `else`: body present without matching condition.
    boolean hasFinalElse = bodyCount > conditionCount;
    DrlxParser.BranchBodyContext finalElseBody = hasFinalElse
            ? ctx.branchBody(bodyCount - 1)
            : null;

    // Reject empty branch bodies.
    for (BranchInput b : branches) {
        if (b.body().ruleItem().isEmpty()) {
            throw new RuntimeException("empty branch body");
        }
    }
    if (hasFinalElse && finalElseBody.ruleItem().isEmpty()) {
        throw new RuntimeException("empty branch body");
    }

    java.util.List<LhsItemIR> orChildren = new java.util.ArrayList<>();
    java.util.List<String> priorConditions = new java.util.ArrayList<>();

    for (BranchInput b : branches) {
        java.util.List<LhsItemIR> andChildren = new java.util.ArrayList<>();
        for (String prior : priorConditions) {
            andChildren.add(new EvalIR("!(" + prior + ")", referencedBindings(prior)));
        }
        andChildren.add(new EvalIR(b.condition(), referencedBindings(b.condition())));
        for (DrlxParser.RuleItemContext ri : b.body().ruleItem()) {
            andChildren.add(visitBranchRuleItem(ri));
        }
        orChildren.add(new GroupElementIR(GroupElementIR.Kind.AND, java.util.List.copyOf(andChildren)));
        priorConditions.add(b.condition());
    }

    if (hasFinalElse) {
        java.util.List<LhsItemIR> andChildren = new java.util.ArrayList<>();
        for (String prior : priorConditions) {
            andChildren.add(new EvalIR("!(" + prior + ")", referencedBindings(prior)));
        }
        for (DrlxParser.RuleItemContext ri : finalElseBody.ruleItem()) {
            andChildren.add(visitBranchRuleItem(ri));
        }
        orChildren.add(new GroupElementIR(GroupElementIR.Kind.AND, java.util.List.copyOf(andChildren)));
    }

    return new GroupElementIR(GroupElementIR.Kind.OR, java.util.List.copyOf(orChildren));
}

private LhsItemIR visitBranchRuleItem(DrlxParser.RuleItemContext itemCtx) {
    if (itemCtx.rulePattern() != null)        return buildPattern(itemCtx.rulePattern());
    if (itemCtx.notElement() != null)         return buildNotElement(itemCtx.notElement());
    if (itemCtx.existsElement() != null)      return buildExistsElement(itemCtx.existsElement());
    if (itemCtx.andElement() != null)         return buildAndElement(itemCtx.andElement());
    if (itemCtx.orElement() != null)          return buildOrElement(itemCtx.orElement());
    if (itemCtx.testElement() != null)        return buildTestElement(itemCtx.testElement());
    if (itemCtx.conditionalBranch() != null)  return buildConditionalBranch(itemCtx.conditionalBranch());
    if (itemCtx.ruleConsequence() != null) {
        throw new RuntimeException("'do' is not allowed inside an if/else branch (Form A)");
    }
    throw new IllegalArgumentException("Unsupported branch rule item: " + itemCtx.getText());
}
```

`referencedBindings` was added in #23 (Task 7) — reuse it. If it's private, mark it package-private or duplicate the regex; either works.

If `buildGroupElementFromChildren` already covers this dispatch in a way that handles `testElement` and `conditionalBranch`, reuse it instead of `visitBranchRuleItem`.

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest`

Expected: PASS (all 5 tests).

- [ ] **Step 6: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
visitor: desugar conditionalBranch to OR(AND(EvalIR, ..., body))

Cumulative guards split into multiple sequential EvalIR nodes (one
per prior-condition negation + own condition). Empty branches are
rejected. Nested CEs handled by visitBranchRuleItem recursion.

Refs #12
EOF
)"
```

---

## Task 3: Empty-branch rejection — parse-error coverage

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java`

- [ ] **Step 1: Append failing test**

```java
@Test
void emptyBranchBodyIsRejected() {
    String rule = """
            package org.drools.drlx.parser;
            import org.drools.drlx.domain.Person;
            import org.drools.drlx.ruleunit.MyUnit;
            unit MyUnit;
            rule R {
                var p : /persons,
                if (p.age > 30) { } else { var j : /juniors[ age == p.age ] },
                do { System.out.println(p); }
            }
            """;
    org.assertj.core.api.Assertions.assertThatThrownBy(
            () -> DrlxRuleAstParseResult.parseSingleRule(rule))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("empty branch body");
}
```

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseParseTest#emptyBranchBodyIsRejected`

Expected: PASS (Task 2 already added the empty-branch check).

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseParseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: empty branch body is rejected

Refs #12
EOF
)"
```

---

## Task 4: Runtime — binary `if/else` end-to-end + test fixtures

**Files:**
- Test: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java` (new)
- Domain fixtures: `Customer`, `Product`, `Rating`, `Rates`, `CreditUnit` (new)

- [ ] **Step 1: Add domain test fixtures**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/domain/Rating.java
package org.drools.drlx.domain;
public enum Rating { LOW, MEDIUM, HIGH }
```

```java
// drlx-parser-core/src/test/java/org/drools/drlx/domain/Rates.java
package org.drools.drlx.domain;
public enum Rates { LOW, MEDIUM, HIGH }
```

```java
// drlx-parser-core/src/test/java/org/drools/drlx/domain/Customer.java
package org.drools.drlx.domain;
public class Customer {
    private final String name;
    private Rating creditRating;
    public Customer(String name, Rating creditRating) {
        this.name = name; this.creditRating = creditRating;
    }
    public String getName() { return name; }
    public Rating getCreditRating() { return creditRating; }
    public void setCreditRating(Rating r) { this.creditRating = r; }
    @Override public String toString() { return name; }
}
```

```java
// drlx-parser-core/src/test/java/org/drools/drlx/domain/Product.java
package org.drools.drlx.domain;
public class Product {
    private final String name;
    private final Rates rate;
    public Product(String name, Rates rate) { this.name = name; this.rate = rate; }
    public String getName() { return name; }
    public Rates getRate() { return rate; }
    @Override public String toString() { return name; }
}
```

```java
// drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/CreditUnit.java
package org.drools.drlx.ruleunit;

import org.drools.ruleunits.api.DataStore;
import org.drools.ruleunits.api.RuleUnitData;
import org.drools.ruleunits.impl.datasources.DataStoreImpl;
import org.drools.drlx.domain.Customer;
import org.drools.drlx.domain.Product;

public class CreditUnit implements RuleUnitData {
    private final DataStore<Customer> customers = new DataStoreImpl<>();
    private final DataStore<Product> products = new DataStoreImpl<>();
    public DataStore<Customer> getCustomers() { return customers; }
    public DataStore<Product> getProducts() { return products; }
}
```

(Match the exact import paths used by existing `MyUnit` — read that file before writing this and mirror its conventions.)

- [ ] **Step 2: Write failing runtime tests**

```java
// drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
package org.drools.drlx.builder.syntax;

import org.drools.drlx.builder.DrlxBuilderTestSupport;
import org.drools.drlx.domain.Customer;
import org.drools.drlx.domain.Product;
import org.drools.drlx.domain.Rates;
import org.drools.drlx.domain.Rating;
import org.junit.jupiter.api.Test;
import org.kie.api.runtime.rule.EntryPoint;
import static org.assertj.core.api.Assertions.assertThat;

class IfElseTest extends DrlxBuilderTestSupport {

    @Test
    void binaryIfElse_lowRating_picksHighRateProduct() {
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
                        var p : /products[ rate == Rates.HIGH ]
                    } else {
                        var p : /products[ rate == Rates.LOW ]
                    },
                    do { System.out.println(c + " " + p); }
                }
                """;
        withSession(rule, (session, listener) -> {
            EntryPoint customers = session.getEntryPoint("customers");
            EntryPoint products = session.getEntryPoint("products");
            customers.insert(new Customer("Alice", Rating.LOW));
            products.insert(new Product("luxury", Rates.HIGH));
            products.insert(new Product("budget", Rates.LOW));
            assertThat(session.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }

    @Test
    void binaryIfElse_highRating_picksLowRateProduct() {
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
                        var p : /products[ rate == Rates.HIGH ]
                    } else {
                        var p : /products[ rate == Rates.LOW ]
                    },
                    do { System.out.println(c + " " + p); }
                }
                """;
        withSession(rule, (session, listener) -> {
            EntryPoint customers = session.getEntryPoint("customers");
            EntryPoint products = session.getEntryPoint("products");
            customers.insert(new Customer("Bob", Rating.HIGH));
            products.insert(new Product("luxury", Rates.HIGH));
            products.insert(new Product("budget", Rates.LOW));
            assertThat(session.fireAllRules()).isEqualTo(1);
            assertThat(listener.getAfterMatchFired()).containsExactly("R1");
        });
    }
}
```

- [ ] **Step 3: Run tests**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest`

Expected: PASS — Tasks 1–2 + #23's runtime should make this work end-to-end.

If FAIL: most likely cause is the regex in `referencedBindings` either over- or under-collecting identifiers. Diagnose by:
- Logging the EvalIR's `referencedBindings` for the LOW guard.
- Logging the `boundVariables` map at the time the runtime builder processes the EvalIR.
- Confirming `c` resolves correctly.

If FAIL with NPE in lambda eval: same root cause as Task 4 of #23 — batch-compile resolution may not be triggered for a programmatic-IR-built rule. Check that `pendingLambdas.compile()` (or equivalent) is invoked before `RuleImpl` is finalized.

- [ ] **Step 4: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/domain/Customer.java \
    drlx-parser-core/src/test/java/org/drools/drlx/domain/Product.java \
    drlx-parser-core/src/test/java/org/drools/drlx/domain/Rating.java \
    drlx-parser-core/src/test/java/org/drools/drlx/domain/Rates.java \
    drlx-parser-core/src/test/java/org/drools/drlx/ruleunit/CreditUnit.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: binary if/else end-to-end (DRLXXXX A example)

Refs #12
EOF
)"
```

---

## Task 5: Runtime — `else if` chain

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

- [ ] **Step 1: Append failing test**

```java
@Test
void elseIfChain_threeRatings() {
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
                    var p : /products[ rate == Rates.HIGH ]
                } else if (c.creditRating == Rating.MEDIUM) {
                    var p : /products[ rate == Rates.MEDIUM ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { System.out.println(c + " " + p); }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.MEDIUM));
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("standard", Rates.MEDIUM));
        products.insert(new Product("budget", Rates.LOW));
        assertThat(session.fireAllRules()).isEqualTo(1);
        assertThat(listener.getAfterMatchFired()).containsExactly("R1");
    });
}
```

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest#elseIfChain_threeRatings`

Expected: PASS (no implementation changes needed — chain handling already in Task 2).

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: else-if chain (3 ratings)

Refs #12
EOF
)"
```

---

## Task 6: Runtime — no-final-else, rule does not fire

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

- [ ] **Step 1: Append failing test**

```java
@Test
void noFinalElse_doesNotFireWhenNoBranchMatches() {
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
                    var p : /products[ rate == Rates.HIGH ]
                } else if (c.creditRating == Rating.MEDIUM) {
                    var p : /products[ rate == Rates.MEDIUM ]
                },
                do { System.out.println(c + " " + p); }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Carol", Rating.HIGH));    // matches no branch
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("standard", Rates.MEDIUM));
        assertThat(session.fireAllRules()).isZero();
        assertThat(listener.getAfterMatchFired()).isEmpty();
    });
}
```

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest#noFinalElse_doesNotFireWhenNoBranchMatches`

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: no-final-else rule does not fire when no branch matches

Refs #12
EOF
)"
```

---

## Task 7: Runtime — same-name binding flow

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

- [ ] **Step 1: Append failing test**

```java
@Test
void sameNameBindingFlowsToConsequence() {
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
                    var p : /products[ rate == Rates.HIGH ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { /* consequence references p; tracked by listener */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.LOW));
        Product hi = new Product("luxury", Rates.HIGH);
        Product lo = new Product("budget", Rates.LOW);
        products.insert(hi);
        products.insert(lo);
        assertThat(session.fireAllRules()).isEqualTo(1);

        // Verify the bound 'p' came from the LOW branch (i.e., the HIGH-rated product).
        // If TrackingAgendaEventListener exposes match objects, use it; otherwise verify
        // by counting fires with the contradiction case (insert only HIGH product, expect
        // 1 fire; insert only LOW product, expect 0 fires for LOW customer).
        // Adjust to whatever the listener API supports.
    });
}
```

If `TrackingAgendaEventListener` doesn't expose tuple/match objects, the simplest substitute is two ordering tests:
- Insert ONLY `Product("luxury", HIGH)` with a LOW customer → expect 1 fire (matched the LOW branch).
- Insert ONLY `Product("budget", LOW)` with a LOW customer → expect 0 fires (LOW customer needed HIGH product).

Use whichever assertion shape best fits the existing listener.

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest#sameNameBindingFlowsToConsequence`

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: same-name binding flows to consequence per branch

Refs #12
EOF
)"
```

---

## Task 8: Runtime — multi-element / nested CE / nested if

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

- [ ] **Step 1: Append three tests**

```java
@Test
void multiElementBranch_implicitAnd() {
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
                    var p1 : /products[ rate == Rates.HIGH ],
                    var p2 : /products[ rate == Rates.MEDIUM ]
                } else {
                    var p1 : /products[ rate == Rates.LOW ],
                    var p2 : /products[ rate == Rates.LOW ]
                },
                do { /* uses p1, p2 */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.LOW));
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("standard", Rates.MEDIUM));
        assertThat(session.fireAllRules()).isEqualTo(1);
    });
}

@Test
void groupElementInsideBranch_not() {
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
                    not /products[ rate == Rates.HIGH ],
                    var p : /products[ rate == Rates.MEDIUM ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { /* uses p */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.LOW));
        products.insert(new Product("standard", Rates.MEDIUM));
        assertThat(session.fireAllRules()).isEqualTo(1);
    });
}

@Test
void nestedIfElseInsideBranch() {
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
                    }
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { /* uses p */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        customers.insert(new Customer("Alice", Rating.LOW));
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("standard", Rates.MEDIUM));
        assertThat(session.fireAllRules()).isEqualTo(1);
    });
}
```

- [ ] **Step 2: Run all three**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest`

Expected: All pass.

If `nestedIfElseInsideBranch` fails: check that `branchBody : '{' ruleItem* '}'` accepts nested `conditionalBranch` (it should — `ruleItem` includes it). If ANTLR reports ambiguity around nested braces, examine the generated parser for token-stream issues.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: multi-element / nested-CE / nested-if branches

Refs #12
EOF
)"
```

---

## Task 9: Property-reactivity natural behavior

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java`

DRLX uses `PropertySpecificOption.ALWAYS` (`DrlxRuleAstRuntimeBuilder.java:105`), so all properties are watched by default. An if-guard like `if (c.creditRating == Rating.LOW) { … }` re-evaluates naturally on `creditRating` updates — no explicit watch list required.

#23's `TestElementTest.test_refiresOnPropertyUpdate` already covers this for the underlying `EvalIR`. This task adds an analogous test through the if/else surface.

- [ ] **Step 1: Append failing test**

```java
@Test
void ifElse_refiresOnOuterScopePropertyUpdate() {
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
                    var p : /products[ rate == Rates.HIGH ]
                } else {
                    var p : /products[ rate == Rates.LOW ]
                },
                do { /* uses p */ }
            }
            """;
    withSession(rule, (session, listener) -> {
        EntryPoint customers = session.getEntryPoint("customers");
        EntryPoint products = session.getEntryPoint("products");
        Customer alice = new Customer("Alice", Rating.HIGH);   // initial: else branch
        org.kie.api.runtime.rule.FactHandle handle = customers.insert(alice);
        products.insert(new Product("luxury", Rates.HIGH));
        products.insert(new Product("budget", Rates.LOW));
        assertThat(session.fireAllRules()).isEqualTo(1);
        listener.getAfterMatchFired().clear();

        alice.setCreditRating(Rating.LOW);
        customers.update(handle, alice);
        // ALWAYS-mode property reactivity — re-evaluates without explicit watch list.
        assertThat(session.fireAllRules()).isEqualTo(1);
    });
}
```

- [ ] **Step 2: Run test**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/pom.xml test -Dtest=IfElseTest#ifElse_refiresOnOuterScopePropertyUpdate`

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/IfElseTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test: if/else re-fires on outer-scope property update

ALWAYS-mode property reactivity covers guard expressions naturally;
no explicit watch list required (mirrors #23's TestElementTest).

Refs #12
EOF
)"
```

---

## Task 10: Polish — full-suite verification + docs

**Files:**
- Modify: `specs/IMPLEMENT_SYNTAX_CANDIDATES.md` (mark item #9 done)

- [ ] **Step 1: Run the full DRLX test suite**

Run: `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test`

Expected: 0 failures. Test count = #23 baseline (~123) + ~10 new for #12.

- [ ] **Step 2: Update syntax-candidates doc**

In `/home/tkobayas/usr/work/mvel3-development/drlx-parser/specs/IMPLEMENT_SYNTAX_CANDIDATES.md`, change row 9:

```
| 9 | ~~**`if`/`else` branching in rule body** (Form A — single trailing consequence)~~ — **IMPLEMENTED in #12**. Form B (per-branch consequences) tracked in #22. | Conditional elements within rule body |
```

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    specs/IMPLEMENT_SYNTAX_CANDIDATES.md

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
docs: mark candidate #9 (if/else Form A) as implemented in #12

Closes #12
EOF
)"
```

`Closes #12` only on this final commit so GitHub auto-closes when merged.

- [ ] **Step 4: Push and open PR**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push origin main

gh pr create --repo tkobayas/drlx-parser \
    --title "feat: if/else branching (Form A) (#12)" \
    --body "$(cat <<'EOF'
## Summary

- Adds `if (cond) { … } else if (cond) { … } else { … }` Form A to DRLX rule bodies — single trailing `do` consequence.
- Visitor desugars to `or(and(EvalIR(guard), …, body), …)` over the existing `or` / `and` infra and the `EvalIR` shipped in #23.
- No new IR for `if/else` itself; downstream sees only the desugared `or`-tree.
- Form B (per-branch consequences) tracked in #22.

## Test plan

- [x] Parse-only: binary, else-if chain, no-else, IR shape (with cumulative guards), empty-branch rejection
- [x] Runtime: binary if/else (LOW/HIGH cases), else-if chain, no-final-else, same-name binding flow, multi-element branch, group-element inside branch, nested if/else, property-reactivity with watch list
- [x] Full DRLX suite passes (~133 tests, 0 failures)

Closes #12
EOF
)"
```

---

## Self-review

### Spec coverage

- ✅ Grammar additions — Task 1
- ✅ Parse-tree → IR desugar — Task 2
- ✅ Empty-branch rejection — Task 3
- ✅ Bindings & scope (inherits `or`) — exercised by Tasks 4, 5, 7, 8
- ✅ Branch with only group element — Task 8
- ✅ Nested if/else — Task 8
- ✅ No-final-else — Task 6
- ✅ Property-reactivity limitation + workaround — Task 9
- ✅ Cumulative-guard split into multiple `EvalIR`s — encoded in Task 2

### Type / signature consistency

- All `EvalIR(String, List<String>)` usages — same shape as #23's contract.
- All `referencedBindings(String)` calls return `List<String>` (binding *names*).
- `visitBranchRuleItem` covers all `ruleItem` alternatives including the `testElement` from #23.

### Risk concentration

Lower risk than the original consolidated plan. The MVEL3 lambda bridging that was the high-risk piece now lives in #23 and is verified before #12 starts. Remaining risks for #12:

- **Task 2 — referencedBindings regex over-collection.** May pick up Java keywords or type names; runtime filtering masks this for valid binding names but could surface false positives if a guard expression mentions a name that *coincidentally* exists as a `boundVariables` key but isn't intended to be referenced. Mitigation: keep the regex broad, rely on runtime filtering.
- **Task 8 — nested if/else inside branch.** Grammar recursion may surprise ANTLR; if ambiguity, simplify by parameterizing `branchBody` differently.

### Sequencing

Tasks 1 → 2 → 3 strictly sequential. Tasks 4–9 each depend on Task 2 but are independent of each other; can be parallelized in subagent-driven mode. Task 10 is final.
