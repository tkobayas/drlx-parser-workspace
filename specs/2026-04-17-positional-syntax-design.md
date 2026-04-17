# DRLX Positional Syntax — Design Spec

**Date:** 2026-04-17
**Status:** Approved (brainstorming complete; awaiting user review of this spec before plan)
**Predecessor:** `plans/Positional_PLAN.md` (Codex-reviewed working plan)

## 1. Goal

Add positional pattern syntax to DRLX so that

```drlx
rule MatchParisLocations {
    Location l : /locations("paris"),
    do { System.out.println(l); }
}
```

binds matches by field-position. The first positional argument resolves to the field annotated `@Position(0)` of the pattern type, the second to `@Position(1)`, and so on, then synthesizes ordinary equality constraints. Positional and slotted constraints can coexist:

```drlx
Location l : /locations("paris")[ district == "Belleville" ]
```

This matches Drools 7.x semantics (`ConstraintExpression.getFieldAtPosition`) and DRLXXXX.md §978.

## 2. Scope

In scope:
- Single-arg, multi-arg positional on the **root** of the oopath only.
- `@Position` resolution against the pattern's effective class (after any inline `#cast`), recursing through the class hierarchy.
- Fail-loud diagnostics for missing / duplicate / inherited-collision `@Position` annotations.
- Coexistence with inline cast and slotted `[...]` constraints.
- Positional args may reference previously bound variables (beta path).

Out of scope (deferred, will be called out at appropriate code locations):
- DRLXXXX `var postCode` out-binding inside positional (§504) — actively unsupported by the grammar choice (uses MVEL `expression`, not `drlxExpression`).
- Positional on non-root oopath chunks like `/a/b("x")` — rejected at parse time by the grammar split.
- MVEL3 `OOPathChunk` AST extension to carry positional args (the JavaParser visitor path is frozen for new DRLX syntax).

## 3. Decisions (sign-off captured during brainstorming)

| # | Decision | Choice |
|---|----------|--------|
| 1 | Where to enforce "positional only on the root" | **Grammar split** — new `oopathRoot` rule allows `(...)`; `oopathChunk` does not |
| 2 | `DrlxToJavaParserVisitor` (frozen) handling of positional | Throw `UnsupportedOperationException` in base; `TolerantDrlxToJavaParserVisitor` overrides to silent-drop so LSP completion keeps working |
| 3 | `@Position` collisions (same-class duplicate or sub/superclass collision) | **Fail loud at build time** with both field locations in the error message |
| 4 | Inner grammar rule for positional args | MVEL `expression` (not `drlxExpression`) — out-binding is a separate future feature |
| 5 | Where the `@Position`-resolution helper lives | Static method on `DrlxLambdaCompiler`, cached in `POSITION_CACHE` (analogue of `DECLARATION_CACHE`) |
| 6 | Test POJO `Location` shape | `@Position(0) String city`, `@Position(1) String district`, plus standard getters/setters/toString and a two-arg ctor — mirrors `Person` style |
| 7 | Test scope | Full set: 3 positive + 5 negative tests |

## 4. Architecture

The canonical pipeline is unchanged in structure; one field threads through end-to-end:

```
DRLX source
  └─> ANTLR (DrlxParser.g4)            ← grammar split: new oopathRoot
        └─> DrlxToRuleAstVisitor       ← reads positional from oopathRoot
              └─> PatternIR            ← + List<String> positionalArgs
                    ├─> DrlxRuleAstParseResult (proto round-trip)
                    └─> DrlxRuleAstRuntimeBuilder
                          └─> DrlxLambdaCompiler.resolvePositionalField
                                → synthesizes "field == (argExpr)" constraints
                                → reuses existing alpha/beta lambda paths
```

Non-canonical paths:
- `DrlxToJavaParserVisitor` (frozen, base class) — gains `visitOopathRoot`; throws when positional `(...)` is present.
- `TolerantDrlxToJavaParserVisitor` (LSP) — overrides `visitOopathRoot` to drop positional silently. The LSP's completion uses the token→AST-node map only and does not require positional fidelity in the AST.

## 5. Grammar changes

**File:** `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`

Current (line 70-77):

```antlr
oopathExpression
    : '/' oopathChunk ('/' oopathChunk)*
    ;

oopathChunk
    : identifier (HASH identifier)? ('[' drlxExpression (',' drlxExpression)* ']')?
    ;
```

Change to:

