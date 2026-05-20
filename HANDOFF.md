# HANDOVER

## Session goals (completed)

**mvel/mvel#432 fixed and merged.** Map evaluator `++`/`--` write-back — 1 commit on `increment-write-back` branch, PR merged upstream. 746 MVEL tests pass, 255 drlx-parser tests pass.

## Current state

- **Project repo** `main`, tip `fa203da`, clean. Pushed (was 8 ahead last session, now current).
- **MVEL repo** branch `increment-write-back`, tip `91e64d0c`, clean. PR merged to `mvel/mvel` main.
- **Workspace** `main`, blog entry + handover uncommitted.
- Issue **#51** closed. Issue **mvel/mvel#432** open (PR merged, issue not closed).

## Immediate next action

**Pick next issue from epic #26.** The MVEL3 detour is done. Optionally update drlx-parser accumulate tests to use `count++` instead of `count = count + 1` workaround (not urgent — both work).

## Key decisions this session

- Postfix `++`/`--` converted to prefix before wrapping in `context.put()` — postfix returns old value, which would write wrong value to map
- List context gets the same treatment as Map context (symmetric)

## References

| Topic | Path |
|---|---|
| MVEL3 issue | https://github.com/mvel/mvel/issues/432 |
| MVEL3 PR | https://github.com/mvel/mvel/pull/433 |
| Blog entry | `blog/2026-05-20-tk02-432-mvel3-increment-writeback.md` |
| Garden entry | `GE-20260520-ee66d2` (postfix write-back gotcha) |
| Epic #26 | https://github.com/tkobayas/drlx-parser/issues/26 |
| Previous handover | `git show a455446:HANDOFF.md` |
