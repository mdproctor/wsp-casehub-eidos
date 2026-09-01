# HANDOFF — eidos

## Last Session

Designed and implemented the agent organizational model (#150) — from research brief through brainstorming (11 design decisions), research doc, implementation plan, and 6 execution batches. Core insight: org structure is YAML-declared (no JPA needed), reconciled via casehub-desiredstate, with three layers — descriptors (who), organization (how they relate), deployment (where they run). Triangulated with engine#1017 YAML-first deployment model and casehub-ops#83 application NodeSpec.

## Immediate Next Step

Annotation surface for org model — `@OrgUnit`, `@Supervises`, Quarkus build extension + recorder. Deferred as M/High. Use `casehub-eidos-annotations/` `EidosAnnotationsProcessor` as the template. Then update CLAUDE.md with the new module structure.

## References

- `research/organizational-model.md` — full design research (workspace)
- `plans/2026-09-01-org-model.md` — implementation plan (workspace)
- `JOURNAL.md` — design journal (workspace)
- `org-api/`, `org-memory/`, `org-runtime/`, `examples/org-scenarios/` — implementation
