# MVEL3 Lambda Registry Refactor — Implementation Plan (Plan 1 of 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor MVEL3 `LambdaRegistry` into four single-purpose components (`LambdaCatalog`, `LambdaPersistenceManager`, `LambdaArtifactStore`, `LambdaRegistryStore`) under a `LambdaRuntime` composition root. Introduce `ArtifactRef(fqn, classFile)` and a static `LambdaArtifactLoader`. Replace the registry-file format with Properties `format.version=2`. Replace the static-initializer with lazy init. Delete `LambdaRegistry` outright. Add MVEL Phase 0 characterization tests M1–M13.

**Architecture:** Four narrow MVEL-internal components composed in `LambdaRuntime`. `MVELCompiler` / `MVELBatchCompiler` keep compile orchestration; PersistenceManager is service-layer (query + association store), not a compile wrapper. DRLX-facing seam is reduced to `ArtifactRef`, `LambdaArtifactLoader`, two static `LambdaRuntime` accessors, and `MVELBatchCompiler.getArtifactRef(handle)`. DRLX-side work is Plan 2.

**Tech Stack:** Java 17+, JUnit 5 (Jupiter), AssertJ, Maven, `KieMemoryCompiler` (existing). No new runtime deps.

**Source spec:** `/home/tkobayas/claude/public/drlx-parser/specs/2026-05-12-mvel-lambda-registry-refactor-design.md`

**Working directory:** `/home/tkobayas/usr/work/mvel3-development/mvel` (branch `lambda-registry-refactor`).

**Commit discipline:** TDD pacing (write-test → see-fail → implement → see-pass → commit). All commits reference `Refs mvel/mvel#428` in the body. After Phase 6 commits, MVEL3 SNAPSHOT must be locally `mvn install`-ed before Plan 2 (DRLX) work.

---

## File Structure

### Files created

| Path | Purpose |
|---|---|
| `src/main/java/org/mvel3/lambdaextractor/ArtifactRef.java` | Record `(String fqn, Path classFile)`. The only cross-repo persistence value object. |
| `src/main/java/org/mvel3/lambdaextractor/RegistrationResult.java` | Record `(int logicalId, int physicalId, boolean reused)`. Returned by `LambdaCatalog.register`. |
| `src/main/java/org/mvel3/lambdaextractor/LambdaCatalog.java` | In-memory dedup + ID allocation + subtype-overload reuse. No FS, no paths. |
| `src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceManager.java` | `physicalId↔ArtifactRef` map; `artifactFor`, `artifactExists`, `attachArtifact`. Triggers `LambdaRuntime.persistSnapshot()` on attach. |
| `src/main/java/org/mvel3/lambdaextractor/LambdaArtifactStore.java` | Dumb byte I/O: `exists(ref)`, `readBytes(ref)`, `deleteAll()`. Root path for `deleteAll` only. |
| `src/main/java/org/mvel3/lambdaextractor/LambdaArtifactLoader.java` | Stateless static helper: `loadOrDefinePersistedClass(ClassManager, ArtifactRef)`. DRLX-safe. |
| `src/main/java/org/mvel3/lambdaextractor/LambdaRuntime.java` | Composition root + lazy init + reset semantics + transitional static accessors + `resetSingletonForTests`. |
| `src/main/java/org/mvel3/lambdaextractor/LambdaRegistryStore.java` | Package-private. Properties v2 save/load. |
| `src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceSnapshot.java` | Record `(CatalogSnapshot catalog, Map<Integer, ArtifactRef> artifacts)`. |
| `src/main/java/org/mvel3/lambdaextractor/CatalogSnapshot.java` | Record `(int nextPhysicalId, int nextLogicalId, List<CatalogEntry> entries)`. |
| `src/main/java/org/mvel3/lambdaextractor/CatalogEntry.java` | Record `(int physicalId, String methodSignature, String normalizedBody)`. |
| `src/main/java/org/mvel3/lambdaextractor/RuntimeConfig.java` | Record-like: `persistenceEnabled`, `persistenceRoot`, `registryFile`, `resetOnTestStartup`. `fromSystemProperties()` factory. |
| `src/main/java/org/mvel3/lambdaextractor/InvalidLambdaRegistryException.java` | Typed `IOException` subclass for malformed / unsupported registry files. |
| `src/test/java/org/mvel3/lambdaextractor/LambdaCatalogTest.java` | M1–M4: dedup, subtype overload (×2), hash collision. |
| `src/test/java/org/mvel3/lambdaextractor/LambdaRegistryStoreTest.java` | M5–M7: round-trip, unsupported version, malformed entry. |
| `src/test/java/org/mvel3/lambdaextractor/MVELCompilerPersistenceTest.java` | M8–M11: fresh/known compile, batch mixed, persistence-disabled. |
| `src/test/java/org/mvel3/lambdaextractor/LambdaRuntimeTest.java` | M12–M13: reset-and-remove-all, lazy-init loads existing file. |

### Files modified

| Path | Change |
|---|---|
| `src/main/java/org/mvel3/MVELCompiler.java` | Rewrite persistence flow (~lines 202–308) to use `LambdaRuntime` + `LambdaCatalog` + `LambdaPersistenceManager` + `LambdaArtifactLoader`. Add `compileInvocationCount()` test-only accessor. |
| `src/main/java/org/mvel3/MVELBatchCompiler.java` | Rewrite persistence flow (~lines 60–120) similarly. Add `getArtifactRef(LambdaHandle)` public method. Add `compileInvocationCount()`. |
| `src/test/java/org/mvel3/lambdaextractor/LambdaRegistryTest.java` | Rewrite against `LambdaCatalog`. Drop direct numeric-ID assertions; assert reuse via `RegistrationResult.reused()`. |
| `src/test/java/org/mvel3/lambdaextractor/LambdaRegistryPersistenceTest.java` | Migrate to `LambdaRegistryStoreTest` (delete this file once tests are ported). |

### Files deleted

| Path | Reason |
|---|---|
| `src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java` | Final end-state of the refactor. Survives temporarily as a thin facade through Phases 1–5; Phase 6 removes it. |

---

## Phase ordering

| Task | Phase | Output | Verification |
|---|---|---|---|
| 1 | Branch setup | Confirm baseline green | Existing test suite passes |
| 2 | Phase 1 | `LambdaCatalog` extracted; `LambdaRegistry` is facade | M1–M4 green |
| 3 | Phase 2 | `LambdaRegistryStore` + Properties v2 + records | M5–M7 green; old format dies |
| 4 | Phase 3 | `LambdaArtifactStore` extracted | All previous tests still green |
| 5 | Phase 4 | `ArtifactRef` + `LambdaPersistenceManager`, facade rewired | All previous tests still green |
| 6 | Phase 5 | `MVELCompiler` + `MVELBatchCompiler` rewritten; `getArtifactRef` + `compileInvocationCount` added | M8–M10 green |
| 7 | Phase 6a | `LambdaRuntime` + `LambdaArtifactLoader` + `RuntimeConfig` | M11–M13 green |
| 8 | Phase 6b | Delete `LambdaRegistry` outright; migrate callers | Full suite green |
| 9 | Final | `mvn install` SNAPSHOT; hand off to Plan 2 | SNAPSHOT in local `.m2`; clean build |

---

## Task 1: Branch setup & baseline verification

**Files:** none (verification only)

- [ ] **Step 1.1: Confirm working tree state**

Run:
```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel status
git -C /home/tkobayas/usr/work/mvel3-development/mvel branch --show-current
```
Expected: `lambda-registry-refactor` branch; only untracked files (`LambdaRegistry_Refactor.md`, `HANDOVER.md`, etc.). No modified tracked files.

- [ ] **Step 1.2: Confirm baseline test suite passes**

Run:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -pl . 2>&1 | tail -20
```
Expected: `BUILD SUCCESS`, all tests pass. Capture the failing test count (should be 0) as the baseline.

- [ ] **Step 1.3: Note current LambdaRegistry test count for reference**

```bash
grep -c "@Test" /home/tkobayas/usr/work/mvel3-development/mvel/src/test/java/org/mvel3/lambdaextractor/LambdaRegistryTest.java
grep -c "@Test" /home/tkobayas/usr/work/mvel3-development/mvel/src/test/java/org/mvel3/lambdaextractor/LambdaRegistryPersistenceTest.java
```
Expected: 5 + 1 = 6 tests. Note this number; target is 6 → ≥19 (6 original behavior preserved or migrated + 13 new M-tests).

---

## Task 2: Phase 1 — Extract `LambdaCatalog`

**Files:**
- Create: `src/main/java/org/mvel3/lambdaextractor/RegistrationResult.java`
- Create: `src/main/java/org/mvel3/lambdaextractor/LambdaCatalog.java`
- Create: `src/test/java/org/mvel3/lambdaextractor/LambdaCatalogTest.java`
- Modify: `src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java` — delegate dedup/ID work to `LambdaCatalog`

- [ ] **Step 2.1: Create `RegistrationResult` record**

`src/main/java/org/mvel3/lambdaextractor/RegistrationResult.java`:
```java
package org.mvel3.lambdaextractor;

/**
 * Result of {@link LambdaCatalog#register(LambdaKey)}.
 * <p>
 * {@code reused} is true if this registration matched an existing entry via
 * exact-key or subtype-overload reuse.
 */
public record RegistrationResult(int logicalId, int physicalId, boolean reused) {}
```

- [ ] **Step 2.2: Write `LambdaCatalogTest` M1 — exact-key dedup**

`src/test/java/org/mvel3/lambdaextractor/LambdaCatalogTest.java`:
```java
package org.mvel3.lambdaextractor;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mvel3.lambdaextractor.LambdaUtils.createLambdaKeyFromMethodDeclarationString;

class LambdaCatalogTest {

    private LambdaCatalog catalog;

    @BeforeEach
    void setup() {
        catalog = new LambdaCatalog();
    }

    @Test
    void M1_dedup_exactKey_reusesPhysicalId() {
        String method = "public boolean eval(org.example.Person p) { return p.getAge() > 20; }";
        LambdaKey k1 = createLambdaKeyFromMethodDeclarationString(method);
        LambdaKey k2 = createLambdaKeyFromMethodDeclarationString(method);

        RegistrationResult r1 = catalog.register(k1);
        RegistrationResult r2 = catalog.register(k2);

        assertThat(r1.reused()).isFalse();
        assertThat(r2.reused()).isTrue();
        assertThat(r2.physicalId()).isEqualTo(r1.physicalId());
        assertThat(r2.logicalId()).isNotEqualTo(r1.logicalId());
    }
}
```

- [ ] **Step 2.3: Run M1 — expect compile failure (`LambdaCatalog` does not exist yet)**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test-compile 2>&1 | tail -5
```
Expected: failure mentioning `LambdaCatalog`.

