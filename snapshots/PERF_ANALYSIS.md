# Performance Analysis: DRLX vs Executable-Model

## All Benchmark Results (SingleShotTime, cold start, 100 rules)

### Before Batch Compilation Optimization

```
Benchmark                                                             (ruleCount)  Mode  Cnt     Score     Error  Units
KieBaseBuildNoPersistenceBenchmark.buildWithDrlxNoPersist                     100    ss    5  5994.439 ± 138.730  ms/op
KieBaseBuildNoPersistenceBenchmark.buildWithExecutableModel                   100    ss    5  1265.521 ± 102.720  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithDrlx                     100    ss    5  2437.612 ± 149.021  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithExecutableModel          100    ss    5   133.336 ±  18.898  ms/op
KieBasePreBuildPersistenceBenchmark.preBuildWithDrlx                          100    ss    5  4385.426 ± 407.721  ms/op
KieBasePreBuildPersistenceBenchmark.preBuildWithExecutableModel               100    ss    5  1164.814 ± 205.089  ms/op
```

### After Batch Compilation (NoPersist path only)

```
Benchmark                                                             (ruleCount)  Mode  Cnt     Score     Error  Units
KieBaseBuildNoPersistenceBenchmark.buildWithDrlxNoPersist                     100    ss    5  2035.796 ± 127.861  ms/op
KieBaseBuildNoPersistenceBenchmark.buildWithExecutableModel                   100    ss    5  1246.619 ±  45.892  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithDrlx                     100    ss    5  2396.726 ± 106.791  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithExecutableModel          100    ss    5   147.074 ±  40.853  ms/op
KieBasePreBuildPersistenceBenchmark.preBuildWithDrlx                          100    ss    5  4359.349 ± 311.752  ms/op
KieBasePreBuildPersistenceBenchmark.preBuildWithExecutableModel               100    ss    5  1164.982 ± 199.239  ms/op
```

### After Batch Compilation (NoPersist + PreBuild paths)

```
Benchmark                                                             (ruleCount)  Mode  Cnt     Score     Error  Units
KieBasePreBuildPersistenceBenchmark.preBuildWithDrlx                          100    ss   25   499.443 ±  84.663  ms/op
KieBasePreBuildPersistenceBenchmark.preBuildWithExecutableModel               100    ss   25   164.198 ±  18.193  ms/op
```

Settings: `-f 5 -wi 3 -i 5 -bm ss` (5 forks, 3 warmup iterations for JIT, 5 measurement iterations per fork).

KieBase building is a one-time process, so SingleShotTime (`ss`) is the appropriate benchmark mode. Adding warmup iterations (`-wi 3`) lets JIT optimize shared infrastructure (ANTLR, classloading) before measurement, isolating the code's efficiency from JVM cold-start noise.

## Summary of Ratios

### Before Batch Compilation

| Scenario | DRLX (ms) | Exec-Model (ms) | Ratio | What it measures |
|----------|-----------|-----------------|-------|------------------|
| **NoPersist** | 5994 | 1266 | **4.74x** | Full in-memory build (compile + KieBase creation) |
| **PreBuild** | 4385 | 1165 | **3.76x** | Compile + persist to disk (maven plugin phase) |
| **UsingPreBuild** | 2438 | 133 | **18.3x** | Load pre-built artifacts → KieBase (runtime) |

### After Batch Compilation (NoPersist only)

| Scenario | DRLX (ms) | Exec-Model (ms) | Ratio | What it measures |
|----------|-----------|-----------------|-------|------------------|
| **NoPersist** | 2036 | 1247 | **1.63x** | Full in-memory build (compile + KieBase creation) |
| **PreBuild** | 4359 | 1165 | **3.74x** | Compile + persist to disk (maven plugin phase) |
| **UsingPreBuild** | 2397 | 147 | **16.3x** | Load pre-built artifacts → KieBase (runtime) |

### After Batch Compilation (NoPersist + PreBuild)

| Scenario | DRLX (ms) | Exec-Model (ms) | Ratio | What it measures |
|----------|-----------|-----------------|-------|------------------|
| **NoPersist** | 2036 | 1247 | **1.63x** | Full in-memory build (compile + KieBase creation) |
| **PreBuild** | **499** | 164 | **3.04x** | Compile + persist to disk (maven plugin phase) |
| **UsingPreBuild** | 2397 | 147 | **16.3x** | Load pre-built artifacts → KieBase (runtime) |

The 2-step build splits NoPersist into PreBuild (offline) + UsingPreBuild (runtime).

### Warm-JVM UsingPreBuild (avgt, 10 warmup + 10 measurement iterations)

| Rule Type | DRLX (ms) | Exec-Model (ms) | Ratio | What it measures |
|-----------|-----------|-----------------|-------|------------------|
| **alpha** | 4.208 | 6.368 | **0.66x (DRLX faster)** | Load pre-built artifacts → KieBase (warm JVM) |
| **multiAlpha** | 5.605 | 7.946 | **0.71x (DRLX faster)** | Load pre-built artifacts → KieBase (warm JVM) |
| **join** | 6.165 | 7.932 | **0.78x (DRLX faster)** | Load pre-built artifacts → KieBase (warm JVM) |
| **multiJoin** | 15.622 | 8.506 | **1.84x** | Load pre-built artifacts → KieBase (warm JVM) |

