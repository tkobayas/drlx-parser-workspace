# DRLX Rule Annotations — Design Spec

**Date:** 2026-04-20
**Status:** Approved — ready for implementation plan
**Scope:** First-pass rule-level annotation support in DRLX (`@Salience`, `@Description`).
**References:**
- `docs/DRLXXXX.md` §"Rule Attributes (Annotations)" (line 317)
- Positional syntax precedent: `specs/2026-04-17-positional-syntax-design.md`, `plans/2026-04-17-positional-syntax-implementation.md`
- `specs/IMPLEMENT_SYNTAX_CANDIDATES.md` (Tier 1, item #2 — previously on hold)

## Background

DRLXXXX.md §"Rule Attributes (Annotations)" specifies that rule attributes are expressed as annotations on `rule`:

```
@Salience(10)
rule CheckAge1 {
    Person p : /persons[ age > 18 ],
    do { System.out.println(p); }
}
```

The spec also says annotations "Need to be imported" and explicitly notes "Supported attributes are to be determined." This feature locks in the first supported set and end-to-end implementation path.

## Decisions (locked)

| # | Decision |
|---|----------|
| 1 | **Supported set:** exactly `@Salience(int)` and `@Description(String)`. Unknown annotations fail loud. Rationale: matches what DRLXXXX.md actually names; easy to extend later. |
| 2 | **Real annotation classes exist** in a new package `org.drools.drlx.annotations` inside `drlx-parser-core`. Mirrors how positional uses the real `org.kie.api.definition.type.Position`. |
| 3 | **Argument form is a literal only.** `@Salience(10)` accepts int literals (positive or negative). `@Description("…")` accepts string literals. Expressions like `@Salience(p.age + 10)` are rejected. |
| 4 | **Duplicates fail loud.** Two `@Salience` on one rule throws with line:col. Consistent with positional `@Position` collision handling. |
| 5 | **Strict import resolution.** `@Salience(10)` requires `import org.drools.drlx.annotations.Salience;` or fully-qualified `@org.drools.drlx.annotations.Salience(10)`. Matches Java semantics and the DRLXXXX.md "Need to be imported" wording. |
| 6 | **Symmetric visitor treatment:** canonical `DrlxToJavaParserVisitor` throws on rule-level annotations; `TolerantDrlxToJavaParserVisitor` (in `drlx-lsp`) silent-drops them. Same split established by positional. |
| 7 | **Test scope:** 4 positive + 5 negative = 9 tests in a new `RuleAnnotationsTest` class under the `syntax/` split. |

## Annotation class definitions

Package `org.drools.drlx.annotations` (new):

```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Salience {
    int value();
}
```

```java
package org.drools.drlx.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Description {
    String value();
}
```

`@Target(TYPE)` is chosen because `rule` is class-like in DRLX. The annotations are never reflected on at runtime — they exist purely as import-resolution targets for the visitor.

## Grammar change (`DrlxParser.g4`)

```antlr
ruleDeclaration
    : annotation* RULE identifier '{' ruleBody '}'
    ;
```

Just `annotation*` added as a prefix. The `annotation` rule is already inherited from `JavaParser.g4` and handles `@Foo`, `@Foo(10)`, `@Foo(value = 10)`, and FQN forms with no new lexer rules required.

The throwing ANTLR error listener on `DrlxRuleBuilder.parseToRuleAst` (added during positional work) already surfaces grammar errors with line:col.

## IR change (`DrlxRuleAstModel.java`)

```java
public record RuleIR(String name,
                     List<RuleAnnotationIR> annotations,
                     List<RuleItemIR> items) {
}

public record RuleAnnotationIR(Kind kind, String rawValue) {
    public enum Kind { SALIENCE, DESCRIPTION }
}
```

Typed enum (not `Map<String,String>`) chosen because the supported set is closed; the visitor resolves the enum once and the runtime builder never redoes name matching. `rawValue` is the string form of the literal (e.g., `"10"` or `Checks adult age` — unquoted after visitor parsing).

## Proto change (`drlx_rule_ast.proto`)

Field 3 added to `RuleParseResult`; two new messages/enum:

```proto
message RuleParseResult {
  string name = 1;
  repeated RuleItemParseResult items = 2;
  repeated RuleAnnotationParseResult annotations = 3;   // NEW
}

message RuleAnnotationParseResult {                     // NEW
  AnnotationKind kind = 1;
  string raw_value = 2;
}

enum AnnotationKind {                                   // NEW
  ANNOTATION_KIND_UNSPECIFIED = 0;
  ANNOTATION_KIND_SALIENCE = 1;
  ANNOTATION_KIND_DESCRIPTION = 2;
}
```

- Field 3 is a forward-compatible addition; existing cached payloads round-trip with empty `annotations`.
- `_UNSPECIFIED = 0` leaves room for future kinds without renumbering.
- Regenerate with bundled `protoc-25.5` (same protoc constraint as positional — 27.1 emits symbols incompatible with the pinned protobuf-java 3.25.5).

## Visitor behavior

### Canonical — `DrlxToRuleAstVisitor`

`buildRule` gets a new first step:

```java
private RuleIR buildRule(DrlxParser.RuleDeclarationContext ctx) {
    String name = ctx.identifier().getText();
    List<RuleAnnotationIR> annotations = buildRuleAnnotations(ctx);   // NEW
    List<RuleItemIR> items = new ArrayList<>();
    if (ctx.ruleBody() != null) {
        ctx.ruleBody().ruleItem().forEach(itemCtx -> items.add(buildItem(itemCtx)));
    }
    return new RuleIR(name, annotations, List.copyOf(items));
}
```

`buildRuleAnnotations` logic:

1. Iterate `ctx.annotation()`.
2. For each:
   - Extract `qualifiedName` from the AST node (handles both `@Salience` and `@org.drools.drlx.annotations.Salience` uniformly).
   - **Resolve to `Kind`:**
     - If the name is fully-qualified: it must equal `org.drools.drlx.annotations.Salience` or `.Description`, else throw *"unsupported DRLX rule annotation '@X' at L:C — supported: @Salience, @Description"*.
     - If the name is a simple identifier: look it up in the compilation unit's `importDeclaration*` list. The import FQN must match one of the two supported types; no match → throw *"unresolved annotation '@X' at L:C — missing import?"*.
3. **Extract `rawValue`:**
   - `SALIENCE`: single `elementValue` must be an int literal, optionally prefixed by a single unary `-` (so `@Salience(10)` and `@Salience(-5)` both work). Any other form — string literal, identifier, binary expression, method call, chained unary, etc. — throws *"@Salience expects int literal, got '…' at L:C"*. Stored as the canonicalised integer string (e.g., `"-5"`).
   - `DESCRIPTION`: single `elementValue` must be a string literal. Non-literal → throw similarly. Stored with surrounding quotes stripped.
4. **Duplicate check:** if two annotations resolve to the same `Kind`, throw *"duplicate @Salience at L:C"*.

**Import map construction:** the compilation unit's `importDeclaration*` is already walked in `buildCompilationUnit(...)`. Build a once-per-unit `Map<String simpleName, String fqn>` and pass it to `buildRule(...)` (or capture as a field on the visitor).

### Canonical java-parser path — `DrlxToJavaParserVisitor` (frozen)

Add a `visitRuleDeclaration` override that throws `RuntimeException` whenever `ctx.annotation()` is non-empty — symmetric with its existing `visitOopathRoot` positional rejection. The existing base-visitor behavior for annotation-free `ruleDeclaration` is preserved via the same "delegate to a `buildXxx` helper" pattern used by positional.

### Tolerant java-parser path — `TolerantDrlxToJavaParserVisitor` (in `drlx-lsp`, cross-repo)

Silent-drop override: process the rule without consuming the annotations (i.e., ignore them as if they were comments). This is a cross-repo follow-up documented in HANDOFF.md — same coordination pattern as positional.

## Runtime builder (`DrlxRuleAstRuntimeBuilder`)

In `buildRule(...)`:

```java
RuleImpl rule = new RuleImpl(parseResult.name());
rule.setResource(rule.getResource());
applyAnnotations(rule, parseResult.annotations());   // NEW

GroupElement ge = GroupElementFactory.newAndInstance();
// ... rest unchanged ...
```

```java
private void applyAnnotations(RuleImpl rule, List<RuleAnnotationIR> annotations) {
    for (RuleAnnotationIR ann : annotations) {
        switch (ann.kind()) {
            case SALIENCE -> rule.setSalience(new SalienceInteger(Integer.parseInt(ann.rawValue())));
            case DESCRIPTION -> rule.addMetaAttribute("Description", ann.rawValue());
        }
    }
}
```

- `SalienceInteger` is `org.drools.base.base.SalienceInteger` (constructor `SalienceInteger(int)` at `drools-base/.../SalienceInteger.java:43`). Already on the classpath via the runtime builder's existing Drools deps.
- `addMetaAttribute(String, Object)` is the standard Drools mechanism for carrying `@Description` metadata; readable via `rule.getMetaData().get("Description")`.
- Empty `annotations` list → no-op (cold-path branch, zero cost, same shape as positional).

## Test plan

New file: `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/RuleAnnotationsTest.java`, extending `DrlxBuilderTestSupport`.

**Positive (4):**

| # | Test | Verifies |
|---|------|----------|
| 1 | `testSalienceAffectsFiringOrder` | Two rules on same fact, `@Salience(10)` vs `@Salience(5)`, higher fires first. Assert order via shared fired-list. |
| 2 | `testDescriptionStoredAsMetadata` | `@Description("Checks adult age")` → `rule.getMetaData().get("Description").equals("Checks adult age")`. |
| 3 | `testSalienceAndDescriptionCombined` | Both on one rule; verifies independent application. |
| 4 | `testFullyQualifiedAnnotationWithoutImport` | `@org.drools.drlx.annotations.Salience(10)` with no import line → applied. |

**Negative (5):**

| # | Test | Expected throw message substring |
|---|------|----------------------------------|
| 5 | `testSalienceWithoutImportFailsLoud` | `"unresolved annotation"` + line:col |
| 6 | `testUnsupportedAnnotationFailsLoud` | `"unsupported DRLX rule annotation"` (e.g., `@NoLoop`) |
| 7 | `testDuplicateSalienceFailsLoud` | `"duplicate @Salience"` |
| 8 | `testSalienceNonIntArgumentFailsLoud` | `@Salience("ten")` → `"expected int literal"` |
| 9 | `testSalienceExpressionArgumentFailsLoud` | `@Salience(p.age + 10)` → `"expected int literal"` |

Test count: 40 → 49.

## Architecture (after implementation)

```
DRLX source
  └─> ANTLR (DrlxParser.g4)              ← ruleDeclaration now accepts annotation*
        └─> DrlxToRuleAstVisitor         ← resolves imports, validates literals, enforces duplicates
              └─> RuleIR(.., annotations, items)
                    ├─> DrlxRuleAstParseResult (proto round-trip — new field 3)
                    └─> DrlxRuleAstRuntimeBuilder
                          └─> applyAnnotations()
                                → SALIENCE     → RuleImpl.setSalience(SalienceInteger)
                                → DESCRIPTION  → RuleImpl.addMetaAttribute("Description", ...)
```

Non-canonical:
- `DrlxToJavaParserVisitor` (frozen) — `visitRuleDeclaration` throws when annotations present.
- `TolerantDrlxToJavaParserVisitor` — cross-repo follow-up in `drlx-lsp` to silent-drop annotations.

## Out of scope (deliberately)

- `@NoLoop`, `@AgendaGroup`, `@RuleflowGroup`, `@LockOnActive`, `@AutoFocus`, `@Enabled`, `@DateEffective`, `@DateExpires`, `@Timer`, `@Duration`, `@Dialect` — classic DRL7 attributes not named in DRLXXXX.md. Easy future extensions once the spec designer commits.
- `@ExistenceDriven` — unit-level (DRLXXXX.md line 402), not rule-level. Separate feature.
- `@Tabled`, `@DataSource(...)` — query-level. Will land with query syntax.
- Expression-based salience (`@Salience("p.age + 10")` or `@Salience(p.age + 10)`) — explicitly rejected in this pass.
- `@Repeatable` semantics — we treat duplicates as an error, not as a list.

## Cross-repo follow-up

`drlx-lsp/.../TolerantDrlxToJavaParserVisitor.java` — add `visitRuleDeclaration` override that silent-drops `ctx.annotation()`, so LSP completion doesn't break when users type `@Salience`. Documented in HANDOFF.md at session end (same pattern positional used).
