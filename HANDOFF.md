# eidos Session Handover — 2026-07-05

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#77 — `AgentCapability.description` field. Nullable String (≤500 chars), second record component. Renders in MARKDOWN (em dash), PROSE (parenthetical), A2A_CARD (declared fallback when enrichment absent/blank). Flyway V6 migration, JPA entity, mapper, YAML registrar all updated. Design review ran (3 rounds, 11 issues, 10 verified, 1 accepted, $10.17).
- Closed eidos#79 — degradation and recovery example. `@QuarkusTest` demonstrating `AgentStateStore` TTL lifecycle via SPI injection (5 tests: ready, degraded, clear, TTL expiry, reason discrimination).
- Closed eidos#72 — stale issue (probe subsumption already implemented in #76).
- Transferred eidos#89 → engine#647 (engine-scoped work). Created eidos#93 as tracking issue.
- Filed eidos#94 (reactive tests for examples) and eidos#95 (eval profiles for description) from design review.
- CLAUDE.md updated for description field and rendering changes.

## Immediate Next Step

Pick from the remaining backlog. #78 (learned specialization lifecycle) or #80 (full probe pipeline) are the next examples in epic #82. Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #78 | Example: learned specialization lifecycle (SUCCESS/DECLINE signals) | M | Med | Epic #82 |
| #80 | Example: full probe pipeline (all six steps) | M | Med | Epic #82 |
| #81 | Example: cost-aware multi-agent routing | M | Med | Epic #82 |
| #93 | track: Engine adoption of behavioral contracts framework (engine#647) | M | Low | Tracking only — blocked by engine#647 |
| #92 | Engine observation logic for delegation/escalation compliance dimensions | M | High | |
| engine#632 | Record behavioral compliance signals from eidos framework | M | Med | Engine dispatch path |
| engine#638 | Surface MatchDegree in AgentCandidate for dispatch-time ranking | S | Low | From #84 design review |
| engine#639 | Routing strategies rank on match quality | M | Med | From #84 design review |
| engine#645 | Observe delegation and escalation compliance dimensions | M | Med | From #87, requires engine policy design |
| engine#647 | Integrate with eidos behavioral contracts framework | M | Med | Transferred from eidos#89 |

## References

- Design review: `~/adr/casehub-eidos/capability-desc-degradation-example-*/`
- Blog: `blog/2026-07-05-mdp01-the-field-that-was-already-missing.md`
- Spec: `specs/2026-07-04-capability-desc-degradation-example-design.md`