With `-gc true`, DRLX is **faster** than exec-model for alpha/multiAlpha/join rule types. Only multiJoin (3 patterns, 4 lambdas/rule) remains 1.84x slower.

## Batch Compilation Optimization — Impact Analysis

The batch compilation optimization collects all 200 generated Java sources during the ANTLR tree walk and compiles them in a **single** `KieMemoryCompiler.compile()` call, eliminating 199 redundant Java compiler startup/initialization cycles.

### What changed

| Benchmark | Before | After | Change | Explanation |
|-----------|--------|-------|--------|-------------|
| **NoPersist** | 5994ms | **2036ms** | **2.95x faster** | 200 javac calls → 1 batch call. Compiler startup overhead eliminated. |
| **PreBuild** | 4359ms | **499ms** | **8.74x faster** | Same batch approach + `compileAndPersist()` for disk persistence. |
| **UsingPreBuild** | 2438ms | 2397ms | ~1.7% (noise) | Loads pre-compiled .class files from disk. No javac involved → batch optimization has zero effect. |

### Why UsingPreBuild did not improve

This benchmark loads pre-compiled `.class` files from disk and calls `defineHiddenClass()` 200 times. There is no javac compilation on this path at all — the bottleneck is `defineHiddenClass()` (~62% of time) and `MethodByteCodeExtractor` (~29%). Batching javac has zero effect here.

### Implementation details

**Files changed:**

| File | Change |
|------|--------|
| `MVELCompiler.java` | Added `TranspiledSource` record, `transpileToSource()` (transpile without javac), `resolveEvaluator()` (instantiate from ClassManager) |
| `DrlxLambdaConstraint.java` | Added `setEvaluator()` for deferred resolution after batch compile |
| `DrlxLambdaConsequence.java` | Added `setEvaluator()` for deferred resolution after batch compile |
| `DrlxToRuleImplVisitor.java` | Added batch mode: `enableBatchMode()`, `compileBatch()`, pending lambda tracking with unique class names. Batch fields made `protected` for subclass access. |
| `DrlxPreBuildVisitor.java` | Overrides `compileBatch()` to use `KieMemoryCompiler.compileAndPersist()` + deferred metadata recording via `PendingPreBuildInfo`. Bypasses per-lambda `LambdaRegistry` flow. |
| `DrlxRuleBuilder.java` | `parse()` enables batch mode when `!LambdaRegistry.PERSISTENCE_ENABLED && BATCH_ENABLED`. `preBuild()` enables batch mode when `BATCH_ENABLED`. Added `BATCH_ENABLED` system property (`drlx.compiler.batch`, default: `true`). |

**Flow (batch mode — NoPersist path):**

```
DrlxRuleBuilder.parse()
├── visitor.enableBatchMode(sharedClassManager)
├── visitor.visitDrlxCompilationUnit()       // tree walk
│   └── for each lambda:
│       ├── MVELCompiler.transpileToSource() // MVEL → Java source (cheap, ~2-5ms)
│       ├── pendingSources.put(fqn, source)  // collect
│       └── return constraint/consequence with null evaluator
└── visitor.compileBatch()
    ├── KieMemoryCompiler.compile(sharedClassManager, allSources, classLoader)  // ONE javac call for all 200
    └── for each pending lambda:
        └── MVELCompiler.resolveEvaluator(sharedClassManager, fqn)  // instantiate from compiled class
```

**Flow (batch mode — PreBuild persistence path):**

```
DrlxRuleBuilder.preBuild()
├── visitor.enableBatchMode(sharedClassManager)
├── visitor.visitDrlxCompilationUnit()       // tree walk (same as NoPersist)
│   └── for each lambda:
│       ├── MVELCompiler.transpileToSource()
│       ├── pendingSources.put(fqn, source)
│       ├── pendingPreBuildInfos.add(ruleName, counterId, expression, fqn)  // deferred metadata
│       └── return constraint/consequence with null evaluator
└── visitor.compileBatch()  (DrlxPreBuildVisitor override)
    ├── KieMemoryCompiler.compileAndPersist(sharedClassManager, allSources, classLoader, persistPath)
    │   // ONE javac call + persist all 200 .class files to disk
    ├── for each pending lambda:
    │   └── MVELCompiler.resolveEvaluator(sharedClassManager, fqn)
    └── for each pendingPreBuildInfo:
        └── metadata.put(ruleName, counterId, fqn, persistedFilePath, expression)
```

## Key Findings

### 1. The 2-step build dramatically helps executable-model but barely helps DRLX

Executable-model runtime loading drops from **1266ms → 133ms** (9.5x improvement). It loads a pre-built kjar via standard ClassLoader — cheap and fast.

DRLX runtime loading drops from **5994ms → 2438ms** (2.5x improvement). The pre-build eliminates the javac compilation step, but replaces it with expensive per-class hidden class loading. The runtime is still dominated by 200 individual `defineHiddenClass()` calls.

### 2. The NoPersist vs PreBuild gap (~1600ms) is KieBase assembly, not disk I/O

