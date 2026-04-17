# Two Persistence Layers: `lambda-registry.dat` vs `drlx-lambda-metadata.properties`

## The two files

### `lambda-registry.dat` — MVEL layer

Location: `target/generated-classes/mvel/lambda-registry.dat` (configurable via `mvel3.compiler.lambda.registry.file`)

- **Key:** content-based — `LambdaKey` (method signature + normalized body, hashed)
- **Value:** `{physicalId, classFilePath, methodSignature, normalisedBody}`
- **Scope:** global, JVM-wide singleton (`LambdaRegistry.INSTANCE`); auto-loads from disk in a static init block
- **Purpose:** *physical class deduplication*. Two different rules (or runs, or projects) containing the same normalized expression (e.g. `speed > 80`) collapse to one `.class` file via the same `physicalId`.
- **Write trigger:** `MVELBatchCompiler.compile()` → `LambdaRegistry.INSTANCE.registerPhysicalPath(physicalId, path)` (`MVELBatchCompiler.java:103`)

### `drlx-lambda-metadata.properties` — drlx-parser layer

Location: `target/generated-classes/mvel/drlx-lambda-metadata.properties`

- **Key:** position-based — `ruleName.counterId` (e.g., `"MyRule.3"` = third lambda emitted while visiting rule `MyRule`)
- **Value:** `{fqn, physicalId, expression}`
- **Scope:** per-DRLX build output
- **Purpose:** *binds a DRLX source slot to a physicalId*. When `DrlxRuleAstRuntimeBuilder` walks the AST and reaches rule `Foo`'s lambda #3, it needs to know "go load physicalId 42" without recompiling the expression.

## Are they duplicated?

### Role-wise: no

They index on different axes:

| | LambdaRegistry | DrlxLambdaMetadata |
|--|--|--|
| Keyed by | lambda source content | source position (rule + counter) |
| Answers | "what file backs this code?" | "what lambda lives at rule Foo slot 3?" |
| Owner | MVEL | DRLX compiler |

Without `DrlxLambdaMetadata`, the runtime build would have to re-generate each lambda's MVEL source, normalize the body, hash it, and look it up in `LambdaRegistry` — saving only the `javac` step. The metadata file lets you skip source generation *and* hashing entirely: you jump straight `(ruleName, counter) → physicalId → classFile`.

### Data-wise: partially

Two of the three fields in the metadata are partly redundant with LambdaRegistry:

- `fqn` — derivable from `LambdaRegistry.getPhysicalPath(physicalId)` (path → fqn)
- `expression` — only used as a drift/safety check in `DrlxLambdaCompiler.tryLoadPreCompiled()` (`DrlxLambdaCompiler.java:190-195`); not load-bearing
- `physicalId` — the only field strictly required

A leaner design could store just `ruleName.counter → physicalId` in the metadata and let LambdaRegistry supply fqn/path. The current `expression` field is defensive (catches "source changed but metadata wasn't regenerated") — worth keeping only if you want that fail-fast check.

## 2-step build: does step 2 need `lambda-registry.dat`?

### Currently: yes — runtime build still touches it

Look at `DrlxLambdaCompiler.loadPreCompiledEvaluator()` (`DrlxLambdaCompiler.java:269-285`):

```java
Path classFilePath = LambdaRegistry.INSTANCE.getPhysicalPath(physicalId);
if (classFilePath == null) {
    throw new IllegalStateException("No persisted class file for physicalId " + physicalId);
}
byte[] bytes = Files.readAllBytes(classFilePath);
```

The runtime build asks LambdaRegistry for the `.class` file path. That path only exists in the in-memory registry because `LambdaRegistry`'s static init loaded `lambda-registry.dat` from disk (`LambdaRegistry.java:42-44`). Delete `lambda-registry.dat` and step 2 fails — `getPhysicalPath()` returns `null`.

### But it's not logically necessary

The path is mechanically reconstructible from `fqn` + the persistence directory. From `MVELBatchCompiler.java:95-97`:

```java
String relativePath = persistenceDir.relativize(persistedFile).toString();
String fileFqn = relativePath.replace('/', '.').replace(".class", "");
```

The path is literally `persistenceDir + fqn.replace('.', '/') + ".class"`. Since `DrlxLambdaMetadata` already stores `fqn`, step 2 could compute the path directly without the registry:

```java
Path classFilePath = persistenceDir.resolve(fqn.replace('.', '/') + ".class");
```

### What would you lose by dropping `lambda-registry.dat` in step 2?

1. **Cross-JVM physicalId stability.** If step 1 and step 2 run in the same JVM, `nextPhysicalId` is in-memory state. Across JVM restarts, the registry file reloads that counter so new lambdas don't collide with existing ones. For pure step-2 (load only, no new compilation), this doesn't matter.
2. **MVEL batch compiler fallback dedup.** If `FALLBACK` mode fires (metadata miss → recompile), `MVELBatchCompiler.add()` uses `LambdaRegistry` to detect "already persisted" lambdas and skip javac. Without the registry, every fallback lambda gets recompiled.
3. **The drift-detection `expression` field** in metadata is load-bearing only if you keep the safety check — it doesn't need the registry either.

## TL;DR

- `LambdaRegistry` = *what classes exist* (content-keyed, JVM-global).
- `DrlxLambdaMetadata` = *which class each DRLX rule slot uses* (position-keyed, per-build).
- They are complementary in role, not duplicates. But `fqn` and `expression` in the metadata partially duplicate LambdaRegistry data — they're shortcuts/safety checks, not logical requirements. Only `physicalId` is essential.
- In a **pure step-2 run with 100% metadata hits**, `lambda-registry.dat` is logically redundant — metadata's `fqn` + a known persistence dir is enough. The current code unnecessarily routes through the registry.
- Keep `lambda-registry.dat` around for (a) same-JVM dev flow where step 1 and step 2 share state, and (b) metadata-miss fallback where MVEL recompiles and wants to dedup against already-persisted classes. Drop either scenario and the registry file becomes dead weight at runtime.
