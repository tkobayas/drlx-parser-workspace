# HANDOVER

## Session goals (completed)

**#52 spec reviewed against Drools DRL runtime, corrected, and implementation plan written.** Critical fix: `handle.getObject()` returns `SubnetworkTuple` in multi-pattern context, not a domain fact. Plan has 11 tasks ready for execution.

## Current state

- **Project repo** `main`, tip `fa203da`, clean. No code changes this session.
- **Workspace** `main`, spec updated + plan created (uncommitted).

## Immediate next action

**Execute the implementation plan** `plans/2026-05-21-52-accumulate-multi-pattern-source-implementation.md` using `executing-plans` skill. Start with Task 1 (grammar change). The plan is self-contained with complete code for every step.

## Key decisions this session

- **`requiredDeclarations` must be empty for multi-source** — Drools uses these for `LogicTransformer.replaceDeclarations()` and property-reactive masks; source declarations belong only in `innerDeclarationCache`
- **`sourceScope` = innerScope minus outerScope** — MVEL3 compiler must see only source bindings, not outer scope
- **`DrlxLambdaAccumulator.tryReverse()` unchanged** — built-in functions reverse by cached value, no tuple access needed
- **`DrlxCustomAccumulator.tryReverse()` needs tuple extraction** — reverse block must re-extract all source facts from tuple via `innerDecls`
- **Garden entry submitted** — `GE-20260521-1265db` documents the SubnetworkTuple gotcha

## References

| Topic | Path |
|---|---|
| Implementation plan | `plans/2026-05-21-52-accumulate-multi-pattern-source-implementation.md` |
| Spec (updated) | `specs/2026-05-20-52-accumulate-multi-pattern-source-design.md` |
| Issue #52 | https://github.com/tkobayas/drlx-parser/issues/52 |
| Epic #26 | https://github.com/tkobayas/drlx-parser/issues/26 |
| Garden entry | `~/.hortora/garden/jvm/GE-20260521-1265db.md` |
| Previous handover | `git show 2c69355:HANDOFF.md` |
