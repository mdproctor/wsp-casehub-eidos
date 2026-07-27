# eidos Session Handover — 2026-07-27

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#101 — goal-based querying: `goalName` on `AgentQuery`, `hasGoal()`/`hasConstraint()` on `AgentDescriptor`, EXISTS subquery in JPA, InMemory stream filter. Platform boundary analysis: routing/termination deferred to engine (engine#784, engine#785).
- Closed eidos#102 — capability name uniqueness: `MAX_CAPABILITIES = 20`, duplicate name check in compact constructor, UNIQUE DDL constraint in V1 base migration.
- Closed eidos#103 — `ConstraintSeverity { HARD, SOFT }` on `AgentConstraint`: severity-discriminated rendering (MARKDOWN `[HARD]`/`[SOFT]`, PROSE grouped, A2A_CARD field), hash coverage, V8 migration.
- Closed eidos#105 — three template composition examples: ordering, shared templates, YAML registration.
- Fixed CI: renamed duplicate V7 Flyway migration to V8.
- Created engine#784 (goal-aware routing) and engine#785 (goal-based termination).
- Design-reviewed (3 rounds, 14 issues, 13 verified, 1 accepted). Garden entry GE-20260726-756909 (ide_replace_text_in_file escape gotcha).

## Immediate Next Step

Pick from epic #82 examples: #78 (learned specialization lifecycle), #80 (full probe pipeline), or #81 (cost-aware routing). Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #78 | Example: learned specialization lifecycle (SUCCESS/DECLINE signals) | M | Med | Epic #82 |
| #80 | Example: full probe pipeline (all five decision steps) | M | Med | Epic #82 |
| #81 | Example: cost-aware multi-agent routing | M | Med | Epic #82 |
| #92 | Engine observation logic for delegation/escalation compliance | M | High | |
| #93 | track: Engine adoption of behavioral contracts (engine#647) | M | Low | Tracking — blocked by engine#647 |
| #94 | Reactive tests for examples | S | Low | Blocked by #78, #80, #81 |
| #101 | Goal-based querying, routing, advanced goal use cases | M | Med | **Closed this session** |

## References

- Design review: `~/adr/casehub-eidos/goals-query-api-enhancements-*/`
- Blog: `blog/2026-07-27-mdp01-fields-a-descriptor-doesnt-ask-about.md`
- Spec: `specs/issue-101-goals-query-api-enhancements/2026-07-26-goals-query-api-enhancements-design.md`
- Garden: `GE-20260726-756909` — ide_replace_text_in_file escape sequence gotcha