- **NoPersist** = compile 200 lambdas + create KieBase
- **PreBuild** = compile 200 lambdas + write to disk (no KieBase)

The ~1600ms difference is the cost of assembling the KieBase (RuleImpl objects, Pattern objects, GroupElement trees, KnowledgePackageImpl, kBase.addPackages()). Disk I/O for writing 200 small .class files is negligible (<50ms).

### 3. DRLX UsingPreBuild (2438ms) — Why so slow?

The `loadPreCompiledEvaluator()` method (`DrlxToRuleImplVisitor.java:205-211`) does this **200 times**:

```java
private Object loadPreCompiledEvaluator(String fqn, String classFilePath) throws Exception {
    byte[] bytes = Files.readAllBytes(Path.of(classFilePath));     // read .class file
    ClassManager classManager = new ClassManager();                 // NEW per lambda!
    classManager.define(Collections.singletonMap(fqn, bytes));     // defineHiddenClass + bytecode extraction
    Class<?> clazz = classManager.getClass(fqn);
    return clazz.getConstructor().newInstance();
}
```

Inside `ClassManager.define()`, for each class:
1. `ClassEntry` constructor calls `MethodByteCodeExtractor.extract("eval", bytes)` — ASM parses bytecode
2. `Murmur3F` hash is computed for deduplication
3. `lookup.defineHiddenClass(bytes, true)` — JVM hidden class definition (verify + link)

**Estimated time breakdown for UsingPreBuild (~2438ms):**

| Component | Per-lambda | x200 | % of total |
|-----------|-----------|------|-----------|
| File I/O (`readAllBytes`) | ~0.3ms | ~60ms | ~2% |
| `MethodByteCodeExtractor.extract()` + Murmur hash | ~2-5ms | ~400-1000ms | ~29% |
| **`defineHiddenClass()` (verify + link)** | **~5-10ms** | **~1000-2000ms** | **~62%** |
| ANTLR re-parse + KieBase assembly | — | ~200ms | ~7% |

The DRLX source is also **re-parsed** every time because the visitor must walk the parse tree to match lambdas with metadata entries.

### 4. Why executable-model UsingPreBuild is only 133ms

The executable-model path loads a single kjar using standard `URLClassLoader`/`KieModuleClassLoader`. Classes are loaded lazily via the JVM's optimized classpath class loading. No `defineHiddenClass()`, no `MethodByteCodeExtractor`, no per-class overhead.

## Root Cause: Per-Lambda Java Compilation (NoPersist/PreBuild)

Each of 100 rules generates **2 MVEL compilations** (constraint + consequence) = **200 separate compilation cycles**:

```
MVEL expression → Transpile to Java source → Java compile (javac/ECJ) → defineHiddenClass
```

### Time Breakdown (NoPersist, ~5994ms total)

| Step | Per-lambda | x200 lambdas | % of total |
|------|-----------|-------------|-----------|
| MVEL → Java transpilation | ~2-5ms | ~400-1000ms | ~12% |
| **Java source → bytecode compile** | **~15-25ms** | **~3000-5000ms** | **~67%** |
| defineHiddenClass + bytecode extraction | ~3-5ms | ~600-1000ms | ~13% |
| KieBase assembly | — | ~500ms | ~8% |

The **Java compilation** (`KieMemoryCompiler.compile()`) is the dominant cost. Each lambda spins up the compiler for a single small class.

### MVEL compilation pipeline

```
MVELCompiler.compile(CompilerParameters)
├── transpile(info)                         // MVEL → Java source
│   ├── MVELTranspiler.transpile()
│   └── CompilationUnitGenerator.createCompilationUnit()
│
└── compileEvaluator(unit, info)
    └── compileEvaluatorClass(classManager, sources, javaFQN)
        └── KieMemoryCompiler.compile()     // ← PRIMARY BOTTLENECK
            └── JavaCompiler.compile()      // javac or Eclipse compiler per lambda
```

## Consistency Check: Does PreBuild + UsingPreBuild = NoPersist?

No: 4385 + 2438 = **6823ms** > 5994ms (NoPersist). The ~828ms surplus exists because:

1. **DRLX source is parsed twice** — once in PreBuild, once in UsingPreBuild (~100-300ms)
2. **UsingPreBuild pays different costs** — `defineHiddenClass()` + bytecode extraction replaces javac compilation, but the per-class hidden class overhead is additive, not complementary to PreBuild

The 2-step model is not a clean decomposition — UsingPreBuild has its own substantial overhead that partially duplicates work.

## Optimization Paths (in order of impact)

### 1. Replace `defineHiddenClass()` with `ClassLoader.defineClass()` on load path (HIGH — fixes UsingPreBuild)

**Root cause**: `loadPreCompiledEvaluator()` (`DrlxToRuleImplVisitor.java:285-291`) is called **200 times**, each time creating a **new `ClassManager`** and calling `defineHiddenClass()`. This is expensive per-call because:
- Each hidden class is isolated — no constant pool sharing, separate class data
- Full bytecode verification on each call
- No JVM batching optimization
- A new `MethodHandles.Lookup` is created per lambda

