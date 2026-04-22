# Multi-element `not(/a, /b)` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the `notElement` grammar + visitor to accept `not(/a[, /b, ...])`, completing the follow-up scope tracked under `#8`.

**Architecture:** Add a labeled second alternative to the `notElement` production; change `DrlxToRuleAstVisitor.buildNotElement` to iterate the (now multi-valued) `ctx.oopathExpression()` list and emit one `PatternIR` per child. Runtime builder, proto schema, and tolerant/frozen visitors are unchanged — the infra was already designed for N children, this landing just exercises N > 1.

**Tech Stack:** Java 17, Maven, ANTLR4 (auto-regen via `antlr4-maven-plugin`), JUnit 5, AssertJ, Drools `GroupElementFactory.newNotInstance()` with multi-child AND semantics.

**Spec:** `/home/tkobayas/claude/public/drlx-parser/specs/2026-04-22-multi-element-not-design.md` (commit `b39114f`).

**Issue:** `#8` (still open post-single-element landing).

---

## File Structure

All paths are relative to `/home/tkobayas/usr/work/mvel3-development/drlx-parser/`.

| Path | Role |
|------|------|
| `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4` | Add `#parenNot` alternative to `notElement` production. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java` | Change `buildNotElement` to iterate oopath children; extract a `buildPatternFromOopath` helper. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java` | Add 3 new tests (`notParens_singleElement`, `notMultiElement_crossProduct`, `notEmpty_failsParse`), delete `notMultiElementForm_failsParse`. |

**Maven rule:** after modifying `drlx-parser-core`, run `mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am install`. ANTLR regen is automatic on `compile`. Do NOT `cd` — use `-f` / `-C` flags.

**Commit convention:** scoped imperative subject (`feat(grammar):`, `feat(visitor):`, `test(not):`), HEREDOC body, `Refs #8` trailer. No `Co-Authored-By`. No emoji. Commit directly on `main`.

---

## Task 1: RED — add failing `notMultiElement_crossProduct` test

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

After this task, the full suite has one RED test (parser rejects `not(/a, /b)` today). The RED flips GREEN in Task 3 after the grammar+visitor land.

- [ ] **Step 1: Append the test**

Insert right before the existing `notMultiElementForm_failsParse` test (will be deleted in Task 5; placement for diff clarity). The test imports `org.drools.drlx.domain.Order` which was added in the prior landing — no new imports.

```java
    @Test
    void notMultiElement_crossProduct() {
        // `not(/persons[age<18], /orders[amount>1000])` — NOT-with-AND semantics:
        // suppresses when BOTH an under-18 person AND a high-value order exist;
        // re-fires when either is removed.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Order;
                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule NoUnderageHighValuePair {
                    not(/persons[ age < 18 ], /orders[ amount > 1000 ]),
                    do { System.out.println("no risky pair"); }
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

            // Neither side present → NOT satisfied, rule fires once.
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).containsExactly("NoUnderageHighValuePair");

            // Only under-18 person → NOT still satisfied (no order match), no new firing.
            fired.clear();
            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isZero();

            // Add high-value order → both sides match, NOT unsatisfied, still no firing.
            orders.insert(new Order("O1", 99, 5000));
            assertThat(kieSession.fireAllRules()).isZero();

            // Remove the order (retract) → NOT satisfied again, rule re-fires.
            // (Use fact handle from a fresh insert so we can retract it.)
            // Simpler: add an orphan adult person — doesn't affect NOT state.
            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isZero();  // still suppressed
        });
    }
```

