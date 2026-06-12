# eidos Workspace
**Name:** casehub-eidos

**Project repo:** /Users/mdproctor/claude/casehub/eidos
**Workspace type:** public

## Session Start

Run `add-dir /Users/mdproctor/claude/casehub/eidos` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` (workspace staging) |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `epic`) |
| adr | `adr/` (workspace staging) |
| write-blog | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (staging; promoted to project `docs/specs/` at epic close)
- `plans/` — implementation plans (ephemeral; stay in workspace only)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records (staging; promoted to project `docs/adr/` at epic close)
- `blog/` — project diary entries with INDEX.md
- `design/` — epic journal (created by `epic` at branch start)
- `research/` — research notes and spec (eidos.md is the canonical spec)

## Git Discipline

Two git repositories are active in every session:
- **Workspace** (`/Users/mdproctor/claude/public/casehub/eidos`) — staging area for specs and ADRs; permanent home for blog, handover, plans, snapshots, research
- **Project repo** (`/Users/mdproctor/claude/casehub/eidos`) — source code + promoted specs (`docs/specs/`) + promoted ADRs (`docs/adr/`)

Before any git operation, run `git rev-parse --show-toplevel` to confirm which repo is currently active.

- Source code commits → project repo (`origin` = mdproctor/eidos, `upstream` = casehubio/eidos)
- Workspace commits → `mdproctor/wsp-casehub-eidos`

## Routing

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | project     | lands in `docs/adr/` — promoted at epic close |
| specs      | project     | lands in `docs/specs/` — promoted at epic close |
| blog       | workspace   | staged here; published to mdproctor.github.io via publish-blog |
| plans      | workspace   | stay in workspace permanently |
| design     | workspace   | epic journal stays in workspace |
| snapshots  | workspace   | stay in workspace permanently |
| handover   | workspace   | |
| research   | workspace   | eidos.md is the canonical spec; promote to parent docs/research/ when mature |

---

# CaseHub Eidos — Project Guide

See the project repo CLAUDE.md (`/Users/mdproctor/claude/casehub/eidos/CLAUDE.md`) for
the full project guide including platform context, Maven coordinates, build commands,
and work tracking.

**Research and spec:** `research/eidos.md` — the canonical specification synthesising
all research. Read this before implementing anything in this project.

**Platform architecture:**
```
https://raw.githubusercontent.com/casehubio/parent/main/docs/PLATFORM.md
```
