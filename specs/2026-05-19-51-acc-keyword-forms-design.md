# #51 Accumulate: acc() keyword forms (2-, 3-, and 5-param)

**Issue:** https://github.com/tkobayas/drlx-parser/issues/51
**Epic:** #26
**Date:** 2026-05-19
**DRLXXXX reference:** §Accumulate (lines 842–890)

## Overview

Add the `acc(...)` keyword syntax for accumulate in DRLX. Three arity variants:

- **2-param:** `acc(source, functions)` — syntactic sugar over existing function-call accumulate
- **3-param:** `acc(source, initVars, action, result)` — custom holder logic with optional `(action, reverse)` pair. The DRLXXXX spec calls this "3 parameters" counting the positions after source (init, action, result); with source included the grammar has 4 positional arguments.
- **5-param:** `acc(source, initVars, action, reverse, result)` — full custom accumulator with explicit reverse. With source included the grammar has 6 positional arguments; the "5" counts the positions after the keyword.

The 2-param form lowers to the existing `AccumulatePatternIR` / `DrlxLambdaAccumulator` path. The 3/5-param forms introduce a new `CustomAccumulateIR` and `DrlxCustomAccumulator` that delegates to MVEL3-compiled evaluators sharing a `Map<String, Object>` context.

## Syntax examples

### 2-param (single function)

```
rule R1 {
    acc(var p : /persons,
        var avgAge = avg(p.age)),
    do { ... }
}
```

### 2-param (grouped functions)

```
rule R1 {
    acc(var p : /persons,
        (var maxAge = max(p.age),
         var minAge = min(p.age))),
    do { ... }
}
```

### 3-param (init + action + result, no reverse)

```
rule R1 {
    acc(var p : /persons,
        int s = 0;,
        s = s + p.age,
        int sum = s),
    do { ... }
}
```

### 3-param with (action, reverse) pair

```
rule R1 {
    acc(var p : /persons,
        int s = 0;,
        (s = s + p.age, s = s - p.age),
        int sum = s),
    do { ... }
}
```

### 5-param (init + action + reverse + result)

```
rule R1 {
    acc(var p : /persons,
        { int count = 0; int total = 0; },
        { total += p.age; count++; },
        { total -= p.age; count--; },
        double avgAge = total / count),
    do { ... }
}
```

## Scope

### In scope

- Grammar: `accKeywordItem` rule in `ruleItem` covering all three arity variants
- IR: `CustomAccumulateIR` and `InitVarIR` records for 3/5-param forms
- Protobuf: `CustomAccumulateParseResult` and `InitVarParseResult` messages in `drlx_rule_ast.proto`; serialization in `DrlxRuleAstParseResult`
- Visitor: dispatch 2-param to existing `AccumulatePatternIR`; 3/5-param to `CustomAccumulateIR`
- Runtime: `DrlxCustomAccumulator` implementing `Accumulator` with MVEL3-compiled action/reverse/result evaluators
- Compilation: `DrlxLambdaCompiler.createCustomAccumulator()` method with batch-compilation support
- Tests: visitor-level, runtime integration, negative

### Out of scope (tracked separately)

- `and(...)` multi-pattern source → #52
- Custom user-imported accumulate functions → #53
- Outer-binding refs in custom accumulate blocks → #54. The visitor rejects any reference to outer-scope bindings in 3/5-param action/reverse/result blocks.
- Multi-result binding (3-param with no explicit result binding). Example from DRLXXXX spec:
  ```
  acc(var p : /persons,
      {int minAge; int maxAge;},
      {minAge = min(minAge, p.age);
       maxAge = max(maxAge, p.age);})
  ```
  This form uses init vars directly as results without an `accResultBinding`. It requires multi-result semantics and does not match the grammar defined here.

## Grammar

New rules added to `DrlxParser.g4`. Note: `boundOopath` requires two identifiers (`identifier identifier (':' | '=') oopathExpression`), so the source binding must include a type prefix (e.g. `var p : /persons`, not bare `p : /persons`).

`acc` is **not** a new lexer token. It is parsed as a contextual keyword: the `accKeywordItem` rule matches an `identifier` whose text equals `"acc"`. This avoids reserving `acc` globally and keeps it available as a variable name elsewhere.

