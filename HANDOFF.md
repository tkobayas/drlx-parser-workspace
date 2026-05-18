# HANDOVER

## Session goals (in progress)

**#50 spec drafted, not yet implemented.** Inline-from form `avg(/persons.age)` design landed after two Codex review rounds. Three workspace commits (`3a725e5` → `63d17e6`); no project-repo work yet, no plan yet.

## Current state

- **Project repo** `main`, tip `50049af` (unchanged from last session), pushed.
- **Workspace** `main`, 3 commits ahead of `origin/main`; push at session end.
- **Spec** `specs/2026-05-18-50-accumulate-inline-from-design.md` — ready for implementation plan.
- Issue **#50** still **open**.

## Immediate next action

**Invoke `writing-plans` skill against the #50 spec.** Spec is approved by the user and stable after two Codex review rounds. The plan is the only step left before implementation can start. Resume tomorrow at this point.

## Key design decisions (locked in the spec)

- **Per-item synthetic source, no fold** — each inline-from accumulator produces its own `AccumulatePatternIR` with a fresh `$inline<N>` source pattern. Aligns with Drools' DRL behavior (`MVELAccumulateBuilder.isMultiFunction()` only folds within one `accumulate(...)` block, never across separate clauses). Reversal from the initial fold-by-textual-oopath choice — confirmed wrong against Drools convention.
- **Grammar**: new `inlineFromOopath: oopathExpression ('.' identifier)?` rule, added as the first alternative of `accumulateCall`. ANTLR dispatches by `/` / `?/` prefix — Java `expression` never starts that way, so no ambiguity.
- **Compose with bound patterns** — `var t : /thresholds, var n = count(/persons), do {...}` parses (bound flushes as a normal LhsItem, inline-from emits its own AccumulatePatternIR). Even same-source case parses, produces two source joins (wasteful but correct).
- **Reject `count(/persons.age)`** at visitor time — runtime silently drops the extractor otherwise.
- **`DrlxRuleAstRuntimeBuilder` unchanged** — change is grammar + visitor only.

## Codex review findings (resolved)

| Round | Finding | Resolution |
|---|---|---|
| 1 | Bound-pattern rejection too broad | Dropped rejection; bound flushes as normal LhsItem |
| 1 | `count(/persons.age)` silently drops `.age` | Visitor rejects with precise message |
| 1 | Test rule used `location` (not on Person) | Changed to `age >= 40` |
| 2 | Multi-function fold misaligned with DRL | Switched to per-item synthetic source (option 1) |

Codex's round-2 finding mistakenly assumed option 1 had been chosen — but after investigating `MVELAccumulateBuilder.java:144` and `AccumulateDescr.java:223`, the DRL convention confirmed option 1 was the right call, so the misunderstanding led to the correct outcome.

## References

| Topic | Path |
|---|---|
| #50 spec (final) | `specs/2026-05-18-50-accumulate-inline-from-design.md` |
| Issue #50 (open) | https://github.com/tkobayas/drlx-parser/issues/50 |
| Parent epic | https://github.com/tkobayas/drlx-parser/issues/26 |
| Prior spec (#49) | `specs/2026-05-18-49-multiaccumulate-folding-design.md` |
| Drools DRL precedent | `~/usr/work/mvel3-development/drools/drools-mvel/src/main/java/org/drools/mvel/builder/MVELAccumulateBuilder.java:144` |
| Previous handover | `git show 7923959:HANDOFF.md` |
