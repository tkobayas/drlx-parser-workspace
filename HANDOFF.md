# HANDOVER

## Session goals (done)

1. ✅ **Executed Plan 2 inline (5 DRLX commits).** Tasks 1–8: DRLX issue tkobayas/drlx-parser#47 created, `DrlxLambdaMetadata` rewritten to Properties v2 with `classFile`, `DrlxPreBuildLambdaCompiler` migrated to `getArtifactRef`, `DrlxLambdaCompiler.loadPreCompiledEvaluator(ArtifactRef)` via `LambdaArtifactLoader`, constants migrated (DrlxRuleBuilder + DrlxCompiler + tests + benchmarks), Phase 0 tests D1–D7 added (D5 covered by existing tests). 176 main + 5 no-persist = **181 tests passing** (was 175 baseline).
2. ✅ **Task 6 dropped on review** (see Gotchas). Plan and spec updated to record the decision. Counter access goes through static `MVELCompiler.compileInvocationCount()` directly; no DRLX accessor needed.
3. ✅ **Phase C cutover complete.** MVEL `lambda-registry-refactor` pushed, PR opened, merged to mvel/mvel `main`. DRLX `main` pushed to `tkobayas/drlx-parser`. Both issues closed (mvel/mvel#428, tkobayas/drlx-parser#47).

## Current state

### MVEL3 — refactor lives on `main`
- Both repos: refactor merged. SNAPSHOT `3.0.0-SNAPSHOT` already installed locally.
- Branch `lambda-registry-refactor` can be deleted at your convenience.

### DRLX — refactor lives on `main`
- 5 commits landed (`8346329 ..abd3dcd`). Run `git -C ~/usr/work/mvel3-development/drlx-parser log --oneline origin/main ^origin/main~5` to list.
- Test suite: 181 passing.

### Workspace — uncommitted plan/spec updates from Task 6 drop
- `plans/2026-05-12-drlx-lambda-boundary-implementation.md` — Task 6 marked SKIPPED; phase-ordering, files-modified, and self-review tables updated.
- `specs/2026-05-12-mvel-lambda-registry-refactor-design.md` — clarifies the counter is **static** on `MVELCompiler` and that `DrlxRuleBuilder.getBatchCompilerForTests()` is not introduced.
- Also `CLAUDE.md` — unrelated writing-style-guide pointer.

## Immediate next action

**Two follow-up findings from the refactor.** Source doc:
`/home/tkobayas/usr/work/mvel3-development/mvel/LambdaRegistry_Refactor_followup.md`

1. **`MVELBatchCompiler` no-persist path still touches global runtime state.**
   When `persistenceDir == null`, the batch path still hits
   `LambdaRuntime.getInstance().catalog().register(...)` and
   `persistenceManager().artifactExists(...)`. The single-compiler path branches
   correctly; the batch path doesn't. Fix: add a top-level no-persist branch in
   `MVELBatchCompiler.add(...)` / `compile(...)` analogous to `MVELCompiler`.
   Sites: `MVELBatchCompiler.java:55`, `:66`; `MVELCompiler.java:219`.

2. **Invalid persisted `classFile` paths bypass typed metadata exceptions.**
   `Path.of(classFile)` throws unchecked `InvalidPathException`, escaping both
   `InvalidLambdaRegistryException` (MVEL) and `InvalidDrlxLambdaMetadataException`
   (DRLX), and on the DRLX side bypassing `DrlxMetadataMismatchMode` routing.
   Fix: wrap `Path.of(...)` in try/catch and convert. Sites:
   `LambdaRegistryStore.java:65`, `DrlxLambdaMetadata.java:91`.

Both are correctness issues. Suggested order: do (1) in MVEL with new tests
(easier to bound), then (2) on both sides in lockstep.

## Gotchas (this session)

- **Plan/impl drift on test-only counter.** Plan 2's D1 test plan called
  `DrlxRuleBuilder.getBatchCompilerForTests().compileInvocationCount()`, but
  Plan 1 had actually implemented the counter as a **static** method on
  `MVELCompiler` (not as an instance method on `MVELBatchCompiler` as the spec
  text suggested). Caught at Task 6; user agreed to drop the bridge accessor
  rather than add a delegating MVEL method, since the counter is JVM-global
  test-only instrumentation. Lesson: validate spec promises against actual Plan
  1 code before consuming them in Plan 2. The check is a single grep — cheap
  insurance.

- **Plan 2 file inventory missed the benchmark module.** Task 1 Step 1.5
  inventory only searched `drlx-parser-core/src`. After Task 5 compile,
  `drlx-parser-benchmark` failed on four files (`KieBaseBuildNoPersistence`,
  `KieSessionFireAllRules`, `KieBasePreBuildPersistence`,
  `KieBaseBuildUsingPreBuildArtifacts`). All four migrated in the same Task 5
  commit — no rework, but the inventory grep should have been repo-wide.

## References

| Topic | Path |
|-------|------|
| Follow-up findings | `~/usr/work/mvel3-development/mvel/LambdaRegistry_Refactor_followup.md` |
| Spec | `specs/2026-05-12-mvel-lambda-registry-refactor-design.md` |
| Plan 1 (MVEL, done) | `plans/2026-05-12-mvel-lambda-registry-refactor-implementation.md` |
| Plan 2 (DRLX + Phase C, done) | `plans/2026-05-12-drlx-lambda-boundary-implementation.md` |
| Codex source plan | `~/usr/work/mvel3-development/mvel/LambdaRegistry_Refactor.md` |
| Previous handover (Plan 1 done) | `git show 1d9cddc:HANDOFF.md` |

## Migration policy & Key commands

*Unchanged — retrieve with* `git show fb86af2:HANDOFF.md`
