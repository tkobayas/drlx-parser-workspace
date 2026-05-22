# HANDOVER

## Session goals (completed)

**#41 queries v1 implemented.** Designed, planned, and shipped query definition (parameterized rules compiled as `QueryImpl` with prefix pattern) and in-rule invocation (positional-argument calls compiled as `QueryElement`). 1 commit, 8 new tests, 278 total tests passing.

## Current state

- **Project repo** `main`, tip `3e7ffc8`, clean, not yet pushed.
- **Workspace** `main`, pushed to origin.
- **GitHub** — #41 updated (v1 scope), #26 epic updated. 6 follow-up query issues created (#56–#61). #56, #57, #59, #60, #61 moved to epic #55 (deferred). #58 (recursive queries) stays in #26.

## Immediate next action

**Push project repo, close #41.** Then pick the next issue from #26. Remaining candidates: #58 (Recursive queries), #30 (Match), #32 (Edge-triggered), #33 (Setter desugaring), #38 (Multiple do blocks), #42 (Windows), #44 (ExistenceDriven). #40 (Group By) and most query extensions in epic #55.

## Key decisions this session

- **Output variable syntax** — `var p` in positional args (grammar: `positionalArg : VAR identifier | expression`). Unbound identifiers without `var` also detected as output at build time.
- **Bare OOPath at ruleItem level** — `/queryName(args),` works alongside bound form. Required scope isolation fix for OR/NOT/EXISTS in `buildLhs()`.
- **Two-pass compilation** — queries compiled first, registered by entry-point name; regular rules compiled second with query registry available.
- **DrlxLambdaBetaConstraint fix** — `decl.getValue()` instead of raw `fh.getObject()` for bound variable extraction. Affects all beta constraints but is safe (SelfReference returns same value).
- **Query result binding** — v1 uses parameter-as-binding-name pattern (`Person result : /persons[...]` where `result` is a query parameter). Spec's positional unification model deferred.

## References

| Topic | Path |
|---|---|
| Spec | `specs/2026-05-22-41-queries-v1-design.md` |
| Plan | `plans/2026-05-22-41-queries-v1-implementation.md` |
| Blog | `blog/2026-05-22-tk01-41-queries-v1.md` |
| Epic #26 | https://github.com/tkobayas/drlx-parser/issues/26 |
