## Design — MVEL3 LambdaRegistry refactor + DRLX boundary cleanup (mvel/mvel#428)

**Issue:** [mvel/mvel#428](https://github.com/mvel/mvel/issues/428) (MVEL side) + new DRLX issue to be created under Epic #26 (DRLX side)
**Epic (DRLX side):** #26
**Date:** 2026-05-12
**Scope:** Two-repo refactor. MVEL3: split `LambdaRegistry` enum into single-purpose components, replace the file format, eliminate static-init side effects. DRLX: consume MVEL's new `ArtifactRef` boundary, drop the cross-repo dependency on `physicalId`.
**Source plan:** `mvel/LambdaRegistry_Refactor.md` (Codex). This spec captures the design after multi-round refinement with the user.

## Motivation

`org.mvel3.lambdaextractor.LambdaRegistry` is a 310-line enum singleton that owns nine responsibilities at once: in-memory dedup, subtype-overload reuse, logical/physical ID allocation, classfile path tracking, registry-file serialization/deserialization, persisted-file cleanup, and a sys-prop-driven static initializer. That conflation is the local problem.

The bigger problem is cross-repo. DRLX persists `(fqn, physicalId, expression)` in `drlx-lambda-metadata.properties` and reads it back at runtime build, calling `LambdaRegistry.INSTANCE.getPhysicalPath(physicalId)` to translate `physicalId` into a classfile path. So `physicalId` — MVEL's internal dedup identity — has become a load-bearing cross-repo persistence API. DRLX's metadata is useless without MVEL's registry file. Two persisted files coupled by an internal foreign key.

The refactor cleans both layers in one window: split `LambdaRegistry`'s responsibilities into four narrow components inside MVEL, and replace `physicalId` in DRLX metadata with an `ArtifactRef(fqn, classFile)` so DRLX's metadata is self-sufficient. Both repos are experimental — no backward compatibility for persisted files.

## Scope

**In scope (MVEL3)**
- New types in `org.mvel3.lambdaextractor`: `ArtifactRef`, `RegistrationResult`, `LambdaCatalog`, `LambdaPersistenceManager`, `LambdaArtifactStore`, `LambdaArtifactLoader`, `LambdaRuntime`, plus internals `LambdaRegistryStore`, `LambdaPersistenceSnapshot`, `CatalogSnapshot`, `CatalogEntry`, `RuntimeConfig`, `InvalidLambdaRegistryException`.
- Final end-state: `LambdaRegistry` deleted outright (no facade left behind). During the migration it temporarily survives as a thin facade through Phases 1–5; Phase 6 removes it.
- Rewrite `MVELCompiler` and `MVELBatchCompiler` persistence flow to share a single decision model (`register → query → load-or-compile → attach`). Compile orchestration stays in the compilers.
- Add `MVELBatchCompiler.getArtifactRef(LambdaHandle)`. Drop `getPhysicalId(handle)` from the DRLX-facing boundary (may be retained package-private if MVEL-internal callers need it).
- Replace the registry file format with Properties (`format.version=2`).
- Replace the static initializer with lazy init via `LambdaRuntime.getInstance()`.
- Add Phase 0 characterization tests M1–M13.

**In scope (DRLX)**
- Rewrite `DrlxLambdaMetadata` (Properties format `v2`; `LambdaEntry(fqn, classFile, expression)`).
- Update `DrlxPreBuildLambdaCompiler.compileBatch` to record `ArtifactRef` from the batch compiler.
- Update `DrlxLambdaCompiler.loadPreCompiledEvaluator(ArtifactRef)` to use `LambdaArtifactLoader`. Drop all calls to `LambdaRegistry.INSTANCE`.
- Migrate `DrlxRuleBuilder` + `DrlxCompiler` constant accesses to `LambdaRuntime.isPersistenceEnabled()` / `defaultPersistencePath()` (transitional static accessors).
- New `InvalidDrlxLambdaMetadataException`.
- Add Phase 0 characterization tests D1–D7. D7 is an architectural guard.

**Out of scope**
- Removing the singleton entirely. `LambdaRuntime.getInstance()` is transitional infrastructure, not the permanent public model. A later cleanup would inject `LambdaRuntime` through compiler constructors.
- Changing lambda identity semantics (`LambdaKey`, normalization, hash strategy).
- Changing `DrlxMetadataMismatchMode` policy. Behavior preserved.
- Backward-compatible reads of old `lambda-registry.dat` (`v1`) or old `drlx-lambda-metadata.properties` (no `format.version`). Both formats die.
- Moving the actual compile call (`KieMemoryCompiler.compileAndPersist`) into `LambdaPersistenceManager`. PM is a service layer; compile orchestration remains in `MVELCompiler` / `MVELBatchCompiler`.
- An abstract `artifactId` (instead of `Path classFile`). Path-based is the first cut per Codex's Decision 2.

## Design decisions

| # | Decision | Rationale |
|---|---|---|
| D1 | Execute the full Codex plan in one development window (the "Revised Preferred Sequence" + Phase 5 + Phase 6) | Both repos experimental; lockstep change avoids transitional formats and quasi-public physicalId. |
| D2 | `LambdaPersistenceManager` owns the `physicalId → ArtifactRef` map | Codex's confirmation. Keeps Catalog focused on dedup and ArtifactStore stateless. |
| D3 | `LambdaPersistenceManager` is service-layer, NOT a compile wrapper | Compilers keep the `register → query → load-or-compile → attach` sequence. PM provides queries/state, not `compileOrLoad`. |
| D4 | Persisted file format: `java.util.Properties` with `format.version=2` | JDK-built-in, readable, no new deps, handles escaping. |
| D5 | Lifecycle: lazy init on first `LambdaRuntime.getInstance()` | Removes static-init side effects without forcing every caller to invoke an explicit init. |
| D6 | DRLX develops directly on `main` against locally-installed MVEL SNAPSHOT | User preference. Cross-repo cutover handled at push-time, not branch-time. |
| D7 | Add Codex's full Phase 0 test list as part of this window | Existing test coverage is thin (6 MVEL + 4 DRLX files); refactor needs a safety net. |
| D8 | `LambdaCatalog.register(LambdaKey)` returns `RegistrationResult(int logicalId, int physicalId, boolean reused)` — no `fqn` | Catalog owns identity/dedup only. Compilers build `ArtifactRef` themselves from their requested FQN + path. |
| D9 | DRLX-facing MVEL surface is exactly: `ArtifactRef`, `LambdaArtifactLoader.loadOrDefinePersistedClass`, `LambdaRuntime.isPersistenceEnabled()` / `defaultPersistencePath()`, `MVELBatchCompiler.getArtifactRef(handle)` | Narrow, greppable. Test D7 enforces. |
| D10 | Distinct exception types per repo: `InvalidLambdaRegistryException` (MVEL) and `InvalidDrlxLambdaMetadataException` (DRLX) | No shared exception type to keep the cross-repo dep surface minimal. |
| D11 | Duplicate `physicalId` on load → malformed, throw | No last-write-wins. Strict fail on either duplicate catalog or duplicate artifact mapping. |
| D12 | Reuse verification uses an explicit `compileInvocationCount()` seam on the compilers | Test-only instrumentation, not part of the long-term compiler API. Stable vs filesystem mtime. |
| D13 | `attachArtifact()` triggers `LambdaRuntime.persistSnapshot()` synchronously when persistence is enabled | Matches today's save-on-each-attach behavior. Batched save at end-of-batch deferred as a future optimization (today's TODO at `LambdaRegistry.java:161`). |
| D14 | Persistence-disabled path bypasses catalog and PM entirely — pure in-memory compile, no dedup | Matches today's `MVELCompiler.compileEvaluatorClass` (non-persistence) branch. With persistence off there's nothing to dedup against (no on-disk artifacts to reuse). |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        LambdaRuntime                            │
│  (composition root; RuntimeConfig from system properties;       │
│   lazy init via getInstance(); reset() + resetAndRemoveAll())   │
│                                                                 │
│  ┌───────────────┐  ┌─────────────────────┐  ┌────────────────┐ │
│  │ LambdaCatalog │  │ LambdaPersistence   │  │ LambdaArtifact │ │
│  │               │  │   Manager           │  │   Store        │ │
│  │ key↔physical  │  │ physical↔Artifact   │  │  exists(ref)   │ │
│  │ subtype       │  │  artifactFor()      │  │  readBytes(ref)│ │
│  │  overload     │  │  artifactExists()   │  │  deleteAll()   │ │
│  │ logical IDs   │  │  attachArtifact()   │  │                │ │
│  └───────────────┘  └──────────┬──────────┘  └────────────────┘ │
│                                │                                │
│                       ┌────────┴──────────────┐                 │
│                       │ LambdaRegistryStore   │ ← internal      │
│                       │  load()/save()        │                 │
│                       │  Properties v2        │                 │
│                       └───────────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
              ▲                                ▲
              │ register/query                 │ same model,
   ┌──────────┴──────┐                 ┌──────┴────────────┐
   │  MVELCompiler   │                 │ MVELBatchCompiler │
   │  (orchestration)│                 │  (bulk compile)   │
   └─────────────────┘                 └─────────┬─────────┘
                                                 │ getArtifactRef(handle)
                                                 ▼
                            ┌────────────────────────────────────┐
                            │ DRLX consumes only:                │
                            │  - ArtifactRef                     │
                            │  - LambdaArtifactLoader (static)   │
                            │  - LambdaRuntime.isPersistenceEnabled() │
                            │  - LambdaRuntime.defaultPersistencePath() │
                            │  - MVELBatchCompiler.getArtifactRef(h)│
                            └────────────────────────────────────┘
