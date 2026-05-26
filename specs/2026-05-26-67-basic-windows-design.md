# #67 Basic Length/Time Windows

**Issue:** https://github.com/tkobayas/drlx-parser/issues/67
**Epic:** #26
**DRLXXXX reference:** Lines 1031-1042

## Goal

Parse DRLX window syntax (`| length[N]`, `| time[Xs]`) and transpile to Drools
`SlidingLengthWindow` / `SlidingTimeWindow` behaviors on the `Pattern` object.

## DRLX Syntax

```
var w : /withdrawals | length[5]
var w : /withdrawals | time[5s]
var w : /withdrawals | time[4d6h5m6s]
```

The `|` operator appears after the oopath expression and before the line
terminator (`,`). It accepts a window type (`length` or `time`) and a parameter
in square brackets.

## Drools Mapping

| DRLX | Drools Runtime |
|------|---------------|
| `\| length[N]` | `pattern.addBehavior(new SlidingLengthWindow(N))` |
| `\| time[Xs]` | `pattern.addBehavior(new SlidingTimeWindow(TimeUtils.parseTimeString(Xs)))` |

No new Drools APIs required. All classes are already on the drlx-parser-core
classpath via the `drools-ruleunits-impl` transitive dependency.

## Design

### 1. Grammar (DrlxParser.g4)

Extend `boundOopath` with an optional `windowFilter` suffix:

```antlr
boundOopath
    : identifier identifier (':' | '=') oopathExpression windowFilter?
    ;

windowFilter
    : BITOR identifier '[' expression ']'
    ;
```

`BITOR` (`|`) is already defined in the inherited JavaLexer. The window type
uses `identifier` (not a keyword) to avoid reserving `length`/`time` as lexer
tokens. The visitor validates that the identifier text is `"length"` or
`"time"`.

### 2. AST Model (DrlxRuleAstModel.java)

Add two fields to `PatternIR`:

```java
public record PatternIR(String typeName,
                        String bindName,
                        String entryPoint,
                        List<String> conditions,
                        String castTypeName,
                        List<String> positionalArgs,
                        boolean passive,
                        List<String> watchedProperties,
                        String windowType,       // "length" | "time" | null
                        String windowParameter)  // "5" | "5s" | "4d6h5m6s" | null
    implements LhsItemIR {
}
```

### 3. Visitor (DrlxToRuleAstVisitor.java)

Extract window metadata in `buildPatternFromBoundOopath()`:

```java
String windowType = null;
String windowParameter = null;
DrlxParser.WindowFilterContext wf = ctx.windowFilter();
if (wf != null) {
    windowType = wf.identifier().getText();
    if (!"length".equals(windowType) && !"time".equals(windowType)) {
        throw new DrlxCompilerException("Unknown window type: " + windowType);
    }
    windowParameter = getText(wf.expression());
}
```

Pass these to the `PatternIR` constructor. All other `buildPattern*` call sites
pass `null, null`.

### 4. Protobuf (drlx_rule_ast.proto)

Add fields 9-10 to `PatternParseResult`:

```protobuf
message PatternParseResult {
  string type_name = 1;
  string bind_name = 2;
  string entry_point = 3;
  repeated string conditions = 4;
  string cast_type_name = 5;
  repeated string positional_args = 6;
  bool passive = 7;
  repeated string watched_properties = 8;
  string window_type = 9;       // NEW
  string window_parameter = 10; // NEW
}
```

### 5. Proto Serialization (DrlxRuleAstParseResult.java)

Update `patternToProto()` to write `windowType` and `windowParameter`, and
`patternFromProto()` to read them back into `PatternIR`.

### 6. Runtime Builder (DrlxRuleAstRuntimeBuilder.java)

After the watch-properties block in `buildPattern()`, add window behavior:

```java
if (parseResult.windowType() != null) {
    switch (parseResult.windowType()) {
        case "time" -> pattern.addBehavior(
            new SlidingTimeWindow(TimeUtils.parseTimeString(parseResult.windowParameter())));
        case "length" -> pattern.addBehavior(
            new SlidingLengthWindow(Integer.parseInt(parseResult.windowParameter())));
    }
}
```

Imports needed:
- `org.drools.core.rule.SlidingTimeWindow`
- `org.drools.core.rule.SlidingLengthWindow`
- `org.drools.base.time.TimeUtils`

### 7. Testing

Integration test in `DrlxRuleBuilderTest`:

- **`testWindowLength`** — DRLX rule with `| length[5]`, verify `Pattern` has
  a `SlidingLengthWindow` behavior with size 5.
- **`testWindowTime`** — DRLX rule with `| time[5s]`, verify `Pattern` has
  a `SlidingTimeWindow` behavior with size 5000ms.
- **`testWindowTimeCompound`** — DRLX rule with `| time[4d6h5m6s]`, verify
  correct millisecond conversion.

## Files Changed

| File | Change |
|------|--------|
| `DrlxParser.g4` | Add `windowFilter` rule, extend `boundOopath` |
| `DrlxRuleAstModel.java` | Add `windowType`, `windowParameter` to `PatternIR` |
| `DrlxToRuleAstVisitor.java` | Extract window filter, validate type |
| `drlx_rule_ast.proto` | Add fields 9-10 to `PatternParseResult` |
| `DrlxRuleAstParseResult.java` | Serialize/deserialize window fields |
| `DrlxRuleAstRuntimeBuilder.java` | Add `SlidingTimeWindow`/`SlidingLengthWindow` behavior |
| `DrlxRuleBuilderTest.java` | Integration tests |

## Out of Scope

- Constraint before/after window (#68)
- Windows with accumulate (#69)
- Windows over group elements (#70)
- Propagation delay (#71)
- Named windows (#72)
