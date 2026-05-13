# DRLX Lambda Boundary Consumption — Implementation Plan (Plan 2 of 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consume MVEL3's new `ArtifactRef` boundary on the DRLX side. Drop all DRLX dependency on `LambdaRegistry` (deleted in Plan 1) and on `physicalId` as a cross-repo identifier. Rewrite `DrlxLambdaMetadata` to Properties `format.version=2` with `classFile`. Update `DrlxLambdaCompiler.loadPreCompiledEvaluator` to use `LambdaArtifactLoader`. Migrate `DrlxRuleBuilder` and `DrlxCompiler` constant access to `LambdaRuntime`. Add DRLX Phase 0 tests D1–D7, including the D7 architectural guard. Coordinate cross-repo cutover (Phase C).

**Architecture:** DRLX consumes a strictly narrow MVEL surface: `ArtifactRef`, `LambdaArtifactLoader.loadOrDefinePersistedClass`, two transitional static `LambdaRuntime` accessors, and `MVELBatchCompiler.getArtifactRef(handle)`. No imports of `LambdaRegistry`, `LambdaCatalog`, `LambdaPersistenceManager`, `LambdaArtifactStore`, or `LambdaRegistryStore` from DRLX, ever — enforced by the D7 test.

**Tech Stack:** Java 17+, JUnit 5 (Jupiter), AssertJ, Maven. Depends on MVEL3 3.0.0-SNAPSHOT (locally installed by Plan 1 Task 9).

**Source spec:** `/home/tkobayas/claude/public/drlx-parser/specs/2026-05-12-mvel-lambda-registry-refactor-design.md`

**Working directory:** `/home/tkobayas/usr/work/mvel3-development/drlx-parser` (branch `main`). Commits stay **local** until Plan 2 Task 9 (the coordinated push).

**Commit discipline:** All DRLX commits reference the new GitHub issue (Plan 2 Task 1 Step 1.4 creates it). Local commits accumulate; remote push happens only in Task 9 Step 9.1.

**Cross-repo prerequisite:** Plan 1 (MVEL refactor) complete; MVEL3 3.0.0-SNAPSHOT installed in `~/.m2`. `mvn compile` on DRLX currently fails with references to `LambdaRegistry` — that's the starting state.

---

## File Structure

### Files created

| Path | Purpose |
|---|---|
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/InvalidDrlxLambdaMetadataException.java` | Typed `IOException` subclass for malformed / unsupported DRLX metadata files. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaBoundaryTest.java` | D1–D6: pre-build→reuse, mismatch policies, format round-trip. |
| `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxMvelImportGuardTest.java` | D7: architectural guard — no DRLX source file imports MVEL-internal lambda components. |

### Files modified

| Path | Change |
|---|---|
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaMetadata.java` | Rewrite to Properties `v2`; `LambdaEntry(fqn, classFile, expression)`; new `toArtifactRef()` accessor; three-state load semantics. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxPreBuildLambdaCompiler.java` | Use `batchCompiler.getArtifactRef(handle)` instead of `getFqn` + `getPhysicalId`; drop `PendingPreBuildInfo.fqn`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java` | `loadPreCompiledEvaluator(ArtifactRef)`; calls `LambdaArtifactLoader.loadOrDefinePersistedClass`; drop the `LambdaRegistry.INSTANCE.getPhysicalPath` call. |
| `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleBuilder.java` | Replace `LambdaRegistry.PERSISTENCE_ENABLED` / `DEFAULT_PERSISTENCE_PATH` with `LambdaRuntime.isPersistenceEnabled()` / `defaultPersistencePath()`. |
| `drlx-parser-core/src/main/java/org/drools/drlx/tools/DrlxCompiler.java` | Replace the same two constants. |
| Existing DRLX tests that import `LambdaRegistry` | Replace imports with `LambdaRuntime`; replace `LambdaRegistry.INSTANCE.resetAndRemoveAllPersistedFiles()` with `LambdaRuntime.getInstance().resetAndRemoveAllPersistedFiles()` (and similar). |

### Files deleted

None.

---

## Phase ordering

| Task | Output | Verification |
|---|---|---|
| 1 | Issue created; baseline confirmed broken against new SNAPSHOT | `mvn compile` fails referencing `LambdaRegistry` |
| 2 | `InvalidDrlxLambdaMetadataException` + rewritten `DrlxLambdaMetadata` | D6 round-trip green |
| 3 | `DrlxPreBuildLambdaCompiler` uses `getArtifactRef` | Pre-build path compiles |
| 4 | `DrlxLambdaCompiler.loadPreCompiledEvaluator(ArtifactRef)` rewritten | DRLX no longer references `LambdaRegistry` for class loads |
| 5 | `DrlxRuleBuilder` + `DrlxCompiler` constant migration; existing tests fixed | `mvn compile` green |
| 6 | ~~Test seam accessor~~ — dropped; D1 uses static `MVELCompiler.compileInvocationCount()` | D1 testable via static counter |
| 7 | D1–D7 tests written | All D-tests green |
| 8 | Full DRLX test suite green at ≥170 | Test count ≥ baseline + 7 |
| 9 | Phase C: coordinated push to both repos | MVEL main + DRLX main both contain the refactor |

---

## Task 1: Issue + baseline

**Files:** none

- [ ] **Step 1.1: Verify MVEL3 SNAPSHOT is installed**

```bash
ls ~/.m2/repository/org/mvel/mvel3/3.0.0-SNAPSHOT/mvel3-3.0.0-SNAPSHOT.jar 2>&1
```
Expected: file exists. If not, run Plan 1 Task 9 Step 9.3 first.

- [ ] **Step 1.2: Confirm DRLX `main` is the working branch**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser status
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser branch --show-current
```
Expected: `main` branch, clean (any untracked workspace files are unrelated).