```

## MVEL3 — component API

```java
// Public, persistence-facing value objects
public record ArtifactRef(String fqn, Path classFile) {}
public record RegistrationResult(int logicalId, int physicalId, boolean reused) {}

// Composition root + lifecycle
public final class LambdaRuntime {
    public static LambdaRuntime getInstance();              // transitional lazy-init singleton
    public LambdaCatalog catalog();
    public LambdaPersistenceManager persistenceManager();

    // Internal/compiler-facing seam. Exposed for compiler-flow access to persistence config
    // (e.g., the top-level config.persistenceEnabled() branch). Read-only.
    // Not intended as long-term external API; same status as getInstance().
    public RuntimeConfig config();

    // Internal/compiler-facing seam. Called by LambdaPersistenceManager.attachArtifact()
    // to flush the snapshot synchronously. No-op when config.persistenceEnabled() is false.
    // Save-per-attach matches today's behavior; batched flushing (deferred save at end of
    // compile batch) is a future optimization, tracked by today's TODO at LambdaRegistry.java:161.
    // Not intended as long-term external API.
    public void persistSnapshot();

    public void reset();                                    // in-memory state only
    public void resetAndRemoveAllPersistedFiles();          // in-memory + classfiles + registry file

    // Transitional static accessors used by DRLX:
    public static boolean isPersistenceEnabled();
    public static Path defaultPersistencePath();

