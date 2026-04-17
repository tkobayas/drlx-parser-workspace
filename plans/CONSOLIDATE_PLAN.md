# Plan: Consolidate Two RuleImpl Build Paths Into One

## Context

Adding a new DRLX syntax feature currently requires changes in 3+ places because there are two parallel paths from ANTLR parse tree to RuleImpl:
- **Path A:** ANTLR → `DrlxToRuleImplVisitor` → RuleImpl (direct)
- **Path B:** ANTLR → RuleAST records → proto → `DrlxRuleAstRuntimeBuilder` → RuleImpl (cached)

We consolidate to **always go through RuleAST records**, making proto persistence optional:

```
ANTLR AST → DrlxToRuleAstVisitor → RuleAST records (in-memory, always)
                                        ├── DrlxRuleAstRuntimeBuilder → RuleImpl
                                        └── proto serialization (optional, for preBuild cache)
```

After consolidation, new syntax features only need changes in **2 places**: the ANTLR→RuleAST visitor and the RuleAST→RuleImpl builder.

## Phase 1: Extract `DrlxLambdaCompiler` from `DrlxToRuleImplVisitor`

Separate the lambda compilation concern from ANTLR visiting. **Do NOT touch `DrlxPreBuildVisitor` yet** (it depends on the full visitor machinery until Phase 3).

Create `DrlxLambdaCompiler.java` with:
- **Fields:** `preBuildMetadata`, `outputDir`, `batchMode`, `batchCompiler`, `pendingLambdas`, `preBuildClassManager`, `loadedClassCache`, `patternId`
- **State management:** `beginRule(String ruleName)` method that resets `lambdaCounter` and sets `currentRuleName` (replaces the implicit state setting that `buildRule()` currently does)
- **Static:** `DECLARATION_CACHE`, `extractDeclarations()`
- **Lambda methods (non-final):** `createLambdaConstraint`, `createBetaLambdaConstraint`, `createLambdaConsequence`, `compileBatch`, batch helpers, `loadPreCompiledEvaluator`
- **Records:** `BoundVariable`, `PendingLambda`
- **Utilities:** `findReferencedBindings`, `getTypeMap`

Update `DrlxToRuleImplVisitor` to delegate to `DrlxLambdaCompiler` (Path A still works).
Update `DrlxRuleAstRuntimeBuilder` to use `DrlxLambdaCompiler` via **composition** instead of extending `DrlxToRuleImplVisitor`.
`DrlxPreBuildVisitor` still extends `DrlxToRuleImplVisitor` (temporary, migrated in Phase 3).

**Verify:** `mvn -pl drlx-parser-core test -Dtest=DrlxRuleBuilderTest` + `DrlxCompilerNoPersistTest`

## Phase 2: Create `DrlxToRuleAstVisitor` + extract RuleAST records

Create `DrlxRuleAstModel.java` in `org.drools.drlx.builder` with the record types moved from `DrlxRuleAstParseResult`:
- `CompilationUnitData`, `RuleData`, `RuleItemData` (sealed interface), `PatternData`, `ConsequenceData`

Create `DrlxToRuleAstVisitor.java` extending `DrlxParserBaseVisitor<Object>`:
- `visitDrlxCompilationUnit()` → returns `CompilationUnitData`
- Consolidates extraction helpers from both `DrlxToRuleImplVisitor` and `DrlxRuleAstParseResult`: `extractConditions`, `extractCastType`, `extractEntryPointFromOopathCtx`, `extractConsequence`
- Eliminates the current duplication of these methods between the two classes

Update imports in `DrlxRuleAstParseResult.java` and `DrlxRuleAstRuntimeBuilder.java` to use `DrlxRuleAstModel.*`.

**Verify:** New unit test for `DrlxToRuleAstVisitor` + existing tests pass.

## Phase 3: Rewire `DrlxRuleBuilder` to single pipeline + simplify persistence

The switchover point — Path A is replaced. Merge the PreBuildVisitor transformation here since it now becomes possible.

**Rewire all entry points in `DrlxRuleBuilder`:**