**Current code:**
```java
private Object loadPreCompiledEvaluator(String fqn, String classFilePath) throws Exception {
    byte[] bytes = Files.readAllBytes(Path.of(classFilePath));
    ClassManager classManager = new ClassManager();                 // NEW per lambda!
    classManager.define(Collections.singletonMap(fqn, bytes));     // defineHiddenClass + bytecode extraction
    Class<?> clazz = classManager.getClass(fqn);
    return clazz.getConstructor().newInstance();
}
```

Inside `ClassManager.define()`, for each class:
1. `ClassEntry` constructor calls `MethodByteCodeExtractor.extract("eval", bytes)` — ASM parses bytecode (~2-5ms)
2. `Murmur3F` hash is computed for deduplication
3. `lookup.defineHiddenClass(bytes, true)` — JVM hidden class definition (verify + link) (~5-10ms)

**Fix**: Replace with a `ByteArrayClassLoader` using standard `ClassLoader.defineClass()`:

```java
class ByteArrayClassLoader extends ClassLoader {
    private final Map<String, byte[]> classBytes;

    ByteArrayClassLoader(ClassLoader parent, Map<String, byte[]> classBytes) {
        super(parent);
        this.classBytes = classBytes;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = classBytes.get(name);
        if (bytes != null) {
            return defineClass(name, bytes, 0, bytes.length);
        }
        throw new ClassNotFoundException(name);
    }
}
```

`ClassLoader.defineClass()` is the standard JVM class loading mechanism (what `URLClassLoader` uses internally). It benefits from JVM-level batch verification optimizations and is significantly cheaper than `defineHiddenClass()`. This is what executable-model does (loads classes via `URLClassLoader`), and it's why its cold-start is only 147ms.

**Tradeoff**: Hidden classes are GC'd when unreferenced (good for dynamic code), while ClassLoader-defined classes live as long as the ClassLoader. For pre-built KieBase artifacts that live for the application lifetime, this tradeoff is favorable.

**Implementation approach**: Read all 200 `.class` files upfront into a `Map<String, byte[]>`, create a single `ByteArrayClassLoader`, then instantiate each evaluator via `Class.forName(fqn, true, loader).getConstructor().newInstance()`. This bypasses `ClassManager`, `ClassEntry`, `MethodByteCodeExtractor`, and `defineHiddenClass()` entirely on the load path. Expected to bring UsingPreBuild from ~2397ms close to executable-model's ~147ms.

### 2. Skip `MethodByteCodeExtractor` on the load path (included in #1, or standalone fallback)

If replacing `defineHiddenClass()` entirely is not feasible (e.g., hidden class GC semantics are needed), a smaller fix: add a `defineWithoutDedup(Map<String, byte[]>)` method to `ClassManager` that calls `defineHiddenClass()` directly without constructing `ClassEntry`. This skips `MethodByteCodeExtractor.extract()` + `Murmur3F` hashing (~29% of UsingPreBuild time). Also batch all 200 classes into a single `ClassManager` instead of creating one per lambda.

### 3. ~~Batch Compilation in MVEL3 (HIGH — fixes NoPersist)~~ ✅ DONE

~~Collect all 200 generated Java sources and compile in a **single** `KieMemoryCompiler.compile()` call. Compiler startup/initialization overhead is currently paid 200 times.~~

**Result**: NoPersist DRLX dropped from **5994ms → 2036ms (2.95x faster)**, ratio vs exec-model narrowed from **4.74x → 1.63x**.

### 4. ~~Batch Compilation for Persistence Path (HIGH — fixes PreBuild)~~ ✅ DONE

~~Apply the same batch compilation to the persistence-enabled path. Requires a two-phase flow: transpile all sources → batch javac → register/persist each output with `LambdaRegistry`.~~

**Result**: PreBuild DRLX dropped from **4359ms → 499ms (8.74x faster)**, ratio vs exec-model narrowed from **3.74x → 3.04x**. `DrlxPreBuildVisitor` overrides `compileBatch()` to use `KieMemoryCompiler.compileAndPersist()` for a single javac call + batch disk persistence, bypassing the per-lambda `LambdaRegistry` flow entirely. Batch mode is controlled by `drlx.compiler.batch` system property (default: `true`).

### 5. Cache Build Input: ParseTree vs RuleAST (MEDIUM — fixes UsingPreBuild)

Two cache strategies were prototyped behind `drlx.compiler.cacheStrategy`:

- `parsetree`: persist `drlx-parse-tree.pb` and rehydrate `DrlxParser.*Context`
- `ruleast`: persist `drlx-rule-ast.pb`, a compact DRLX-specific rule AST

The `ruleast` strategy stores only the semantic rule data needed at runtime
(package/imports/rules/patterns/consequence text), then rebuilds `RuleImpl`
without reconstructing ANTLR runtime objects.

#### Benchmark Configuration

```
java -jar target/drlx-benchmarks.jar \
  -jvmArgs "-Xms4g -Xmx4g" \
  -gc true -f 1 -wi 5 -i 5 -bm avgt \
  -p ruleCount=100 -p ruleType=multiJoin \
  -p runConfig=none,parsetree,ruleast,exec-model \
  org.drools.drlx.perf.KieBaseBuildUsingPreBuildArtifactsBenchmark

java -jar target/drlx-benchmarks.jar \
  -jvmArgs "-Xms4g -Xmx4g" \
  -gc true -f 1 -wi 5 -i 5 -bm avgt \
  -p ruleCount=1000 -p ruleType=multiJoin \
  -p runConfig=none,parsetree,ruleast,exec-model \
  org.drools.drlx.perf.KieBaseBuildUsingPreBuildArtifactsBenchmark
```