- [ ] **Step 2.4: Create `LambdaCatalog`**

`src/main/java/org/mvel3/lambdaextractor/LambdaCatalog.java`:
```java
package org.mvel3.lambdaextractor;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * In-memory lambda dedup + ID allocation. Pure: no filesystem access,
 * no parsing of persisted files, no system-property reads.
 * <p>
 * Subtype-overload reuse: a new {@link LambdaKey} whose parameter types are
 * subtypes of an existing key's parameters reuses the existing physical ID.
 */
public final class LambdaCatalog {

    private final Map<LambdaKey, Integer> physicalIdByKey = new HashMap<>();
    private final Map<Integer, List<LambdaKey>> keysByHash = new HashMap<>();
    private final Map<Integer, Integer> physicalByLogical = new HashMap<>();
    private int nextPhysicalId = 0;
    private int nextLogicalId = 0;

    public RegistrationResult register(LambdaKey key) {
        int logicalId = nextLogicalId++;

        Integer existingExact = physicalIdByKey.get(key);
        if (existingExact != null) {
            physicalByLogical.put(logicalId, existingExact);
            return new RegistrationResult(logicalId, existingExact, true);
        }

        Integer reusedSubtype = findSubtypeOverloadReuse(key);
        if (reusedSubtype != null) {
            physicalIdByKey.put(key, reusedSubtype);
            physicalByLogical.put(logicalId, reusedSubtype);
            return new RegistrationResult(logicalId, reusedSubtype, true);
        }

        int physicalId = nextPhysicalId++;
        physicalIdByKey.put(key, physicalId);
        keysByHash.computeIfAbsent(key.hashCode(), h -> new ArrayList<>()).add(key);
        physicalByLogical.put(logicalId, physicalId);
        return new RegistrationResult(logicalId, physicalId, false);
    }

    private Integer findSubtypeOverloadReuse(LambdaKey key) {
        List<LambdaKey> candidates = keysByHash.get(key.hashCode());
        if (candidates == null) return null;
        for (LambdaKey target : candidates) {
            if (!target.getNormalisedBody().equals(key.getNormalisedBody())) continue;
            LambdaKey.MethodSignatureInfo targetInfo = target.getMethodSignatureInfo();
            LambdaKey.MethodSignatureInfo currentInfo = key.getMethodSignatureInfo();
            if (!targetInfo.returnType.equals(currentInfo.returnType)) continue;
            if (!targetInfo.methodName.equals(currentInfo.methodName)) continue;
            if (targetInfo.parameterTypes.size() != currentInfo.parameterTypes.size()) continue;
            if (allParamsAssignable(targetInfo, currentInfo)) {
                return physicalIdByKey.get(target);
            }
        }
        return null;
    }

    private static boolean allParamsAssignable(LambdaKey.MethodSignatureInfo target, LambdaKey.MethodSignatureInfo current) {
        for (int i = 0; i < target.parameterTypes.size(); i++) {
            if (!target.parameterTypes.get(i).isAssignableFrom(current.parameterTypes.get(i))) return false;
        }
        return true;
    }

    /** Test/internal: clears all in-memory state. */
    public void clear() {
        physicalIdByKey.clear();
        keysByHash.clear();
        physicalByLogical.clear();
        nextPhysicalId = 0;
        nextLogicalId = 0;
    }

    /** Used by LambdaRuntime to rehydrate from a loaded snapshot. */
    void applySnapshot(CatalogSnapshot snapshot) {
        clear();
        this.nextPhysicalId = snapshot.nextPhysicalId();
        this.nextLogicalId = snapshot.nextLogicalId();
        for (CatalogEntry entry : snapshot.entries()) {
            LambdaKey key = LambdaUtils.createLambdaKeyFromMethodDeclarationString(
                    entry.methodSignature() + " " + entry.normalizedBody());
            physicalIdByKey.put(key, entry.physicalId());
            keysByHash.computeIfAbsent(key.hashCode(), h -> new ArrayList<>()).add(key);
        }
    }

    /** Used by LambdaRuntime to produce a snapshot for serialization. */
    CatalogSnapshot toSnapshot() {
        List<CatalogEntry> entries = new ArrayList<>();
        for (Map.Entry<LambdaKey, Integer> e : physicalIdByKey.entrySet()) {
            entries.add(new CatalogEntry(e.getValue(), e.getKey().getMethodSignature(), e.getKey().getNormalisedBody()));
        }
        entries.sort((a, b) -> Integer.compare(a.physicalId(), b.physicalId()));
        return new CatalogSnapshot(nextPhysicalId, nextLogicalId, entries);
    }
}
```

**Note:** `applySnapshot` / `toSnapshot` reference `CatalogSnapshot` and `CatalogEntry` which don't exist yet. Stub them out as empty records in this task so the code compiles; Task 3 fills them in properly.

- [ ] **Step 2.5: Create `CatalogSnapshot` and `CatalogEntry` stubs**

`src/main/java/org/mvel3/lambdaextractor/CatalogSnapshot.java`:
```java
package org.mvel3.lambdaextractor;

import java.util.List;

public record CatalogSnapshot(int nextPhysicalId, int nextLogicalId, List<CatalogEntry> entries) {}
```

`src/main/java/org/mvel3/lambdaextractor/CatalogEntry.java`:
```java
package org.mvel3.lambdaextractor;

public record CatalogEntry(int physicalId, String methodSignature, String normalizedBody) {}
```

- [ ] **Step 2.6: Run M1 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=LambdaCatalogTest#M1_dedup_exactKey_reusesPhysicalId 2>&1 | tail -10
```
Expected: 1 test passed.

- [ ] **Step 2.7: Add M2 — subtype overload, supertype-first → subtype reuses**

Append to `LambdaCatalogTest.java`:
```java
@Test
void M2_dedup_subtypeOverload_supertypeFirst_subtypeReuses() {
    String supertypeMethod = "public boolean eval(java.lang.Object o) { return o != null; }";
    String subtypeMethod = "public boolean eval(java.lang.String o) { return o != null; }";

    LambdaKey supertype = createLambdaKeyFromMethodDeclarationString(supertypeMethod);
    LambdaKey subtype = createLambdaKeyFromMethodDeclarationString(subtypeMethod);

    RegistrationResult r1 = catalog.register(supertype);
    RegistrationResult r2 = catalog.register(subtype);

    assertThat(r1.reused()).isFalse();
    assertThat(r2.reused()).isTrue();
    assertThat(r2.physicalId()).isEqualTo(r1.physicalId());
}
```

- [ ] **Step 2.8: Run M2 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=LambdaCatalogTest#M2_dedup_subtypeOverload_supertypeFirst_subtypeReuses 2>&1 | tail -5
```

- [ ] **Step 2.9: Add M3 — subtype-first → supertype does NOT collapse**

```java
@Test
void M3_dedup_subtypeOverload_subtypeFirst_supertypeDoesNotReuse() {
    String subtypeMethod = "public boolean eval(java.lang.String o) { return o != null; }";
    String supertypeMethod = "public boolean eval(java.lang.Object o) { return o != null; }";

    LambdaKey subtype = createLambdaKeyFromMethodDeclarationString(subtypeMethod);
    LambdaKey supertype = createLambdaKeyFromMethodDeclarationString(supertypeMethod);

    RegistrationResult r1 = catalog.register(subtype);
    RegistrationResult r2 = catalog.register(supertype);

    assertThat(r1.reused()).isFalse();
    assertThat(r2.reused()).isFalse();
    assertThat(r2.physicalId()).isNotEqualTo(r1.physicalId());
}
```

- [ ] **Step 2.10: Add M4 — hash collision but distinct keys**

```java
@Test
void M4_hashCollision_distinctKeys_keepSeparatePhysicalIds() {
    String method1 = "public boolean eval(java.lang.Object o) { return o != null; }";
    String method2 = "public int compute(java.lang.Object o) { return 1; }";

    LambdaKey k1 = createLambdaKeyFromMethodDeclarationString(method1);
    LambdaKey k2 = createLambdaKeyFromMethodDeclarationString(method2);
    k2.forceHash(k1.hashCode());     // package-private; same package

    RegistrationResult r1 = catalog.register(k1);
    RegistrationResult r2 = catalog.register(k2);

    assertThat(r1.physicalId()).isNotEqualTo(r2.physicalId());
    assertThat(r2.reused()).isFalse();
}
```

