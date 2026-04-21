# DRLX GroupElement Infrastructure + `not /pattern` — Design Spec

**Date:** 2026-04-21
**Status:** Approved — ready for implementation plan
**Scope:** Generalize `RuleIR` to a tree shape that supports nested group elements, and land `not /pattern` (single-element form) as the first real consumer of that tree.
**References:**
- `docs/DRLXXXX.md` §"'not' / 'exists'" (line 595-600)
- `specs/IMPLEMENT_SYNTAX_CANDIDATES.md` Tier 2, items #4 and #5
- GitHub issues: parent epic #4; this landing covers #7 (infra) and #8 (`not`); siblings #9 `exists`, #11 `or`/`and`, #12 `if`/`else`, #13 property-reactive
- Precedents:
  - Positional: `specs/2026-04-17-positional-syntax-design.md`
  - Rule annotations: `specs/2026-04-20-rule-annotations-design.md`

## Background

Today `RuleIR.items` is a flat `List<RuleItemIR>` where `RuleItemIR` is a sealed interface permitting `PatternIR` and `ConsequenceIR`. The top-level AND is implicit — the runtime builder wraps all patterns in a single `GroupElementFactory.newAndInstance()`. There is no way to express nested group elements, so `not`, `exists`, `or`, `and`, and `if`/`else` are all blocked on this shape.

DRLXXXX.md §"'not' / 'exists'" (line 599) locks the user-visible syntax:

```
rule R1 {
    not /persons[ age < 18 ],
    do { ... }
}
```

Single-element form is parens-omitted (spec line 597). Multi-element form `not(/a, /b)` is spec'd but deferred to a follow-up issue.

## Decisions (locked)

| # | Decision |
|---|----------|
| 1 | **Scope of this landing:** infrastructure (tree-shape IR + proto + visitor + runtime builder) **plus** `not /pattern` as the first consumer. Option (B) from brainstorming. Rationale: infra-only (A) ships dead code that's hard to review; infra + `not` + `exists` (C) doubles test surface for the first merge. (B) gives the refactor a concrete driver. |
| 2 | **IR shape:** split LHS from RHS explicitly. New sealed interface `LhsItemIR permits PatternIR, GroupElementIR`. `ConsequenceIR` becomes a direct field (`rhs`) on `RuleIR`, no longer a sealed-interface peer. Rationale: the current mixed-list convention ("consequence must be last, exactly once") is unenforced; the type system can enforce it for free. |
| 3 | **Grammar scope for `not` in this landing:** single-element only — `not /pattern`. No paren form `not(…)`; no `exists`. Rationale: YAGNI — single-element exercises the recursive IR/proto just fine; multi-element is purely grammar extension and can ship on its own under a follow-up. |
| 4 | **Bindings inside `not`: forbid, fail loud.** `not var p : /persons[...]` → visitor error. Rationale: for (α) there's no useful case; inner bindings could never escape `not` anyway. Keeps scope-tracking trivial (no scope frames needed yet). |
| 5 | **Frozen visitor symmetry:** `DrlxToJavaParserVisitor` throws on the new `notElement` (same style as positional / rule annotations). `TolerantDrlxToJavaParserVisitor` adds a silent-drop override to preserve LSP completion. |
| 6 | **Proto recursion is unversioned.** Old `.pb` cache files fail to parse against the new schema — acceptable, cache regenerates from source. No migration shim. |
| 7 | **Test scope:** 3 happy + 2 negative + 1 frozen-visitor + 1 LSP tolerance = **7 new tests**. The existing ~50-test suite continues to run against the refactored `RuleIR(lhs, rhs)` and serves as the regression guard for the infra rewire. Expected end state: **57 tests**, all passing. |

## Architecture overview