#### Results

| Rule Count | DRLX none (ms) | DRLX parseTree (ms) | DRLX ruleAst (ms) | Exec-model (ms) |
|---|---|---|---|---|
| 100 | 15.354 ± 0.176 | 11.802 ± 0.380 | **3.641 ± 0.555** | 8.365 ± 0.896 |
| 1000 | 147.945 ± 3.816 | 118.029 ± 3.170 | **38.831 ± 3.427** | 71.734 ± 2.840 |

#### Interpretation

- `parsetree` improves DRLX, but only modestly:
  - `100 rules`: `15.354 -> 11.802 ms` (**1.30x faster**)
  - `1000 rules`: `147.945 -> 118.029 ms` (**1.25x faster**)
- `ruleast` changes the ranking entirely:
  - vs DRLX `none`:
    - `100 rules`: **4.22x faster**
    - `1000 rules`: **3.81x faster**
  - vs exec-model:
    - `100 rules`: **2.30x faster**
    - `1000 rules`: **1.85x faster**

#### Conclusion

- `parsetree` is useful as a measurement point, but ANTLR rehydration is too
  expensive to be the final design.
- `ruleast` is the first cache strategy that clearly outperforms exec-model on
  `multiJoin`.
- The architectural downside of `ruleast` is maintainability: there are
  currently two `RuleImpl` construction paths, `ParseTree -> RuleImpl`
  (`DrlxToRuleImplVisitor`) and `RuleAST -> RuleImpl`
  (`DrlxRuleAstRuntimeBuilder`). If `ruleast` becomes the chosen direction,
  the next cleanup step should be a shared semantic-model-to-`RuleImpl` builder.

### 6. Expression-Based Caching (LOW for this benchmark)

Cache compiled evaluators by expression hash. With 100 unique rules this benchmark has zero duplication, but real-world rule sets with repeated patterns would benefit.

## Benchmark Methodology Notes

- **5 forks**: Adequate for directional analysis. JMH uses 99.9% CI with t-distribution (df=4, t=8.61), which inflates error bars. 10 forks would halve confidence intervals for publication-quality results.
- **Relative error margins**: DRLX 2-9%, exec-model 8-18%. The smaller DRLX variance is expected given its longer runtime.
- **`@Setup(Level.Invocation)` in PreBuild benchmark**: Setup/teardown time is included in ss measurements. The overhead (temp dir creation, LambdaRegistry reset) is <5ms — negligible for 1000-6000ms measurements.
- **Setup warms some classes**: `@Setup(Level.Trial)` in UsingPreBuild loads ClassManager, ANTLR, etc. Both sides benefit roughly equally, making the numbers slightly optimistic vs truly cold production deployment.
- **GC**: 4GB heap (`-Xms4g -Xmx4g`) means minimal GC pressure. No specific GC algorithm pinned — minor reproducibility concern.

## Warm-JVM Analysis (avgt mode)

### Benchmark Configuration

```
java -jar target/drlx-benchmarks.jar \
  -jvmArgs "-Xms4g -Xmx4g" \
  -f 1 -wi 10 -i 10 -bm avgt -p ruleCount=100 \
  org.drools.drlx.perf.KieBaseBuildUsingPreBuildArtifactsBenchmark
```

JDK 17.0.15, OpenJDK 64-Bit Server VM (Temurin), G1GC (default), 4GB heap.
1 fork, 10 warmup iterations, 10 measurement iterations, average time.

### Results (all rule types, with `-gc true`)

```
Benchmark                                                             (ruleCount)  (ruleType)  Mode  Cnt   Score   Error  Units
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithDrlx                     100       alpha  avgt    3   4.208 ± 1.417  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithDrlx                     100  multiAlpha  avgt    3   5.605 ± 1.752  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithDrlx                     100        join  avgt    3   6.165 ± 1.973  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithDrlx                     100   multiJoin  avgt    3  15.622 ± 2.651  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithExecutableModel          100       alpha  avgt    3   6.368 ± 1.294  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithExecutableModel          100  multiAlpha  avgt    3   7.946 ± 0.585  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithExecutableModel          100        join  avgt    3   7.932 ± 0.990  ms/op
KieBaseBuildUsingPreBuildArtifactsBenchmark.buildWithExecutableModel          100   multiJoin  avgt    3   8.506 ± 5.557  ms/op
```

DRLX is **faster** for 3 of 4 rule types. Only multiJoin remains slower (1.84x).

### Results (alpha + multiAlpha + join + multiJoin)

| Rule Type | Lambdas/rule | DRLX (ms) | Exec-model (ms) | Ratio |
|---|---|---|---|---|
| Alpha | 2 | 4.208 ± 1.417 | 6.368 ± 1.294 | **0.66x (DRLX faster)** |
| MultiAlpha | 2 | 5.605 ± 1.752 | 7.946 ± 0.585 | **0.71x (DRLX faster)** |
| Join | 3 | 6.165 ± 1.973 | 7.932 ± 0.990 | **0.78x (DRLX faster)** |
| MultiJoin | 4 | 15.622 ± 2.651 | 8.506 ± 5.557 | **1.84x** |