- [ ] **Step 2.11: Run M2–M4 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=LambdaCatalogTest 2>&1 | tail -10
```
Expected: 4 tests passed.

- [ ] **Step 2.12: Make `LambdaRegistry` delegate to `LambdaCatalog` (transitional facade)**

Edit `src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java`:

1. Delete only the **dedup** state — `entriesByKey`, `hashToKeys`, `logicalToPhysical`, `nextPhysicalId`, `nextLogicalId` — and the dedup helpers (`reuseIfSubtypeOverload`, `isAllParamsAssignable`). These live in `LambdaCatalog` now.
2. **KEEP** `entriesByPhysicalId` and the `RegistryEntry` inner class. They continue to hold artifact-path state in the facade until Phase 4 (when `LambdaPersistenceManager` takes over). Otherwise `registerPhysicalPath` / `getPhysicalPath` / `isPersisted` would throw immediately on every call and the test suite breaks mid-refactor.
3. **Add `fqn` field** to `RegistryEntry`. Today's RegistryEntry has `key`, `physicalId`, `path` — no FQN. We need FQN to build `ArtifactRef` later. Simplest path: track it explicitly from Phase 1 onwards.
   ```java
   private static final class RegistryEntry {
       private final int physicalId;
       private String fqn;        // NEW — set by registerPhysicalPath
       private Path path;
       private RegistryEntry(int physicalId) { this.physicalId = physicalId; }
   }
   ```
4. Add `private final LambdaCatalog catalog = new LambdaCatalog();` and route registration through it. `registerLambda` populates `entriesByPhysicalId` so that `registerPhysicalPath` can find the entry later:
   ```java
   public int registerLambda(int ignoredLogicalId, LambdaKey key) {
       RegistrationResult result = catalog.register(key);
       entriesByPhysicalId.computeIfAbsent(result.physicalId(), RegistryEntry::new);
       return result.physicalId();
   }
   ```
5. **Change `registerPhysicalPath` signature to take FQN** (`int physicalId, String fqn, Path path`):
   ```java
   public void registerPhysicalPath(int physicalId, String fqn, Path path) {
       RegistryEntry entry = entriesByPhysicalId.get(physicalId);
       if (entry == null) throw new IllegalStateException("Unknown physical ID " + physicalId);
       entry.fqn = fqn;
       entry.path = path;
       if (PERSISTENCE_ENABLED) persistToDisk();
   }
   ```
6. `getPhysicalPath(int)` and `isPersisted(int)` unchanged (still read `entriesByPhysicalId`).
7. `getNextLogicalId()` — keep, but it's now informational only; the catalog allocates logicalId inside `register`. Returns `-1` to signal "ignored":
   ```java
   public int getNextLogicalId() { return -1; }
   ```
8. `loadFromDisk()` — leave unchanged in Phase 1 (still reads the legacy pipe/Base64 format). Phase 2 rewrites it. To make the legacy load populate the new `RegistryEntry(physicalId)` constructor: replace `RegistryEntry entry = new RegistryEntry(key, physicalId);` with `RegistryEntry entry = new RegistryEntry(physicalId);`. The legacy format does not carry FQN; `entry.fqn` stays null on loads of v1 files. Subsequent `registerPhysicalPath` calls would re-populate FQN. (This is acceptable because Phase 2 will replace the format and load semantics entirely.)

**Update MVELCompiler caller** (`src/main/java/org/mvel3/MVELCompiler.java`, around line 302):
```diff
- LambdaRegistry.INSTANCE.registerPhysicalPath(physicalId, persistedFiles.get(0));
+ LambdaRegistry.INSTANCE.registerPhysicalPath(physicalId, newJavaFQN, persistedFiles.get(0));
```
`newJavaFQN` is already in scope inside `compileEvaluatorClassWithPersistence`.

**Update MVELBatchCompiler caller** (`src/main/java/org/mvel3/MVELBatchCompiler.java`, around line 103):
```diff
- LambdaRegistry.INSTANCE.registerPhysicalPath(h.physicalId, path);
+ LambdaRegistry.INSTANCE.registerPhysicalPath(h.physicalId, h.fqn, path);
```
`h.fqn` is already on `LambdaHandle` (set at line 80 when the handle was created).

Keep `getPhysicalId(int logicalId)` — refactor it to call the catalog's internal map via a new package-private accessor `LambdaCatalog.physicalForLogical(int)`:

In `LambdaCatalog.java`, add:
```java
/** Test/facade-only helper. */
public Integer physicalForLogical(int logicalId) {
    return physicalByLogical.get(logicalId);
}
```

In `LambdaRegistry.java`:
```java
public int getPhysicalId(int logicalId) {
    return catalog.physicalForLogical(logicalId);
}
```

- [ ] **Step 2.13: Update existing `LambdaRegistryTest` to remain green via the facade**

The existing test (`LambdaRegistryTest.testRegisterLambda_SameLambdaDifferentVariableNames_ShouldSharePhysicalId`, etc.) calls `registry.getPhysicalId(1)` and `registry.getPhysicalId(2)` after `registry.registerLambda(1, k1)`. With the facade, `registerLambda` now ignores the `logicalId` argument and the catalog assigns its own. So those `getPhysicalId(1)` lookups will return `null` (or fail with NPE).

Fix: change the existing tests to bind logicalId from the return of `registerLambda`. Since today's `registerLambda` returns `physicalId`, not logicalId, we'd need a temporary shim. **Simpler:** mark the conflicting old tests `@Disabled("superseded by LambdaCatalogTest in Phase 1")` for now; they will be rewritten in Step 2.14 below.

- [ ] **Step 2.14: Run full test suite — green except disabled tests**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | tail -20
```
Expected: BUILD SUCCESS; previously-passing tests still pass; the 5 original LambdaRegistryTest tests are now `@Disabled` (or rewritten — see step 2.15); 4 new LambdaCatalogTest tests pass.

- [ ] **Step 2.15: Rewrite the 5 disabled `LambdaRegistryTest` tests against the catalog**

For each disabled test, port it to `LambdaCatalogTest.java` using the same assertions but via `catalog.register(key)` and `RegistrationResult`. Drop assertions that compare exact numeric IDs starting from 0 (per spec Risk R6); keep reuse-vs-different semantics. After porting, delete the now-redundant tests from `LambdaRegistryTest.java`.

Note: There are 5 original tests. The exact subset that exercises subtype-overload behavior may overlap M2/M3 — in that case keep the more comprehensive version, delete the duplicate.

- [ ] **Step 2.16: Run full suite — expect all green, no disabled tests**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | tail -10
```

- [ ] **Step 2.17: Commit Phase 1**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add \
    src/main/java/org/mvel3/lambdaextractor/RegistrationResult.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaCatalog.java \
    src/main/java/org/mvel3/lambdaextractor/CatalogSnapshot.java \
    src/main/java/org/mvel3/lambdaextractor/CatalogEntry.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java \
    src/test/java/org/mvel3/lambdaextractor/LambdaCatalogTest.java \
    src/test/java/org/mvel3/lambdaextractor/LambdaRegistryTest.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
Phase 1: Extract LambdaCatalog from LambdaRegistry

Moves LambdaKey↔physicalId, subtype-overload reuse, and ID allocation
into a new pure-in-memory LambdaCatalog. LambdaRegistry becomes a thin
facade delegating to it. Adds RegistrationResult record. Introduces
CatalogSnapshot/CatalogEntry stubs filled in by Phase 2.

Adds M1–M4 characterization tests in LambdaCatalogTest; ports the
existing LambdaRegistryTest assertions to use the new API.

Refs mvel/mvel#428

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Phase 2 — Extract `LambdaRegistryStore` + Properties v2

**Files:**
- Create: `src/main/java/org/mvel3/lambdaextractor/InvalidLambdaRegistryException.java`
- Create: `src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceSnapshot.java`
- Create: `src/main/java/org/mvel3/lambdaextractor/LambdaRegistryStore.java` (package-private)
- Create: `src/test/java/org/mvel3/lambdaextractor/LambdaRegistryStoreTest.java` (M5–M7)
- Modify: `src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java` — use the new store, drop `loadFromDisk`/`persistToDisk` internals
- Delete: `src/test/java/org/mvel3/lambdaextractor/LambdaRegistryPersistenceTest.java`

- [ ] **Step 3.1: Create `InvalidLambdaRegistryException`**

`src/main/java/org/mvel3/lambdaextractor/InvalidLambdaRegistryException.java`:
```java
package org.mvel3.lambdaextractor;

import java.io.IOException;

/**
 * Thrown when the lambda registry file is malformed, contains an unsupported
 * {@code format.version}, references a missing catalog entry from an artifact
 * entry, or contains duplicate physical IDs.
 */
public class InvalidLambdaRegistryException extends IOException {
    public InvalidLambdaRegistryException(String message) { super(message); }
    public InvalidLambdaRegistryException(String message, Throwable cause) { super(message, cause); }
}
```

- [ ] **Step 3.2: Create `LambdaPersistenceSnapshot` and an artifact-map placeholder**

`src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceSnapshot.java`:
```java
package org.mvel3.lambdaextractor;

import java.util.Map;

/** Transfer object between LambdaRuntime and LambdaRegistryStore. */
public record LambdaPersistenceSnapshot(CatalogSnapshot catalog, Map<Integer, ArtifactRef> artifacts) {}
```

This references `ArtifactRef` which doesn't exist yet. Create a temporary stub:

`src/main/java/org/mvel3/lambdaextractor/ArtifactRef.java`:
```java
package org.mvel3.lambdaextractor;

import java.nio.file.Path;

/**
 * Persistence-facing value object. Lambda's persisted classfile + FQN.
 * Used both by MVEL internally and by DRLX as the cross-repo persistence
 * artifact reference.
 */
public record ArtifactRef(String fqn, Path classFile) {}
```

(Although `ArtifactRef` is logically Phase 4, the type is needed by `LambdaPersistenceSnapshot` here.)

- [ ] **Step 3.3: Write M5 (round-trip) test**

`src/test/java/org/mvel3/lambdaextractor/LambdaRegistryStoreTest.java`:
```java
package org.mvel3.lambdaextractor;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class LambdaRegistryStoreTest {

    @Test
    void M5_registryStore_writeReadRoundTrip(@TempDir Path tmp) throws IOException {
        Path file = tmp.resolve("lambda-registry.dat");

        CatalogSnapshot catalog = new CatalogSnapshot(2, 3, List.of(
                new CatalogEntry(0, "public boolean eval(java.lang.Object obj)", "{ return obj != null; }"),
                new CatalogEntry(1, "public boolean eval(java.lang.String s)", "{ return s.length() > 0; }")
        ));
        Map<Integer, ArtifactRef> artifacts = Map.of(
                0, new ArtifactRef("org.mvel3.GenA", tmp.resolve("GenA.class")),
                1, new ArtifactRef("org.mvel3.GenB", tmp.resolve("GenB.class"))
        );
        LambdaPersistenceSnapshot snapshot = new LambdaPersistenceSnapshot(catalog, artifacts);

        LambdaRegistryStore store = new LambdaRegistryStore(file);
        store.save(snapshot);
        LambdaPersistenceSnapshot loaded = store.load();

        assertThat(loaded.catalog().nextPhysicalId()).isEqualTo(2);
        assertThat(loaded.catalog().nextLogicalId()).isEqualTo(3);
        assertThat(loaded.catalog().entries()).hasSize(2);
        assertThat(loaded.artifacts()).isEqualTo(artifacts);
    }
}
```

- [ ] **Step 3.4: Run M5 — expect compile fail (no `LambdaRegistryStore`)**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test-compile 2>&1 | tail -5
```

- [ ] **Step 3.5: Implement `LambdaRegistryStore`**

