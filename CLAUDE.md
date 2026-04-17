# drlx-parser Workspace

**Project repo:** /home/tkobayas/usr/work/mvel3-development/drlx-parser
**Workspace type:** public

## Session Start

Run `add-dir /home/tkobayas/usr/work/mvel3-development/drlx-parser` before any other work.

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

---

## Project Type

**Type:** java

## Local CLAUDE.md reference
Read `./CLAUDE.local.md` for user-local-specific notes and instructions that are not meant to be shared outside the team. This may include:
- Dependency source code locations on the local filesystem