    // Test-only hook:
    static void resetSingletonForTests();                   // clears the cached instance
}

// In-memory dedup
public final class LambdaCatalog {
    public RegistrationResult register(LambdaKey key);
    // internals: entriesByKey, hashToKeys, logicalToPhysical, nextPhysicalId, nextLogicalId
    // applySnapshot(CatalogSnapshot), clear(), toSnapshot() — used by LambdaRuntime
}

// Artifact-association service
public final class LambdaPersistenceManager {
    public Optional<ArtifactRef> artifactFor(int physicalId);
    public boolean artifactExists(int physicalId);         // map has ref AND artifactStore.exists(ref)
    public void attachArtifact(int physicalId, ArtifactRef artifact);  // triggers LambdaRuntime.persistSnapshot() synchronously
    // applyArtifacts(Map<Integer, ArtifactRef>), clear(), snapshot() — used by LambdaRuntime
}

// Dumb byte I/O
public final class LambdaArtifactStore {
    public LambdaArtifactStore(Path persistenceRoot);       // root used only for deleteAll()
    public boolean exists(ArtifactRef ref);                 // Files.exists(ref.classFile())
    public byte[] readBytes(ArtifactRef ref) throws IOException;
    public void deleteAll() throws IOException;
}

// Stateless static helper — the DRLX-facing class-load utility
public final class LambdaArtifactLoader {
    public static Class<?> loadOrDefinePersistedClass(ClassManager cm, ArtifactRef ref) throws IOException {
        if (cm.getClasses().containsKey(ref.fqn())) {
            return cm.getClass(ref.fqn());                  // idempotent: skip duplicate define
        }
        byte[] bytes = Files.readAllBytes(ref.classFile());
        cm.define(Map.of(ref.fqn(), bytes));
        return cm.getClass(ref.fqn());
    }
}

