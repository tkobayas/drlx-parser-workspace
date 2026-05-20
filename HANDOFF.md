# HANDOVER

## Session goals (completed)

**#51 implemented.** Grammar, IR, visitor, protobuf, runtime accumulator, lambda compiler, runtime builder — 8 commits, 255 tests pass, 0 failures.

**MVEL3 `++`/`--` bug found.** Map evaluator doesn't write back increment/decrement to context map. Workaround: `count = count + 1`. Bug documented in `.claude/Mvel3_increment_bug.md` for upstream GH issue filing.

## Current state

- **Project repo** `main`, tip `fa203da`, unpushed (8 commits ahead). Clean working tree.
- **Workspace** `main`, plan uncommitted (`plans/2026-05-20-51-acc-keyword-forms-implementation.md`).
- Issue **#51** open, implementation complete, ready to close.

## Immediate next action

**Close #51** on GitHub. Then **push project repo** (8 commits ahead of origin). Pick next issue from epic #26.

**MVEL3 bug:** file GH issue on mvel repo using `.claude/Mvel3_increment_bug.md` as description. Root cause: `MVELToJavaRewriter.rewriteNode()` `UnaryExpr` case missing `context.put()` write-back for `PREFIX_INCREMENT`/`POSTFIX_INCREMENT`/`PREFIX_DECREMENT`/`POSTFIX_DECREMENT`.

## Key decisions this session

- `resolveInitVarType()` returns boxed classes (not primitives) — MVEL3 rejects primitives in `.out()`
- `++`/`--` operators avoided in acc() tests due to MVEL3 bug — use `count += 1` form
- Outer-binding rejection lives in runtime builder (not visitor) — needs `outerScope` map

## References

| Topic | Path |
|---|---|
| #51 spec | `specs/2026-05-19-51-acc-keyword-forms-design.md` |
| #51 plan | `plans/2026-05-20-51-acc-keyword-forms-implementation.md` |
| MVEL3 bug doc | `.claude/Mvel3_increment_bug.md` |
| Blog entry | `blog/2026-05-20-tk01-51-acc-keyword-forms.md` |
| Garden entry | `GE-20260520-ac03d9` (mvel3-map-evaluator-increment-writeback) |
| Issue #51 | https://github.com/tkobayas/drlx-parser/issues/51 |
| Epic #26 | https://github.com/tkobayas/drlx-parser/issues/26 |
| Previous handover | `git show 92932d9:HANDOFF.md` |