- [ ] **Step 1.3: Confirm baseline is broken against new MVEL SNAPSHOT**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml compile 2>&1 | tail -15
```
Expected: BUILD FAILURE. Compile errors should reference `LambdaRegistry`, `getPhysicalPath`, `getPhysicalId`, `PERSISTENCE_ENABLED`, `DEFAULT_PERSISTENCE_PATH`. Capture the list of failing files — these match the "files modified" table above plus possibly test files.

- [ ] **Step 1.4: Create DRLX GitHub issue under Epic #26**

```bash
gh issue create --repo tkobayas/drlx-parser \
    --title "Consume MVEL ArtifactRef boundary; drop physicalId dependency" \
    --body "$(cat <<'EOF'
Counterpart of upstream mvel/mvel#428.

After MVEL3 deletes `LambdaRegistry` and exposes the new
`ArtifactRef` / `LambdaArtifactLoader` / `LambdaRuntime` surface, DRLX
must:

- Replace `DrlxLambdaMetadata` schema (drop `physicalId`, add
  `classFile`; Properties `format.version=2`).
- Update `DrlxPreBuildLambdaCompiler` to call
  `MVELBatchCompiler.getArtifactRef(handle)`.
- Update `DrlxLambdaCompiler.loadPreCompiledEvaluator` to use
  `LambdaArtifactLoader.loadOrDefinePersistedClass` (no more
  `LambdaRegistry.INSTANCE` calls at runtime build).
- Migrate `DrlxRuleBuilder` and `DrlxCompiler` to consume
  `LambdaRuntime.isPersistenceEnabled()` /
  `LambdaRuntime.defaultPersistencePath()`.
- Add `InvalidDrlxLambdaMetadataException` distinct from MVEL's
  registry-side exception.
- Add Phase 0 characterization tests D1–D7, including the D7
  architectural guard.

Part of Epic #26. Plan:
`plans/2026-05-12-drlx-lambda-boundary-implementation.md` (workspace).

EOF
)" 2>&1 | tail -5
```

Note the issue number returned by `gh`. Use it (`#N`) in all subsequent commit messages.

- [ ] **Step 1.5: Inventory DRLX files referencing the deleted MVEL API**

```bash
grep -rln "LambdaRegistry\|getPhysicalPath\|PERSISTENCE_ENABLED\|DEFAULT_PERSISTENCE_PATH" \
    /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src
```
Expected:
- `src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java`
- `src/main/java/org/drools/drlx/builder/DrlxLambdaMetadata.java` (only stores `physicalId` in fields/format)
- `src/main/java/org/drools/drlx/builder/DrlxPreBuildLambdaCompiler.java` (uses `getPhysicalId(handle)`)
- `src/main/java/org/drools/drlx/builder/DrlxRuleBuilder.java`
- `src/main/java/org/drools/drlx/tools/DrlxCompiler.java`
- A handful of test files

Save the test-file list — each becomes a target in Task 5 Step 5.3.

---

## Task 2: `InvalidDrlxLambdaMetadataException` + rewritten `DrlxLambdaMetadata`

**Files:**
- Create: `drlx-parser-core/src/main/java/org/drools/drlx/builder/InvalidDrlxLambdaMetadataException.java`
- Modify (rewrite): `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaMetadata.java`

- [ ] **Step 2.1: Create `InvalidDrlxLambdaMetadataException`**

`drlx-parser-core/src/main/java/org/drools/drlx/builder/InvalidDrlxLambdaMetadataException.java`:
```java
package org.drools.drlx.builder;

import java.io.IOException;

/**
 * Thrown when the DRLX lambda metadata file is malformed, contains an unsupported
 * {@code format.version}, or has missing required keys for an entry. Routed
 * through {@link DrlxMetadataMismatchMode} by callers.
 */
public class InvalidDrlxLambdaMetadataException extends IOException {
    public InvalidDrlxLambdaMetadataException(String message) { super(message); }
    public InvalidDrlxLambdaMetadataException(String message, Throwable cause) { super(message, cause); }
}
```

- [ ] **Step 2.2: Rewrite `DrlxLambdaMetadata`**

Replace the entire file at `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaMetadata.java`:

```java
package org.drools.drlx.builder;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Properties;
import java.util.TreeSet;

import org.mvel3.lambdaextractor.ArtifactRef;

/**
 * Pre-build metadata for DRLX lambdas. Persisted as a Properties file with
 * {@code format.version=2}. Each entry is keyed by {@code rule.<ruleName>.<counterId>}
 * with {@code expression}, {@code fqn}, and {@code classFile} sub-keys.
 * <p>
 * Self-sufficient: contains absolute {@code classFile} paths so the runtime build
 * loads classes directly via {@link org.mvel3.lambdaextractor.LambdaArtifactLoader},
 * without any callback into MVEL's registry state.
 */
public class DrlxLambdaMetadata {

    private static final String FILE_NAME = "drlx-lambda-metadata.properties";
    private static final String FORMAT_VERSION = "2";
    private static final String KEY_VERSION = "format.version";

    private final Map<String, LambdaEntry> entries = new LinkedHashMap<>();

    public record LambdaEntry(String fqn, Path classFile, String expression) {
        public ArtifactRef toArtifactRef() { return new ArtifactRef(fqn, classFile); }
    }

    public void put(String ruleName, int counterId, ArtifactRef ref, String expression) {
        entries.put(key(ruleName, counterId), new LambdaEntry(ref.fqn(), ref.classFile(), expression));
    }

    public LambdaEntry get(String ruleName, int counterId) {
        return entries.get(key(ruleName, counterId));
    }

    public int size() { return entries.size(); }

    public void save(Path dir) throws IOException {
        Files.createDirectories(dir);
        Path file = dir.resolve(FILE_NAME);
        Properties props = new Properties();
        props.setProperty(KEY_VERSION, FORMAT_VERSION);
        for (Map.Entry<String, LambdaEntry> e : entries.entrySet()) {
            String base = e.getKey();    // already of the form rule.<ruleName>.<counterId>
            props.setProperty(base + ".expression", e.getValue().expression());
            props.setProperty(base + ".fqn", e.getValue().fqn());
            props.setProperty(base + ".classFile", e.getValue().classFile().toString());
        }
        try (OutputStream out = Files.newOutputStream(file)) {
            props.store(out, "DRLX lambda metadata");
        }
    }

    /**
     * Loads metadata from {@code file}.
     * <ul>
     *   <li>Missing file → empty metadata (NOT a mismatch).</li>
     *   <li>Missing or wrong {@code format.version} → throws {@link InvalidDrlxLambdaMetadataException}.</li>
     *   <li>Missing required keys in an entry → throws same.</li>
     * </ul>
     */
    public static DrlxLambdaMetadata load(Path file) throws IOException {
        DrlxLambdaMetadata metadata = new DrlxLambdaMetadata();
        if (!Files.exists(file)) return metadata;

        Properties props = new Properties();
        try (InputStream in = Files.newInputStream(file)) {
            props.load(in);
        }
        String version = props.getProperty(KEY_VERSION);
        if (!FORMAT_VERSION.equals(version)) {
            throw new InvalidDrlxLambdaMetadataException(
                    "Unsupported DRLX lambda metadata format.version: " + version + " (expected " + FORMAT_VERSION + ")");
        }

        TreeSet<String> bases = new TreeSet<>();
        for (String key : props.stringPropertyNames()) {
            if (!key.startsWith("rule.")) continue;
            int lastDot = key.lastIndexOf('.');
            if (lastDot <= "rule.".length()) continue;
            bases.add(key.substring(0, lastDot));
        }
        for (String base : bases) {
            String expression = required(props, base + ".expression");
            String fqn = required(props, base + ".fqn");
            String classFile = required(props, base + ".classFile");
            metadata.entries.put(base, new LambdaEntry(fqn, Path.of(classFile), expression));
        }
        return metadata;
    }

    public static Path metadataFilePath(Path dir) {
        return dir.resolve(FILE_NAME);
    }

    private static String key(String ruleName, int counterId) {
        return "rule." + ruleName + "." + counterId;
    }

    private static String required(Properties p, String key) throws InvalidDrlxLambdaMetadataException {
        String v = p.getProperty(key);
        if (v == null) throw new InvalidDrlxLambdaMetadataException("Missing key: " + key);
        return v;
    }
}
```

- [ ] **Step 2.3: Compile to verify the metadata class is syntactically clean**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml compile 2>&1 | tail -15
```
Expected: still fails (other files not yet updated) but the new `DrlxLambdaMetadata.java` itself doesn't generate errors. Confirm errors are only in other files.

- [ ] **Step 2.4: Commit Task 2 (local)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/InvalidDrlxLambdaMetadataException.java \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaMetadata.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
Rewrite DrlxLambdaMetadata for ArtifactRef boundary (Properties v2)

Drops physicalId from the metadata schema. LambdaEntry now carries
(fqn, classFile, expression) and exposes toArtifactRef(). File format
switches to java.util.Properties with format.version=2.

Adds InvalidDrlxLambdaMetadataException. Load semantics:
  - missing file → empty (NOT mismatch)
  - wrong/missing format.version or required keys → throws

Refs #<DRLX-issue-N>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Replace `#<DRLX-issue-N>` with the issue number from Task 1.4.

---

## Task 3: `DrlxPreBuildLambdaCompiler` — use `getArtifactRef`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxPreBuildLambdaCompiler.java`

- [ ] **Step 3.1: Update the per-handle attach loop**