// Internal to LambdaRuntime — not visible to compilers
final class LambdaRegistryStore {
    LambdaPersistenceSnapshot load() throws IOException;
    void save(LambdaPersistenceSnapshot snapshot) throws IOException;
}
record LambdaPersistenceSnapshot(CatalogSnapshot catalog, Map<Integer, ArtifactRef> artifacts) {}
record CatalogSnapshot(int nextPhysicalId, int nextLogicalId, List<CatalogEntry> entries) {}
record CatalogEntry(int physicalId, String methodSignature, String normalizedBody) {}

final class RuntimeConfig {
    static RuntimeConfig fromSystemProperties();
    boolean persistenceEnabled();
    Path persistenceRoot();
    Path registryFile();
    boolean resetOnTestStartup();
}

// Typed exception for malformed/unsupported registry files
public final class InvalidLambdaRegistryException extends IOException { /* ... */ }
```

### `LambdaRuntime.initialize()` (package-private, called by `getInstance()`)

```java
void initialize() {
    if (config.resetOnTestStartup()) {
        resetAndRemoveAllPersistedFiles();              // skip load entirely
        return;
    }
    if (config.persistenceEnabled() && Files.exists(config.registryFile())) {
        LambdaPersistenceSnapshot snapshot = registryStore.load();
        catalog.applySnapshot(snapshot.catalog());
        persistenceManager.applyArtifacts(snapshot.artifacts());
    }
}
```

Note: `resetOnTestStartup` now fires on first `getInstance()` (was: on `LambdaRegistry` class load). Behavior is unchanged for typical callers; tests combining `resetOnTestStartup=true` with `resetSingletonForTests()` will reset multiple times in one JVM — expected.

### Singleton + system-property mutation in tests

Cached singleton means that toggling `mvel3.compiler.lambda.persistence`, `...persistence.path`, `...registry.file`, or `...resetOnTestStartup` after first `getInstance()` does not take effect.

Production constraint: do not mutate these system properties after first use.

Test constraint: tests that vary config must call `LambdaRuntime.resetSingletonForTests()` between setups before the next `getInstance()`.

### Compiler flow (shared decision model)

The flow has a top-level branch on `config.persistenceEnabled()`. The persistence-disabled path bypasses the catalog and PM entirely — no in-memory registration, no dedup, no artifact tracking. This matches today's `MVELCompiler.compileEvaluatorClass` (non-persistence) vs `compileEvaluatorClassWithPersistence` (persistence) split.

```java
LambdaRuntime rt = LambdaRuntime.getInstance();

if (!rt.config().persistenceEnabled()) {
    // Pure in-memory compile. No catalog, no PM, no registry-file write,
    // no classfile persistence. Matches today's compileEvaluatorClass branch.
    Class<?> clazz = compileInMemory(classManager, classLoader, unit, fqn);
    return wrapEvaluator(clazz);
}

// Persistence-enabled path: register, query, load-or-compile-and-attach.
RegistrationResult reg = rt.catalog().register(lambdaKey);
String renamedFqn = applyPhysicalIdToFqn(originalFqn, reg.physicalId());

if (rt.persistenceManager().artifactExists(reg.physicalId())) {
    // In-session cache check is folded into LambdaArtifactLoader:
    //   it calls classManager.getClasses().containsKey(fqn) before redefining.
    ArtifactRef ref = rt.persistenceManager().artifactFor(reg.physicalId()).orElseThrow();
    Class<?> clazz = LambdaArtifactLoader.loadOrDefinePersistedClass(classManager, ref);
    return wrapEvaluator(clazz);
}

// Fresh compile path. The compiler owns the actual compile call:
//   MVELCompiler:      single-lambda KieMemoryCompiler.compileAndPersist
//   MVELBatchCompiler: bulk compileAndPersist, then per-handle attach loop
Path classFile = compileAndPersist(classManager, classLoader, unit, renamedFqn,
                                   rt.config().persistenceRoot());
