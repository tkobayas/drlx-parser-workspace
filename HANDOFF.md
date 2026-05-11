# HANDOVER

## Session goals (done)

1. ✅ **DrlxRuleUnitInstance wrapper** — built test infra (wrapper, TestDataObserver, MyUnit promoted to RuleUnitData) to enable #37-style rules calling `DataStore.add/remove` from consequence bodies. Project commits `b034bf7..f99b693`.
2. ✅ **#37 part 1** — DRLX now registers every public unit-class field as a global on the KiePackage, declares an entry point for every DataSource field, and resolves globals at MVEL3 eval time. Mirrors `PackageModel.addRuleUnitVariable` upstream. Project commits `8528cf3..cab2862`.
3. ✅ **Issue triage** — closed #37 (add/remove done), filed #45 (`update(T)` coercion), closed #46 as duplicate of pre-existing #34, updated epic #26 body.

## Current state

### Test suite
- **DRLX: 162 passing, 0 failures.** Up from 146 baseline at session start.

### GitHub issues
- **Epic #26**: 14 open sub-issues, 4 closed (#27–#29 + newly #37)
- High priority: **#45** (update(T) coercion — new), #39 (accumulate), #40 (groupBy), #41 (queries)
- The `with`-block compact update for `alerts.update(t{...})` is covered by #34 (Low Priority), now annotated with the dependency on #45

### Migration policy
*Unchanged — retrieve with* `git show 10e0a0f:HANDOFF.md`

## Immediate next action

User-pick. Natural follow-ups from today: **#45** (`update(T)` coercion — small parser/visitor change, scope already analyzed in #37 close comment) or any of #39/#40/#41. The disabled-then-enabled `DataStoreCrudTest` is now the test fixture for #45's happy-path tests.

## Gotchas (this session, resolved)

- Mirroring half of an upstream registration produces silent-but-broken end-to-end behaviour. `PackageModel.addRuleUnitVariable` does globals + entry points in one method; DRLX did only entry points (and only for LHS-pattern sources). The gap was invisible until #37 introduced consequence-only DataSource fields.
- Compile-time MVEL3 type-map awareness is not eval-time variable provisioning. `DrlxLambdaConsequence` needs both: globalNames in the type map AND `valueResolver.getGlobal(name)` calls at eval time. The spec covered the first.
- Garden gotcha captured: GE-20260511-264416 — `Write` tool fails after `git mv` until new path is `Read`. Local garden at `~/.hortora/garden` (initialized this session, no remote).

For historical gotchas: *unchanged — retrieve with* `git show 10e0a0f:HANDOFF.md`

## References

| Topic | Path |
|-------|------|
| Today's blog entry | `blog/2026-05-11-tk01-37-globals-were-half-the-gap.md` |
| Wrapper spec / plan | `specs/2026-05-11-drlx-rule-unit-instance-design.md`, `plans/2026-05-11-drlx-rule-unit-instance-implementation.md` |
| Globals spec / plan | `specs/2026-05-11-drlx-unit-field-globals-design.md`, `plans/2026-05-11-drlx-unit-field-globals-implementation.md` |
| DRLXXXX spec | `docs/DRLXXXX.md` (in project repo) |
| Previous handover | `git show 8c1fade:HANDOFF.md` |

## Key commands

*Unchanged — retrieve with:* `git show 10e0a0f:HANDOFF.md`