`src/main/java/org/mvel3/lambdaextractor/LambdaRegistryStore.java`:
```java
package org.mvel3.lambdaextractor;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.TreeMap;

/**
 * Properties-based persistence for the lambda registry. Internal to LambdaRuntime;
 * compilers should not interact with this class directly.
 * <p>
 * Format: {@code format.version=2}. See the design doc for the schema.
 */
final class LambdaRegistryStore {

    private static final String FORMAT_VERSION = "2";
    private static final String KEY_VERSION = "format.version";
    private static final String KEY_NEXT_PHYSICAL = "catalog.nextPhysicalId";
    private static final String KEY_NEXT_LOGICAL = "catalog.nextLogicalId";

    private final Path file;

    LambdaRegistryStore(Path file) {
        this.file = file;
    }

    LambdaPersistenceSnapshot load() throws IOException {
        Properties props = new Properties();
        try (InputStream in = Files.newInputStream(file)) {
            props.load(in);
        }
        String version = props.getProperty(KEY_VERSION);
        if (!FORMAT_VERSION.equals(version)) {
            throw new InvalidLambdaRegistryException(
                    "Unsupported lambda-registry format.version: " + version + " (expected " + FORMAT_VERSION + ")");
        }

        int nextPhysical = requiredInt(props, KEY_NEXT_PHYSICAL);
        int nextLogical = requiredInt(props, KEY_NEXT_LOGICAL);

        // Catalog entries
        Map<Integer, CatalogEntry> catalogByPhysicalId = new TreeMap<>();
        Map<Integer, Integer> catalogIndexByIndex = parseIndexed(props, "catalog.entry.");
        for (Integer index : catalogIndexByIndex.keySet()) {
            int physicalId = requiredInt(props, "catalog.entry." + index + ".physicalId");
            String methodSignature = requiredString(props, "catalog.entry." + index + ".methodSignature");
            String normalizedBody = requiredString(props, "catalog.entry." + index + ".normalizedBody");
            if (catalogByPhysicalId.containsKey(physicalId)) {
                throw new InvalidLambdaRegistryException("Duplicate catalog physicalId: " + physicalId);
            }
            catalogByPhysicalId.put(physicalId, new CatalogEntry(physicalId, methodSignature, normalizedBody));
        }

        // Artifacts
        Map<Integer, ArtifactRef> artifacts = new HashMap<>();
        Map<Integer, Integer> artifactIndexByIndex = parseIndexed(props, "artifact.");
        for (Integer index : artifactIndexByIndex.keySet()) {
            int physicalId = requiredInt(props, "artifact." + index + ".physicalId");
            String fqn = requiredString(props, "artifact." + index + ".fqn");
            String classFile = requiredString(props, "artifact." + index + ".classFile");
            if (!catalogByPhysicalId.containsKey(physicalId)) {
                throw new InvalidLambdaRegistryException(
                        "Artifact physicalId " + physicalId + " has no matching catalog entry");
            }
            if (artifacts.containsKey(physicalId)) {
                throw new InvalidLambdaRegistryException("Duplicate artifact physicalId: " + physicalId);
            }
            artifacts.put(physicalId, new ArtifactRef(fqn, Path.of(classFile)));
        }

        return new LambdaPersistenceSnapshot(
                new CatalogSnapshot(nextPhysical, nextLogical, new ArrayList<>(catalogByPhysicalId.values())),
                artifacts);
    }

    void save(LambdaPersistenceSnapshot snapshot) throws IOException {
        Files.createDirectories(file.getParent());
        Properties props = new Properties();
        props.setProperty(KEY_VERSION, FORMAT_VERSION);
        props.setProperty(KEY_NEXT_PHYSICAL, Integer.toString(snapshot.catalog().nextPhysicalId()));
        props.setProperty(KEY_NEXT_LOGICAL, Integer.toString(snapshot.catalog().nextLogicalId()));

        int i = 0;
        for (CatalogEntry entry : snapshot.catalog().entries()) {
            props.setProperty("catalog.entry." + i + ".physicalId", Integer.toString(entry.physicalId()));
            props.setProperty("catalog.entry." + i + ".methodSignature", entry.methodSignature());
            props.setProperty("catalog.entry." + i + ".normalizedBody", entry.normalizedBody());
            i++;
        }
        int j = 0;
        for (Map.Entry<Integer, ArtifactRef> e : new TreeMap<>(snapshot.artifacts()).entrySet()) {
            props.setProperty("artifact." + j + ".physicalId", Integer.toString(e.getKey()));
            props.setProperty("artifact." + j + ".fqn", e.getValue().fqn());
            props.setProperty("artifact." + j + ".classFile", e.getValue().classFile().toString());
            j++;
        }

        try (OutputStream out = Files.newOutputStream(file)) {
            props.store(out, "MVEL3 lambda registry");
        }
    }

    private static int requiredInt(Properties p, String key) throws InvalidLambdaRegistryException {
        String v = p.getProperty(key);
        if (v == null) throw new InvalidLambdaRegistryException("Missing key: " + key);
        try { return Integer.parseInt(v); }
        catch (NumberFormatException nfe) {
            throw new InvalidLambdaRegistryException("Non-integer value for " + key + ": " + v, nfe);
        }
    }
    private static String requiredString(Properties p, String key) throws InvalidLambdaRegistryException {
        String v = p.getProperty(key);
        if (v == null) throw new InvalidLambdaRegistryException("Missing key: " + key);
        return v;
    }

    /** Returns the set of distinct integer indexes used in keys matching {@code prefix<N>.*}. */
    private static Map<Integer, Integer> parseIndexed(Properties p, String prefix) {
        Map<Integer, Integer> indexes = new TreeMap<>();
        for (String key : p.stringPropertyNames()) {
            if (!key.startsWith(prefix)) continue;
            String rest = key.substring(prefix.length());
            int dot = rest.indexOf('.');
            if (dot <= 0) continue;
            try {
                int n = Integer.parseInt(rest.substring(0, dot));
                indexes.put(n, n);
            } catch (NumberFormatException ignored) { }
        }
        return indexes;
    }
}
```

- [ ] **Step 3.6: Run M5 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=LambdaRegistryStoreTest#M5_registryStore_writeReadRoundTrip 2>&1 | tail -5
```

- [ ] **Step 3.7: Add M6 (unsupported version)**

Append to `LambdaRegistryStoreTest.java`:
```java
@Test
void M6_registryStore_unsupportedVersion_throws(@TempDir Path tmp) throws IOException {
    Path file = tmp.resolve("lambda-registry.dat");
    Files.writeString(file, "format.version=1\ncatalog.nextPhysicalId=0\ncatalog.nextLogicalId=0\n");

    LambdaRegistryStore store = new LambdaRegistryStore(file);

    assertThatThrownBy(store::load)
            .isInstanceOf(InvalidLambdaRegistryException.class)
            .hasMessageContaining("Unsupported");
}
```

- [ ] **Step 3.8: Add M7 (malformed: duplicate physicalId + missing key + dangling artifact)**

```java
@Test
void M7a_registryStore_duplicateCatalogPhysicalId_throws(@TempDir Path tmp) throws IOException {
    Path file = tmp.resolve("lambda-registry.dat");
    Files.writeString(file, String.join("\n",
            "format.version=2",
            "catalog.nextPhysicalId=2",
            "catalog.nextLogicalId=2",
            "catalog.entry.0.physicalId=0",
            "catalog.entry.0.methodSignature=public boolean eval(java.lang.Object o)",
            "catalog.entry.0.normalizedBody={ return o != null; }",
            "catalog.entry.1.physicalId=0",     // duplicate
            "catalog.entry.1.methodSignature=public boolean eval(java.lang.String s)",
            "catalog.entry.1.normalizedBody={ return s != null; }",
            ""));
    assertThatThrownBy(() -> new LambdaRegistryStore(file).load())
            .isInstanceOf(InvalidLambdaRegistryException.class)
            .hasMessageContaining("Duplicate catalog physicalId");
}

@Test
void M7b_registryStore_missingRequiredKey_throws(@TempDir Path tmp) throws IOException {
    Path file = tmp.resolve("lambda-registry.dat");
    Files.writeString(file, "format.version=2\n");
    assertThatThrownBy(() -> new LambdaRegistryStore(file).load())
            .isInstanceOf(InvalidLambdaRegistryException.class)
            .hasMessageContaining("Missing key");
}

@Test
void M7c_registryStore_danglingArtifactReference_throws(@TempDir Path tmp) throws IOException {
    Path file = tmp.resolve("lambda-registry.dat");
    Files.writeString(file, String.join("\n",
            "format.version=2",
            "catalog.nextPhysicalId=1",
            "catalog.nextLogicalId=1",
            "catalog.entry.0.physicalId=0",
            "catalog.entry.0.methodSignature=public boolean eval(java.lang.Object o)",
            "catalog.entry.0.normalizedBody={ return o != null; }",
            "artifact.0.physicalId=99",     // no matching catalog entry
            "artifact.0.fqn=org.mvel3.Gen",
            "artifact.0.classFile=/tmp/Gen.class",
            ""));
    assertThatThrownBy(() -> new LambdaRegistryStore(file).load())
            .isInstanceOf(InvalidLambdaRegistryException.class)
            .hasMessageContaining("no matching catalog entry");
}
```

- [ ] **Step 3.9: Run M6, M7a/b/c — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=LambdaRegistryStoreTest 2>&1 | tail -10
```

- [ ] **Step 3.10: Rewire `LambdaRegistry` facade to use the new store with the real artifact map**

Edit `LambdaRegistry.java`:
- Replace `persistToDisk()` to build a real `LambdaPersistenceSnapshot` using both the catalog snapshot AND the artifact map derived from `entriesByPhysicalId` (each `RegistryEntry` now has FQN from Phase 1):
  ```java
  private void persistToDisk() {
      try {
          Files.createDirectories(REGISTRY_FILE.getParent());
          Map<Integer, ArtifactRef> artifacts = new HashMap<>();
          for (RegistryEntry entry : entriesByPhysicalId.values()) {
              if (entry.fqn != null && entry.path != null) {
                  artifacts.put(entry.physicalId, new ArtifactRef(entry.fqn, entry.path));
              }
          }
          new LambdaRegistryStore(REGISTRY_FILE).save(
                  new LambdaPersistenceSnapshot(catalog.toSnapshot(), artifacts));
      } catch (IOException e) {
          throw new RuntimeException("Failed to persist lambda registry", e);
      }
  }
  ```
- Replace `loadFromDisk()` to read the new format and populate both catalog state and `entriesByPhysicalId`:
  ```java
  private void loadFromDisk() {
      if (!Files.exists(REGISTRY_FILE)) return;
      try {
          LambdaPersistenceSnapshot snapshot = new LambdaRegistryStore(REGISTRY_FILE).load();
          catalog.applySnapshot(snapshot.catalog());
          for (Map.Entry<Integer, ArtifactRef> e : snapshot.artifacts().entrySet()) {
              RegistryEntry entry = new RegistryEntry(e.getKey());
              entry.fqn = e.getValue().fqn();
              entry.path = e.getValue().classFile();
              entriesByPhysicalId.put(e.getKey(), entry);
          }
      } catch (IOException e) {
          throw new RuntimeException("Failed to load lambda registry", e);
      }
  }
  ```
- Delete `parseIntSafe`, `encode`, `decode`. Delete the `REGISTRY_VERSION` field.

Round-trip is intact throughout Phase 2: artifact paths are persisted in the new format and restored on load. Phase 4 takes ownership of the artifact map from `entriesByPhysicalId` and hands it to `LambdaPersistenceManager`.

- [ ] **Step 3.11: Delete `LambdaRegistryPersistenceTest.java`**

The old test was 1 test against the legacy pipe/Base64 format. Its semantics are covered by M5–M7 against the new format.

```bash
rm /home/tkobayas/usr/work/mvel3-development/mvel/src/test/java/org/mvel3/lambdaextractor/LambdaRegistryPersistenceTest.java
```

- [ ] **Step 3.12: Run full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 3.13: Commit Phase 2**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add \
    src/main/java/org/mvel3/lambdaextractor/InvalidLambdaRegistryException.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceSnapshot.java \
    src/main/java/org/mvel3/lambdaextractor/ArtifactRef.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaRegistryStore.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java \
    src/test/java/org/mvel3/lambdaextractor/LambdaRegistryStoreTest.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel rm \
    src/test/java/org/mvel3/lambdaextractor/LambdaRegistryPersistenceTest.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