- [ ] **Step 2: Run the target test — expect RED**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='NotTest#notMultiElement_crossProduct' 2>&1 | tail -15
```

Expected: FAILURE. Surefire shows a parse error like `DRLX parse error at line N:M - missing ',' at '('` or similar — the current grammar does not accept `not(`.

- [ ] **Step 3: Commit (RED)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): failing happy-path for multi-element not (RED)

Adds notMultiElement_crossProduct — `not(/persons[age<18],
/orders[amount>1000])`. Fails today because the grammar rejects the
paren form; flips GREEN once the grammar + visitor follow-up lands.

Refs #8
EOF
)"
```

---

## Task 2: Grammar + Visitor — atomic change

**Files:**
- Modify: `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

Both changes go in one commit. The grammar change makes ANTLR regenerate `NotElementContext.oopathExpression()` as `List<OopathExpressionContext>` instead of singular `OopathExpressionContext`, which breaks the current visitor's type-level expectation. Splitting these would leave the build red between commits.

- [ ] **Step 1: Extend the grammar**

In `DrlxParser.g4`, replace the existing `notElement` block (lines 57-62):

**Before:**

```antlr
// 'not' group element — single-element form only in this landing.
// DRLX spec §"'not' / 'exists'" line 599. Multi-element `not(/a, /b)`
// deferred to a follow-up issue. Trailing ',' mirrors rulePattern.
notElement
    : NOT oopathExpression ','
    ;
```

**After:**

```antlr
// 'not' group element. Bare form `not /a,` for a single child; paren
// form `not(/a[, /b, ...]),` for either single-in-parens or multi-element.
// DRLX spec §"'not' / 'exists'" line 597. Trailing ',' after the closing
// ')' (or after the bare oopath) is the ruleItem terminator.
notElement
    : NOT oopathExpression ','                                        # singleNot
    | NOT '(' oopathExpression (',' oopathExpression)* ')' ','        # parenNot
    ;
```

- [ ] **Step 2: Update the visitor — new helper + iterate children**

In `DrlxToRuleAstVisitor.java`, replace the existing `buildNotElement` method:

**Before:**

```java
    private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx) {
        DrlxParser.OopathExpressionContext oopathCtx = ctx.oopathExpression();
        String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
        String castTypeName = extractCastType(oopathCtx);
        List<String> conditions = extractConditions(oopathCtx);
        List<String> positionalArgs = extractPositionalArgs(oopathCtx);
        PatternIR inner = new PatternIR("", "", entryPoint, conditions, castTypeName, positionalArgs);
        return new GroupElementIR(GroupElementIR.Kind.NOT, List.of(inner));
    }
```

**After:**

```java
    private GroupElementIR buildNotElement(DrlxParser.NotElementContext ctx) {
        List<LhsItemIR> children = new ArrayList<>();
        for (DrlxParser.OopathExpressionContext oopathCtx : ctx.oopathExpression()) {
            children.add(buildPatternFromOopath(oopathCtx));
        }
        return new GroupElementIR(GroupElementIR.Kind.NOT, List.copyOf(children));
    }

    private PatternIR buildPatternFromOopath(DrlxParser.OopathExpressionContext oopathCtx) {
        String entryPoint = extractEntryPointFromOopathCtx(oopathCtx);
        String castTypeName = extractCastType(oopathCtx);
        List<String> conditions = extractConditions(oopathCtx);
        List<String> positionalArgs = extractPositionalArgs(oopathCtx);
        return new PatternIR("", "", entryPoint, conditions, castTypeName, positionalArgs);
    }
```

Note: `ArrayList` and `List` are already imported at the top of the file (verify with `grep "^import java.util" src/main/java/.../DrlxToRuleAstVisitor.java`); `LhsItemIR` is already imported. No new imports needed.

- [ ] **Step 3: Run the RED test — expect GREEN**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='NotTest#notMultiElement_crossProduct' 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`, `Tests run: 1, Failures: 0, Errors: 0`. If the test still fails with a parse error, verify the grammar change saved correctly (`grep "parenNot" drlx-parser-core/src/main/antlr4/...`) — ANTLR regen runs during `compile`, so the updated `DrlxParser.java` should reflect the new alternative.