ArtifactRef ref = new ArtifactRef(renamedFqn, classFile);
rt.persistenceManager().attachArtifact(reg.physicalId(), ref);
// attachArtifact triggers rt.persistSnapshot() synchronously — registry file written here.
return wrapEvaluator(classManager.getClass(renamedFqn));
```

Same decision model in both compilers; different compile shape (single vs bulk). `MVELBatchCompiler` runs the load-or-attach decision per-handle in a loop after the single bulk `compileAndPersist` call.

### Test-only compiler instrumentation

```java
public final class MVELCompiler {
    /** Test-only instrumentation. NOT part of the long-term public compiler API.
     *  Bumped only on the actual compile path, not on reuse. */
    public int compileInvocationCount();
}
public final class MVELBatchCompiler {
    /** Test-only. Bumped once per actual bulk compile invocation. */
    public int compileInvocationCount();
}
```

`public` visibility because DRLX-side test D1 needs to observe this from a different package. The javadoc labels them clearly as test-only.

To bridge from DRLX tests to MVEL's compile counter:

```java
public class DrlxRuleBuilder {
    /** Test-only accessor. Allows tests to inspect MVELBatchCompiler.compileInvocationCount(). */
    public MVELBatchCompiler getBatchCompilerForTests();
}
```

Used by M9 / M10 (MVEL side) and D1 (DRLX side) to assert reuse-vs-recompile semantics without relying on filesystem mtime.

## DRLX — boundary consumption

```java
public class DrlxLambdaMetadata {
    private static final String FILE_NAME = "drlx-lambda-metadata.properties";
    private static final String FORMAT_VERSION = "2";

    public record LambdaEntry(String fqn, Path classFile, String expression) {
        public ArtifactRef toArtifactRef() { return new ArtifactRef(fqn, classFile); }
    }

    public void put(String ruleName, int counterId, ArtifactRef ref, String expression);
    public LambdaEntry get(String ruleName, int counterId);
    public void save(Path dir) throws IOException;
    public static DrlxLambdaMetadata load(Path file)
        throws IOException, InvalidDrlxLambdaMetadataException;
}
```

### `DrlxLambdaCompiler.loadPreCompiledEvaluator` after refactor

```java
protected Object loadPreCompiledEvaluator(ArtifactRef ref) throws Exception {
    Class<?> clazz = loadedClassCache.get(ref.fqn());
    if (clazz == null) {
        if (preBuildClassManager == null) preBuildClassManager = new ClassManager();
        clazz = LambdaArtifactLoader.loadOrDefinePersistedClass(preBuildClassManager, ref);
        loadedClassCache.put(ref.fqn(), clazz);
    }
    return clazz.getConstructor().newInstance();
}
```

If `ref.classFile()` does not exist, `Files.readAllBytes` inside `LambdaArtifactLoader` throws `IOException`. `tryLoadPreCompiled` already catches `Exception` and routes through `handleMetadataMismatch` (FAIL_FAST / FALLBACK). Behavior is identical to today's missing-physicalPath case — same control flow, different trigger.

`loadedClassCache` and `preBuildClassManager` remain DRLX-local concerns.

### `DrlxPreBuildLambdaCompiler.compileBatch` after refactor

```java
@Override
public void compileBatch(ClassLoader classLoader) {
    super.compileBatch(classLoader);
    for (PendingPreBuildInfo info : pendingPreBuildInfos) {
        ArtifactRef ref = batchCompiler.getArtifactRef(info.handle());
        metadata.put(info.ruleName(), info.counterId(), ref, info.expression());
    }
    pendingPreBuildInfos.clear();
}
```

`PendingPreBuildInfo` loses its `fqn` field — it's now obtained via `getArtifactRef`.

### Constant migration

```java
// DrlxRuleBuilder:24, DrlxRuleBuilder:128
- import org.mvel3.lambdaextractor.LambdaRegistry;
+ import org.mvel3.lambdaextractor.LambdaRuntime;
- Path persistDir = LambdaRegistry.PERSISTENCE_ENABLED ? LambdaRegistry.DEFAULT_PERSISTENCE_PATH : null;
+ Path persistDir = LambdaRuntime.isPersistenceEnabled() ? LambdaRuntime.defaultPersistencePath() : null;