Phase 2: Extract LambdaRegistryStore + Properties v2 format

Replaces the pipe/Base64 registry file format with java.util.Properties
(format.version=2). Adds LambdaRegistryStore (package-private),
LambdaPersistenceSnapshot, ArtifactRef stub, and
InvalidLambdaRegistryException. LambdaRegistry's loadFromDisk/persistToDisk
delegate to the store; encode/decode/parseIntSafe deleted.

Adds M5–M7 tests (round-trip, unsupported version, malformed entries).
Old v1 format dies; no backward-compat readers per spec.

Refs mvel/mvel#428

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Phase 3 — Extract `LambdaArtifactStore`

**Files:**
- Create: `src/main/java/org/mvel3/lambdaextractor/LambdaArtifactStore.java`
- Modify: `src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java` — delegate path/byte/cleanup work to the store

- [ ] **Step 4.1: Create `LambdaArtifactStore`**

`src/main/java/org/mvel3/lambdaextractor/LambdaArtifactStore.java`:
```java
package org.mvel3.lambdaextractor;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.stream.Stream;

/**
 * Dumb byte-level I/O for persisted lambda classfiles. The {@code persistenceRoot}
 * is used only by {@link #deleteAll()}; {@link #exists(ArtifactRef)} and
 * {@link #readBytes(ArtifactRef)} work directly off the path embedded in the
 * {@link ArtifactRef}.
 */
public final class LambdaArtifactStore {

    private final Path persistenceRoot;

    public LambdaArtifactStore(Path persistenceRoot) {
        this.persistenceRoot = persistenceRoot;
    }

    public boolean exists(ArtifactRef ref) {
        return Files.exists(ref.classFile());
    }

    public byte[] readBytes(ArtifactRef ref) throws IOException {
        return Files.readAllBytes(ref.classFile());
    }

    public void deleteAll() throws IOException {
        if (!Files.exists(persistenceRoot)) return;
        try (Stream<Path> walk = Files.walk(persistenceRoot)) {
            walk.sorted((a, b) -> b.getNameCount() - a.getNameCount())
                .forEach(path -> {
                    try { Files.deleteIfExists(path); }
                    catch (IOException ignored) { /* best-effort */ }
                });
        }
    }
}
```

- [ ] **Step 4.2: Rewire `LambdaRegistry` cleanup to the store**

In `LambdaRegistry.java`, replace the body of `resetAndRemoveAllPersistedFiles()`:

```java
public synchronized void resetAndRemoveAllPersistedFiles() {
    LOG.info("Clean up Lambda Registry and persisted files at {}", DEFAULT_PERSISTENCE_PATH);
    try {
        new LambdaArtifactStore(DEFAULT_PERSISTENCE_PATH).deleteAll();
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
    catalog.clear();
    entriesByPhysicalId.clear();
}
```

(The `entriesByPhysicalId` map still exists in `LambdaRegistry` as part of the artifact-path tracking — that goes away in Phase 4.)

- [ ] **Step 4.3: Run full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | tail -10
```

- [ ] **Step 4.4: Commit Phase 3**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add \
    src/main/java/org/mvel3/lambdaextractor/LambdaArtifactStore.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
Phase 3: Extract LambdaArtifactStore

Centralises file-existence checks, byte reads, and persistence-directory
cleanup into a dedicated dumb-I/O component. LambdaRegistry's
resetAndRemoveAllPersistedFiles delegates to it.

Refs mvel/mvel#428

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Phase 4 — Introduce `LambdaPersistenceManager`, plumb artifact map

**Files:**
- Create: `src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceManager.java`
- Modify: `src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java` — facade routes path queries through PM; remove `entriesByPhysicalId`
- Modify: `src/main/java/org/mvel3/lambdaextractor/LambdaRegistryStore.java` — `save` now receives the real artifact map (already wired in Phase 2)

- [ ] **Step 5.1: Create `LambdaPersistenceManager`**

`src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceManager.java`:
```java
package org.mvel3.lambdaextractor;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

/**
 * Owns the {@code physicalId → ArtifactRef} association. Coordinates with
 * {@link LambdaArtifactStore} for on-disk existence checks. Triggers
 * {@link LambdaRuntime#persistSnapshot()} synchronously on {@link #attachArtifact}.
 * <p>
 * Compilers use this for compile-vs-reuse state queries. Not a compile wrapper —
 * compile orchestration remains in {@link org.mvel3.MVELCompiler} and
 * {@link org.mvel3.MVELBatchCompiler}.
 */
public final class LambdaPersistenceManager {

    private final LambdaArtifactStore artifactStore;
    private final LambdaRuntime runtime;     // for persistSnapshot()
    private final Map<Integer, ArtifactRef> artifacts = new HashMap<>();

    LambdaPersistenceManager(LambdaArtifactStore artifactStore, LambdaRuntime runtime) {
        this.artifactStore = artifactStore;
        this.runtime = runtime;
    }

    public Optional<ArtifactRef> artifactFor(int physicalId) {
        return Optional.ofNullable(artifacts.get(physicalId));
    }

    /** True iff an artifact is attached to {@code physicalId} AND its classfile exists on disk. */
    public boolean artifactExists(int physicalId) {
        ArtifactRef ref = artifacts.get(physicalId);
        return ref != null && artifactStore.exists(ref);
    }

    /** Attaches an artifact. Triggers synchronous registry-file save via the runtime. */
    public void attachArtifact(int physicalId, ArtifactRef ref) {
        artifacts.put(physicalId, ref);
        runtime.persistSnapshot();
    }

    /** Internal: rehydrate from a loaded snapshot. */
    void applyArtifacts(Map<Integer, ArtifactRef> loaded) {
        artifacts.clear();
        artifacts.putAll(loaded);
    }

    /** Internal: clear all artifact mappings (catalog reset path). */
    void clear() {
        artifacts.clear();
    }

    /** Internal: produce the snapshot view for serialization. */
    Map<Integer, ArtifactRef> snapshot() {
        return Map.copyOf(artifacts);
    }
}
```

Note: `LambdaRuntime` is referenced but not yet defined as a class. Add a temporary stub so this file compiles:

`src/main/java/org/mvel3/lambdaextractor/LambdaRuntime.java`:
```java
package org.mvel3.lambdaextractor;

/** Stub — full implementation lands in Phase 6. */
public final class LambdaRuntime {
    void persistSnapshot() { /* no-op stub */ }
}
```

- [ ] **Step 5.2: Wire facade to PM; migrate artifact map from `entriesByPhysicalId` to PM**

Edit `LambdaRegistry.java`. Add a PM (held by the facade) and route all artifact queries through it. The `registerPhysicalPath` signature already takes FQN (from Phase 1), so no path-to-FQN reconstruction is needed:

```java
// In LambdaRegistry:
private final LambdaArtifactStore artifactStore = new LambdaArtifactStore(DEFAULT_PERSISTENCE_PATH);
private final LambdaRuntime stubRuntime = new LambdaRuntime();    // Phase 6 replaces with the real runtime
private final LambdaPersistenceManager persistenceManager = new LambdaPersistenceManager(artifactStore, stubRuntime);

public void registerPhysicalPath(int physicalId, String fqn, Path path) {
    persistenceManager.attachArtifact(physicalId, new ArtifactRef(fqn, path));
    if (PERSISTENCE_ENABLED) {
        persistToDisk();
    }
}

public Path getPhysicalPath(int physicalId) {
    return persistenceManager.artifactFor(physicalId).map(ArtifactRef::classFile).orElse(null);
}

public boolean isPersisted(int physicalId) {
    return persistenceManager.artifactExists(physicalId);
}
```

Update `persistToDisk()` to read the artifact map from PM (no longer from `entriesByPhysicalId`):
```java
private void persistToDisk() {
    try {
        Files.createDirectories(REGISTRY_FILE.getParent());
        new LambdaRegistryStore(REGISTRY_FILE).save(
                new LambdaPersistenceSnapshot(catalog.toSnapshot(), persistenceManager.snapshot()));
    } catch (IOException e) {
        throw new RuntimeException("Failed to persist lambda registry", e);
    }
}
```

Update `loadFromDisk()` to hand the artifact map to PM instead of `entriesByPhysicalId`:
```java
private void loadFromDisk() {
    if (!Files.exists(REGISTRY_FILE)) return;
    try {
        LambdaPersistenceSnapshot snapshot = new LambdaRegistryStore(REGISTRY_FILE).load();
        catalog.applySnapshot(snapshot.catalog());
        persistenceManager.applyArtifacts(snapshot.artifacts());
    } catch (IOException e) {
        throw new RuntimeException("Failed to load lambda registry", e);
    }
}
```

Update `registerLambda` — it no longer needs to seed `entriesByPhysicalId`:
```java
public int registerLambda(int ignoredLogicalId, LambdaKey key) {
    return catalog.register(key).physicalId();
}
```

**Now delete** the `entriesByPhysicalId` field and the `RegistryEntry` inner class — `LambdaPersistenceManager` is authoritative for artifact state from this point on.

Update `resetAndRemoveAllPersistedFiles`:
```java
public synchronized void resetAndRemoveAllPersistedFiles() {
    try { artifactStore.deleteAll(); }
    catch (IOException e) { throw new RuntimeException(e); }
    catalog.clear();
    persistenceManager.clear();
    try { Files.deleteIfExists(REGISTRY_FILE); }
    catch (IOException e) { throw new RuntimeException(e); }
}
```

- [ ] **Step 5.3: Run full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | tail -10
```

- [ ] **Step 5.4: Commit Phase 4**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add \
    src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceManager.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaRuntime.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
Phase 4: Add LambdaPersistenceManager + plumb artifact map

Introduces LambdaPersistenceManager owning the physicalId↔ArtifactRef
association. LambdaRegistry's registerPhysicalPath / getPhysicalPath /
isPersisted facade methods route through PM. Snapshot now carries the
real artifact map. LambdaRuntime exists as a stub; Phase 6 fleshes it out.

Refs mvel/mvel#428

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Phase 5 — Rewrite compilers, add `getArtifactRef`, add `compileInvocationCount`

**Files:**
- Modify: `src/main/java/org/mvel3/MVELCompiler.java` (~lines 195–308)
- Modify: `src/main/java/org/mvel3/MVELBatchCompiler.java` (~lines 60–120)
- Create: `src/test/java/org/mvel3/lambdaextractor/MVELCompilerPersistenceTest.java` (M8–M11)

- [ ] **Step 6.1: Add `compileInvocationCount` to `MVELCompiler`**