```antlr
oopathExpression
    : '/' oopathRoot ('/' oopathChunk)*
    ;

oopathRoot
    : identifier (HASH identifier)?
      ('(' expression (',' expression)* ')')?
      ('[' drlxExpression (',' drlxExpression)* ']')?
    ;

oopathChunk
    : identifier (HASH identifier)?
      ('[' drlxExpression (',' drlxExpression)* ']')?
    ;
```

Notes:
- Positional block sits between the entry-point identifier and the slotted `[...]` block. Order matches DRLXXXX §507 ("`()` must come first").
- `oopathChunk` is unchanged in behavior; only its callers move from "any chunk" to "non-root chunks only."
- An empty `()` is grammatically rejected by `expression (',' expression)*` (at least one expression required). Mirrors Drools DRL practice.
- Verify after regeneration: `OopathRootContext.expression()` returns `List<ExpressionContext>` (standard ANTLR convention for repeated alternatives).

## 6. IR record changes

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`

Extend `PatternIR` (line 24) with one component:

```java
public record PatternIR(String typeName,
                        String bindName,
                        String entryPoint,
                        List<String> conditions,
                        String castTypeName,
                        List<String> positionalArgs) implements RuleItemIR {}
```

Always non-null; pass `List.of()` when there are no positional args.

Construction sites that compile-fail until updated (use this as the work checklist):
1. `DrlxToRuleAstVisitor.buildPattern` (line 84)
2. `DrlxRuleAstParseResult.load` (line 75)

## 7. Visitor changes — canonical

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

Add a static helper next to `extractCastType` (line 104):

```java
private static List<String> extractPositionalArgs(DrlxParser.OopathExpressionContext ctx) {
    DrlxParser.OopathRootContext root = ctx.oopathRoot();
    if (root == null || root.expression() == null || root.expression().isEmpty()) {
        return List.of();
    }
    return root.expression().stream()
            .map(ParserRuleContext::getText)
            .toList();
}
```

`extractEntryPointFromOopathCtx` and `extractCastType` are also adjusted to read from `oopathRoot()` instead of `oopathChunk(0)` (the root identifier and inline cast moved out of `oopathChunk`).

`buildPattern` now constructs `PatternIR` with the new field:

```java
List<String> positionalArgs = extractPositionalArgs(oopathCtx);
return new PatternIR(typeName, bindName, entryPoint, conditions, castTypeName, positionalArgs);
```

`extractConditions` continues to read from the **last** chunk — for a root-only oopath that's `oopathRoot`; with navigation chunks it's still the final `oopathChunk`. That logic needs a small update: read from `oopathRoot` if there are no navigation chunks, otherwise from the last `oopathChunk`.

## 8. Proto schema

**File:** `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`

Next free field number on `PatternParseResult` is 6:

```proto
message PatternParseResult {
  string type_name = 1;
  string bind_name = 2;
  string entry_point = 3;
  repeated string conditions = 4;
  string cast_type_name = 5;
  repeated string positional_args = 6;
}
```

Each entry stores the raw source text of one `expression` (e.g. `"paris"`, `42`, `prefix + "x"`), the same convention as `conditions`. Resolution is deferred to the runtime builder where `TypeResolver` is available — mirrors how `cast_type_name` stays symbolic.

After editing, regenerate `DrlxRuleAstProto.java` and check it in (the inline-cast commit `0ce0b60` follows this convention):

```bash
/tmp/protoc/bin/protoc --java_out=drlx-parser-core/src/main/java \
    --proto_path=drlx-parser-core/src/main/proto \
    drlx-parser-core/src/main/proto/drlx_rule_ast.proto
```

Backwards compatibility: existing `.pb` caches predate this field; proto3 default for `repeated` is empty list, so old caches round-trip cleanly.

## 9. Persistence — `DrlxRuleAstParseResult`

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`

Two symmetric edits:

- `load` (line 75): pass `List.copyOf(pattern.getPositionalArgsList())` as the new `PatternIR` argument.
- `toProtoItem` (line 102): `p.positionalArgs().forEach(pb::addPositionalArgs)`.

## 10. Resolution — `@Position` lookup

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java`

New static method, sibling to `extractDeclarations` (line 51) and `DECLARATION_CACHE` (line 49):

```java
private static final ConcurrentHashMap<Class<?>, Map<Integer, String>> POSITION_CACHE =
        new ConcurrentHashMap<>();

public static String resolvePositionalField(Class<?> patternType, int index) {
    Map<Integer, String> map = POSITION_CACHE.computeIfAbsent(patternType,
            DrlxLambdaCompiler::buildPositionMap);
    String name = map.get(index);
    if (name == null) {
        throw new RuntimeException(
            "Unable to find @Position(" + index + ") field for class " + patternType.getName());
    }
    return name;
}