Init variable declarations use `localVariableDeclaration` (from the Java/MVEL3 grammar). The `var` form is **rejected** at the visitor level — init vars require explicit types, since MVEL3 `Declaration.of()` needs a concrete `Class<?>` and type inference from initializers is unreliable for complex expressions. The trailing `;` is part of the grammar rule (from `blockStatement : localVariableDeclaration ';'`), so the dedicated `accInitVar` rule includes it explicitly.

Multi-declarators (e.g. `int a = 0, b = 1;`) are split by the visitor into separate `InitVarIR` entries. Missing initializers (e.g. `int a;`) are mapped to Java type defaults (`0` for int, `0.0` for double, `null` for reference types).

```antlr
ruleItem
    : rulePattern
    | accumulateItem ','
    | accKeywordItem ','          // NEW
    | notElement ','
    | existsElement ','
    | andElement ','
    | orElement ','
    | testElement ','
    | conditionalBranch ','
    | ruleConsequence
    ;

accKeywordItem
    : identifier '(' accSource ',' accBody ')'   // identifier text must be "acc"
    ;

accSource
    : boundOopath
    ;

accBody
    : accFunctionList                                                    // 2-param
    | accInitVars ',' accActionBlock ',' accResultBinding                // 3-param
    | accInitVars ',' accActionBlock ',' accActionBlock ',' accResultBinding  // 5-param
    ;

accFunctionList
    : accumulateItem                                 // single function
    | '(' accumulateItem (',' accumulateItem)* ')'   // grouped
    ;

accInitVars
    : accInitVar                                     // single: int s = 0;
    | '{' accInitVar+ '}'                            // multi: { int count = 0; int total = 0; }
    ;

accInitVar
    : localVariableDeclaration ';'                   // int s = 0; (var rejected at visitor)
    ;

accActionBlock
    : expression                                     // single: s = s + p.age
    | '(' expression ',' expression ')'              // paired (action, reverse)
    | '{' statement+ '}'                             // multi: { total += p.age; count++; }
    ;

accResultBinding
    : typeType identifier '=' expression             // double avgAge = total / count
    ;
```

`accResultBinding` requires an explicit type (`typeType`, not `VAR`). Custom accumulators have no function registry to infer the result type from, so `var` is not permitted for 3/5-param result bindings. (2-param uses the existing `accumulateItem` rule which supports `var` via the function registry.)

### Semantic validation in visitor

After parsing `accKeywordItem`, the visitor checks `identifier.getText().equals("acc")`. If not, the visitor throws a parse error. This keeps `acc` contextual — it is only special at the `ruleItem` level.

## IR (Intermediate Representation)

### Existing (unchanged)

`AccumulatePatternIR` and `AccumulatorIR` remain for bare function calls and 2-param `acc(...)`.

### New records

```java
public record CustomAccumulateIR(
    PatternIR source,              // single pattern source
    List<InitVarIR> initVars,      // declared holder variables
    String actionBlock,            // MVEL3 statement(s)
    String reverseBlock,           // nullable — absent = no reverse support
    String resultTypeName,         // "double", "int" — explicit type required (no "var")
    String resultBindName,         // "avgAge", "sum"
    String resultExpression,       // "total / count", "s"
    List<String> referencedBindings
) implements LhsItemIR {
}

public record InitVarIR(
    String typeName,               // "int", "double"
    String name,                   // "count", "total"
    String initializer             // "0", "new ResultsHolder()"
) {
}
```

`LhsItemIR` sealed interface adds `CustomAccumulateIR` to its permits list.

`source` is `PatternIR` (not `LhsItemIR`) — multi-pattern `and(...)` source is deferred to #52.

### Type resolution for init vars and result bindings

Both `InitVarIR.typeName` and `CustomAccumulateIR.resultTypeName` store the type as written in source (e.g. `"int"`, `"double"`, `"String"`). The runtime builder resolves them to `Class<?>` via this contract:

1. **Java primitives:** `int` → `Integer.class`, `long` → `Long.class`, `double` → `Double.class`, `float` → `Float.class`, `short` → `Short.class`, `byte` → `Byte.class`, `boolean` → `Boolean.class`, `char` → `Character.class`. Boxed types are used since `Map<String, Object>` cannot hold unboxed primitives.
2. **Fully-qualified names:** resolved via `Class.forName(typeName)`.
3. **Simple names:** resolved via the compilation unit's `TypeResolver` (which incorporates imports). E.g. `String` resolves via `java.lang.*`, and `ResultsHolder` resolves if imported.
4. **Failure:** `"cannot resolve type 'X' in custom accumulate — use a fully-qualified name or add an import"`.