```
DRLX source
  └─> ANTLR                       ← new `notElement` production
        └─> DrlxToRuleAstVisitor  ← builds GroupElementIR(NOT, [PatternIR])
              └─> RuleIR(name, annotations, lhs: List<LhsItemIR>, rhs: ConsequenceIR)
                    ├─> proto round-trip (recursive LhsItem message)
                    └─> DrlxRuleAstRuntimeBuilder
                          └─> buildLhs(List<LhsItemIR>, GroupElement parent)
                                → PatternIR              → Pattern, add to parent
                                → GroupElementIR(NOT)    → newNotInstance(), recurse, add
```

Non-canonical path: `DrlxToJavaParserVisitor` throws on `notElement` (frozen). `TolerantDrlxToJavaParserVisitor` silent-drops it so LSP completion survives.

## IR / data model changes

`drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`:

```java
public record CompilationUnitIR(String packageName, List<String> imports, List<RuleIR> rules) {}

public record RuleIR(String name,
                     List<RuleAnnotationIR> annotations,
                     List<LhsItemIR> lhs,
                     ConsequenceIR rhs) {}

public sealed interface LhsItemIR permits PatternIR, GroupElementIR {}

public record PatternIR(String typeName,
                        String bindName,
                        String entryPoint,
                        List<String> conditions,
                        String castTypeName,
                        List<String> positionalArgs) implements LhsItemIR {}

public record GroupElementIR(Kind kind, List<LhsItemIR> children) implements LhsItemIR {
    public enum Kind { NOT /* EXISTS, AND, OR later */ }
}

public record ConsequenceIR(String block) {}

public record RuleAnnotationIR(Kind kind, String rawValue) {
    public enum Kind { SALIENCE, DESCRIPTION }
}
```

Changes from current:
- `RuleItemIR` (sealed: `PatternIR | ConsequenceIR`) is **removed**.
- `LhsItemIR` (sealed: `PatternIR | GroupElementIR`) replaces it on the LHS.
- `PatternIR` now implements `LhsItemIR` instead of `RuleItemIR`.
- `ConsequenceIR` stops implementing the sealed interface — direct field `rhs` on `RuleIR`.
- `GroupElementIR.Kind` enum ships with only `NOT`. `EXISTS`, `AND`, `OR` anticipated but not exposed (added in their respective issues).

Call-site impact:
- `DrlxToRuleAstVisitor` stops appending `ConsequenceIR` to a flat list; sets `rhs` directly.
- `DrlxRuleAstRuntimeBuilder` outer loop walks `lhs` instead of `items`; consequence wire-up moves out of the loop into a single `setConsequence(...)` call.
- `RuleIR.items()` → `.lhs()` / `.rhs()` everywhere it's read.

## Grammar change (`DrlxParser.g4`)

```antlr
ruleItem
    : rulePattern
    | notElement
    | ruleConsequence
    ;

// Single-element only for this landing (spec § 'not'/'exists', line 599).
// Multi-element `not(/a, /b)` form deferred to a follow-up issue.
notElement
    : NOT oopathExpression
    ;
```

Lexer (`DrlxLexer.g4` / `Mvel3Lexer.g4`): add `NOT : 'not' ;` as a keyword, symmetric with the existing `RULE`, `UNIT`, `DO` tokens.

Trailing-comma handling stays at the `ruleBody` / `ruleItem` level — comma required between items, optional after last, same as `rulePattern` today. No comma inside `notElement` itself.

Deliberate grammar omissions:
- No paren form — `not(…)` parses as "no viable alternative" in this landing.
- No `var` prefix inside `notElement`'s `oopathExpression` — grammar stays permissive; the visitor enforces with a specific error message.

## Visitor changes

**`DrlxToRuleAstVisitor` (canonical — builds IR):**
- New override: `visitNotElement(NotElementContext ctx)` → returns `GroupElementIR(Kind.NOT, List.of(pattern))` where `pattern` is the `PatternIR` built from the inner `oopathExpression`.
- `visitRuleBody` walks `ruleItem`s, collects `LhsItemIR`s into `lhs`, sets aside `ConsequenceIR` into `rhs`. Duplicate consequences still fail loud (existing behaviour preserved).
- When the inner pattern of `notElement` would produce a `PatternIR` with a non-null `bindName` — that is, the user wrote `var p : /persons[...]` or the older `Type p : /persons[...]` form — throw:
  ```
  @not does not allow bindings — a bound identifier is forbidden inside 'not' (binding would be unreachable outside). Rule: <ruleName>.
  ```

