# drlx-parser Workspace

**Project repo:** /home/tkobayas/usr/work/mvel3-development/drlx-parser
**Workspace type:** public

## Session Start

Run `add-dir /home/tkobayas/usr/work/mvel3-development/drlx-parser` before any other work.
Run `add-dir /home/tkobayas/usr/work/mvel3-development/drools` next.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `epic`) |
| adr | `adr/` |
| write-blog | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together
- **Push workspace after handover** — once `HANDOFF.md` is committed at session end, push the workspace to `origin` if a remote exists and local is ahead. Fast-forward only; if behind or diverged, ask before acting. Project repo pushes remain explicit per request.

## Routing

Per-artifact routing destinations (optional). If absent, all artifacts route to the project repo.

| Artifact   | Destination |
|------------|-------------|
| adr        | project     |
| blog       | project     |
| design     | project     |
| snapshots  | project     |

Valid destinations: `project` · `workspace` · `alternative ~/path/to/repo/`

## Context Management

If the conversation is getting very long or you notice context pressure,
proactively suggest writing a handover before continuing.

## Work Tracking

**Issue tracking:** enabled
**GitHub repo:** tkobayas/drlx-parser
**Changelog:** GitHub Releases (run `gh release create --generate-notes` at milestones)

**Automatic behaviours (Claude follows these at all times in this project):**
- **Before implementation begins** — when the user says "implement", "start coding",
  "execute the plan", "let's build", or similar: check if an active issue or epic
  exists. If not, run issue-workflow Phase 1 to create one **before writing any code**.
- **Before writing any code** — check if an issue exists for what's about to be
  implemented. If not, draft one and assess epic placement (issue-workflow Phase 2)
  before starting. Also check if the work spans multiple concerns.
- **Before any commit** — run issue-workflow Phase 3 (via git-commit) to confirm
  issue linkage and check for split candidates. This is a fallback — the issue
  should already exist from before implementation began.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done).
  If the user explicitly says to skip ("commit as is", "no issue"), ask once to confirm
  before proceeding — it must be a deliberate choice, not a default.

---

## Project Type

**Type:** java

## Local CLAUDE.md reference
Read `./CLAUDE.local.md` for user-local-specific notes and instructions that are not meant to be shared outside the team. This may include:
- Dependency source code locations on the local filesystem