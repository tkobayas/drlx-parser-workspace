# Positional Syntax Implementation Plan

> **Revision note (Codex review applied):**
> - §2b added: positional restricted to the first/root oopath chunk (grammar split preferred).
> - §3 rewritten: `@Position` lookup precomputes a hierarchy-wide map and fails on duplicates/inherited collisions (was "first wins").
> - §8 synthesizes `fieldName == (argExpr)` with defensive parens (was unparenthesized).
> - §8 ordering justified as source-order preservation (removed speculative alpha-node sharing claim).
> - §8b added: `DrlxToJavaParserVisitor` and `TolerantDrlxToJavaParserVisitor` also traverse `oopathChunk` — decide how to handle. (`DrlxToDescrVisitor` has since been removed as no longer needed.)
> - §10 expanded: negative/diagnostic tests promoted from optional to required.
> - §11.5 clarified: `var postCode` out-binding is actively unsupported by the grammar (uses `expression`, not `drlxExpression`), not merely deferred.

## 1. Current pipeline state (confirmed)

`DrlxToRuleImplVisitor` **no longer exists** in the tree. `drlx-parser-core/src/main/java/org/drools/drlx/builder/` contains only `DrlxToRuleAstVisitor` (ANTLR → IR) and `DrlxRuleAstRuntimeBuilder` (IR → `RuleImpl`), with `DrlxLambdaCompiler` held by composition. The canonical pipeline is:

```
ANTLR parse
  └─> DrlxToRuleAstVisitor           (single ANTLR-touching step)
        └─> DrlxRuleAstModel records (in-memory IR)
              ├─> DrlxRuleAstParseResult  (optional proto cache on disk)
              └─> DrlxRuleAstRuntimeBuilder ── uses ──> DrlxLambdaCompiler ──> RuleImpl
```

Consequence for this plan: there is exactly ONE visitor/proto/runtime set to change, not two. The structure of the work mirrors the inline-cast commit (`0ce0b60`) but touches `DrlxToRuleAstVisitor` instead of the now-deleted `DrlxToRuleImplVisitor`.

## 2. Grammar changes

**File:** `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`

Current rule (line 75-77):
```
oopathChunk
    : identifier (HASH identifier)? ('[' drlxExpression (',' drlxExpression)* ']')?
    ;
```

Proposed change — insert an optional positional block `(...)` **after** the optional `#cast` and **before** the `[...]` slotted block:

```
oopathChunk
    : identifier (HASH identifier)?
      ('(' expression (',' expression)* ')')?
      ('[' drlxExpression (',' drlxExpression)* ']')?
    ;
```

Rationale / decisions:

- **Grammar position order matches spec line 507** — "They may be mixed, `()` must come first." In `oopathChunk` the cast segment is first, positional second, slotted third.
- **Use `expression` (not `drlxExpression`) inside `()`**. Positional args are values/identifiers (e.g. `"paris"`, `var postCode`), not `name : value` bindings. Bindings in positional slots are a later feature (spec line 504 `var postCode`); for this first cut we accept the MVEL `expression` rule, which is rich enough (string literals, numeric literals, identifiers, method calls, etc).
- **Ambiguity check**: `()` is also used for method calls and cast expressions inside the `[...]` body, but those are parsed by the `expression` rule *inside* `drlxExpression`, not at `oopathChunk` level. The new `(expression (',' expression)*)?` sits between the `identifier` (entry-point name) and `[`, so there is no parser ambiguity: the token immediately following the entry-point identifier is either `#`, `(`, `[`, `/` (next chunk), or end-of-chunk. `(` unambiguously starts the positional block because nothing else in `oopathChunk` starts with `(` at that spot.
- **Empty positional `()`** is not allowed by this rule (at least one `expression` required). That mirrors Drools DRL practice — an empty positional block is meaningless.

One thing the implementer must verify: the ANTLR-generated `DrlxParser.OopathChunkContext` will gain an `expression()` accessor. Confirm the generated accessor name (likely `expression()` returning `List<ExpressionContext>`) after regenerating the parser.

### Decision: positional only on the FIRST chunk (Codex review)

The grammar above allows `(...)` on every `oopathChunk`, but positional args apply to the pattern type, not to path navigation. If we accept this grammar as-is and extract only from the first chunk (§4), later-chunk positional like `/a/b("x")` parses but is silently dropped.

**Fix:** split the grammar into a root oopath rule and a navigation-chunk rule, so positional is only grammatically valid on the root:

```
oopathExpression
    : oopathRoot ('/' oopathChunk)*
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

Alternatively, keep the single `oopathChunk` rule and add a visitor-level validation that rejects positional on non-root chunks with a clear error. Pick whichever is less intrusive on the existing visitor structure; the grammar split is cleaner.

## 3. Field-by-index resolution — recommendation: `@Position` annotation

Three candidates considered:

| Option | Pros | Cons |
|---|---|---|
| (a) `@Position(n)` on fields | Matches Drools 7.x semantics (`ConstraintExpression.getFieldAtPosition`, confirmed at `drools/drools-model/drools-model-codegen/src/main/java/org/drools/model/codegen/execmodel/generator/drlxparse/ConstraintExpression.java:81-94`). Matches DRLXXXX.md line 978 which explicitly shows `@Position(0)` and `@Position(1)`. Explicit — position order is immune to field reordering. Inherited-safe: recurses into superclass. | POJO authors must annotate. Reflection on every pattern build (one-time; cache it). |
| (b) Declared-field order | Zero annotation burden | Fragile — reordering fields silently breaks rules. `Class.getDeclaredFields()` order is JVM-implementation-defined (Javadoc says "not sorted"). Existing test POJOs would work by accident today. |
| (c) Canonical-constructor parameter order | Clean for records | Does not work for regular classes (no canonical ctor). `Person` has multiple ctors. |

**Recommendation: (a) `@Position`**, copying Drools' behavior verbatim. Use `org.kie.api.definition.type.Position` (the same annotation class Drools already uses — file at `drools/kie-api/src/main/java/org/kie/api/definition/type/Position.java`, available on the drlx-parser classpath via the `drools-base`/`kie-api` dependency chain already pulled in).

Lookup algorithm (revised per Codex review — do NOT use "first wins"):

```
precompute Map<Integer, Field> for the entire class hierarchy:
  for each class C from patternClass up to (but excluding) Object:
    for each Field f in C.getDeclaredFields():
      if f has @Position(n):
        if map already contains n:
          throw "Duplicate @Position(n) on class X: field1=..., field2=..."
        map.put(n, f)

at resolution time: map.get(i) → fieldName; if null, throw
  "Unable to find @Position(i) field for class X".
```

This rejects two silent-failure cases the first-wins approach hides:
- **Duplicate in one class**: two fields with the same `@Position(0)` — treated as a config error, not "first wins via `getDeclaredFields()` order" (which is JVM-implementation-defined).
- **Inherited collision**: superclass `@Position(0)` and subclass `@Position(0)` — treated as a config error. Do NOT silently let subclass win; the DRLXXXX spec doesn't sanction override semantics, so fail loudly.

Gaps are fine: `@Position(0)` and `@Position(2)` can coexist; index `1` errors with "not found".

Note: the existing test POJOs (`Person`, `Address`, `Employee`) do **not** have `@Position`. They can stay unchanged — only the new `Location` test POJO needs annotations.

## 4. DrlxToRuleAstVisitor changes

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`

Changes to `buildPattern(RulePatternContext)` (line 77-85):

1. Call a new static `extractPositionalArgs(OopathExpressionContext ctx)` helper that:
   - Looks at the **first** `oopathChunk` (positional applies to the pattern type, not to path navigation chunks — consistent with how cast lives on the first chunk).
   - Returns `chunks.get(0).expression()` mapped through `getText()` into `List<String>`; empty list if none.
2. Pass this list as a new `positionalArgs` field to the extended `PatternIR` constructor.

The helper lives next to the existing `extractCastType` method (line 104-114) and uses the same pattern.

Method signature to add:
```
private List<String> extractPositionalArgs(DrlxParser.OopathExpressionContext ctx)
```

## 5. IR record changes

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java` (line 24)

Extend `PatternIR` to add a `List<String> positionalArgs` component. New signature:

```
public record PatternIR(String typeName,
                        String bindName,
                        String entryPoint,
                        List<String> conditions,
                        String castTypeName,
                        List<String> positionalArgs) implements RuleItemIR {}
```

All `new PatternIR(...)` call sites (3 of them: `DrlxToRuleAstVisitor.buildPattern`, `DrlxRuleAstParseResult.load`) need updating.

## 6. Proto schema

**File:** `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`

Current `PatternParseResult` (line 27-33): next free field number is 6. Add:

```
message PatternParseResult {
  string type_name = 1;
  string bind_name = 2;
  string entry_point = 3;
  repeated string conditions = 4;
  string cast_type_name = 5;
  repeated string positional_args = 6;
}
```

**Encoding justification:** each positional arg is stored as the raw source text of the corresponding `expression` (e.g., `"paris"`, `42`, `someVar`). This matches how `conditions` are already encoded (line 31) — raw expression strings compiled later by MVEL. Keeping the format uniform means:
- Same serialization mechanism.
- Same handling of quoting/escaping (none — it's just a string).
- Index in the `repeated` list equals the positional index (position 0 is `positional_args(0)`, etc.).

After editing the proto, regenerate `DrlxRuleAstProto.java` (the commit `0ce0b60` checks in the generated code — this module appears to do the same).

## 7. DrlxRuleAstParseResult changes

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java`