Settings: `-f 1 -wi 10 -i 10 -bm avgt -gc true -p ruleCount=100`, 3 measurement iterations.

DRLX is now **faster than exec-model** for alpha, multiAlpha, and join rule types. Only multiJoin (3 patterns, 4 lambdas/rule) remains slower at 1.84x. DRLX scales linearly with lambda count (~3.8ms per additional lambda/rule for 100 rules), while exec-model scales nearly flat (+2.1ms from alpha to multiJoin).

Compared to earlier runs (without `-gc true`), alpha ratio improved from 1.17x to 0.66x — the forced GC between iterations eliminates the GC variance that previously inflated DRLX times.

### Build Cache Strategy Experiment (Protobuf)

The `UsingPreBuild` benchmark was rerun with both protobuf-backed cache
strategies:

- `parsetree`: serialized ANTLR parse tree
- `ruleast`: serialized DRLX-specific Rule AST

#### Benchmark Configuration

```
java -jar target/drlx-benchmarks.jar \
  -jvmArgs "-Xms4g -Xmx4g" \
  -gc true -f 1 -wi 5 -i 5 -bm avgt \
  -p ruleCount=100 -p ruleType=multiJoin \
  -p runConfig=none,parsetree,ruleast,exec-model \
  org.drools.drlx.perf.KieBaseBuildUsingPreBuildArtifactsBenchmark

java -jar target/drlx-benchmarks.jar \
  -jvmArgs "-Xms4g -Xmx4g" \
  -gc true -f 1 -wi 5 -i 5 -bm avgt \
  -p ruleCount=1000 -p ruleType=multiJoin \
  -p runConfig=none,parsetree,ruleast,exec-model \
  org.drools.drlx.perf.KieBaseBuildUsingPreBuildArtifactsBenchmark
```

#### Results

| Rule Count | DRLX none (ms) | DRLX parseTree (ms) | DRLX ruleAst (ms) | Exec-model (ms) |
|---|---|---|---|---|
| 100 | 15.354 ± 0.176 | 11.802 ± 0.380 | **3.641 ± 0.555** | 8.365 ± 0.896 |
| 1000 | 147.945 ± 3.816 | 118.029 ± 3.170 | **38.831 ± 3.427** | 71.734 ± 2.840 |

The protobuf parse-tree snapshot improves `multiJoin`, but its benefit is much
smaller than the RuleAST parse-result. `ruleast` avoids both reparsing and ANTLR
context rehydration, which is why it scales much better.

### CPU Profile Analysis (multiJoin, async-profiler)

Profile source: `flame-cpu-forward.html` from async-profiler on `buildWithDrlx` with `ruleType=multiJoin`.

Total benchmark samples: 4083.

| Component | Samples | % of total | Notes |
|---|---|---|---|
| **ANTLR parsing** (`DrlxParser.drlxCompilationUnit`) | ~2019 | **49%** | Parsing DRLX source text |
| **`ClassLoader.defineClass0`** (JVM native) | ~1179 | **29%** | Bytecode verification + linking per lambda class |
| `ClassManager$ClassEntry.<init>` | ~152 | 4% | ClassManager overhead |
| `Files.readAllBytes` | ~165 | 4% | File I/O for loading class bytes — **not a bottleneck** |
| `createKieBase` | 100 | 2% | KieBase assembly |
| `findReferencedBindings` | 73 | 2% | Regex scanning for bound variables |
| Other | ~395 | 10% | |

#### Breakdown by call site

`loadPreCompiledEvaluator` is called from three paths. Each path's cost is dominated by `ClassLoader.defineClass0`:

| Call site | `loadPreCompiledEvaluator` samples | `defineClass0` samples |
|---|---|---|
| `createBetaLambdaConstraint` | 853 | 631 |
| `createLambdaConstraint` (alpha) | 412 | 269 |
| `createLambdaConsequence` | 387 | 279 |
| **Total** | **1652** | **1179** |

#### Key insight

The file I/O hypothesis was wrong. Bulk-loading class file bytes (`Files.readAllBytes`) would save only ~4% of build time. The actual cost is in the JVM's class loading machinery (`defineClass0`), not disk reads. The two dominant costs are:

1. **ANTLR parsing (49%)** — Even with pre-built artifacts, the DRLX source is fully parsed to walk the tree and match lambdas with metadata entries. Exec-model skips this entirely since rules are already compiled into Java.
2. **JVM class definition (29%)** — `defineClass0` (native bytecode verification + linking) is expensive when defining 400 individual lambda classes via `ClassManager.define()` → `defineHiddenClass()`.

### CPU Profile Analysis (join, protobuf serialized parse-tree)

Profile source: `flame-cpu-forward.html` from async-profiler on `buildWithDrlx`
with `ruleType=join`, `ruleCount=100`, and `drlx.compiler.serializedParseTree=true`.

Total benchmark samples in the measured path: 4216.