// DrlxCompiler:12, DrlxCompiler:50, DrlxCompiler:79 — same pattern
```

### Metadata mismatch / absence distinction

| Condition | Treatment |
|---|---|
| Metadata file missing | **Not a mismatch.** Empty metadata returned. DRLX compiles fresh, no warning. |
| `format.version` missing or `!= "2"` | **Mismatch.** Throws `InvalidDrlxLambdaMetadataException`. Routed through `DrlxMetadataMismatchMode`. |
| Required keys missing or unparseable | **Mismatch.** Same routing. |
| Entry exists but expression differs from rule | **Mismatch.** Existing logic in `tryLoadPreCompiled` — unchanged. |
| Entry exists but `classFile` deleted on disk | **Mismatch.** `LambdaArtifactLoader` throws `IOException`; caught in `tryLoadPreCompiled` and routed. |

Principle: *absence is not mismatch; corruption or staleness is.*

## File formats (Properties, `format.version=2`)

### MVEL `lambda-registry.dat`

```properties
format.version=2

catalog.nextPhysicalId=2
catalog.nextLogicalId=3

catalog.entry.0.physicalId=0
catalog.entry.0.methodSignature=public boolean eval(java.lang.Object obj)
catalog.entry.0.normalizedBody={ return obj != null; }

catalog.entry.1.physicalId=1
catalog.entry.1.methodSignature=public boolean eval(java.lang.Object obj, java.util.Map ctx)
catalog.entry.1.normalizedBody={ return ((Integer) ctx.get("age")) > 18; }

artifact.0.physicalId=0
artifact.0.fqn=org.mvel3.GeneratorEvaluator___0
artifact.0.classFile=/abs/path/.../GeneratorEvaluator___0.class

artifact.1.physicalId=1
artifact.1.fqn=org.mvel3.GeneratorEvaluator___1
artifact.1.classFile=/abs/path/.../GeneratorEvaluator___1.class
```

**Schema rules:**
- `catalog.entry.<N>.*` and `artifact.<N>.*` use 0-based iteration indexes. `physicalId` is stored explicitly inside each entry (not derived from the index) to tolerate future non-contiguous IDs.
- Properties value escaping (handles `=`, `:`, `\`, newlines, leading whitespace) is sufficient for `methodSignature` and `normalizedBody`. No Base64.
- Referential integrity: every `artifact.<N>.physicalId` must reference an existing `catalog.entry.<M>.physicalId`. Missing reference → malformed.
- Duplicate `physicalId` in catalog entries → malformed. Throw.
- Duplicate `physicalId` in artifact entries → malformed. Throw. (No last-write-wins.)
- Missing required keys → malformed. Throw.
- `format.version` missing or `!= "2"` → throw `InvalidLambdaRegistryException`.
- Missing file → empty registry (not an error).

### DRLX `drlx-lambda-metadata.properties`

```properties
format.version=2

rule.CheckAge1.0.expression=age > 18
rule.CheckAge1.0.fqn=org.mvel3.GeneratorEvaluator___0
rule.CheckAge1.0.classFile=/abs/path/.../GeneratorEvaluator___0.class