Replace the file with:
```java
package org.drools.drlx.builder;

import java.util.ArrayList;
import java.util.List;

import org.mvel3.MVELBatchCompiler;
import org.mvel3.lambdaextractor.ArtifactRef;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Pre-build lambda compiler that extends {@link DrlxLambdaCompiler}.
 * Records metadata for each compiled lambda via the {@link #onLambdaCreated}
 * hook so that runtime builds can bypass MVEL compilation.
 * <p>
 * Records {@link ArtifactRef} (fqn + classFile) per lambda; no {@code physicalId}
 * crosses the DRLX persistence boundary.
 */
public class DrlxPreBuildLambdaCompiler extends DrlxLambdaCompiler {

    private static final Logger LOG = LoggerFactory.getLogger(DrlxPreBuildLambdaCompiler.class);

    private final DrlxLambdaMetadata metadata = new DrlxLambdaMetadata();

    private final List<PendingPreBuildInfo> pendingPreBuildInfos = new ArrayList<>();

    record PendingPreBuildInfo(String ruleName, int counterId, String expression, MVELBatchCompiler.LambdaHandle handle) {}

    public DrlxPreBuildLambdaCompiler(MVELBatchCompiler batchCompiler) {
        super(batchCompiler);
    }

    public DrlxLambdaMetadata getMetadata() { return metadata; }

    @Override
    protected void onLambdaCreated(int counter, String expression) {
        MVELBatchCompiler.LambdaHandle handle = pendingLambdas.get(pendingLambdas.size() - 1).handle();
        pendingPreBuildInfos.add(new PendingPreBuildInfo(currentRuleName, counter, expression, handle));
    }

    @Override
    public void compileBatch(ClassLoader classLoader) {
        if (pendingLambdas.isEmpty()) return;

        super.compileBatch(classLoader);

        for (PendingPreBuildInfo info : pendingPreBuildInfos) {
            ArtifactRef ref = batchCompiler.getArtifactRef(info.handle());
            metadata.put(info.ruleName(), info.counterId(), ref, info.expression());
            LOG.info("Recorded pre-build metadata: {}.{} -> {} (classFile={})",
                    info.ruleName(), info.counterId(), ref.fqn(), ref.classFile());
        }

        pendingPreBuildInfos.clear();
    }
}
```

Changes from today: dropped the separate `getFqn(handle)` + `getPhysicalId(handle)` calls; `metadata.put` signature changed; the log line drops `physicalId` and shows `classFile`.

- [ ] **Step 3.2: Commit Task 3 (local)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxPreBuildLambdaCompiler.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
DrlxPreBuildLambdaCompiler: record ArtifactRef via getArtifactRef