private static Map<Integer, String> buildPositionMap(Class<?> clz) {
    Map<Integer, String> map = new HashMap<>();
    Map<Integer, String> ownerByPosition = new HashMap<>();   // for diagnostics
    for (Class<?> c = clz; c != null && c != Object.class; c = c.getSuperclass()) {
        for (Field f : c.getDeclaredFields()) {
            Position pos = f.getAnnotation(Position.class);
            if (pos == null) continue;
            int n = pos.value();
            if (map.containsKey(n)) {
                throw new RuntimeException(
                    "Duplicate @Position(" + n + ") on " + clz.getName()
                    + ": " + ownerByPosition.get(n) + "#" + map.get(n)
                    + " and " + c.getName() + "#" + f.getName());
            }
            map.put(n, f.getName());
            ownerByPosition.put(n, c.getName());
        }
    }
    return map;
}
```

`Position` is `org.kie.api.definition.type.Position` — already on the classpath via `drools-base`/`kie-api`.

Behavior:
- Walks subclass first, then superclass, up to but excluding `Object`.
- Same-class duplicate (`@Position(0)` on two fields of one class) → error.
- Inherited collision (`@Position(0)` on subclass and superclass) → error.
- Gaps are allowed: `@Position(0)` and `@Position(2)` coexist; `index 1` errors with "not found".
- Cache stores immutable maps after the first build per class. Cache poisoning on error is impossible because the error throws before `map.put` completes the first call (the `computeIfAbsent` lambda exits via exception and no entry is stored — confirm in implementation, but standard `ConcurrentHashMap` semantics).

## 11. Runtime synthesis — `DrlxRuleAstRuntimeBuilder.buildPattern`

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

Edit `buildPattern` (line 83). After the existing `declarations` extraction (line 99) and **before** the `conditions` loop (line 100), insert a positional loop:

```java
for (int i = 0; i < parseResult.positionalArgs().size(); i++) {
    String argExpr = parseResult.positionalArgs().get(i);
    String fieldName = DrlxLambdaCompiler.resolvePositionalField(patternClass, i);
    String synthesized = fieldName + " == (" + argExpr + ")";    // defensive parens
    List<BoundVariable> refs = lambdaCompiler.findReferencedBindings(synthesized, boundVariables);
    Constraint constraint = refs.isEmpty()
            ? lambdaCompiler.createLambdaConstraint(synthesized, patternClass, declarations)
            : lambdaCompiler.createBetaLambdaConstraint(synthesized, patternClass, declarations, refs);
    pattern.addConstraint(constraint);
}
```

Key points:
- **Synthesized form** is `<fieldName> == (<argExpr>)` — defensive parens around `argExpr` so future MVEL operator-precedence quirks (ternaries, casts, arithmetic) cannot reinterpret the equality.
- **Ordering**: positional constraints are added before slotted constraints in the `Pattern`, matching the syntactic order `(...)[...]`. Keeps lambda numbering deterministic. AND semantics make this functionally irrelevant; we choose it for source-order preservation, not optimization.
- **Cast interaction**: `patternClass` is already the effective type (line 86's `castTypeName != null ? castTypeName : typeName`), so positional resolves against the cast class. Correct.
- **Beta path**: a positional arg referencing a previously bound variable (e.g. `prefix + "paris"` where `prefix` is bound) routes through `createBetaLambdaConstraint` automatically via the existing `findReferencedBindings` check. No additional work.

## 12. Non-canonical visitor changes

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java`

`visitOopathExpression` (line 426) currently iterates `ctx.oopathChunk()`. With the grammar split, the iteration becomes: visit the single `oopathRoot` first, then iterate the (now navigation-only) `oopathChunk` list.

Refactor: extract the body of `visitOopathChunk` (line 447) into a private `buildOOPathChunk(SimpleName field, SimpleName inlineCast, NodeList<DrlxExpression> conditions)` helper that handles `OOPathChunk` construction and parent-relationship wiring. Then:

```java
@Override
public Node visitOopathRoot(DrlxParser.OopathRootContext ctx) {
    if (ctx.expression() != null && !ctx.expression().isEmpty()) {
        throw new UnsupportedOperationException(
            "Positional syntax is not supported in DrlxToJavaParserVisitor — "
            + "use DrlxToRuleAstVisitor for DRLX→RuleImpl. "
            + "Note: this visitor is frozen for new DRLX syntax.");
    }
    SimpleName field = new SimpleName(ctx.identifier(0).getText());
    SimpleName inlineCast = ctx.identifier().size() > 1
            ? new SimpleName(ctx.identifier(1).getText()) : null;
    NodeList<DrlxExpression> conditions = new NodeList<>();
    if (ctx.drlxExpression() != null) {
        for (DrlxParser.DrlxExpressionContext drlxCtx : ctx.drlxExpression()) {
            conditions.add((DrlxExpression) visit(drlxCtx));
        }
    }
    return buildOOPathChunk(field, inlineCast, conditions);
}
```

`visitOopathExpression` (line 426) also updates: visit the single `oopathRoot` first, then iterate `oopathChunk` for navigation chunks.

**File:** `drlx-lsp` repo, `TolerantDrlxToJavaParserVisitor` (read-only reference path here; the actual edit lives in `drlx-lsp`):

```java
@Override
public Node visitOopathRoot(DrlxParser.OopathRootContext ctx) {
    // LSP completion only needs the token→AST-node map; positional args are
    // not represented in MVEL3 OOPathChunk and are silently dropped.
    // Body mirrors DrlxToJavaParserVisitor.visitOopathRoot's non-throw path:
    // identifier(0) → field, identifier(1)? → inlineCast, drlxExpression* → conditions,
    // then buildOOPathChunk(...). The (...) positional block is ignored.
    SimpleName field = new SimpleName(ctx.identifier(0).getText());
    SimpleName inlineCast = ctx.identifier().size() > 1
            ? new SimpleName(ctx.identifier(1).getText()) : null;
    NodeList<DrlxExpression> conditions = new NodeList<>();
    if (ctx.drlxExpression() != null) {
        for (DrlxParser.DrlxExpressionContext drlxCtx : ctx.drlxExpression()) {
            conditions.add((DrlxExpression) visit(drlxCtx));
        }
    }
    return buildOOPathChunk(field, inlineCast, conditions);
}
```

This requires a coordinated change in `drlx-lsp` — see §17 for sequencing constraints.

## 13. Test fixtures

All under `drlx-parser-core/src/test/java/org/drools/drlx/domain/`:

| File | Purpose |
|------|---------|
| `Location.java` | Happy-path POJO: `@Position(0) String city`, `@Position(1) String district`, getters/setters, `toString`, two-arg ctor, no-arg ctor (mirrors `Person`). |
| `PlainLocation.java` | Negative: same shape as `Location` minus the `@Position` annotations. |
| `DuplicatePositionLocation.java` | Negative: two fields both annotated `@Position(0)`. |
| `BasePositioned.java` | Negative inheritance fixture: declares `@Position(0) String foo`. |
| `ChildPositioned.java` extends `BasePositioned` | Negative inheritance fixture: declares `@Position(0) String bar`. |

Following `Person`'s convention: private fields with public getters/setters, no public field exposure.

## 14. Tests

**File:** `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleBuilderTest.java`

Pattern after `testInlineCast` (line 278). Each test builds a `DrlxRuleBuilder`, calls `build(rule)`, opens a session, inserts fixtures, asserts `fireAllRules()` count.

Positive (3):

| Test | DRLX (excerpt) | Inserted | Expected fires |
|------|----------------|----------|----------------|
| `testPositionalSyntax` | `Location l : /locations("paris")` | `("paris","Belleville")`, `("paris","Montmartre")`, `("london","Soho")` | 2 |
| `testPositionalAndSlotted` | `Location l : /locations("paris")[ district == "Belleville" ]` | same 3 | 1 |
| `testPositionalSyntaxTwoArgs` | `Location l : /locations("paris", "Belleville")` | same 3 | 1 |

Negative (5) — assert `assertThatThrownBy(() -> builder.build(rule)).isInstanceOf(RuntimeException.class).hasMessageContaining(...)`:

| Test | Trigger | Expected error fragment |
|------|---------|--------------------------|
| `testPositionalMissingAnnotation` | `/plainLocations("x")` against `PlainLocation` | "Unable to find @Position(0) field for class ...PlainLocation" |
| `testPositionalDuplicatePositionFails` | `DuplicatePositionLocation` referenced as pattern type | "Duplicate @Position(0) on ...DuplicatePositionLocation" + both field names |
| `testPositionalInheritedCollisionFails` | `ChildPositioned` referenced as pattern type | "Duplicate @Position(0) on ...ChildPositioned" + both class+field tokens |
| `testPositionalWithComplexExpression` | `Location l : /locations(prefix + "paris")` after binding `String prefix` to a fact in a prior pattern | Asserts positive: 1 fire (exercises beta path + paren synthesis) |
| `testPositionalOnNonRootChunkFails` | `/locations/sub("x")` | ANTLR parse exception (assert message references non-viable alternative or similar) |

