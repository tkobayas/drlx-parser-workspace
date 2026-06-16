# Named Windows — Design Spec (#72)

**Issue:** #72 (epic #79)
**DRLXXXX reference:** Lines 1075–1078

## Syntax

### Declaration (package level)

```
window WithdrawalWindow {
    /withdrawals | time(10s)
}
```

- `window` keyword followed by a PascalCase name (same convention as query declarations)
- Body is an OOPath expression + mandatory window filter
- No binding in the declaration — just the type source and behavior
- OOPath may include constraints: `/withdrawals[name == "special"] | time(10s)`

### Reference (in rules)

```
rule R1 {
    var w : /withdrawalWindow[amount > 1000],
    do { ... }
}
```

- Window name used as an OOPath root with lowercase first letter (same convention as query references)
- Declaration `WithdrawalWindow` → reference `/withdrawalWindow` (first letter lowered)
- Additional constraints via `[]` applied on top of the window's pattern
- Identical syntax to regular OOPath — the runtime builder resolves whether the name is a type or a window by matching against declared window names (lowercasing the first letter of the declaration name for comparison)

## Drools runtime mapping

The drools runtime already supports named windows — no new APIs needed:

| DRLX concept | Drools object |
|---|---|
| Window declaration | `WindowDeclaration` (name + compiled `Pattern` with behaviors) |
| Window reference in rule | `Pattern` with `WindowReference(name)` as source |
| Package storage | `InternalKnowledgePackage.addWindowDeclaration()` |

## Grammar & Lexer

### Lexer (`DrlxLexer.g4`)

Add `WINDOW` keyword:

```
WINDOW : 'window';
```

### Parser (`DrlxParser.g4`)

Add `windowDeclaration` rule and update `drlxCompilationUnit`:

```
drlxCompilationUnit
    : packageDeclaration? importDeclaration* unitDeclaration
      windowDeclaration* ruleDeclaration*
    ;

windowDeclaration
    : WINDOW identifier '{' oopathExpression windowFilter '}'
    ;
```

- Window declarations placed between `unitDeclaration` and `ruleDeclaration*`
- `windowFilter` is mandatory (a named window without a behavior has no purpose)
- Body reuses existing `oopathExpression` and `windowFilter` rules

## IR (AST model)

### New record

```java
public record WindowDeclarationIR(String name, PatternIR pattern) { }
```

### Updated `CompilationUnitIR`

```java
public record CompilationUnitIR(String packageName,
                                String unitName,
                                List<String> imports,
                                List<WindowDeclarationIR> windowDeclarations,
                                List<RuleIR> rules) { }
```

The window body is parsed into a `PatternIR` — same record already used for inline windows (carries `typeName`, `conditions`, `windowType`, `windowParameter`).

On the reference side, no IR changes. `/withdrawalWindow[amount > 1000]` produces a normal `PatternIR` with `typeName = "withdrawalWindow"`. Resolution happens at runtime build time.

## Visitor changes (`DrlxToRuleAstVisitor`)

- **`visitWindowDeclaration()`** — extracts window name from identifier, builds a `PatternIR` from `oopathExpression` + `windowFilter` (reusing existing pattern-building logic minus the binding), wraps in `WindowDeclarationIR`
- **`visitDrlxCompilationUnit()`** — iterates `ctx.windowDeclaration()`, collects into `CompilationUnitIR`

## Runtime builder changes (`DrlxRuleAstRuntimeBuilder`)

### Declaration side

For each `WindowDeclarationIR`:

1. Resolve the pattern type from the IR's `typeName`
2. Compile constraints and window behavior (`SlidingTimeWindow` / `SlidingLengthWindow`)
3. Create `WindowDeclaration(name, namespace)`, set the compiled `Pattern`
4. Register via `knowledgePackage.addWindowDeclaration(windowDecl)`
5. Process all window declarations before rules

### Reference side

When building a pattern from `PatternIR`, before normal type resolution:

1. Check if `typeName` matches any registered `WindowDeclaration` name
2. If match: use the window declaration's pattern type for the `Pattern`, set `WindowReference(name)` as the pattern source, apply additional constraints from the rule's `PatternIR`
3. If no match: proceed with normal OOPath type resolution (existing behavior)

## Protobuf changes (`drlx_rule_ast.proto`)

```protobuf
message WindowDeclarationParseResult {
  string name = 1;
  PatternParseResult pattern = 2;
}

message CompilationUnitParseResult {
  string source_hash = 1;
  string package_name = 2;
  repeated string imports = 3;
  repeated RuleParseResult rules = 4;
  string unit_name = 5;
  repeated WindowDeclarationParseResult window_declarations = 6;
}
```

`DrlxRuleAstParseResult` serialization/deserialization updated to round-trip window declarations.

## Testing

### Visitor-level tests

- Parses window declaration → correct `WindowDeclarationIR` (name, pattern type, window type/parameter)
- Parses window declaration with constraints in body → constraints carried through
- Parses rule referencing a window → normal `PatternIR` with window name as typeName

### Session-level tests

- Declare a time window, reference from rule, fire with pseudo clock → events outside window expire
- Declare a length window, reference from rule → only N most recent events match
- Reference with additional constraints → constraints filter within the window
- Multiple rules referencing the same named window → both rules fire correctly