Two edits, symmetric to the cast-type handling:

1. **`load` method (line 72-81)** — when constructing `PatternIR`, also pass `List.copyOf(pattern.getPositionalArgsList())`.
2. **`toProtoItem` method (line 101-110)** — when building the `PatternParseResult.Builder`, call `p.positionalArgs().forEach(pb::addPositionalArgs)`.

No new methods, no signature changes beyond the record field.

## 8. DrlxRuleAstRuntimeBuilder changes — the core of the work

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`

Edit `buildPattern(PatternIR, TypeResolver, Map<String, BoundVariable>)` (line 83-109).

After the current "conditions" loop (which builds `Constraint`s from `parseResult.conditions()`), insert a sibling loop that converts each positional arg to a constraint expression and then builds it the same way:

Pseudo-flow:
```
Class<?> patternClass = ((ClassObjectType) pattern.getObjectType()).getClassType();
for (int i = 0; i < parseResult.positionalArgs().size(); i++) {
    String argExpr = parseResult.positionalArgs().get(i);
    String fieldName = resolvePositionalField(patternClass, i);
        // helper: walks getDeclaredFields(), matches @Position(value==i),
        // recurses into superclass, throws on miss
    String synthesizedExpression = fieldName + " == (" + argExpr + ")";
    // then treat it identically to a regular slotted condition
    List<BoundVariable> refs = lambdaCompiler.findReferencedBindings(synthesizedExpression, boundVariables);
    Constraint constraint = refs.isEmpty()
            ? lambdaCompiler.createLambdaConstraint(synthesizedExpression, patternClass, declarations)
            : lambdaCompiler.createBetaLambdaConstraint(synthesizedExpression, patternClass, declarations, refs);
    pattern.addConstraint(constraint);
}
```

Key decisions:

- **Synthesize `<fieldName> == (<argExpr>)` as a single expression string** (parentheses around `argExpr` defensively per Codex review — `fieldName == argExpr` is too trusting if `argExpr` contains ternaries, casts, or other operators whose precedence could interact with `==`). Then pipe through the same `createLambdaConstraint` / `createBetaLambdaConstraint` path as slotted conditions. This reuses the MVEL lambda compilation infrastructure with no changes — positional constraints become ordinary alpha/beta constraints indistinguishable from `district == "paris"` once synthesized.
- **`castTypeName` interaction**: when a cast is present, `patternClass` (after line 86's `effectiveTypeName` resolution) is already the cast class, so positional resolution correctly looks at the cast class's `@Position` fields. That's the right behavior — positional applies to the effective type.
- **Positional placed before slotted constraints in the `Pattern`**: semantics are unchanged (AND conjunction in Drools). The reason is **source-order preservation** — in the DRLX surface syntax `(...)[...]`, positional is syntactically first, so constraint order should match. This also keeps lambda numbering deterministic across runs. (The original plan cited "alpha-node sharing" as justification; Codex flagged this as speculative without evidence — dropped.)

Add private helper method `resolvePositionalField(Class<?>, int)` to this class, or — better for reusability — add it as a `public static` on `DrlxLambdaCompiler` alongside `extractDeclarations` (line 51-67). Either location is fine; putting it on `DrlxLambdaCompiler` keeps reflection-on-Class utilities together. **Recommendation**: new static method on `DrlxLambdaCompiler`, cached in a `ConcurrentHashMap<Class<?>, Map<Integer,String>>` analogous to `DECLARATION_CACHE`.

## 8b. Non-canonical visitor blast radius (Codex review)

Grammar changes affect all visitors that traverse `oopathChunk`. After removing `DrlxToDescrVisitor`, the remaining concern is:

- `drlx-parser-core/src/main/java/org/drools/drlx/parser/DrlxToJavaParserVisitor.java` — constructs MVEL3 `OOPathChunk` AST nodes at line 437-449 (`OOPathChunk(null, field, inlineCast, conditions)`). Has no notion of positional args.
- `TolerantDrlxToJavaParserVisitor` (extends the above) — used by drlx-lsp for code completion via the token→AST-node map.

**LSP / tolerant path**: Agent analysis confirmed that drlx-lsp's completion flow at `drlx-lsp/drlx-completion/.../DrlxCompletionHelper.java:99-112` uses only the token→AST-node map for cursor-scope resolution. Missing positional args in the AST does NOT break completion — typing `/locations("pa` and triggering completion inside `(...)` still works because the map only needs correct token-to-previous-node mappings. **Safe to ignore positional in the tolerant visitor's AST output.**

**Non-tolerant path (`DrlxToJavaParserVisitor`)**: feeds Drools exec-model codegen. If this path is still exercised, positional would silently disappear. Decision required:

1. **Update `DrlxToJavaParserVisitor`** to propagate positional to the `OOPathChunk` AST (requires extending MVEL3's `OOPathChunk` class — check `/home/tkobayas/usr/work/mvel3-development/mvel/` for the current fields and whether the MVEL project accepts an extension).
2. **Throw explicitly** when positional is encountered, making the limitation visible rather than silent.
3. **Leave as silent drop** — acceptable only if this visitor is exercised purely for AST generation in tooling that doesn't care about constraint semantics (e.g., formatters, IDE rendering).

Recommendation: option 2 (throw) unless there's confirmation the JavaParser path is actively consumed for rule execution.

## 9. Domain model — new test POJO

**File (new):** `drlx-parser-core/src/test/java/org/drools/drlx/domain/Location.java`

Model the Drools ruleunits `Location.java` (`drools/drools-ruleunits/drools-ruleunits-impl/src/test/java/org/drools/ruleunits/impl/domain/Location.java`) but adapted to the DRLX spec's example (city + district):

Fields to expose (matching DRLX spec examples in lines 275, 499, 509):
- `@Position(0) private String city;` — the positional arg `"paris"` maps here
- `private String district;` — slotted `district == "Belleville"` uses this

Plus a standard constructor `Location(String city, String district)`, getters, setters, `toString()`, and a no-arg constructor for consistency with `Employee`.

Should `district` also be `@Position(1)`? Yes — harmless and lets future tests use two-arg positional. Keep parallel to the Drools precedent.

## 10. Tests

**File:** `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxRuleBuilderTest.java`

Two new tests, directly mirroring `testInlineCast` / `testInlineCastWithoutConstraints` (lines 268-341):

### `testPositionalSyntax`
DRLX:
```
rule MatchParisLocations {
    Location l : /locations("paris"),
    do { System.out.println(l); }
}
```
Inserts three `Location`s into the `locations` entry point: `("paris","Belleville")`, `("paris","Montmartre")`, `("london","Soho")`. Asserts `fireAllRules()` returns **2**.

### `testPositionalAndSlotted`
DRLX:
```
rule MatchParisBelleville {
    Location l : /locations("paris")[ district == "Belleville" ],
    do { System.out.println(l); }
}
```
Same three `Location`s. Asserts `fireAllRules()` returns **1**. This test specifically exercises that positional `city == "paris"` AND slotted `district == "Belleville"` are both applied as alpha constraints.

### `testPositionalSyntaxTwoArgs`
`/locations("paris", "Belleville")` — verifies multi-arg positional (positions 0 and 1) both get mapped. Asserts `1`.

### Negative / diagnostic tests (Codex review — add, don't defer)

Positional is built on reflection + synthesized expressions; happy-path-only tests will miss the most likely failure modes. Required additions:

- `testPositionalMissingAnnotation` — pattern `/plainPojos("x")` against a POJO with no `@Position(n)`; expect a clear error naming the class and the missing position.
- `testPositionalDuplicatePositionFails` — POJO with two fields annotated `@Position(0)`; expect build-time error naming both fields.
- `testPositionalInheritedCollisionFails` — POJO extending another where both declare `@Position(0)`; expect build-time error.
- `testPositionalWithComplexExpression` — `/locations(prefix + "paris")` where `prefix` is a pre-bound var; verifies the parenthesized synthesis handles non-trivial RHS and the beta-constraint path fires.
- `testPositionalOnNonRootChunkFails` (if we chose the grammar-split option in §2b) — `/a/b("x")` should fail to parse; if we chose visitor-validation, it should error with a clear message, not be silently dropped.

## 11. Risks / open questions

1. **Grammar ambiguity with existing `expression` inside `[...]`**: None. `(...)` inside the slotted block is parsed by `expression`, which is a separate context from the `oopathChunk` positional block. The new positional `(...)` sits at chunk level, between the entry-point identifier and `[`. Verified by reading `DrlxParser.g4` lines 70-84.

2. **Ambiguity with `(` in subsequent chunk tokens**: After `oopathChunk` comes either `/` (another chunk), `,` (end-of-pattern), or `]` (we're inside a constraint — but we're not). Not ambiguous.

3. **`this` binding**: Positional constraints synthesize `<fieldName> == <expr>` expressions that the existing MVEL lambda compiler already handles; no `this` scoping concerns.

4. **Inheritance (Employee → Person) and `@Position`**: The Drools reference (`ConstraintExpression.getFieldAtPosition` lines 88-90) recurses into the superclass. Our `resolvePositionalField` must do the same. This means positional args work transparently across inherited `@Position` fields. Note: if a subclass redefines a `@Position(n)` annotation the subclass wins (search walks child first). Current plan follows this.

5. **Beta constraints (var references from prior patterns)**: The plan correctly routes through `findReferencedBindings` → `createBetaLambdaConstraint` if the positional arg references a previously bound variable. Note that the grammar uses `expression` (MVEL) inside `(...)`, so DRLXXXX's `var postCode` out-binding inside positional (spec line 504) is **actively unsupported by the grammar**, not merely deferred — `var postCode` is DRLX-specific binding syntax, not a valid MVEL `expression`. Supporting it would require switching the inner rule to `drlxExpression` and teaching the visitor to extract out-bindings, which is a separate feature. Be explicit about this in the plan and docs.

6. **Multi-arg positional `("paris", "France")`**: **Yes, support it from day one.** The grammar rule (`expression (',' expression)*`) naturally handles it, and the runtime loop over `positionalArgs()` is already indexed. Cost is zero; lack of support later would be surprising.

7. **`@Position` annotation not found on any field at a given index**: Throw a clear `RuntimeException` with message like `"Unable to find @Position(0) field for class org.drools.drlx.domain.Location — positional argument cannot be resolved"`. Make sure the test POJO **has** the annotation.

8. **Field visibility**: `getDeclaredFields()` returns private fields. The synthesized `<fieldName> == <expr>` expression uses the *field name*, which MVEL resolves via either the field or the JavaBean getter. `DrlxLambdaCompiler.extractDeclarations` already uses JavaBean property names from `Introspector.getBeanInfo`. For `Location.city` with a `getCity()`, the property name is `city` — matches the field name. Works. For an annotated field without a corresponding getter, the test POJO would fail at MVEL-compile time. The Drools `Location` exposes `thing` as both a public field AND a getter, so it works both ways; our new `Location` should follow `Person`'s convention (private field + getter/setter).

9. **Proto-cache-staleness with old parse results**: Existing `.pb` caches from before this change lack `positional_args`; proto3 default = empty list, so old caches still round-trip correctly without positional args. Non-breaking.

10. **Whether to emit the positional-synthesized expression in the `conditions` proto field versus a separate `positional_args` field**: I chose a separate field. Rationale: the semantics differ (positional requires field-by-index resolution at RuleAST→RuleImpl time, and must resolve against the effective class which depends on cast type that's part of the same proto record). Inlining would require premature class-resolution during parsing, coupling the IR to the classloader. Keeping `positional_args` as raw argument expressions defers resolution to the runtime builder where `TypeResolver` is available, which mirrors how `cast_type_name` is kept symbolic.

## Implementation step order (for the implementer)

Phases to keep commits bisectable:

1. **Grammar** — edit `DrlxParser.g4`, regenerate ANTLR parser (Maven build), confirm the generated `OopathChunkContext.expression()` accessor.
2. **IR record** — extend `PatternIR` with `positionalArgs`, update all call sites (compile will fail until you hit them all — use that as your checklist).
3. **Visitor** — `DrlxToRuleAstVisitor.buildPattern` populates the new field via `extractPositionalArgs`.
4. **Proto** — edit `drlx_rule_ast.proto`, regenerate `DrlxRuleAstProto.java`, update `DrlxRuleAstParseResult.load` and `toProtoItem`.
5. **Runtime** — add `resolvePositionalField` helper in `DrlxLambdaCompiler`; extend `DrlxRuleAstRuntimeBuilder.buildPattern` to synthesize and build positional constraints.
6. **Test POJO** — add `Location.java`.
7. **Tests** — `testPositionalSyntax`, `testPositionalAndSlotted`; optionally `testPositionalSyntaxTwoArgs`.

Each phase compiles but is semantically inert until phase 5 wires things up; phases 1-4 can be a single commit if preferred, but keeping 5-7 together is a natural milestone.

## Critical Files for Implementation

- `drlx-parser-core/src/main/antlr4/org/drools/drlx/parser/DrlxParser.g4`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxToRuleAstVisitor.java`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java`
- `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleAstModel.java`
- `drlx-parser-core/src/main/proto/drlx_rule_ast.proto`