In `MVELCompiler.java`, add a counter field and accessor:
```java
private final java.util.concurrent.atomic.AtomicInteger compileInvocations = new java.util.concurrent.atomic.AtomicInteger();

/**
 * Test-only instrumentation. NOT part of the long-term public compiler API.
 * Bumped only on the actual compile path (fresh compile-and-persist or fresh in-memory),
 * not on reuse-from-disk.
 */
public int compileInvocationCount() { return compileInvocations.get(); }
```

Bump it in `compileEvaluatorClass(...)` and on the `else` branch of `compileEvaluatorClassWithPersistence(...)` (the `compileAndPersist` path). Do **not** bump on the `isPersisted == true` reuse path.

- [ ] **Step 6.2: Add `compileInvocationCount` and `getArtifactRef` to `MVELBatchCompiler`**

In `MVELBatchCompiler.java`:
```java
private final java.util.concurrent.atomic.AtomicInteger compileInvocations = new java.util.concurrent.atomic.AtomicInteger();
public int compileInvocationCount() { return compileInvocations.get(); }
```

Bump it inside the bulk compile branch (where `KieMemoryCompiler.compileAndPersist` is actually called), not on the reuse branch.

Add `getArtifactRef`:
```java
import org.mvel3.lambdaextractor.ArtifactRef;
import org.mvel3.lambdaextractor.LambdaRegistry;
import java.nio.file.Path;

/**
 * Returns the persisted artifact reference for a handle, for cross-repo consumers
 * (DRLX). Internal callers should still use {@link #getFqn(LambdaHandle)} /
 * {@link #getPhysicalId(LambdaHandle)} where appropriate.
 */
public ArtifactRef getArtifactRef(LambdaHandle handle) {
    String fqn = getFqn(handle);
    Path classFile = LambdaRegistry.INSTANCE.getPhysicalPath(getPhysicalId(handle));
    if (classFile == null) {
        throw new IllegalStateException("No artifact attached for handle " + handle);
    }
    return new ArtifactRef(fqn, classFile);
}
```

(`getPhysicalId(handle)` still exists as a package-private/internal accessor. After Phase 6 it remains internal; DRLX never calls it.)

- [ ] **Step 6.3: Write M8 (fresh compile attaches)**

`src/test/java/org/mvel3/lambdaextractor/MVELCompilerPersistenceTest.java`:
```java
package org.mvel3.lambdaextractor;

import java.nio.file.Files;
import java.nio.file.Path;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mvel3.CompilerParameters;
import org.mvel3.Evaluator;
import org.mvel3.MVEL;
import org.mvel3.MVELCompiler;

import static org.assertj.core.api.Assertions.assertThat;

class MVELCompilerPersistenceTest {

    @BeforeEach
    void cleanState() {
        LambdaRegistry.INSTANCE.resetAndRemoveAllPersistedFiles();
    }

    @Test
    void M8_compiler_freshLambda_persistsAndAttaches() {
        MVELCompiler compiler = new MVELCompiler();
        int before = compiler.compileInvocationCount();

        Evaluator<Object, Void, Boolean> evaluator = compiler.compile(
                MVEL.pojo(String.class).<Boolean>out(Boolean.class).expression("length() > 0").build());

        assertThat(compiler.compileInvocationCount()).isEqualTo(before + 1);
        Path registryFile = LambdaRegistry.DEFAULT_PERSISTENCE_PATH.resolve("lambda-registry.dat");
        assertThat(Files.exists(registryFile)).isTrue();
        assertThat(evaluator.evaluate("hello", null)).isTrue();
    }
}
```

