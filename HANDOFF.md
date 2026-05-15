# HANDOVER

## Session goals (done)

**#39 v1 shipped.** Executed Tasks 6–11 of `plans/2026-05-13-drlx-accumulate-implementation.md` inline (continuing yesterday's Tasks 1–5). Issue closed at project HEAD `538185b`. drlx-parser-core test suite: 182 → 209 green (+27).

| Task | Commit | Outcome |
|---|---|---|
| 6 | `de442ae` | `AccumulateFunctionRegistry` (5 built-ins) + `DrlxLambdaAccumulator` (wraps Drools `AccumulateFunction` + optional extractor) |
| 7 | `dbab883` | Runtime-builder lowering: `AccumulatePatternIR` → N × `SingleAccumulate`; reflection extractor for `binding.property`; `MyUnit.results` field for test capture |
| 8 | `03fe3df` | Multi-function + count + sum end-to-end tests |
| 9 | `538185b` | Error-path tests: qualified-name, unknown-fn, source-scope isolation, v1 expression-limit |
| 10 | (regression) | Full module green; all 3 reactor modules compile |
| 11 | pushed + closed | 8 commits pushed to `origin/main`; #39 closed with v1-shipped summary |

## Current state

- **Project repo** `main`, tip `538185b`, pushed.
- **Workspace** `main`, tip `14f9cce` (blog entry); 3 garden entries committed at `~/.hortora/garden` (`cac4fcd`).

## Immediate next action

**Choose the next epic-#26 child to work on.** Open issues from the accumulate-v1 deferrals:
- MVEL3-backed extractor (lift v1's `binding.property`-only limit — the `complexExtractorExpressionRejectedAsV1Limitation` test in `AccumulateTest.java` is the contract)
- `MultiAccumulate` folding (N×SingleAccumulate → one node)
- Inline-from form (`avg(/persons.age)`)
- `acc()` keyword forms (2/3/5-param)
- Multi-pattern source via `and(...)`
- Custom user-imported functions

Pick one, run brainstorming → spec → plan as usual. Or pick a different #26 child entirely — `gh issue list --repo tkobayas/drlx-parser --label "parent:#26"` for the full list.

## Plan deviations (worth knowing)

- **Task 7 extractor:** Plan called for an MVEL3 batch-compile path on `DrlxLambdaCompiler.createValueExtractor`. I substituted reflection-based simple `binding.property` extraction with a v1-limit test that locks the boundary. Lifting this is the natural Task 7 follow-up.
- **Task 9 negative tests:** Added a fourth test (`complexExtractorExpressionRejectedAsV1Limitation`) beyond the plan's three, to encode the extractor boundary.

## Gotchas (this session) — submitted to garden

- `SumAccumulateFunction` returns `Double` regardless of input type — `min`/`max` preserve type but `sum` doesn't. Garden: `GE-20260515-dc014f`.
- Sealed-interface permits forbid transient builder types; fold inline using caller-local state. Garden: `GE-20260515-d6a406`.
- `SelfReferenceClassFieldReader` lives in `org.drools.base.base.extractors`, not where its `ReadAccessor` interface lives. Garden: `GE-20260515-92a9e8`.

## References

| Topic | Path |
|---|---|
| Today's blog | `blog/2026-05-15-tk01-39-accumulate-v1-shipped.md` |
| Spec | `specs/2026-05-13-drlx-accumulate-design.md` |
| Plan | `plans/2026-05-13-drlx-accumulate-implementation.md` |
| Issue (closed) | https://github.com/tkobayas/drlx-parser/issues/39 |
| Parent epic | https://github.com/tkobayas/drlx-parser/issues/26 |
| Yesterday's handover | `git show HEAD~2:HANDOFF.md` |