- `parse(String)`: 
  ```
  CompilationUnitData ast = new DrlxToRuleAstVisitor(tokens).visit(ctx);
  DrlxLambdaCompiler lambdaCompiler = new DrlxLambdaCompiler();
  lambdaCompiler.enableBatchMode(batchCompiler);
  List<KiePackage> packages = new DrlxRuleAstRuntimeBuilder(lambdaCompiler).build(ast);
  lambdaCompiler.compileBatch(classLoader);
  return packages;
  ```

- `build(String)`: delegates to `parse()` + `createKieBase()`

- `build(ctx, tokens, metadata)` (with pre-built metadata):
  ```
  CompilationUnitData ast = new DrlxToRuleAstVisitor(tokens).visit(ctx);
  DrlxLambdaCompiler lambdaCompiler = new DrlxLambdaCompiler();
  lambdaCompiler.setPreBuildMetadata(metadata);
  List<KiePackage> packages = new DrlxRuleAstRuntimeBuilder(lambdaCompiler).build(ast);
  return packages;
  ```

- `preBuild(String, Path)`:
  ```
  CompilationUnitData ast = new DrlxToRuleAstVisitor(tokens).visit(ctx);
  // persist RuleAST to proto (if cache strategy enabled)
  DrlxRuleAstParseResult.save(ast, sourceHash, outputDir);
  DrlxPreBuildLambdaCompiler preBuildCompiler = new DrlxPreBuildLambdaCompiler(...);
  new DrlxRuleAstRuntimeBuilder(preBuildCompiler).build(ast);
  preBuildCompiler.compileBatch(classLoader);
  ```

- `buildFromCache()`: unchanged (load proto → `CompilationUnitData` → `DrlxRuleAstRuntimeBuilder`)

**Transform `DrlxPreBuildVisitor`** → rename to `DrlxPreBuildLambdaCompiler extends DrlxLambdaCompiler`. It no longer needs ANTLR visiting because the ANTLR walk is handled by `DrlxToRuleAstVisitor`.

**Simplify `DrlxRuleAstParseResult`:**
- Update `save()` to accept `CompilationUnitData` instead of ANTLR contexts
- Remove ANTLR-to-proto conversion methods (`toProtoRule`, `toProtoPattern`, `extractConditions`, `extractCastType`, `extractEntryPointFromOopathCtx`)
- `load()` unchanged (proto → records)

**`DrlxBuildCacheStrategy`** semantics simplify: controls only proto **persistence**, not whether RuleAST IR is used.

**Verify:** All tests pass — `DrlxRuleBuilderTest`, `DrlxCompilerTest`, `DrlxCompilerNoPersistTest`.

## Phase 4: Delete `DrlxToRuleImplVisitor`

- Remove `DrlxToRuleImplVisitor.java`
- Clean up any remaining dead code
- Run all tests

## Files summary

| Action | File | Phase |
|--------|------|-------|
| Create | `DrlxLambdaCompiler.java` | 1 |
| Create | `DrlxRuleAstModel.java` | 2 |
| Create | `DrlxToRuleAstVisitor.java` | 2 |
| Modify | `DrlxToRuleImplVisitor.java` (delegate to LambdaCompiler) | 1 |
| Modify | `DrlxRuleAstRuntimeBuilder.java` (composition) | 1 |
| Modify | `DrlxRuleAstParseResult.java` (remove inner types + ANTLR methods) | 2, 3 |
| Modify | `DrlxRuleBuilder.java` (rewire) | 3 |
| Rename | `DrlxPreBuildVisitor` → `DrlxPreBuildLambdaCompiler` | 3 |
| Delete | `DrlxToRuleImplVisitor.java` | 4 |

## Verification

After each phase:
```bash
mvn -pl drlx-parser-core -am install -DskipTests
mvn -pl drlx-parser-core test -Dtest=DrlxRuleBuilderTest
mvn -pl drlx-parser-core test  # DrlxCompilerNoPersistTest
```

After Phase 3 (full switchover):
```bash
mvn -pl drlx-parser-core test -Dtest=DrlxCompilerTest
```