**`DrlxToJavaParserVisitor` (frozen — Java-AST path):**
- New override: `visitNotElement` throws:
  ```
  `not` is not supported in DrlxToJavaParserVisitor — use DrlxToRuleAstVisitor for DRLX→RuleImpl.
  Note: this visitor is frozen for new DRLX syntax.
  ```
  Same message shape as the existing positional and rule-annotations rejections.

**`TolerantDrlxToJavaParserVisitor` (LSP path):**
- New override: `visitNotElement` silent-drops. Returns the inner pattern's JavaParser representation unwrapped (or a neutral placeholder) so the tokenId→AST-node map stays populated on the inner expression — LSP completion inside `not /persons[ age > 18. ]` must keep working as the user types.
- Covered by a new tolerant test (incomplete rule with `not /persons[...]` + unfinished consequence).

## Runtime builder changes

`DrlxRuleAstRuntimeBuilder.buildRule` refactors to split LHS construction from consequence wire-up:

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
        switch (item) {
            case PatternIR patternIr -> {
                Pattern pattern = buildPattern(patternIr, typeResolver, boundVariables);
                parent.addChild(pattern);
                Declaration declaration = pattern.getDeclaration();
                if (declaration != null) {
                    Class<?> patternClass = ((ClassObjectType) pattern.getObjectType()).getClassType();
                    boundVariables.put(declaration.getIdentifier(),
                        new BoundVariable(declaration.getIdentifier(), patternClass, pattern));
                }
            }
            case GroupElementIR group -> {
                GroupElement ge = switch (group.kind()) {
                    case NOT -> GroupElementFactory.newNotInstance();
                };
                buildLhs(group.children(), ge, typeResolver, boundVariables);
                parent.addChild(ge);
            }
        }
    }
}
```

**Binding-scope note for (α):** bindings are forbidden inside `not`, so the recursive `buildLhs` call cannot introduce new entries into `boundVariables` via an inner `PatternIR` with a `bindName`. Outer bindings remain visible (the map is passed in), which is correct — they can still be referenced from constraints inside the NOT pattern, and Drools beta-joins handle that naturally at the NOT group's left input. No scope-frame push/pop is required in this landing.

`findReferencedBindings` stays as-is: flat regex over a constraint string against an accumulated top-level binding set. Scope-aware binding tracking is deferred until nested `var` becomes legal (i.e. under `and`/`or`/multi-element `not`) — revisit with #11.

## Proto changes

`drlx-parser-core/src/main/proto/drlx_rule_ast.proto`:

```proto
message Rule {
    string name = 1;
    reserved 2;                              // old flat `items` field number — retired
    repeated RuleAnnotation annotations = 3; // existing
    repeated LhsItem lhs = 4;                // NEW
    Consequence rhs = 5;                     // NEW — pulled out of items
}

message LhsItem {
    oneof kind {
        Pattern pattern = 1;
        GroupElement group = 2;
    }
}

message GroupElement {
    GroupElementKind kind = 1;
    repeated LhsItem children = 2;           // recursive
}

