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
- JPAF research analysis: mapped Jungian personality framework against eidos architecture. Created two epics:
  - eidos#107 (Jungian personality framework — 10 child issues: #108-#117)
  - engine#790 (personality-adaptive routing — 6 child issues: #791-#796)

## Immediate Next Step

JPAF vocabulary is the natural next pick — eidos#108 (JungianFunctionTerm) is S/Low, zero API change, immediately useful. Or continue epic #82 examples: #78, #80, #81. Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #107 | **epic:** Jungian personality framework | XL | High | 10 child issues (#108-#117) |
| #108 | vocab: Jungian cognitive function vocabulary | S | Low | Epic #107 — no API change |
| #109 | vocab: Cross-vocabulary equivalences (Jung ↔ DISC ↔ Belbin) | S | Med | Epic #107 — depends on #108 |
| #110 | vocab: MBTI type vocabulary via specialization | S | Med | Epic #107 — depends on #108 |
| #111 | feat: Weighted disposition profile on AgentDescriptor | M | High | Epic #107 — major API change |
| #114 | docs: Update personality-frameworks.md | S | Low | Epic #107 |
| #78 | Example: learned specialization lifecycle | M | Med | Epic #82 |
| #80 | Example: full probe pipeline | M | Med | Epic #82 |
| #81 | Example: cost-aware multi-agent routing | M | Med | Epic #82 |
| #92 | Engine observation logic for delegation/escalation compliance | M | High | |
| #94 | Reactive tests for examples | S | Low | Blocked by #78, #80, #81 |

## Cross-Module

**Enabled** (we delivered, downstream can now proceed):
- `engine` — goal-aware routing (engine#784) and goal-based termination (engine#785) can now use `AgentQuery.byGoal()` and `descriptor.goals()` · M · Med

**Blocking** (eidos must deliver before engine can start):
- `engine` — personality-adaptive routing (engine#790) blocked by eidos#107 (Jungian vocabulary + weighted profiles + DispositionHealth SPI) · XL · High

## References

- Design review: `~/adr/casehub-eidos/goals-query-api-enhancements-*/`
- Blog: `blog/2026-07-27-mdp01-fields-a-descriptor-doesnt-ask-about.md`
- Spec: `specs/issue-101-goals-query-api-enhancements/2026-07-26-goals-query-api-enhancements-design.md`
- Garden: `GE-20260726-756909` — ide_replace_text_in_file escape sequence gotcha
- JPAF paper: https://arxiv.org/abs/2601.10025