rule.CheckAge1.1.expression=name != null
rule.CheckAge1.1.fqn=org.mvel3.GeneratorEvaluator___1
rule.CheckAge1.1.classFile=/abs/path/.../GeneratorEvaluator___1.class
```

**Schema rules:**
- Key pattern: `rule.<ruleName>.<counterId>.{expression,fqn,classFile}`.
- `counterId` resets per rule, matching today's behavior.
- Missing file → empty metadata (not an error).
- `format.version` missing or `!= "2"` → throw `InvalidDrlxLambdaMetadataException`.
- Missing required keys → malformed, throw same.

## Phase 0 characterization tests

### MVEL3 — `src/test/java/org/mvel3/lambdaextractor/`

| # | Test name | Asserts |
|---|---|---|
| M1 | `dedup_exactKey_reusesPhysicalId` | Same `LambdaKey` twice → same `physicalId`, `reused=true`. |
| M2 | `dedup_subtypeOverload_supertypeFirst_subtypeReuses` | Supertype registered first; subtype call reuses. |
| M3 | `dedup_subtypeOverload_subtypeFirst_supertypeDoesNotReuse` | Subtype registered first; supertype does not collapse. |
| M4 | `hashCollision_distinctKeys_keepSeparatePhysicalIds` | Same `hashCode()`, distinct content → distinct `physicalId`. |
| M5 | `registryStore_writeReadRoundTrip` | `save(snapshot)` then `load()` → snapshots equal. |
| M6 | `registryStore_unsupportedVersion_throws` | `format.version=1` → `InvalidLambdaRegistryException`. |
| M7 | `registryStore_malformedEntry_throws` | Missing `physicalId`, duplicate `physicalId`, dangling artifact ref → throws. |
| M8 | `compiler_freshLambda_persistsAndAttaches` | `MVELCompiler` for new lambda: classfile exists at `ref.classFile()`; registry file written by the `attachArtifact → persistSnapshot` chain (synchronous save). |
| M9 | `compiler_knownLambda_reusesPersisted` | Second compile of same lambda: `compileInvocationCount` unchanged. |
| M10 | `batchCompiler_mixed_freshAndKnown` | Batch with N new + M known: `compileInvocationCount` advances by 1 (one bulk compile call); new ones persisted, known ones loaded. |
| M11 | `runtime_persistenceDisabled_noFileWrites` | `mvel3.compiler.lambda.persistence=false` → no registry file, no classfiles persisted; catalog and PM remain empty (compiler bypasses both entirely). Uses `resetSingletonForTests()`. |
| M12 | `runtime_resetAndRemoveAll_cleansEverything` | Generated classes + registry file deleted; catalog + PM in-memory empty. |
| M13 | `runtime_lazyInit_loadsExistingFile` | Pre-existing valid registry file is loaded on first `getInstance()`. Uses `resetSingletonForTests()`. |

### DRLX — `drlx-parser-core/src/test/java/org/drools/drlx/builder/`

| # | Test name | Asserts |
|---|---|---|
| D1 | `preBuildThenRuntime_loadsWithoutCompile` | Pre-build writes metadata + classfiles; second build with same DRLX picks up evaluators without re-compiling. Asserts via `DrlxRuleBuilder.getBatchCompilerForTests().compileInvocationCount() == 0` after the runtime build (no compile occurred). |
| D2 | `metadata_missingFile_freshCompile` | No metadata file → DRLX compiles fresh, no error, no warning. |
| D3 | `metadata_unsupportedVersion_failFast` | `format.version=1` → `FAIL_FAST` throws, `FALLBACK` logs + compiles. |
| D4 | `metadata_staleClassFile_failFast` | Metadata references a deleted classfile → mismatch policy. |
| D5 | `metadata_expressionMismatch_failFast` | Expression differs from rule → existing behavior, preserved post-refactor. |
| D6 | `metadata_format_roundTrip` | `save()` then `load()` → entries equal. |
| D7 | `drlx_noLambdaRegistryImports` | **Architectural guard.** Grep test (or compile-time check via custom annotation processor) that asserts no DRLX source file imports `org.mvel3.lambdaextractor.LambdaRegistry` (deleted), `LambdaCatalog`, `LambdaPersistenceManager`, `LambdaArtifactStore`, or `LambdaRegistryStore`. Allowed: `ArtifactRef`, `LambdaArtifactLoader`, `LambdaRuntime`, `MVELBatchCompiler`. |

## Migration sequence

### Phase A — MVEL3 (branch `lambda-registry-refactor`)

1. Add MVEL Phase 0 tests M1–M13 (some target today's API; others target the new API and get enabled as the phases complete).
2. Phase 1: extract `LambdaCatalog`; keep `LambdaRegistry` as thin facade.
3. Phase 2: extract `LambdaRegistryStore`; switch to Properties `v2`. M5/M6/M7 turn green.
4. Phase 3: extract `LambdaArtifactStore`.
5. Phase 4: add `ArtifactRef`; add `LambdaPersistenceManager`. Facade routes through new components.
6. Phase 5: rewrite `MVELCompiler` + `MVELBatchCompiler` to use new components. Add `MVELBatchCompiler.getArtifactRef(handle)`. Add `compileInvocationCount()` test-only seam.
7. Phase 6: introduce `LambdaRuntime` composition root + lazy init. Delete `LambdaRegistry` outright. Add `LambdaArtifactLoader`. Add `LambdaRuntime.isPersistenceEnabled()` / `defaultPersistencePath()`. Add `resetSingletonForTests()`.
8. Local `mvn install` of MVEL3 3.0.0-SNAPSHOT after each meaningful checkpoint.

### Phase B — DRLX (working directly on `main` against locally-installed SNAPSHOT, commits stay local until Phase C)

9. Add DRLX Phase 0 tests D1–D7. Most can land before refactor and just track today's behavior.
10. Rewrite `DrlxLambdaMetadata` (Properties `v2`, drop `physicalId`, add `classFile`).
11. Update `DrlxPreBuildLambdaCompiler.compileBatch` to call `getArtifactRef`.
12. Update `DrlxLambdaCompiler.loadPreCompiledEvaluator(ArtifactRef)` + `tryLoadPreCompiled` call site.
13. Update `DrlxRuleBuilder` + `DrlxCompiler` to use `LambdaRuntime` static accessors. Remove all `LambdaRegistry` imports.
14. Delete `physicalId`-related code paths. Rewrite tests asserting via classFile/FQN reuse (not ID equality).
15. Run DRLX suite; must stay at or above 170 passing.

### Phase C — coordinated cutover (single window)

1. Final `mvn install` of MVEL SNAPSHOT.
2. Final DRLX test run, green against current MVEL branch.
3. Push MVEL `lambda-registry-refactor`; merge PR to MVEL `main`.
4. Push DRLX `main` commits.

CI on `main` of either repo never sees an inconsistent state. This is the cost of keeping DRLX off a feature branch — discipline at push-time.

## Risks

| # | Risk | Mitigation |
|---|---|---|
| R1 | Singleton + sys-prop coupling makes tests flaky | Mandatory `resetSingletonForTests()` between sys-prop mutations; documented constraint; M11/M13 use it. |
| R2 | DRLX `main` carries commits that don't compile against MVEL `main` until Phase C step 3 | Commits stay local until coordinated push; verified locally against installed SNAPSHOT. |
| R3 | Phase 0 tests rely on internals that the refactor reshapes (e.g., direct catalog access) | Tests written against the new API first; legacy `LambdaRegistry`-API tests migrated as the refactor progresses. |
| R4 | `ClassManager.define` semantics on duplicate FQN | `LambdaArtifactLoader` checks `cm.getClasses().containsKey(fqn)` first to avoid redefining. |
| R5 | Old persisted files break local dev workflows | `mvn clean` regenerates; no migration code. Spec'd as accepted cost. |
| R6 | Tests that assert exact numeric ID sequences | Rewrite to assert reuse-vs-difference via `RegistrationResult.reused` or via classFile/FQN equality. |
| R7 | Phase 5 rewrite touches both `MVELCompiler` and `MVELBatchCompiler` in the same window | Phase 0 M8/M9/M10 cover both compilers; refactor under the safety net. |

## References

- Source plan: `mvel/LambdaRegistry_Refactor.md` (Codex)
- MVEL files touched: `LambdaRegistry.java` (deleted), `MVELCompiler.java`, `MVELBatchCompiler.java`, plus new files under `org.mvel3.lambdaextractor`
- DRLX files touched: `DrlxLambdaMetadata.java`, `DrlxLambdaCompiler.java`, `DrlxPreBuildLambdaCompiler.java`, `DrlxRuleBuilder.java`, `DrlxCompiler.java`
- Today's `LambdaRegistry`: 310 LOC, 9 responsibilities (in-memory dedup, subtype overload, ID allocation, classfile path tracking, registry serialize/deserialize, cleanup, sys-prop static init, singleton accessor)
- Today's MVEL `LambdaRegistry` callers (in-MVEL): `MVELCompiler` (5 call sites), `MVELBatchCompiler` (3 call sites)
- Today's DRLX `LambdaRegistry` callers: `DrlxLambdaCompiler.loadPreCompiledEvaluator` (`getPhysicalPath`), `DrlxRuleBuilder` (`PERSISTENCE_ENABLED`, `DEFAULT_PERSISTENCE_PATH`), `DrlxCompiler` (same two constants)