enum GroupElementKind {
    GE_KIND_UNSPECIFIED = 0;
    NOT = 1;
    // EXISTS = 2; AND = 3; OR = 4;          reserved for follow-ups
}
```

Cache invalidation: `DrlxBuildCacheStrategy.RULE_AST` persists `drlx-rule-ast.pb` files. After this change, existing cached artifacts won't survive — they'll fail to parse or produce incoherent rules. Acceptable: cache regenerates from source on next build, and the compiler is too young to have external persisted caches. No version bump; the schema itself is the breakpoint.

`DrlxRuleAstParseResult.toProto(RuleIR)` / `fromProto(Rule)` updated to walk the tree recursively. `LhsItem` translation is a thin switch.

## Error / negative cases

| Input | Where enforced | Behaviour |
|-------|----------------|-----------|
| `not var p : /persons[...]` or `not Person p : /persons[...]` | `DrlxToRuleAstVisitor.visitNotElement` | Throws `UnsupportedOperationException` with `@not does not allow bindings …` message. |
| `not(/a, /b)` (multi-element) | Grammar | Parse error — `not(` is not in the `notElement` production for this landing. Surfaces as native ANTLR "no viable alternative". |
| `not` with no following expression | Grammar | Parse error — `oopathExpression` required. |
| `exists /pattern` | Grammar | Parse error — `exists` token not yet in any production; reserved for #9. |
| `not` encountered by frozen `DrlxToJavaParserVisitor` | Visitor | Throws with the "visitor is frozen for new DRLX syntax" message style. |

No warning-level behaviour. Nothing silently ignored in the canonical pipeline. Tolerant visitor silent-drops (documented LSP tolerance contract).

## Testing scope

TDD-ordered. Each test lands as its own commit, matching the cadence used for positional and rule-annotations.

| # | Kind | Test | Proves |
|---|------|------|--------|
| 1 | happy | `notSuppressesMatch` | `not /persons[age < 18]` → rule fires when no under-18 present; does not fire when one exists. Core semantic proof of NOT. |
| 2 | happy | `notWithOuterBinding` | `var p : /persons[...], not /orders[customerId == p.id]` → beta-joined NOT; outer binding resolves inside. |
| 3 | happy | `protoRoundTrip_withNot` | Build `RuleIR` with `GroupElementIR(NOT, [PatternIR])`, serialize + deserialize, assert structural equality. |
| 4 | negative | `notWithInnerBinding_failsLoud` | `not var p : /persons[...]` → clear `UnsupportedOperationException` with the spec'd message. |
| 5 | negative | `notMultiElementForm_failsParse` | `not(/a, /b)` → parser error. Documents multi-element is deferred. |
| 6 | frozen-visitor | `javaParserVisitor_throwsOnNot` | `DrlxToJavaParserVisitor` throws on `notElement`. Symmetric with existing positional / rule-annotations tests. |
| 7 | LSP tolerance | `incompleteRule_WithNot` | DRLX unit form with `not /persons[ age > 18. ]` + unfinished consequence through `TolerantDrlxToJavaParserVisitor`; no throw, completion marker reachable on the inner pattern. Mirrors the annotations tolerance test. |

No infra-only tests needed — the ~50-test existing suite runs against the refactored `RuleIR(lhs, rhs)` shape and serves as the regression guard for the IR / runtime-builder / proto rewire.

Expected end state: **50 existing + 7 new = 57 tests**, all passing.

## Non-goals / deferred

Explicitly out of scope for this landing:

- **Multi-element `not(/a, /b)`** — follow-up issue under #8 (can ship independently). Same `GroupElementIR(NOT)` shape; grammar extension only.
- **`exists /pattern`** — #9. Same infra; `Kind.EXISTS` enum slot reserved, not exposed.
- **`and()` / `or()` group CEs** — #11. `Kind.AND` / `Kind.OR` slots anticipated.
- **`if` / `else` in rule body** — #12.
- **Property-reactive watch list** — #13.
- **Scope-aware `findReferencedBindings`** — only needed once `var` becomes legal inside group elements. Revisit with #11.
- **Cross-repo `drlx-lsp` changes** — none. Tolerant override lives in `drlx-parser`; `drlx-lsp` consumes transparently (same as the rule-annotations follow-up this session).
- **Cache migration** — no schema-version bump, no migration shim.
- **`DESIGN.md` sync** — the Parser/IR tables (DESIGN.md lines 180-188) will be updated to mention `LhsItemIR` and `GroupElementIR` as part of the implementation commits via `java-update-design`.