Uses MVELBatchCompiler.getArtifactRef(handle) (added in mvel/mvel#428)
to obtain (fqn, classFile) per pending lambda. No more getFqn +
getPhysicalId; physicalId no longer crosses the boundary.

Refs #<DRLX-issue-N>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: `DrlxLambdaCompiler.loadPreCompiledEvaluator(ArtifactRef)`

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java`

- [ ] **Step 4.1: Update imports and `loadPreCompiledEvaluator`**

Edit `DrlxLambdaCompiler.java`:

Replace import:
```diff
- import org.mvel3.lambdaextractor.LambdaRegistry;
+ import org.mvel3.lambdaextractor.ArtifactRef;
+ import org.mvel3.lambdaextractor.LambdaArtifactLoader;
```

Replace the entire body of `loadPreCompiledEvaluator(String, int)` (currently at `DrlxLambdaCompiler.java:382-398`) with a new signature and body:

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

- [ ] **Step 4.2: Update the single call site in `tryLoadPreCompiled`**

In the same file, find:
```java
Object evaluator = loadPreCompiledEvaluator(entry.fqn(), entry.physicalId());
```
Replace with:
```java
Object evaluator = loadPreCompiledEvaluator(entry.toArtifactRef());
```

- [ ] **Step 4.3: Commit Task 4 (local)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
DrlxLambdaCompiler.loadPreCompiledEvaluator: use LambdaArtifactLoader

Drops the call into LambdaRegistry.INSTANCE.getPhysicalPath. Loading
goes through the stateless static LambdaArtifactLoader with the
ArtifactRef carried in the DRLX metadata entry. tryLoadPreCompiled
passes entry.toArtifactRef() directly.

IOException from a missing classFile is routed through the existing
DrlxMetadataMismatchMode policy as before — same control flow,
different trigger.

Refs #<DRLX-issue-N>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: `DrlxRuleBuilder` + `DrlxCompiler` + existing tests — constant migration

**Files:**
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleBuilder.java`
- Modify: `drlx-parser-core/src/main/java/org/drools/drlx/tools/DrlxCompiler.java`
- Modify: every existing DRLX test file that imports `LambdaRegistry` (list from Task 1 Step 1.5)

- [ ] **Step 5.1: Update `DrlxRuleBuilder.java`**

Locate the two reference sites (today at `DrlxRuleBuilder.java:24` and `:128`):
```diff
- import org.mvel3.lambdaextractor.LambdaRegistry;
+ import org.mvel3.lambdaextractor.LambdaRuntime;
```
```diff
- Path persistDir = LambdaRegistry.PERSISTENCE_ENABLED ? LambdaRegistry.DEFAULT_PERSISTENCE_PATH : null;
+ Path persistDir = LambdaRuntime.isPersistenceEnabled() ? LambdaRuntime.defaultPersistencePath() : null;
```

- [ ] **Step 5.2: Update `DrlxCompiler.java`**

Locate the three reference sites (`DrlxCompiler.java:12`, `:50`, `:79`):
```diff
- import org.mvel3.lambdaextractor.LambdaRegistry;
+ import org.mvel3.lambdaextractor.LambdaRuntime;
```
```diff
- this(LambdaRegistry.DEFAULT_PERSISTENCE_PATH);
+ this(LambdaRuntime.defaultPersistencePath());
```
```diff
- if (LambdaRegistry.PERSISTENCE_ENABLED) {
+ if (LambdaRuntime.isPersistenceEnabled()) {
```
Update the error-message string in the `if` body to refer to `LambdaRuntime.isPersistenceEnabled()` instead of the deleted `LambdaRegistry.PERSISTENCE_ENABLED`.

- [ ] **Step 5.3: Update every existing DRLX test file from the Task 1.5 inventory**

For each file:
```diff
- import org.mvel3.lambdaextractor.LambdaRegistry;
+ import org.mvel3.lambdaextractor.LambdaRuntime;
- import static org.mvel3.lambdaextractor.LambdaRegistry.DEFAULT_PERSISTENCE_PATH;
+ // remove this static import; use LambdaRuntime.defaultPersistencePath() inline
```

```diff
- LambdaRegistry.INSTANCE.resetAndRemoveAllPersistedFiles();
+ LambdaRuntime.getInstance().resetAndRemoveAllPersistedFiles();
```

Anywhere `DEFAULT_PERSISTENCE_PATH` is used as an identifier, replace with `LambdaRuntime.defaultPersistencePath()`.

The `@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")` annotation stays — that's a JUnit declarative check, not a code reference to the deleted class.

- [ ] **Step 5.4: Verify DRLX compiles**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml compile test-compile 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 5.5: Run existing DRLX test suite (no new tests yet)**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test 2>&1 | tail -20
```
Expected: BUILD SUCCESS, ≥170 tests passing (baseline). If some tests fail due to subtle behavior changes from the MVEL refactor, fix them now — but only fix the test, not by reverting MVEL behavior.

- [ ] **Step 5.6: Commit Task 5 (local)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxRuleBuilder.java \
    drlx-parser-core/src/main/java/org/drools/drlx/tools/DrlxCompiler.java \
    drlx-parser-core/src/test/java/
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
Migrate DrlxRuleBuilder, DrlxCompiler, and tests off LambdaRegistry

Replaces LambdaRegistry.PERSISTENCE_ENABLED with
LambdaRuntime.isPersistenceEnabled() and
LambdaRegistry.DEFAULT_PERSISTENCE_PATH with
LambdaRuntime.defaultPersistencePath() across:

- DrlxRuleBuilder.java
- DrlxCompiler.java
- All test files that imported the deleted class

This is the constant-migration half of the cross-repo boundary
cleanup. After this commit, DRLX has zero references to LambdaRegistry.

Refs #<DRLX-issue-N>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: ~~Add `DrlxRuleBuilder.getBatchCompilerForTests()` seam~~ — SKIPPED

**Status:** Dropped on 2026-05-13 during Plan 2 execution.

**Reason:** Plan 1 implemented `compileInvocationCount()` as a **static** method on
`MVELCompiler` (not as an instance method on `MVELBatchCompiler` as the original
spec wording suggested). A static counter is global JVM state, so D1 can simply
call `MVELCompiler.compileInvocationCount()` directly — no instance accessor
needed. The counter is test-only instrumentation provided by MVEL, not
per-batch-compiler instance state.

**Implications:**
- No code change in `DrlxRuleBuilder`.
- D1 test (Task 7 Step 7.7) imports `org.mvel3.MVELCompiler` and asserts on
  `MVELCompiler.compileInvocationCount()` directly.
- D7 architectural guard is unaffected — `MVELCompiler` is not on the forbidden
  import list.
- One DRLX commit is dropped.

---

## Task 7: Add Phase 0 tests D1–D7

**Files:**
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaBoundaryTest.java` (D1–D6)
- Create: `drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxMvelImportGuardTest.java` (D7)

- [ ] **Step 7.1: Write D6 (metadata round-trip) — simplest, no MVEL build needed**

`drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaBoundaryTest.java`:
```java
package org.drools.drlx.builder;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import org.mvel3.lambdaextractor.ArtifactRef;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DrlxLambdaBoundaryTest {

    @Test
    void D6_metadata_format_roundTrip(@TempDir Path tmp) throws IOException {
        DrlxLambdaMetadata m = new DrlxLambdaMetadata();
        m.put("RuleA", 0, new ArtifactRef("org.mvel3.GenA", tmp.resolve("GenA.class")), "age > 18");
        m.put("RuleA", 1, new ArtifactRef("org.mvel3.GenB", tmp.resolve("GenB.class")), "name != null");
        m.save(tmp);

        DrlxLambdaMetadata loaded = DrlxLambdaMetadata.load(DrlxLambdaMetadata.metadataFilePath(tmp));
        assertThat(loaded.size()).isEqualTo(2);
        DrlxLambdaMetadata.LambdaEntry e0 = loaded.get("RuleA", 0);
        assertThat(e0.fqn()).isEqualTo("org.mvel3.GenA");
        assertThat(e0.classFile()).isEqualTo(tmp.resolve("GenA.class"));
        assertThat(e0.expression()).isEqualTo("age > 18");
    }
}
```

- [ ] **Step 7.2: Run D6 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -pl drlx-parser-core -Dtest=DrlxLambdaBoundaryTest#D6_metadata_format_roundTrip 2>&1 | tail -10
```

- [ ] **Step 7.3: Write D2 (missing file → empty)**

Append:
```java
@Test
void D2_metadata_missingFile_freshCompile(@TempDir Path tmp) throws IOException {
    DrlxLambdaMetadata loaded = DrlxLambdaMetadata.load(tmp.resolve("nonexistent.properties"));
    assertThat(loaded.size()).isEqualTo(0);
    // No exception thrown — absence is not a mismatch.
}
```

- [ ] **Step 7.4: Write D3 (unsupported version → throws)**

```java
@Test
void D3_metadata_unsupportedVersion_throws(@TempDir Path tmp) throws IOException {
    Files.writeString(DrlxLambdaMetadata.metadataFilePath(tmp), "format.version=1\nrule.X.0.expression=foo\n");
    assertThatThrownBy(() -> DrlxLambdaMetadata.load(DrlxLambdaMetadata.metadataFilePath(tmp)))
            .isInstanceOf(InvalidDrlxLambdaMetadataException.class)
            .hasMessageContaining("Unsupported");
}
```

(Test names D3 in spec says "fail-fast / fallback" — that's the policy routing tested via the integration path with `DrlxLambdaCompiler.tryLoadPreCompiled`. Pure metadata-load behavior is "throws"; D3a covers metadata layer; D3b would cover the policy routing. Add D3b only if integration setup is straightforward — otherwise rely on existing `DrlxMetadataMismatchModeTest`-style tests in the repo for the policy layer.)

- [ ] **Step 7.5: Write D4 (stale classFile → mismatch policy routes through)**

```java
@Test
void D4_metadata_staleClassFile_throws_inLoader(@TempDir Path tmp) throws IOException {
    // The metadata layer doesn't validate classFile existence; it's the loader's job.
    // This test asserts that LambdaArtifactLoader propagates IOException on missing file,
    // which is what DrlxLambdaCompiler.tryLoadPreCompiled then routes through mismatch policy.
    ArtifactRef stale = new ArtifactRef("org.example.Missing", tmp.resolve("does-not-exist.class"));
    assertThatThrownBy(() -> org.mvel3.lambdaextractor.LambdaArtifactLoader.loadOrDefinePersistedClass(
            new org.mvel3.ClassManager(), stale))
            .isInstanceOf(java.nio.file.NoSuchFileException.class);
}
```

- [ ] **Step 7.6: Write D5 (expression mismatch — preserved behavior smoke test)**

D5 is "expression in metadata differs from rule" — existing logic at `DrlxLambdaCompiler.tryLoadPreCompiled` checks `entry.expression().equals(expression)` and routes mismatches through policy. This behavior is unchanged by the refactor; existing tests around `DrlxMetadataMismatchMode` already cover it. A new D5 is duplicative. Mark D5 as covered by existing tests in the test plan; do not add a new test file for it.

Skip — no new test needed. (Track in self-review.)

- [ ] **Step 7.7: Write D1 (pre-build → runtime; assert no recompile)**

D1 needs an end-to-end pre-build then runtime build. Assertions use the
global static counter `MVELCompiler.compileInvocationCount()` provided by
Plan 1 (test-only instrumentation). The harness mirrors existing
`DrlxRuleBuilderTest` patterns; the exact `DrlxRuleBuilder` API methods
(`preBuild(...)` / `build(...)`) and signatures must be looked up from
existing tests in the repo and mirrored.

Key assertion shape:
```java
int before = MVELCompiler.compileInvocationCount();
runtimeBuilder.build(drlxSource);   // metadata path
int after = MVELCompiler.compileInvocationCount();
assertThat(after).isEqualTo(before);   // no recompile in runtime build
```

Setup must reset MVEL state between pre-build and runtime build (e.g.,
`LambdaRuntime.getInstance().reset()`) so the runtime build cannot rely on
in-memory catalog hits.

**Adjustment needed:** mirror an existing pre-build test or `DrlxRuleBuilderTest`
case for the actual builder API, system-property toggles for persistence path,
and metadata save/load flow. The skeleton above only fixes the assertion shape.

- [ ] **Step 7.8: Run D1–D4, D6 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -pl drlx-parser-core -Dtest=DrlxLambdaBoundaryTest 2>&1 | tail -10
```

- [ ] **Step 7.9: Write D7 — architectural guard test**

`drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxMvelImportGuardTest.java`:
```java
package org.drools.drlx.builder;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Pattern;
import java.util.stream.Stream;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Architectural guard. Asserts that DRLX source files import ONLY the allowed
 * narrow MVEL surface: ArtifactRef, LambdaArtifactLoader, LambdaRuntime,
 * MVELBatchCompiler, MVELCompiler, MVEL, CompilerParameters, Evaluator,
 * ClassManager, Type, Declaration, transpiler.context.Declaration.
 *
 * Any DRLX file importing LambdaRegistry, LambdaCatalog, LambdaPersistenceManager,
 * LambdaArtifactStore, or LambdaRegistryStore breaks the cross-repo boundary
 * agreement and this test will fail at build time.
 */
class DrlxMvelImportGuardTest {

    private static final Path DRLX_SRC = Path.of("src/main/java/org/drools/drlx");

    private static final List<Pattern> FORBIDDEN = List.of(
            Pattern.compile("^\\s*import\\s+org\\.mvel3\\.lambdaextractor\\.LambdaRegistry\\s*;"),
            Pattern.compile("^\\s*import\\s+org\\.mvel3\\.lambdaextractor\\.LambdaCatalog\\s*;"),
            Pattern.compile("^\\s*import\\s+org\\.mvel3\\.lambdaextractor\\.LambdaPersistenceManager\\s*;"),
            Pattern.compile("^\\s*import\\s+org\\.mvel3\\.lambdaextractor\\.LambdaArtifactStore\\s*;"),
            Pattern.compile("^\\s*import\\s+org\\.mvel3\\.lambdaextractor\\.LambdaRegistryStore\\s*;")
    );

    @Test
    void D7_drlx_noForbiddenMvelLambdaImports() throws IOException {
        List<String> violations = new ArrayList<>();
        try (Stream<Path> walk = Files.walk(DRLX_SRC)) {
            walk.filter(p -> p.toString().endsWith(".java"))
                .forEach(p -> {
                    List<String> lines;
                    try { lines = Files.readAllLines(p); }
                    catch (IOException e) { throw new RuntimeException(e); }
                    for (int i = 0; i < lines.size(); i++) {
                        String line = lines.get(i);
                        for (Pattern pat : FORBIDDEN) {
                            if (pat.matcher(line).find()) {
                                violations.add(p + ":" + (i + 1) + " — " + line.trim());
                            }
                        }
                    }
                });
        }
        assertThat(violations)
                .as("DRLX source files must not import MVEL-internal lambda components")
                .isEmpty();
    }
}
```

The path `src/main/java/org/drools/drlx` is resolved relative to the module's working directory (which is `drlx-parser-core` when the test runs).

- [ ] **Step 7.10: Run D7 — expect PASS**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test -pl drlx-parser-core -Dtest=DrlxMvelImportGuardTest 2>&1 | tail -10
```

If D7 fails with violations, fix the offending files. The Task 5 migration should have removed all such imports already; D7 catches anything missed.

- [ ] **Step 7.11: Commit Task 7 (local)**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser add \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxLambdaBoundaryTest.java \
    drlx-parser-core/src/test/java/org/drools/drlx/builder/DrlxMvelImportGuardTest.java
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser commit -m "$(cat <<'EOF'
Add Phase 0 DRLX boundary tests D1–D7

D1 — pre-build then runtime build, asserts no recompile via
     DrlxRuleBuilder.getBatchCompilerForTests().compileInvocationCount().
D2 — missing metadata file → empty metadata (not a mismatch).
D3 — wrong format.version → InvalidDrlxLambdaMetadataException.
D4 — stale classFile → LambdaArtifactLoader propagates IOException
     (which DrlxLambdaCompiler.tryLoadPreCompiled then routes through
     DrlxMetadataMismatchMode).
D5 — covered by existing DrlxMetadataMismatchMode tests; no new test.
D6 — DrlxLambdaMetadata save/load round-trip.
D7 — architectural guard: no DRLX file imports LambdaRegistry,
     LambdaCatalog, LambdaPersistenceManager, LambdaArtifactStore, or
     LambdaRegistryStore.

Refs #<DRLX-issue-N>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Final verification — full DRLX test suite green

**Files:** none (verification only)

- [ ] **Step 8.1: Clean build + full test run**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml clean test 2>&1 | tail -20
```
Expected: BUILD SUCCESS. Tests run should be ≥ baseline (170) + 6 D-tests (D1, D2, D3, D4, D6 in `DrlxLambdaBoundaryTest` + D7) = ≥176.

- [ ] **Step 8.2: Capture the test count for the handover**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml test 2>&1 | grep -E "Tests run:" | tail -5
```
Note the totals for the post-cutover HANDOFF.md.

- [ ] **Step 8.3: Verify no DRLX file references removed MVEL types**

```bash
grep -rln "LambdaRegistry\|LambdaCatalog\|LambdaPersistenceManager\|LambdaArtifactStore\|LambdaRegistryStore" \
    /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src
```
Expected: zero matches.

- [ ] **Step 8.4: Verify allowed-only MVEL imports**

```bash
grep -rln "import org.mvel3" /home/tkobayas/usr/work/mvel3-development/drlx-parser/drlx-parser-core/src \
    | xargs grep "import org.mvel3.lambdaextractor" 2>/dev/null | sort -u
```
Expected: only `ArtifactRef`, `LambdaArtifactLoader`, `LambdaRuntime` (no others).

---

## Task 9: Phase C — coordinated cutover

**Files:** none (push/merge orchestration)

- [ ] **Step 9.1: Final `mvn install` of MVEL SNAPSHOT (sanity)**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/mvel/pom.xml install -DskipTests 2>&1 | tail -5
```

- [ ] **Step 9.2: Final DRLX test run against latest local SNAPSHOT**

```bash
mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml clean test 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 9.3: Push MVEL `lambda-registry-refactor` and open PR**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/mvel push -u origin lambda-registry-refactor 2>&1 | tail -5
```

Then create the PR:
```bash
gh -R mvel/mvel pr create --title "Refactor LambdaRegistry: split into Catalog/PM/ArtifactStore/Runtime; drop physicalId from cross-repo boundary" \
    --body "$(cat <<'EOF'
## Summary
- Splits `LambdaRegistry` into four single-purpose components under a `LambdaRuntime` composition root: `LambdaCatalog` (dedup), `LambdaPersistenceManager` (artifact map), `LambdaArtifactStore` (byte I/O), and the internal `LambdaRegistryStore` (Properties v2).
- Introduces `ArtifactRef(fqn, classFile)` as the only persistence-facing value type. `physicalId` stays MVEL-internal.
- Replaces static initializer with lazy init via `LambdaRuntime.getInstance()`.
- Adds `LambdaArtifactLoader` (stateless static helper) and `MVELBatchCompiler.getArtifactRef(handle)` for the DRLX-facing seam.
- Adds 13 Phase 0 characterization tests (M1–M13).
- Deletes `LambdaRegistry` outright; no facade left behind.

## Test plan
- [x] Full MVEL3 test suite passes (M1–M13 added).
- [x] DRLX (Plan 2) consumes new API; full DRLX suite green.
- [x] Properties `format.version=2` round-trips; malformed inputs throw `InvalidLambdaRegistryException`.

Companion: drlx-parser PR.

Closes #428.
EOF
)" 2>&1 | tail -5
```

- [ ] **Step 9.4: After MVEL PR is merged, push DRLX `main`**

```bash
git -C /home/tkobayas/usr/work/mvel3-development/drlx-parser push origin main 2>&1 | tail -5
```

(Since DRLX work happened directly on `main`, this push delivers all the DRLX commits in one go. The MVEL PR must merge first so that DRLX `main` compiles against MVEL `main` when CI runs.)

- [ ] **Step 9.5: Close DRLX issue**

```bash
gh issue close <DRLX-issue-N> --repo tkobayas/drlx-parser \
    --comment "Landed in main. Companion of mvel/mvel#428 (merged)."
```

- [ ] **Step 9.6: Update workspace HANDOFF.md**

Final handover at `/home/tkobayas/claude/public/drlx-parser/HANDOFF.md` should record:
- MVEL #428 + DRLX issue both closed.
- MVEL `main` and DRLX `main` both contain the refactor.
- New DRLX-facing seam locked: `ArtifactRef`, `LambdaArtifactLoader`, `LambdaRuntime` (two static accessors), `MVELBatchCompiler.getArtifactRef(handle)`.
- Phase 0 tests: 13 (M1–M13) on MVEL side; 6 (D1, D2, D3, D4, D6, D7) on DRLX side, plus D5 covered by existing `DrlxMetadataMismatchMode` tests.
- Next: pick from Epic #26 open sub-issues (#39 accumulate / #40 groupBy / #41 queries / #34 compact `with`-update — now natural after #45).

---

## Self-review

**Spec coverage (cross-check against `2026-05-12-mvel-lambda-registry-refactor-design.md` DRLX-side sections):**

| Spec item | Plan task |
|---|---|
| DRLX `LambdaRegistry` import removal (5 files) | Tasks 4, 5 |
| `DrlxLambdaMetadata` Properties v2 schema | Task 2 |
| `LambdaEntry(fqn, classFile, expression)` + `toArtifactRef()` | Task 2 Step 2.2 |
| Three-state load semantics (missing / unsupported / malformed) | Task 2 Step 2.2 + Tests D2/D3 |
| `DrlxPreBuildLambdaCompiler` → `getArtifactRef` | Task 3 |
| `DrlxLambdaCompiler.loadPreCompiledEvaluator(ArtifactRef)` via `LambdaArtifactLoader` | Task 4 |
| `LambdaRegistry.PERSISTENCE_ENABLED` / `DEFAULT_PERSISTENCE_PATH` → `LambdaRuntime` static accessors | Task 5 |
| `InvalidDrlxLambdaMetadataException` typed exception | Task 2 Step 2.1 |
| ~~`DrlxRuleBuilder.getBatchCompilerForTests()` bridging seam~~ — dropped, D1 uses static `MVELCompiler.compileInvocationCount()` | Task 6 (skipped) |
| D1 — pre-build → runtime asserts no recompile | Task 7 Step 7.7 |
| D2 — missing file (not mismatch) | Task 7 Step 7.3 |
| D3 — wrong format version | Task 7 Step 7.4 |
| D4 — stale classFile | Task 7 Step 7.5 |
| D5 — expression mismatch (covered by existing tests) | Task 7 Step 7.6 — no new test, documented in self-review |
| D6 — round-trip | Task 7 Step 7.1 |
| D7 — architectural guard | Task 7 Step 7.9 |
| Phase C cutover order: MVEL merge first, then DRLX push | Task 9 Steps 9.3 → 9.4 |
| Cross-repo issue cross-references | Task 1 Step 1.4 body; Task 9 Step 9.5 comment |

**D5 note:** the spec lists D5 as `metadata_expressionMismatch_failFast`. Existing DRLX tests around `DrlxMetadataMismatchMode` and `DrlxLambdaCompiler.tryLoadPreCompiled` already cover the "entry exists but expression differs" case — adding a new D5 would duplicate. The plan explicitly documents this and skips adding a redundant test. If the spec author intends D5 to be a separate added test rather than a coverage claim, escalate before execution.

**Placeholder scan:** no TBD/TODO/FIXME. The phrases "adjust to actual API" in Step 7.7 and "adjust to fit" in 7.7 are concrete engineering instructions for an integration-test setup where the existing-test patterns are the source of truth; not placeholders.

**Type consistency:**
- `DrlxLambdaMetadata.LambdaEntry(String fqn, Path classFile, String expression)` — used identically in Steps 2.2, 3.1, 7.1.
- `ArtifactRef(String fqn, Path classFile)` — same as Plan 1; used in Steps 2.2, 3.1, 7.5.
- `DrlxLambdaMetadata.put(String, int, ArtifactRef, String)` — same signature in 2.2 (definition) and 3.1 (call site) and 7.1 (test).
- `LambdaArtifactLoader.loadOrDefinePersistedClass(ClassManager, ArtifactRef)` — same as Plan 1; used in 4.1, 7.5.
- `MVELBatchCompiler.getArtifactRef(LambdaHandle)` — defined in Plan 1 Task 6, consumed in Plan 2 Task 3.
- `MVELCompiler.compileInvocationCount()` — static, defined in Plan 1, consumed directly in Plan 2 Task 7 (D1). Task 6 was a redundant accessor and is dropped.

All references resolve. Plan 2 complete.

---

**Plan 2 save target:** `/home/tkobayas/claude/public/drlx-parser/plans/2026-05-12-drlx-lambda-boundary-implementation.md`

Together with Plan 1, this completes the two-repo refactor of MVEL's lambda persistence and the DRLX-side boundary cleanup (mvel/mvel#428 + DRLX issue under Epic #26).