- [ ] **Step 4: Run the full suite — prove no regressions**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am install 2>&1 | grep -E "Tests run:|BUILD" | tail -15
```

Expected: `BUILD SUCCESS`. The existing `notMultiElementForm_failsParse` test now **fails** because the parser accepts the input — that's expected and gets cleaned up in Task 5. Every other test should still pass. Final line for drlx-parser-core default run: `Tests run: 62, Failures: 1, Errors: 0` (the failing test being `notMultiElementForm_failsParse`).

If any other test regresses, the cause is almost certainly that ANTLR's generated method signature for `oopathExpression()` on `NotElementContext` changed in a way that broke a different caller — grep the codebase for `NotElementContext` before deeper debugging.

- [ ] **Step 5: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4 \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
feat(grammar): accept paren form not(/a[, /b, ...])

Adds a second labelled alternative parenNot to notElement. Bare
form singleNot stays unchanged. The visitor now iterates ctx's
oopathExpression() list and emits one PatternIR per child —
GroupElementIR(NOT, [children...]) with children.size() >= 1.

Infra (runtime builder, proto schema, tolerant/frozen visitors)
needs no change: buildLhs already recurses, proto has repeated
LhsItem children. Flips notMultiElement_crossProduct GREEN.

Refs #8
EOF
)"
```

---

## Task 3: Happy test — `notParens_singleElement`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

Proves spec Decision #1 — `not(/a)` is legal and behaves identically to the bare `not /a`.

- [ ] **Step 1: Append the test**

Insert before the (soon-to-be-deleted) `notMultiElementForm_failsParse` test:

```java
    @Test
    void notParens_singleElement() {
        // Spec Decision #1 — single-element in parens is accepted and behaves
        // identically to the bare form. `not(/persons[age<18])` ≡ `not /persons[age<18]`.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule OnlyAdultsParen {
                    not(/persons[ age < 18 ]),
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

            persons.insert(new Person("Alice", 30));
            assertThat(kieSession.fireAllRules()).isEqualTo(1);
            assertThat(fired).containsExactly("OnlyAdultsParen");

            fired.clear();
            persons.insert(new Person("Charlie", 10));
            assertThat(kieSession.fireAllRules()).isZero();
        });
    }
```

- [ ] **Step 2: Run the target test**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am test \
    -Dtest='NotTest#notParens_singleElement' 2>&1 | tail -10
```

Expected: passes. If it fails with a parse error, the grammar's `(',' oopathExpression)*` (zero-or-more) isn't matching the no-extra-oopath case — verify the `*` (not `+`) quantifier in Task 2 Step 1.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): single element in parens accepted

notParens_singleElement — `not(/persons[age<18])` is legal and
behaves identically to the bare form. Proves spec Decision #1
(permissive reading of DRLXXXX.md line 597).

Refs #8
EOF
)"
```

---

## Task 4: Negative test — `notEmpty_failsParse`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

- [ ] **Step 1: Append the test**

Insert before `notMultiElementForm_failsParse` (still to be deleted in Task 5):

```java
    @Test
    void notEmpty_failsParse() {
        // `not()` — parens with zero inner oopaths. Grammar's first
        // oopathExpression is not optional, so this is a parse error.
        final String rule = """
                package org.drools.drlx.parser;

                import org.drools.drlx.domain.Person;
                import org.drools.drlx.ruleunit.MyUnit;

                unit MyUnit;

                rule EmptyNot {
                    not(),
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
    -Dtest='NotTest#notEmpty_failsParse' 2>&1 | tail -10
```

Expected: passes — the grammar rejects `not()` because the first `oopathExpression` is mandatory.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): empty not() rejected at parse time

notEmpty_failsParse — not() with zero inner oopaths is a parse
error; the grammar requires at least one oopathExpression inside
the parens.

Refs #8
EOF
)"
```

---

## Task 5: Delete obsolete `notMultiElementForm_failsParse`

**Files:**
- Modify: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java`

This test asserted the parser **rejects** `not(/a, /b)`. The grammar now accepts it, so the assertion is wrong. Delete it outright — the new `notMultiElement_crossProduct` test covers the happy-path semantics.

- [ ] **Step 1: Delete the method**

Remove the entire `notMultiElementForm_failsParse` test method from `NotTest.java`. Also delete the blank line above it to avoid accidental double-blanks.