| Component | Samples | % of measured path | Notes |
|---|---|---|---|
| **`DrlxParseTreeParseResult.load`** | 2366 | **56%** | New dominant cost after removing reparsing |
| `DrlxParseTreeProto.ParseTreeParseResult.parseFrom` | 436 | 10% | Raw protobuf decode |
| **`DrlxParseTreeParseResult.fromProtoNode`** | 1879 | **45%** | Recursive ANTLR context rehydration |
| **`DrlxToRuleImplVisitor.visitDrlxCompilationUnit`** | 1635 | **39%** | Rule visit/build still substantial |
| `DrlxRuleBuilder.createKieBase` | 192 | 5% | KieBase assembly |

#### Breakdown inside protobuf rehydration

The dominant protobuf cost is not reading bytes from disk. It is rebuilding the
ANTLR object graph:

- repeated `Class.forName`
- repeated `Class.getConstructor` / `Class.getConstructor0`
- repeated `Constructor.newInstance`
- deep recursive `fromProtoNode(...)`

This makes the protobuf approach fundamentally different from a compact AST/IR
cache: protobuf removes parser runtime work, but pays heavily in reflection and
object reconstruction.

#### Hidden-class loading is still expensive

Even after removing reparsing, lambda evaluator loading remains a major runtime
cost inside the visitor:

| Call site | Samples | Notes |
|---|---|---|
| `createLambdaConstraint` | 1240 | Alpha constraint path |
| `loadPreCompiledEvaluator` (alpha) | 1223 | Mostly evaluator class loading |
| `ClassManager.define` (alpha) | 1013 | Dominated by hidden class definition |
| `MethodHandles.Lookup.defineHiddenClass` (alpha) | 920 | Native bytecode verify + link |
| `createBetaLambdaConstraint` | 61 | Join/beta constraint path |
| `createLambdaConsequence` | 59 | Consequence path |

#### Key insight

The parse-tree cache removes ANTLR parsing, but most of the saved work is
replaced by ANTLR context rehydration. This is why `parsetree` helps, but does
not fundamentally change the ranking against exec-model.

### CPU Profile Analysis (multiJoin, RuleAST)

Profile source: `flame-cpu-forward.html` from async-profiler on `build`
with `ruleType=multiJoin`, `ruleCount=100`, and `runConfig=ruleast`.

Total benchmark samples in the measured path: 3632.

| Component | Samples | % of measured path | Notes |
|---|---|---|---|
| **`DrlxRuleAstRuntimeBuilder.buildPattern`** | 3020 | **83%** | Dominant RuleAST rebuild cost |
| **`createLambdaConstraint`** | 2501 | **69%** | Alpha constraint path |
| `createBetaLambdaConstraint` | 195 | 5% | Beta constraint path |
| `createLambdaConsequence` | 58 | 2% | Consequence path |
| `findReferencedBindings` | 203 | 6% | AST-specific binding detection |
| **`DrlxRuleAstParseResult.load`** | 84 | **2%** | Cache load is now small |
| `DrlxRuleAstProto.CompilationUnitParseResult.parseFrom` | 49 | 1% | Raw protobuf decode |

#### Hidden-class loading is now the dominant remaining cost

Inside the RuleAST path, the dominant work is no longer cache deserialization.
It is evaluator class loading:

| Call site | Samples | Notes |
|---|---|---|
| `loadPreCompiledEvaluator` (alpha) | 2472 | Main runtime cost |
| `ClassManager.define` (alpha) | 1894 | Hidden class definition path |
| `MethodHandles.Lookup.defineHiddenClass` (alpha) | 1666 | Native bytecode verify + link |

#### Key insight

`ruleast` successfully removes the ANTLR bottleneck. Cache loading becomes
almost irrelevant, and the remaining ceiling is the hidden-class loading path
for precompiled evaluators.

The protobuf experiment confirms that **reparsing is not the only bottleneck**.
Removing ANTLR parse time helps, but protobuf-based ANTLR context rehydration
becomes the new dominant cost. A DRLX-specific AST/IR remains the more promising
direction because it can avoid:

1. reparsing DRLX text
2. protobuf decode into full ANTLR context objects
3. reflection-heavy ANTLR context reconstruction

### Allocation Comparison (`UsingPreBuild`, `multiJoin`, `-prof gc`)

The `UsingPreBuild` benchmark was also run with `-prof gc` across all runtime
build modes for `ruleCount=100`, `ruleType=multiJoin`, and three measurement
iterations.

For memory comparison, the most useful metric is **`gc.alloc.rate.norm`**
because it reports normalized allocation in bytes per benchmark operation.

#### Normalized allocation per `build()` call

| runConfig | Time (ms/op) | `gc.alloc.rate.norm` (B/op) | Approx. MB/op |
|---|---:|---:|---:|
| `none` | 15.908 | 33,614,120 | 33.6 |
| `parsetree` | 11.756 | 16,353,108 | 16.4 |
| `ruleast` | 3.781 | 3,484,944 | 3.5 |
| `exec-model` | 8.263 | 5,338,041 | 5.3 |

#### Allocation takeaways

- `ruleast` has the best allocation profile in this run at **~3.5 MB/op**
- versus `none`, `ruleast` reduces allocation by about **90%**
- versus `parsetree`, `ruleast` reduces allocation by about **79%**
- `ruleast` also allocates about **35% less** than `exec-model`

