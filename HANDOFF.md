# HANDOVER

## Session goals (done)

1. ✅ **Brainstormed mvel/mvel#428** (Codex's `LambdaRegistry_Refactor.md` as input). 12 rounds resolved the design's open questions: `LambdaPersistenceManager` owns `physicalId→ArtifactRef`, PM is service-layer not compile-wrapper, file format = Properties `format.version=2`, lifecycle = lazy `LambdaRuntime.getInstance()`. Spec at `specs/2026-05-12-mvel-lambda-registry-refactor-design.md`.
2. ✅ **Wrote two implementation plans**: Plan 1 (MVEL, 9 tasks) at `plans/2026-05-12-mvel-lambda-registry-refactor-implementation.md`. Plan 2 (DRLX + Phase C cutover, 9 tasks) at `plans/2026-05-12-drlx-lambda-boundary-implementation.md`. Caught two plan blockers during review: Phase 1 sequencing (kept `entriesByPhysicalId` alive through Phase 4) and FQN reconstruction (passed FQN through `registerPhysicalPath(int, String, Path)`).
3. ✅ **Executed Plan 1 inline (8 commits on `lambda-registry-refactor`).** MVEL3 733 tests passing (was 722). `LambdaRegistry` deleted outright. 13 Phase 0 tests added (M1–M13). DRLX-facing seam locked: `ArtifactRef`, `LambdaArtifactLoader.loadOrDefinePersistedClass`, `LambdaRuntime.isPersistenceEnabled() / defaultPersistencePath()`, `MVELBatchCompiler.getArtifactRef(handle)`. MVEL3 3.0.0-SNAPSHOT installed.
4. ✅ Today's blog: `blog/2026-05-12-tk02-428-half-the-refactor-plus-a-bridge.md`. Garden: 4 entries (`GE-20260512-enum1`, `-mvelreset`, `-mvelvisib`, `-pasfacad`).

## Current state

### MVEL3 (branch `lambda-registry-refactor`, NOT YET PUSHED)
- Commits `b91af14..<HEAD>` (8 commits). Run `git -C ~/usr/work/mvel3-development/mvel log --oneline lambda-registry-refactor ^main` to list.
- Test suite: 733 passing, 0 failing, 117 skipped.
- DRLX `main` does **not** compile against this SNAPSHOT yet — that's the entry point for Plan 2.

### DRLX (`main`, unchanged this session)
- Still references the deleted `LambdaRegistry` in `DrlxLambdaCompiler.java`, `DrlxRuleBuilder.java`, `drlx/tools/DrlxCompiler.java`. Confirmed by smoke `mvn compile` — fails on those three files exactly. Plan 2 Task 1 expects this state.

## Immediate next action

**Resume Plan 2 from Task 1.** Working dir: `/home/tkobayas/usr/work/mvel3-development/drlx-parser`, branch `main`. Plan path: `plans/2026-05-12-drlx-lambda-boundary-implementation.md`. First step: verify MVEL SNAPSHOT in `~/.m2`, confirm DRLX baseline broken, **create the DRLX GitHub issue under Epic #26** (Plan 2 Task 1 Step 1.4 has the body). All DRLX commits stay local until Phase C cutover (Plan 2 Task 9) — coordinated push to both repos at the end.

## Gotchas (this session)

- **Java enum init order.** `INSTANCE` is constructed before `static final` fields are initialized. Instance-field initializer that reads a static field fails with "illegal reference to static field from initializer." Fix: wire in a `static { }` block after the constants. Garden: `GE-20260512-enum1`.
- **`mvel3.compiler.lambda.resetOnTestStartup=true`** is set globally in MVEL `pom.xml` surefire config. Combined with lazy init + `Files.walk(persistenceRoot)`-from-root in `resetAndRemoveAllPersistedFiles`, this nukes any test's `@TempDir` parent on first `getInstance()`. Tests that redirect `persistence.path` MUST also set `resetOnTestStartup=false`. Garden: `GE-20260512-mvelreset`.
- **Generated evaluator class visibility.** `MVEL.<T>pojo(T.class, ...)` against a `public static` inner of a package-private test class fails: `org.mvel3.GeneratorEvaluator___N` can't reach `org.mvel3.lambdaextractor.MyTest`. Make the outer test class `public`. Garden: `GE-20260512-mvelvisib`.

## Techniques

- **Pass-through facade as a one-commit migration bridge.** Used between Phase 6a (introduce `LambdaRuntime`) and Phase 6b (delete `LambdaRegistry`). The facade became state-less delegation for one commit; tests stayed green; next commit migrated callers and deleted the facade. Generally useful for "rename a central class with N callers." Garden: `GE-20260512-pasfacad`.

For historical gotchas: *unchanged — retrieve with* `git show 98b59ab:HANDOFF.md` and `git show fb86af2:HANDOFF.md`.

## References

| Topic | Path |
|-------|------|
| Today's blog | `blog/2026-05-12-tk02-428-half-the-refactor-plus-a-bridge.md` |
| Spec | `specs/2026-05-12-mvel-lambda-registry-refactor-design.md` |
| Plan 1 (MVEL, done) | `plans/2026-05-12-mvel-lambda-registry-refactor-implementation.md` |
| Plan 2 (DRLX, tomorrow) | `plans/2026-05-12-drlx-lambda-boundary-implementation.md` |
| Codex source plan | `~/usr/work/mvel3-development/mvel/LambdaRegistry_Refactor.md` |
| Garden entries | `~/.hortora/garden/jvm/GE-20260512-{enum1,mvelreset,mvelvisib,pasfacad}.md` |
| Previous handover | `git show fb86af2:HANDOFF.md` |

## Migration policy & Key commands

*Unchanged — retrieve with* `git show fb86af2:HANDOFF.md`