`testPositionalWithComplexExpression` is functionally a positive test of the beta path — included in this list because it specifically exercises a code path that happy-path tests miss (beta + paren-defensive synthesis).

## 15. Implementation phases (commits stay bisectable)

1. **Grammar** — edit `DrlxParser.g4`, regenerate ANTLR parser, confirm `OopathRootContext.expression()` accessor name.
2. **IR record** — extend `PatternIR` with `positionalArgs`. Compile errors at all call sites = checklist.
3. **Visitor** — `DrlxToRuleAstVisitor.buildPattern` populates `positionalArgs` via `extractPositionalArgs`; update `extractEntryPointFromOopathCtx`, `extractCastType`, and `extractConditions` to read from `oopathRoot` where appropriate.
4. **Proto** — edit `drlx_rule_ast.proto`, regenerate `DrlxRuleAstProto.java`, update `DrlxRuleAstParseResult.load`/`toProtoItem`.
5. **Resolution + runtime** — add `resolvePositionalField` + `POSITION_CACHE` to `DrlxLambdaCompiler`. Extend `DrlxRuleAstRuntimeBuilder.buildPattern` with positional loop.
6. **Non-canonical visitors** — update `DrlxToJavaParserVisitor` (visit `oopathRoot`, throw on positional) and coordinate `TolerantDrlxToJavaParserVisitor` override in `drlx-lsp`.
7. **Test fixtures** — add `Location`, `PlainLocation`, `DuplicatePositionLocation`, `BasePositioned`, `ChildPositioned`.
8. **Tests** — add the 8 tests in `DrlxRuleBuilderTest`.

Phases 1-4 are semantically inert; phase 5 wires it up; phases 6-8 round it out. Phases 1-4 can be a single commit if preferred. Phase 6 (LSP) can land separately if the lsp repo move is not synchronous, since the base-class throw is gated by the user actually invoking `DrlxToJavaParserVisitor` — which the LSP path only does via the tolerant subclass.

## 16. Critical files

- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`
- `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
- `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java`
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/Location.java` (new)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/PlainLocation.java` (new)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/DuplicatePositionLocation.java` (new)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/BasePositioned.java` (new)
- `drlx-parser-core/src/test/java/org/drools/drlx/domain/ChildPositioned.java` (new)
- `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleBuilderTest.java`
- `drlx-lsp/.../TolerantDrlxToJavaParserVisitor.java` (coordinated change)

## 17. Risks and open items

- **`DrlxToJavaParserVisitor` freeze enforcement is documentation-only** — Javadoc warns, but nothing mechanically prevents a drive-by edit. Not new in this design.
- **MVEL3 `OOPathChunk` lacks positional fields** — if a future need arises to thread positional through the JavaParser AST path (e.g. for IDE-based rule rendering), MVEL3 would need extension. Not blocking.
- **`drlx-lsp` coordination is required, not optional** — phase 6 touches both `drlx-parser` (base visitor throw) and `drlx-lsp` (tolerant subclass override). The tolerant subclass inherits any new `visitOopathRoot` we add to the base; without the override, the inherited throw fires when an LSP user types positional syntax in any DRLX file. ANTLR-level tolerance does not catch a `RuntimeException` from a visitor method — it would propagate. Sequencing: either land the LSP override first (it can be a no-op against the old grammar, harmless) and then the parser change, or coordinate both in lockstep with version pinning. Document this in the implementation plan as a phase-6 hazard.
- **`computeIfAbsent` exception semantics** — when `buildPositionMap` throws on a duplicate, `ConcurrentHashMap.computeIfAbsent` does not store an entry (per JDK spec: "If the function itself throws an (unchecked) exception, the exception is rethrown, and no mapping is recorded"). The next call repeats the build and throws again — fail-fast for malformed POJOs.

## 18. Reference

- Drools precedent for `@Position`: `drools/drools-model/drools-model-codegen/src/main/java/org/drools/model/codegen/execmodel/generator/drlxparse/ConstraintExpression.java:81-94`
- `@Position` annotation: `drools/kie-api/src/main/java/org/kie/api/definition/type/Position.java`
- Inline-cast precedent commit: `0ce0b60`
- Working plan superseded by this spec: `plans/Positional_PLAN.md`