(Adjust `MVEL.pojo(...)` shape to match the project's current API if needed — verify against an existing test like `LambdaRegistryTest`.)

- [ ] **Step 6.4: Run M8 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=MVELCompilerPersistenceTest#M8_compiler_freshLambda_persistsAndAttaches 2>&1 | tail -10
```

- [ ] **Step 6.5: Write M9 (known lambda reuses, no recompile)**

```java
@Test
void M9_compiler_knownLambda_reusesPersisted() {
    MVELCompiler first = new MVELCompiler();
    Evaluator<Object, Void, Boolean> e1 = first.compile(
            MVEL.pojo(String.class).<Boolean>out(Boolean.class).expression("length() > 0").build());

    MVELCompiler second = new MVELCompiler();
    int before = second.compileInvocationCount();
    Evaluator<Object, Void, Boolean> e2 = second.compile(
            MVEL.pojo(String.class).<Boolean>out(Boolean.class).expression("length() > 0").build());

    assertThat(second.compileInvocationCount()).isEqualTo(before);  // no compile on reuse
    assertThat(e2.evaluate("hello", null)).isTrue();
}
```

- [ ] **Step 6.6: Run M9 — expect PASS**

- [ ] **Step 6.7: Write M10 (batch mixed) and M11 (persistence-disabled)**

M10:
```java
@Test
void M10_batchCompiler_mixed_freshAndKnown() {
    // Pre-seed: compile lambda A once.
    new MVELCompiler().compile(MVEL.pojo(String.class)
            .<Boolean>out(Boolean.class).expression("length() > 0").build());

    MVELBatchCompiler batch = new MVELBatchCompiler();
    int before = batch.compileInvocationCount();
    batch.add(MVEL.pojo(String.class).<Boolean>out(Boolean.class).expression("length() > 0").build());   // known
    batch.add(MVEL.pojo(String.class).<Boolean>out(Boolean.class).expression("isEmpty()").build());      // new
    batch.compile(MVELCompilerPersistenceTest.class.getClassLoader());

    assertThat(batch.compileInvocationCount()).isEqualTo(before + 1);   // exactly one bulk compile
}
```

M11 (uses system property):
```java
@Test
void M11_runtime_persistenceDisabled_noFileWrites(@TempDir Path tmp) {
    String key = "mvel3.compiler.lambda.persistence";
    String prev = System.getProperty(key);
    System.setProperty(key, "false");
    try {
        // Force re-init by clearing the singleton-like state. In Phase 6 this becomes
        // LambdaRuntime.resetSingletonForTests(); for now, the facade reads the prop on
        // each call, so the existing reset is sufficient.
        LambdaRegistry.INSTANCE.resetAndRemoveAllPersistedFiles();

        MVELCompiler compiler = new MVELCompiler();
        compiler.compile(MVEL.pojo(String.class)
                .<Boolean>out(Boolean.class).expression("length() > 0").build());

        Path registryFile = LambdaRegistry.DEFAULT_PERSISTENCE_PATH.resolve("lambda-registry.dat");
        assertThat(Files.exists(registryFile)).isFalse();
    } finally {
        if (prev == null) System.clearProperty(key);
        else System.setProperty(key, prev);
    }
}
```

**Caveat:** `PERSISTENCE_ENABLED` in today's code is a `static final boolean` initialised once at class load. M11 will not see the mutation until Phase 6 introduces `RuntimeConfig.fromSystemProperties()` + `resetSingletonForTests()`. Mark M11 with `@Disabled("Enabled in Phase 6 after LambdaRuntime is introduced")` for this task; un-disable in Task 7.

- [ ] **Step 6.8: Run M8–M10 — expect PASS; M11 disabled**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=MVELCompilerPersistenceTest 2>&1 | tail -10
```

- [ ] **Step 6.9: Now refactor `MVELCompiler.compileEvaluator` to use `LambdaCatalog` + `LambdaPersistenceManager` directly (not through the facade)**

In `MVELCompiler.java`, replace the persistence block (lines 202–308) with the shared-decision-model flow. Pseudocode (adapt to existing types):

```java
private <C, W, O> Evaluator<C, W, O> compileEvaluator(CompilationUnit unit, CompilerParameters<C, W, O> info) {
    String javaFQN = evaluatorFullQualifiedName(unit);
    ClassManager clsManager = info.classManager() != null ? info.classManager() : new ClassManager();

    if (!LambdaRegistry.PERSISTENCE_ENABLED) {                      // Phase 6 replaces with LambdaRuntime.getInstance().config()
        compileEvaluatorClass(clsManager, info.classLoader(), unit, javaFQN);
        compileInvocations.incrementAndGet();
        Class<Evaluator<C, W, O>> def = clsManager.getClass(javaFQN);
        return createEvaluatorInstance(def);
    }

    LambdaCatalog catalog = LambdaRegistry.INSTANCE.catalog();      // add catalog() accessor on facade for now
    LambdaPersistenceManager pm = LambdaRegistry.INSTANCE.persistenceManager();

    LambdaRegistration reg = registerAndRename(unit, javaFQN);     // unchanged — still renames the class
    int physicalId = reg.physicalId();
    String newFqn = reg.newFqn();

    if (pm.artifactExists(physicalId)) {
        ArtifactRef ref = pm.artifactFor(physicalId).orElseThrow();
        LambdaArtifactLoader.loadOrDefinePersistedClass(clsManager, ref);
    } else {
        Map<String, String> sources = Map.of(newFqn, PrintUtil.printNode(unit));
        List<Path> persistedFiles = KieMemoryCompiler.compileAndPersist(
                clsManager, sources, info.classLoader(), null, LambdaRegistry.DEFAULT_PERSISTENCE_PATH);
        compileInvocations.incrementAndGet();
        pm.attachArtifact(physicalId, new ArtifactRef(newFqn, persistedFiles.get(0)));
    }

    Class<Evaluator<C, W, O>> def = clsManager.getClass(newFqn);
    return createEvaluatorInstance(def);
}
```

**Note:** `LambdaArtifactLoader` doesn't exist yet — Task 7 creates it. For this task, inline the equivalent two-line operation:
```java
if (!clsManager.getClasses().containsKey(ref.fqn())) {
    byte[] bytes = Files.readAllBytes(ref.classFile());
    clsManager.define(Map.of(ref.fqn(), bytes));
}
```
Replace with `LambdaArtifactLoader.loadOrDefinePersistedClass(clsManager, ref);` in Task 7.

Add `catalog()` and `persistenceManager()` accessors to `LambdaRegistry` (facade-temporary):
```java
public LambdaCatalog catalog() { return catalog; }
public LambdaPersistenceManager persistenceManager() { return persistenceManager; }
```

Delete `compileEvaluatorClassWithPersistence` (its content is now inlined in `compileEvaluator`).

- [ ] **Step 6.10: Refactor `MVELBatchCompiler` persistence flow similarly**

Read `MVELBatchCompiler.java` lines 60–120 and apply the same decision model in the bulk-compile loop. Bump `compileInvocations` once around the bulk `compileAndPersist` invocation.

- [ ] **Step 6.11: Run full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | tail -10
```

- [ ] **Step 6.12: Commit Phase 5**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add \
    src/main/java/org/mvel3/MVELCompiler.java \
    src/main/java/org/mvel3/MVELBatchCompiler.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java \
    src/test/java/org/mvel3/lambdaextractor/MVELCompilerPersistenceTest.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
Phase 5: Rewrite MVELCompiler + MVELBatchCompiler persistence flow

Both compilers now share the same decision model (register → query →
load-or-compile → attach), going through LambdaCatalog and
LambdaPersistenceManager directly. Persistence-disabled path bypasses
catalog/PM entirely.

Adds MVELBatchCompiler.getArtifactRef(handle) — the DRLX-facing seam
that replaces getPhysicalId for cross-repo consumers. Adds test-only
compileInvocationCount() accessor on both compilers.

Adds M8–M10 tests; M11 disabled (un-disabled in Phase 6).

Refs mvel/mvel#428

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Phase 6a — `LambdaRuntime` + `LambdaArtifactLoader` + `RuntimeConfig`

**Files:**
- Create: `src/main/java/org/mvel3/lambdaextractor/RuntimeConfig.java`
- Create: `src/main/java/org/mvel3/lambdaextractor/LambdaArtifactLoader.java`
- Modify (rewrite): `src/main/java/org/mvel3/lambdaextractor/LambdaRuntime.java` (was Phase 4 stub)
- Modify: `src/main/java/org/mvel3/lambdaextractor/LambdaPersistenceManager.java` — `attachArtifact` now actually persists via runtime
- Create: `src/test/java/org/mvel3/lambdaextractor/LambdaRuntimeTest.java` (M12–M13)
- Un-disable M11 in `MVELCompilerPersistenceTest.java`

- [ ] **Step 7.1: Create `RuntimeConfig`**

`src/main/java/org/mvel3/lambdaextractor/RuntimeConfig.java`:
```java
package org.mvel3.lambdaextractor;

import java.nio.file.Path;

/**
 * Lambda runtime configuration. Sourced from system properties at
 * {@link LambdaRuntime#getInstance()} time.
 */
public record RuntimeConfig(boolean persistenceEnabled, Path persistenceRoot,
                            Path registryFile, boolean resetOnTestStartup) {

    public static RuntimeConfig fromSystemProperties() {
        boolean persistenceEnabled = Boolean.parseBoolean(
                System.getProperty("mvel3.compiler.lambda.persistence", "true"));
        Path persistenceRoot = Path.of(
                System.getProperty("mvel3.compiler.lambda.persistence.path", "target/generated-classes/mvel"));
        Path registryFile = Path.of(System.getProperty("mvel3.compiler.lambda.registry.file",
                persistenceRoot.resolve("lambda-registry.dat").toString()));
        boolean resetOnTestStartup = Boolean.getBoolean("mvel3.compiler.lambda.resetOnTestStartup");
        return new RuntimeConfig(persistenceEnabled, persistenceRoot, registryFile, resetOnTestStartup);
    }
}
```

- [ ] **Step 7.2: Create `LambdaArtifactLoader`**

`src/main/java/org/mvel3/lambdaextractor/LambdaArtifactLoader.java`:
```java
package org.mvel3.lambdaextractor;

import java.io.IOException;
import java.nio.file.Files;
import java.util.Map;

import org.mvel3.ClassManager;

/**
 * Stateless helper for loading a persisted lambda classfile into a {@link ClassManager}.
 * Used by both MVEL compilers and DRLX consumers (cross-repo). Idempotent:
 * skips the define if the FQN is already loaded in the manager.
 */
public final class LambdaArtifactLoader {

    private LambdaArtifactLoader() {}

    public static Class<?> loadOrDefinePersistedClass(ClassManager cm, ArtifactRef ref) throws IOException {
        if (cm.getClasses().containsKey(ref.fqn())) {
            return cm.getClass(ref.fqn());
        }
        byte[] bytes = Files.readAllBytes(ref.classFile());
        cm.define(Map.of(ref.fqn(), bytes));
        return cm.getClass(ref.fqn());
    }
}
```

- [ ] **Step 7.3: Rewrite `LambdaRuntime` as the real composition root**

`src/main/java/org/mvel3/lambdaextractor/LambdaRuntime.java`:
```java
package org.mvel3.lambdaextractor;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;

/**
 * Composition root for MVEL3 lambda persistence. Lazy-initialised via
 * {@link #getInstance()}. Holds {@link LambdaCatalog}, {@link LambdaPersistenceManager},
 * {@link LambdaArtifactStore}, and an internal {@link LambdaRegistryStore}.
 * <p>
 * {@code getInstance()} and the static accessors below are transitional
 * infrastructure, not the desired long-term public model. A later cleanup
 * would inject {@code LambdaRuntime} through compiler constructors.
 */
public final class LambdaRuntime {

    private static volatile LambdaRuntime instance;

    private final RuntimeConfig config;
    private final LambdaCatalog catalog;
    private final LambdaPersistenceManager persistenceManager;
    private final LambdaArtifactStore artifactStore;
    private final LambdaRegistryStore registryStore;

    private LambdaRuntime(RuntimeConfig config) {
        this.config = config;
        this.catalog = new LambdaCatalog();
        this.artifactStore = new LambdaArtifactStore(config.persistenceRoot());
        this.persistenceManager = new LambdaPersistenceManager(artifactStore, this);
        this.registryStore = new LambdaRegistryStore(config.registryFile());
    }

    public static LambdaRuntime getInstance() {
        LambdaRuntime r = instance;
        if (r == null) {
            synchronized (LambdaRuntime.class) {
                r = instance;
                if (r == null) {
                    r = new LambdaRuntime(RuntimeConfig.fromSystemProperties());
                    r.initialize();
                    instance = r;
                }
            }
        }
        return r;
    }

    public LambdaCatalog catalog() { return catalog; }
    public LambdaPersistenceManager persistenceManager() { return persistenceManager; }
    public RuntimeConfig config() { return config; }

    /** Internal/compiler-facing seam. Not intended as long-term public API. */
    public void persistSnapshot() {
        if (!config.persistenceEnabled()) return;
        try {
            registryStore.save(new LambdaPersistenceSnapshot(catalog.toSnapshot(), persistenceManager.snapshot()));
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to persist lambda registry", e);
        }
    }

    public void reset() {
        catalog.clear();
        persistenceManager.clear();
    }

    public synchronized void resetAndRemoveAllPersistedFiles() {
        reset();
        try {
            artifactStore.deleteAll();
            Files.deleteIfExists(config.registryFile());
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    void initialize() {
        if (config.resetOnTestStartup()) {
            resetAndRemoveAllPersistedFiles();
            return;
        }
        if (config.persistenceEnabled() && Files.exists(config.registryFile())) {
            try {
                LambdaPersistenceSnapshot snapshot = registryStore.load();
                catalog.applySnapshot(snapshot.catalog());
                persistenceManager.applyArtifacts(snapshot.artifacts());
            } catch (IOException e) {
                throw new UncheckedIOException("Failed to load lambda registry", e);
            }
        }
    }

    // Transitional static accessors used by DRLX:
    public static boolean isPersistenceEnabled() { return getInstance().config.persistenceEnabled(); }
    public static Path defaultPersistencePath() { return getInstance().config.persistenceRoot(); }

    /** Test-only: clears the cached singleton so the next getInstance() re-reads system properties. */
    public static synchronized void resetSingletonForTests() {
        instance = null;
    }
}
```

- [ ] **Step 7.4: Write M12 (`resetAndRemoveAllPersistedFiles`)**

`src/test/java/org/mvel3/lambdaextractor/LambdaRuntimeTest.java`:
```java
package org.mvel3.lambdaextractor;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import static org.assertj.core.api.Assertions.assertThat;

class LambdaRuntimeTest {

    @BeforeEach
    void resetSingleton() {
        LambdaRuntime.resetSingletonForTests();
    }

    @Test
    void M12_runtime_resetAndRemoveAll_cleansEverything(@TempDir Path tmp) throws IOException {
        System.setProperty("mvel3.compiler.lambda.persistence.path", tmp.toString());
        System.setProperty("mvel3.compiler.lambda.registry.file",
                tmp.resolve("lambda-registry.dat").toString());
        LambdaRuntime.resetSingletonForTests();

        LambdaRuntime rt = LambdaRuntime.getInstance();
        Path dummyClass = tmp.resolve("Dummy.class");
        Files.writeString(dummyClass, "x");
        rt.persistenceManager().attachArtifact(0, new ArtifactRef("X.Y", dummyClass));

        Path registryFile = tmp.resolve("lambda-registry.dat");
        assertThat(Files.exists(registryFile)).isTrue();
        assertThat(Files.exists(dummyClass)).isTrue();

        rt.resetAndRemoveAllPersistedFiles();

        assertThat(Files.exists(registryFile)).isFalse();
        assertThat(Files.exists(dummyClass)).isFalse();
        assertThat(rt.persistenceManager().artifactFor(0)).isEmpty();
    }
}
```

- [ ] **Step 7.5: Run M12 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=LambdaRuntimeTest#M12_runtime_resetAndRemoveAll_cleansEverything 2>&1 | tail -10
```

- [ ] **Step 7.6: Write M13 (lazy init loads existing file)**

```java
@Test
void M13_runtime_lazyInit_loadsExistingFile(@TempDir Path tmp) throws IOException {
    System.setProperty("mvel3.compiler.lambda.persistence.path", tmp.toString());
    System.setProperty("mvel3.compiler.lambda.registry.file",
            tmp.resolve("lambda-registry.dat").toString());
    LambdaRuntime.resetSingletonForTests();

    // Pre-write a valid v2 registry file.
    LambdaRegistryStore store = new LambdaRegistryStore(tmp.resolve("lambda-registry.dat"));
    CatalogSnapshot catalog = new CatalogSnapshot(1, 1, java.util.List.of(
            new CatalogEntry(0, "public boolean eval(java.lang.Object o)", "{ return o != null; }")));
    Map<Integer, ArtifactRef> artifacts = Map.of(0, new ArtifactRef("org.example.Gen", tmp.resolve("Gen.class")));
    store.save(new LambdaPersistenceSnapshot(catalog, artifacts));

    LambdaRuntime rt = LambdaRuntime.getInstance();

    assertThat(rt.persistenceManager().artifactFor(0))
            .isPresent()
            .hasValueSatisfying(ref -> assertThat(ref.fqn()).isEqualTo("org.example.Gen"));
}
```

- [ ] **Step 7.7: Run M13 — expect PASS**

- [ ] **Step 7.8: Un-disable M11 in `MVELCompilerPersistenceTest`; update it to use `LambdaRuntime.resetSingletonForTests()`**

Replace the M11 body's old reset call with:
```java
System.setProperty("mvel3.compiler.lambda.persistence", "false");
LambdaRuntime.resetSingletonForTests();
```
And use `LambdaRuntime.defaultPersistencePath()` for path resolution rather than `LambdaRegistry.DEFAULT_PERSISTENCE_PATH` (which we're about to delete in Task 8).

Remove the `@Disabled` annotation.

- [ ] **Step 7.9: Run M11 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test -Dtest=MVELCompilerPersistenceTest#M11_runtime_persistenceDisabled_noFileWrites 2>&1 | tail -10
```

- [ ] **Step 7.10: Update `LambdaPersistenceManager.attachArtifact` to actually call the real runtime**

In `LambdaPersistenceManager.java`, the constructor already receives `LambdaRuntime runtime` (added in Phase 4). The `runtime.persistSnapshot()` call already exists but pointed at the stub. Now it points at the real method (the stub was replaced in Step 7.3).

Verify by reading `LambdaPersistenceManager.attachArtifact` — no code change needed here; the upgrade happens by virtue of `LambdaRuntime`'s new implementation.

- [ ] **Step 7.11: Run full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | tail -10
```

- [ ] **Step 7.12: Commit Phase 6a**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add \
    src/main/java/org/mvel3/lambdaextractor/RuntimeConfig.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaArtifactLoader.java \
    src/main/java/org/mvel3/lambdaextractor/LambdaRuntime.java \
    src/test/java/org/mvel3/lambdaextractor/LambdaRuntimeTest.java \
    src/test/java/org/mvel3/lambdaextractor/MVELCompilerPersistenceTest.java
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
Phase 6a: LambdaRuntime composition root + LambdaArtifactLoader

Introduces LambdaRuntime as the composition root holding all four
components, with lazy init from RuntimeConfig.fromSystemProperties().
Adds resetSingletonForTests() test hook. Adds LambdaArtifactLoader
stateless helper for cross-repo class-load.

LambdaPersistenceManager.attachArtifact now triggers a real synchronous
snapshot save via runtime.persistSnapshot().

Adds M12–M13 tests; un-disables M11.

Refs mvel/mvel#428

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Phase 6b — Delete `LambdaRegistry`; migrate callers

**Files:**
- Modify: `src/main/java/org/mvel3/MVELCompiler.java` — replace `LambdaRegistry.*` references with `LambdaRuntime.getInstance().*` and `LambdaArtifactLoader.loadOrDefinePersistedClass`
- Modify: `src/main/java/org/mvel3/MVELBatchCompiler.java` — same
- Delete: `src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java`

- [ ] **Step 8.1: Migrate `MVELCompiler` away from `LambdaRegistry`**

Replace every `LambdaRegistry.INSTANCE.foo()` and `LambdaRegistry.X` constant access:

| Before | After |
|---|---|
| `LambdaRegistry.PERSISTENCE_ENABLED` | `LambdaRuntime.getInstance().config().persistenceEnabled()` |
| `LambdaRegistry.DEFAULT_PERSISTENCE_PATH` | `LambdaRuntime.getInstance().config().persistenceRoot()` |
| `LambdaRegistry.INSTANCE.catalog()` | `LambdaRuntime.getInstance().catalog()` |
| `LambdaRegistry.INSTANCE.persistenceManager()` | `LambdaRuntime.getInstance().persistenceManager()` |
| Inlined load+define (Task 6.9) | `LambdaArtifactLoader.loadOrDefinePersistedClass(cm, ref)` |

Remove `LambdaRegistration` record if no longer needed (or move it to a package-private utility — only `registerAndRename` uses it).

- [ ] **Step 8.2: Migrate `MVELBatchCompiler` away from `LambdaRegistry`**

Same as 8.1. Also update `getArtifactRef`:
```java
public ArtifactRef getArtifactRef(LambdaHandle handle) {
    String fqn = getFqn(handle);
    Path classFile = LambdaRuntime.getInstance().persistenceManager()
            .artifactFor(getPhysicalId(handle))
            .map(ArtifactRef::classFile)
            .orElseThrow(() -> new IllegalStateException("No artifact attached for handle " + handle));
    return new ArtifactRef(fqn, classFile);
}
```

- [ ] **Step 8.3: Migrate test files that still import `LambdaRegistry`**

```bash
grep -rln "LambdaRegistry" /home/tkobayas/usr/work/mvel3-development/mvel/src/test/java
```

For each match: replace `LambdaRegistry.INSTANCE.resetAndRemoveAllPersistedFiles()` with `LambdaRuntime.getInstance().resetAndRemoveAllPersistedFiles()`. Replace `LambdaRegistry.DEFAULT_PERSISTENCE_PATH` with `LambdaRuntime.defaultPersistencePath()`. Replace `LambdaRegistry.PERSISTENCE_ENABLED` with `LambdaRuntime.isPersistenceEnabled()`.

- [ ] **Step 8.4: Confirm `LambdaRegistry` has no remaining callers**

```bash
grep -rln "LambdaRegistry" /home/tkobayas/usr/work/mvel3-development/mvel/src
```
Expected: only `LambdaRegistry.java` itself (the file we're about to delete).

- [ ] **Step 8.5: Delete `LambdaRegistry.java`**

```bash
rm /home/tkobayas/usr/work/mvel3-development/mvel/src/main/java/org/mvel3/lambdaextractor/LambdaRegistry.java
```

- [ ] **Step 8.6: Run full suite**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | tail -20
```
Expected: BUILD SUCCESS. All tests green.

- [ ] **Step 8.7: Commit Phase 6b**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel add -A
git -C /home/tkobayas/usr/work/mvel3-development/mvel commit -m "$(cat <<'EOF'
Phase 6b: Delete LambdaRegistry; migrate all callers to LambdaRuntime

Final end-state of the refactor. LambdaRegistry is gone. Compilers
(MVELCompiler, MVELBatchCompiler) and tests now consume LambdaRuntime
directly via getInstance(), and the DRLX-facing seam is established:

  - ArtifactRef
  - LambdaArtifactLoader.loadOrDefinePersistedClass
  - LambdaRuntime.isPersistenceEnabled() / defaultPersistencePath()
  - MVELBatchCompiler.getArtifactRef(handle)

No facade left behind. Static-initializer side effects fully removed.

Closes mvel/mvel#428

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Final verification + `mvn install` SNAPSHOT

**Files:** none (verification + install only)

- [ ] **Step 9.1: Confirm full test suite green**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml clean test 2>&1 | tail -20
```
Expected: BUILD SUCCESS, 0 failures, 0 errors.

- [ ] **Step 9.2: Confirm test count went up appropriately**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml test 2>&1 | grep -E "Tests run:"
```
Expected: at least baseline-count + 13 (M1–M13).

- [ ] **Step 9.3: Install MVEL3 3.0.0-SNAPSHOT to local `~/.m2`**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml install -DskipTests 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 9.4: Verify the SNAPSHOT artifact**

```bash
ls -la ~/.m2/repository/org/mvel/mvel3/3.0.0-SNAPSHOT/ 2>&1 | head -10
```
Expected: a recent `mvel3-3.0.0-SNAPSHOT.jar` file.

- [ ] **Step 9.5: Verify DRLX consumes the updated MVEL3 without crashing on the old API surface**

Smoke test by building DRLX once against the locally-installed SNAPSHOT:
```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml compile 2>&1 | tail -10
```
Expected: failure with compile errors referencing `LambdaRegistry` — these are the symbols Plan 2 (DRLX) replaces. This confirms Plan 1 is done and Plan 2 is the natural next step.

- [ ] **Step 9.6: Update HANDOFF.md (workspace)**

This concludes Plan 1. The workspace handover should record:
- MVEL `lambda-registry-refactor` branch carries Phases 1–6 commits.
- MVEL3 3.0.0-SNAPSHOT is installed locally.
- DRLX `main` does not yet build against new SNAPSHOT — Plan 2 fixes this.
- Plan 2 path: `plans/2026-05-12-drlx-lambda-boundary-implementation.md` (to be written).

(This is documentation, not a code step — handle outside the plan if running via subagent-driven execution.)

---

## Self-review

**Spec coverage (cross-check against `2026-05-12-mvel-lambda-registry-refactor-design.md`):**

| Spec item | Plan task |
|---|---|
| D1 — execute full Codex plan in one window | Tasks 1–9 cover MVEL side of the window |
| D2 — PM owns physicalId→ArtifactRef map | Task 5 (Phase 4) creates PM with this map |
| D3 — PM is service-layer, not compile wrapper | Task 6 (Phase 5) — compilers keep orchestration |
| D4 — Properties v2 format | Task 3 (Phase 2) |
| D5 — Lazy init on `getInstance()` | Task 7 (Phase 6a) |
| D6 — DRLX develops on main against SNAPSHOT | Step 9.5 verifies the SNAPSHOT exists; Plan 2 consumes it |
| D7 — Phase 0 tests M1–M13 | Tasks 2 (M1–M4), 3 (M5–M7), 6 (M8–M11), 7 (M12–M13) |
| D8 — `RegistrationResult` has no fqn | Task 2 Step 2.1 |
| D9 — DRLX-facing seam narrow | Verified after Task 8; D7 (architectural guard) lives in Plan 2 |
| D10 — distinct exception types | Task 3 (Phase 2) creates `InvalidLambdaRegistryException` |
| D11 — duplicate physicalId is malformed | Task 3 Step 3.8 (M7a) |
| D12 — `compileInvocationCount` test seam | Task 6 Steps 6.1, 6.2 |
| D13 — `attachArtifact` triggers `persistSnapshot` | Task 7 (real impl); Task 5 (stub wiring) |
| D14 — persistence-disabled bypasses catalog/PM | Task 6 Step 6.9 (top-level branch) |
| File formats sample (registry file) | Task 3 schema implementation |
| Lifecycle (`initialize` with reset-first ordering) | Task 7 Step 7.3 |
| MVEL `LambdaRegistry` deleted outright | Task 8 Step 8.5 |

All spec items mapped. No gaps.

**Placeholder scan:** searched for TBD/TODO/FIXME — none in this plan. All steps contain executable content. The phrase "Adjust `MVEL.pojo(...)` shape to match the project's current API if needed" in Step 6.3 is a real engineering instruction (verify against an existing test before running), not a placeholder.

**Type consistency:**
- `LambdaCatalog.register(LambdaKey)` returns `RegistrationResult(int logicalId, int physicalId, boolean reused)` — used identically in Steps 2.2, 2.7, 2.9, 2.10.
- `ArtifactRef(String fqn, Path classFile)` — used identically across Tasks 3, 5, 6, 7, 8.
- `LambdaPersistenceManager.artifactExists(int physicalId)` — used in Tasks 5, 6, 7. No accidental `isPersisted` renaming.
- `LambdaRuntime.getInstance()` — singular entry point; never `LambdaRuntime.instance()` or similar drift.
- `LambdaArtifactLoader.loadOrDefinePersistedClass(ClassManager, ArtifactRef)` — same signature in Task 7 definition and all use sites (Tasks 6.9, 8.1).

---

**Plan 1 complete. Save target:** `/home/tkobayas/claude/public/drlx-parser/plans/2026-05-12-mvel-lambda-registry-refactor-implementation.md`

Plan 2 (DRLX-side consumption + Phase C cutover) follows separately.