- [ ] **Step 2: Run the full suite — must be all GREEN**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am install 2>&1 | grep -E "Tests run:|BUILD" | tail -15
```

Expected: `BUILD SUCCESS`. Per-class counts:
- `NotTest`: **7** (was 5; −1 deleted + 3 new = 5 − 1 + 3 = 7). Test names:
  - `notSuppressesMatch` (unchanged)
  - `notWithOuterBinding` (unchanged)
  - `protoRoundTrip_withNot` (unchanged)
  - `notWithInnerBinding_failsParse` (unchanged)
  - `notParens_singleElement` ← Task 3
  - `notMultiElement_crossProduct` ← Task 1
  - `notEmpty_failsParse` ← Task 4
  - *(deleted: `notMultiElementForm_failsParse`)*
- Everything else unchanged from the last session's post-landing state.
- Grand total default-test run: **63** (was 61 + 3 new − 1 deleted = 63). No-persist run: 5.

- [ ] **Step 3: Commit**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/NotTest.java

git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
test(not): drop obsolete notMultiElementForm_failsParse

The test asserted the parser rejects not(/a, /b); the grammar now
accepts that form. notMultiElement_crossProduct covers the
happy-path semantics.

Refs #8
EOF
)"
```

---

## Task 6: Final verification + issue update

- [ ] **Step 1: Full install + test suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml \
    -pl drlx-parser-core -am install 2>&1 | tail -25
```

Expected: `BUILD SUCCESS`, grand total **68 tests passing** (default-test 63 + no-persist 5), 0 failures.

- [ ] **Step 2: Git log sanity check**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser log --oneline origin/main..HEAD | head -10
```

Expected: 5 new commits since the last push, starting with `test(not): failing happy-path for multi-element not (RED)` and ending with `test(not): drop obsolete notMultiElementForm_failsParse`. All trailing `Refs #8`.

- [ ] **Step 3: Update #8 and close**

Multi-element is the last remaining scope under #8. Once this lands, #8 can close.

```bash
gh issue comment 8 --repo tkobayas/drlx-parser --body "Multi-element not(/a, /b) landed. Grammar extension + visitor loop + 3 new tests. Spec Decision #1 (single-in-parens accepted), Decision #4 (no inner bindings) preserved. Full suite: 68 tests passing."
gh issue close 8 --repo tkobayas/drlx-parser
```

- [ ] **Step 4: HANDOFF update**

Update `/home/tkobayas/claude/public/drlx-parser/HANDOFF.md` via the `handover` skill or by hand. Key content: commit range for this landing, test count (68), #8 closed, next candidate feature (likely #9 `exists` — same infra, can ship both bare + paren in one landing).

---

## Appendix — notes for the implementer

- **Why atomic grammar + visitor in Task 2:** ANTLR's `NotElementContext.oopathExpression()` return type flips from singular to `List<>` when the production references the rule more than once. Splitting grammar from visitor would leave the Java code uncompilable between commits.
- **`notMultiElement_crossProduct` semantics:** Drools' `newNotInstance()` with two pattern children implements NOT(exists A AND exists B). A match for either side alone doesn't satisfy the AND, so NOT stays satisfied and the rule fires. Both sides matching breaks the NOT and suppresses firing.
- **Why `*` not `+` in `(',' oopathExpression)*`:** the `*` lets `not(/a)` match the paren form with zero additional oopaths. Using `+` would force `not(/a, /b)` as the minimum for the paren form, losing Decision #1's single-in-parens acceptance.
- **No proto regen needed.** `GroupElementProto.repeated LhsItem children` already handles N >= 1 children. Verify with `grep -A 3 "message GroupElementProto" drlx-parser-core/src/main/proto/drlx_rule_ast.proto` only if the runtime fails to serialize.
- **Frozen visitor untouched.** `DrlxToJavaParserVisitor.visitNotElement` already throws regardless of parens. The tolerant variant also already works — it dispatches to `ctx.oopathExpression(0)` which still resolves to the first child (now of the multi-valued list). Decision #6 in the spec.
- **#9 `exists` can reuse buildNotElement shape.** The same pattern — bare + paren labelled alternatives, iterate children, emit `GroupElementIR(EXISTS, [...])` — drops in with minimal changes once `Kind.EXISTS` is exposed.