This mirrors the existing `resultClassFor()` logic in `DrlxRuleAstRuntimeBuilder` (lines 542–564) but without the function-registry fallback (which custom accumulators don't have).

## Protobuf persistence

New proto messages in `drlx_rule_ast.proto`:

```protobuf
message LhsItemParseResult {
  oneof kind {
    PatternParseResult pattern = 1;
    GroupElementParseResult group = 2;
    EvalParseResult eval = 3;
    AccumulatePatternParseResult accumulate_pattern = 4;
    CustomAccumulateParseResult custom_accumulate = 5;    // NEW
  }
}

message CustomAccumulateParseResult {
  PatternParseResult source = 1;
  repeated InitVarParseResult init_vars = 2;
  string action_block = 3;
  string reverse_block = 4;           // empty string = absent
  string result_type_name = 5;
  string result_bind_name = 6;
  string result_expression = 7;
  repeated string referenced_bindings = 8;
}

message InitVarParseResult {
  string type_name = 1;
  string name = 2;
  string initializer = 3;
}
```

`DrlxRuleAstParseResult.toProtoLhs()` adds a `CustomAccumulateIR` branch; `fromProtoLhs()` deserializes it. This ensures the prebuild/cache path works for rules using `acc(...)`.

## Visitor

The `DrlxToRuleAstVisitor.buildRule()` dispatch loop adds a branch for `accKeywordItem`:

- **2-param:** extract source as `PatternIR`, extract function list as `List<AccumulatorIR>`, emit `AccumulatePatternIR`.
- **3-param:** extract source, parse `accInitVars` into `List<InitVarIR>`, extract action block text (and optional reverse from paired form), extract result binding, emit `CustomAccumulateIR`.
- **5-param:** same as 3-param but action and reverse are separate `accActionBlock` positions. The visitor rejects the `(expression, expression)` paired form in either the action or reverse position of a 5-param `acc(...)` — paired blocks are only valid in the 3-param form where they combine action and reverse into one position. Error: `"paired (action, reverse) block is not valid in 5-param acc — use separate action and reverse positions"`.

The visitor extracts action/reverse/result as raw text strings — MVEL3 compilation is deferred to runtime-builder time, consistent with consequences and value extractors.

The visitor extracts `InitVarIR` structurally from the `localVariableDeclaration` parse tree: type name, variable name, and initializer expression text. These are stored as data, not as executable code.

**Init var name validation:** The visitor rejects:
- Init var names that equal the source binding name (e.g. `int p = 0` when the source is `var p : /persons`) — this would cause `accumulate()` to overwrite the holder variable when inserting the source fact, and the `finally` cleanup would delete it entirely.
- Duplicate init var names (e.g. `{ int x = 0; int x = 1; }`) — the second declaration would silently overwrite the first.

Both produce clear errors: `"init var 'p' conflicts with source binding name"` and `"duplicate init var name 'x'"`.

**Literal initializer type compatibility:** The visitor validates that each init var's literal is assignment-compatible with its declared type, following Java widening rules:
- Numeric widening: `int s = 0` (ok), `long s = 0` (ok, int → long), `double s = 0` (ok, int → double)
- No narrowing: `int s = 0L` → error (`"cannot assign long literal to int"`)
- Null only for reference types: `int s = null` → error (`"cannot assign null to primitive type int"`)
- Type mismatch: `boolean b = "true"` → error (`"cannot assign String literal to boolean"`)
- Overflow/range: since init literals are parsed programmatically (not by MVEL3), the runtime builder validates ranges for integer types. `int s = 999999999999` → error (`"literal value out of range for int"`). `byte b = 128` → error. Uses `Long.parseLong` then checks target-type bounds before narrowing.

Incompatible combinations are rejected with: `"init var 'name': cannot assign <literal-type> to <declared-type>"`.

**Reference validation by block type:**

- **Action / reverse blocks** may reference the source binding and init vars. Outer-scope bindings (from preceding patterns) are rejected: `"outer-binding reference 'x' in custom accumulate is not yet supported (see #54)"`.
- **Result expression** may reference init vars only. The source binding is not available at `getResult()` time (there is no current source fact). If the result expression references the source binding, the visitor throws: `"result expression cannot reference source binding 'p' — only init vars are available"`.
- **Outer-scope bindings** in any block are rejected as above (#54).

## Runtime builder

### DrlxCustomAccumulator

A single reusable `Accumulator` implementation:

```java
public final class DrlxCustomAccumulator implements Accumulator {

    private Evaluator<Map<String,Object>, Void, ?> actionEval;
    private Evaluator<Map<String,Object>, Void, ?> reverseEval;  // nullable
    private Evaluator<Map<String,Object>, Void, Object> resultEval;
    private final List<InitVarIR> initVars;
    private final String srcBindingName;

    public boolean supportsReverse() { return reverseEval != null; }
}
```

Lifecycle:
- `createContext()` → new `HashMap<String, Object>`
- `init()` → populate map from `InitVarIR` entries: for each init var, `map.put(name, defaultValue)` where `defaultValue` is derived from the parsed initializer (literal evaluation). No MVEL3 evaluator needed — the init vars are structural data, not executable code.
- `accumulate()` → put `handle.getObject()` (the source fact) into map under `srcBindingName`, run `actionEval`, **remove `srcBindingName` from map** (in a `finally` block), **return the source fact**. The returned value is passed back to `tryReverse()` by the Drools engine. Removing the source binding ensures the map only retains init vars between calls.
- `tryReverse()` → if `reverseEval != null`, put the `value` parameter (the source fact returned by `accumulate()`) into map under `srcBindingName`, run `reverseEval`, **remove `srcBindingName` from map** (in a `finally` block), return true; otherwise return false. Using the `value` parameter (not `handle.getObject()`) matches the Drools `Accumulator` contract as implemented in `DrlxLambdaAccumulator`.
- `getResult()` → run `resultEval` on the map, return value. The source binding is not in the map — only init vars are available, enforced by the `accumulate()`/`tryReverse()` cleanup above and consistent with the visitor's reference validation.

**Init var initialization:** For this issue, init var initializers are restricted to **literal values and Java type defaults only**. The visitor validates that each `InitVarIR.initializer` is either:
- A numeric literal (`0`, `0.0`, `0L`, `0.0f`)
- A boolean literal (`true`, `false`)
- A string literal (`"..."`)
- A char literal (`'x'`)
- `null`
- Absent (mapped to Java type default: `0` for int, `0.0` for double, `false` for boolean, `null` for reference types)

Complex initializers like `new ResultsHolder()` are **rejected** with an error: `"custom accumulate init vars must be literals — complex initializers are not yet supported"`. This avoids introducing an ad-hoc compilation path outside the batch/prebuild model. Support for complex initializers can be added later by modelling them as proper `EvaluatorSink` slots.

The map is populated programmatically from the parsed literal values — no MVEL3 evaluator is needed for init.

### Batch compilation in DrlxLambdaCompiler

New method:

```java
public DrlxCustomAccumulator createCustomAccumulator(
    CustomAccumulateIR ir,
    Class<?> srcClass,
    String srcBindingName)
```

`DrlxCustomAccumulator` holds three evaluator slots (action, reverse, result). Each slot is backed by a separate inner `EvaluatorSink` wrapper registered as its own `PendingLambda`. This follows the existing pattern where each lambda gets its own `PendingLambda(handle, sink)` entry.

Concrete approach: three lightweight sink wrappers (e.g. `DrlxCustomAccumulator.ActionSink`, `ReverseSink`, `ResultSink`), each implementing `EvaluatorSink.bindEvaluator()` to set its slot on the parent `DrlxCustomAccumulator`. The `createCustomAccumulator` method:

1. Build MVEL3 `Declaration` objects from `InitVarIR` list (holder vars) — these declare the types available in the map context
2. **Normalize** action block text before compilation. Single-expression form (from `accActionBlock : expression`) gets a trailing `;` appended. Braced form (`'{' statement+ '}'`) gets outer braces trimmed (same as `trimBraces` for consequences). Then compile via `MVEL.map(holderDecls + srcDecl).block(normalizedText + "\n return null;")` with out type `String.class` → register `PendingLambda(handle, actionSink)`. The `return null;` suffix and `String` out-type work around MVEL3's `Void`/`void` mismatch (same pattern as `DrlxLambdaConsequence`).
3. **Normalize** and compile reverse block (if non-null) same way → register `PendingLambda(handle, reverseSink)`
4. Resolve the result type from `resultTypeName` (using the type resolution contract above) to get `Class<?> resultClass`. Compile result expression via `MVEL.map(holderDecls).expression(resultExpr)` with out type `resultClass` → register `PendingLambda(handle, resultSink)`. This ensures MVEL3 produces the correct numeric shape (e.g. `Double` for `double avgAge = total / count`, not `Integer` from integer division).
5. Return `DrlxCustomAccumulator` (evaluators are null initially; bound by `compileBatch()`)

Each slot also integrates with `tryLoadPreCompiled()` / `onLambdaCreated()` for the prebuild cache path. The lambda counter increments once per slot (two or three increments per custom accumulator depending on whether reverse is present).

### collectPatternClasses

`DrlxRuleAstRuntimeBuilder.collectPatternClasses()` adds a `CustomAccumulateIR` branch to resolve the source pattern type for type-declaration registration and property-reactive setup:

```java
} else if (item instanceof CustomAccumulateIR customAcc) {
    classes.add(resolvePatternType(customAcc.source(), typeResolver, entryPointTypes, unitClass));
}
```

### Wiring into SingleAccumulate

The `DrlxRuleAstRuntimeBuilder` handles `CustomAccumulateIR`:
1. Build source pattern from `source` (single `PatternIR`)
2. Call `lambdaCompiler.createCustomAccumulator(ir, srcClass, srcBindingName)`
3. Wrap in `SingleAccumulate(srcPattern, required, accumulator)`
4. Create typed result `Pattern` with `resultTypeName` / `resultBindName`
5. Add to parent `GroupElement`
6. Register result binding in `outerScope`: resolve `resultClass`, fetch the `Declaration` from the wrapper pattern, and `outerScope.put(resultBindName, new BoundVariable(resultBindName, resultClass, wrapPattern, decl))`. This mirrors the existing `buildAccumulatePattern` epilogue and ensures later constraints/consequences can resolve the result binding (e.g. `avgAge`).

## MVEL3 variable sharing

The custom accumulator uses MVEL3's map-based evaluator pattern for the action, reverse, and result blocks. Compiled evaluators extract variables from a `Map<String, Object>` at the start, execute compiled code, and write modified variables back. The same map instance is passed to action → reverse → result calls, so mutations persist across the accumulate lifecycle.

Init var values are set programmatically on the map (no MVEL3 evaluator), so there is no risk of MVEL3 local variable declarations shadowing map context slots.

This approach is proven by MVEL3's `testMapEvaluatorReturns` test and matches drlx-parser's existing compile-early pattern.

## Test plan

### Visitor-level tests (AccumulateVisitorTest)

- 2-param single function → `AccumulatePatternIR` with one `AccumulatorIR`
- 2-param grouped functions → `AccumulatePatternIR` with N `AccumulatorIR`s
- 3-param (init + action + result) → `CustomAccumulateIR`, null reverse
- 3-param with (action, reverse) pair → `CustomAccumulateIR`, non-null reverse
- 5-param with braced blocks → `CustomAccumulateIR`, multi-statement init/action/reverse
- Multiple `InitVarIR` entries from braced init block

### Runtime integration tests (AccumulateTest)

- 2-param acc produces same result as bare function call (avg = 40.0 for [20, 40, 60])
- 3-param sum without reverse: insert facts, fire, check result
- 3-param sum with reverse: insert, fire, retract one fact, fire again, check updated result
- 5-param avg: insert [20, 40, 60], verify result = 40.0
- 5-param with retraction: verify reverse path updates correctly

### Protobuf round-trip tests

- `CustomAccumulateIR` → proto → `CustomAccumulateIR` round-trip for 3-param and 5-param forms
- Prebuild/cache path works end-to-end with custom accumulate rules

### Negative tests

- acc() with missing source → parse error
- 3-param with missing result binding → parse error
- Malformed init block → meaningful error message
- `var` in custom accumulate result binding → error (explicit type required)
- `var` in custom accumulate init var → error (explicit type required)
- Source binding referenced in result expression → error (only init vars allowed)
- Paired `(action, reverse)` block in 5-param form → error
- Outer-binding reference in action block → error pointing to #54
- Init var name equals source binding → error (name collision)
- Duplicate init var names → error
- Complex initializer `new Foo()` in init var → error (literals only)
- Type-incompatible literal `int s = null` → error
- Narrowing literal `int s = 0L` → error
- Unresolvable type name in init/result → meaningful error with fix suggestion
- Multi-declarator `int a = 0, b = 1;` → splits into two `InitVarIR` entries
- Missing initializer `int a;` → `InitVarIR` with default value

## DRLXXXX spec fix

Line 888: change `total / count` to `double avgAge = total / count` — the result must always have a typed binding.
