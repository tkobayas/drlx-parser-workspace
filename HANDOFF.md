# HANDOVER

## Session goals (done)

1. ✅ **Spec for #39** (Accumulate, parent epic #26) — design completed and written to `specs/2026-05-13-drlx-accumulate-design.md`. Two rounds of Codex review applied (count() zero-arg, scope ownership, qualified-name silent-aliasing, single vs multi lowering wording).
2. ✅ **Implementation plan for #39** — bite-sized TDD plan at `plans/2026-05-13-drlx-accumulate-implementation.md` (11 tasks, ~700 lines). One round of Codex review applied (sealed-interface vs `PendingAccumulatorIR`).

No code has been written in the project repo yet.

## Locked decisions for v1

- **Scope:** forms 1 + 2 only (simple-function + multi-function). No `acc(...)` keyword. Built-ins: `avg`, `sum`, `min`, `max`, `count`.
- **Binding form:** `(VAR | typeType) identifier '=' accumulateCall` — both `var` and typed (`int`, `double`, `BigDecimal`, generics OK).
- **Lowering:** N × `SingleAccumulate` sharing cloned source patterns. MultiAccumulate folding is a fast-follow.
- **Source visibility:** `p` is internal to the accumulate; `avgAge` (result) is visible. Build-time error on late `p` reference.
- **Qualified names:** parse but build-time reject in v1 (no silent aliasing). Custom-function support deferred.
- **Validation:** all binding-scope errors are build-time (visitor does no scope check).

## Current state

### Project repo
- Branch `main`, tip `46f7d2f`. No new commits this session.

### Workspace
- Spec + plan committed in this session's wrap (see commit at session end). No other uncommitted artifacts.

## Immediate next action

**Execute Task 1 of `plans/2026-05-13-drlx-accumulate-implementation.md`** — confirm baseline tests pass (`mvn -f /home/tkobayas/usr/work/mvel3-development/drlx-parser/pom.xml -pl drlx-parser-core -am test`, expect 182 green), post the v1-scope comment on issue #39, then proceed to Task 2 (grammar).

User's execution preference for tomorrow not yet chosen — offer **subagent-driven** (recommended) vs **inline** at session start.

## Gotchas (this session)

- **Sealed interfaces and transient visitor types**: the original plan had `PendingAccumulatorIR implements LhsItemIR`, which can't compile against the sealed permits list. Fixed by inline folding in the visitor (no helper LhsItemIR subtype). See Task 4.5 of the plan.
- **DRLXXXX lines 824/829** omit `var` for simple-form accumulate. Verified spec-wide: this is a drafting slip, not intentional. Every other accumulate-related example uses `var` (lines 839, 846, 851).

## References

| Topic | Path |
|-------|------|
| Spec | `specs/2026-05-13-drlx-accumulate-design.md` |
| Plan | `plans/2026-05-13-drlx-accumulate-implementation.md` |
| Issue | https://github.com/tkobayas/drlx-parser/issues/39 |
| Parent epic | https://github.com/tkobayas/drlx-parser/issues/26 |
| Spec source (DRLXXXX §Accumulate) | `~/usr/work/mvel3-development/drlx-parser/docs/DRLXXXX.md` lines 820–890 |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