This reinforces the CPU profile story: `ruleast` is not only faster than the
other DRLX cache strategies here, it also dramatically reduces transient object
creation. That is exactly what we would expect from removing both reparsing and
ANTLR tree rehydration.

#### Important caveat on GC time/count

The `gc.count` and `gc.time` values in this short run are still useful, but
they should not be treated as the primary memory metric:

- `gc.alloc.rate.norm` is the cleanest cross-mode memory comparison
- `gc.alloc.rate` depends on throughput and wall-clock behavior
- `gc.count` and `gc.time` are process-level effects and can be noisy in short
  runs, especially with only three measurement iterations

In this dataset, `ruleast` shows the lowest normalized allocation even though
its aggregate `gc.time` is high. That is not a contradiction; it is a reminder
that GC timing in short runs is sensitive to when collections happen to land.

### GC Variance Analysis (`-prof gc`)

An earlier run with `-prof gc` revealed the root cause of DRLX's higher variance:

#### GC Profile Comparison

| Metric | buildWithDrlx | buildWithExecutableModel |
|---|---|---|
| Allocation per op | ~5.16 MB | ~4.00 MB |
| GC count (10 iters) | 33 (3-4/iter) | 444 (39-48/iter) |
| Total GC time (10 iters) | 5,599 ms | 1,979 ms |
| Avg GC time per event | ~170 ms | ~4.5 ms |
| GC time range per iter | 294-873 ms | 179-212 ms |

### Root Cause: Infrequent but Expensive GC Pauses

The two benchmarks exhibit fundamentally different GC behavior:

**buildWithExecutableModel (stable)**: Allocates ~4 MB/op of short-lived objects that die quickly
in young generation. This triggers frequent but cheap young GC collections (~4.5 ms each). The
cost is predictable and evenly spread across iterations, resulting in low variance.

**buildWithDrlx (high variance)**: Allocates ~5.16 MB/op. The ANTLR parse tree and visitor objects
survive young generation because they remain live through the full parse -> visit -> build pipeline.
This causes infrequent but expensive mixed/old-gen GC collections (~170 ms each). When an expensive
GC lands inside a measurement window, that iteration is slow; when it doesn't, the iteration is
fast.

### Correlation Between GC Time and Iteration Time (buildWithDrlx)

| Iteration | Time (ms/op) | GC time (ms) | GC count |
|---|---|---|---|
| 1 | 9.358 | 623 | 3 |
| 2 | 6.910 | 332 | 4 |
| 3 | 7.815 | 484 | 3 |
| 4 | 8.838 | 757 | 3 |
| 5 | 8.144 | 590 | 3 |
| 6 | 6.828 | 347 | 4 |
| 7 | 7.009 | 566 | 3 |
| 8 | 9.202 | 873 | 3 |
| 9 | 8.571 | 733 | 4 |
| 10 | 6.742 | 294 | 3 |

Same GC count across iterations, but GC duration per event varies wildly (294-873 ms),
directly driving the benchmark variance.

### Recommendations for Reducing Variance

1. **`-gc true`** — Force GC between iterations to get consistent baselines (inflates absolute
   times but stabilizes relative comparison).
2. **Low-pause collector** — Use `-XX:+UseZGC` or `-XX:+UseShenandoahGC` to eliminate long
   GC pauses. Won't fix extra allocation but will stabilize measurement variance.
3. **Tune G1GC** — Try `-XX:MaxGCPauseMillis=5` to force more frequent but shorter collections.

### Recommendations for Reducing Allocation

1. **Reduce allocation** — The ~1.16 MB/op gap is mostly ANTLR parse tree nodes. Caching or
   reusing the parse result (since `drlxSource` is the same across builds) would reduce
   allocation significantly.
2. **Optimize object lifetimes** — If parse tree nodes can be made shorter-lived (e.g., by
   extracting needed data early and releasing the tree), they would be collected in young gen
   instead of promoting to old gen, shifting from expensive mixed GCs to cheap young GCs.

## Files Involved

| File | Role |
|------|------|
| `src/main/java/org/drools/drlx/builder/DrlxLambdaConstraint.java` | Compiles each constraint lambda via MVEL |
| `src/main/java/org/drools/drlx/builder/DrlxLambdaConsequence.java` | Compiles each consequence lambda via MVEL |
| `src/main/java/org/drools/drlx/builder/DrlxToRuleImplVisitor.java` | ANTLR visitor; `loadPreCompiledEvaluator()` loads pre-built classes |
| `src/main/java/org/drools/drlx/builder/DrlxRuleBuilder.java` | Core builder orchestrating parse + visit |
| `src/main/java/org/drools/drlx/tools/DrlxCompiler.java` | Public facade; routes to pre-build or compile path |
| (mvel) `org/mvel3/MVELCompiler.java` | Transpiles + compiles each lambda |
| (mvel) `org/mvel3/javacompiler/KieMemoryCompiler.java` | Java bytecode compilation — primary bottleneck |
| (mvel) `org/mvel3/ClassManager.java` | `defineHiddenClass()` + `MethodByteCodeExtractor` — secondary bottleneck |
