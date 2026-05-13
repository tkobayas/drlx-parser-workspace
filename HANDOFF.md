# HANDOVER

## Session goals (done)

1. ✅ **Follow-up 1** — `MVELBatchCompiler` no-persist mode now stays off the global `LambdaRuntime` state. Branches on `persistenceDir == null`, dedups and renames through a batch-local `LambdaCatalog`. Tests M12 (probe-register catalog untouched) and M13 (no-persist batch ignores previously persisted artifact) added. Landed on MVEL via PR `#430` (`70d95695`).
2. ✅ **Follow-up 2** — `Path.of(classFile)` now wrapped in try/catch on both sides; converted to `InvalidLambdaRegistryException` / `InvalidDrlxLambdaMetadataException`. Tests M7d (MVEL) and D3b (DRLX) added. Landed on MVEL via PR `#431` (`ef8fcd9e`), on DRLX `main` directly (`46f7d2f`).

Both findings from the post-Plan-2 follow-up doc are closed. The original two-issue refactor (`mvel/mvel#428`, `tkobayas/drlx-parser#47`) is fully wrapped.

## Current state

### MVEL
- All work on `main`. Final tip: `ef8fcd9e`. Test suite: 737 passing.

### DRLX
- All work on `main`. Final tip: `46f7d2f`. Test suite: 177 main + 5 no-persist = **182 passing** (was 181 baseline → +1 D3b).

### Workspace
- Blog entry `2026-05-13-tk02-428-no-persist-and-nul-byte.md` committed (`85c5f60`).
- Two cross-project garden entries submitted (commit `060f964` in `~/.hortora/garden`): `GE-20260513-fbfeba` (Java `\uXXXX` in comments) and `GE-20260513-a0d2d6` (probe-register technique).
- No uncommitted artifacts beyond this handover.

## Immediate next action

**No specific next thread is in flight.** The lambda-registry epic is closed. Candidate directions, in priority order suggested by current state:

1. **New rule-syntax features** — per memory `project_status.md`, "next: more rule syntax". Pick an open issue on `tkobayas/drlx-parser`.
2. **Drools 10.1.0 alignment** — only if specific gaps surfaced that haven't been logged as issues yet.
3. **MVEL3 batch-compiler API hardening** — `MVELBatchCompiler.getArtifactRef(...)` still calls into the global persistence manager unconditionally. Not a bug today (DRLX only invokes it on the persistence path), but worth a guard if the no-persist contract ever leaks into a caller.

## Gotchas (this session)

- **`\u0000` in Java comments still trips JLS §3.3.** Writing a `\u0000` inside a `//` comment to document the escape silently injected a NUL byte into the source file, which broke the next Edit-tool search and would have made the build hostile to byte-level scanners. The lexer pre-processes Unicode escapes before recognising comments. Fix: `\\u0000` (odd backslash count before `\u` → not an escape). Captured as garden entry `GE-20260513-fbfeba`.

- **Map-mode evaluators don't get unique class names from `CompilationUnitGenerator`.** `createMapEvaluatorUnit` returns the template unit unchanged (no `renameTemplateClass` call). Two distinct Map-mode expressions therefore produce the same FQN; only the persistence-path `_<physicalId>` rename makes them unique. FQN-based dedup is correct only for paths that go through `registerAndRename`. Project-specific; lives in this blog rather than the cross-project garden.

## References

| Topic | Path |
|-------|------|
| Today's blog | `blog/2026-05-13-tk02-428-no-persist-and-nul-byte.md` |
| Garden entries (this session) | `~/.hortora/garden/jvm/GE-20260513-fbfeba.md`, `~/.hortora/garden/tools/GE-20260513-a0d2d6.md` |
| Plan 1 (MVEL, done) | `plans/2026-05-12-mvel-lambda-registry-refactor-implementation.md` |
| Plan 2 (DRLX + Phase C, done) | `plans/2026-05-12-drlx-lambda-boundary-implementation.md` |
| Spec | `specs/2026-05-12-mvel-lambda-registry-refactor-design.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
