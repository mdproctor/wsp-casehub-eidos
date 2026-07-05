# eidos Session Handover — 2026-07-05

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#95 — two data-engineer eval profiles (autonomous/directed) with capability descriptions on all capabilities. AUTONOMY variant pair (third axis covered). Synthetic A2A_CARD case in EvalDataset. Design review ran (3 rounds, 12 issues, 9 verified, 3 accepted, $11.68).
- Filed eidos#97 — deferred from #95 design review: isolate A2A_CARD declared-description fallback path (enrichment always runs in eval, so declared description fallback is untested).
- CLAUDE.md updated for description field rendering and eval profile count.

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
| #94 | Reactive tests for examples | S | Low | Blocked by #78, #80, #81 |
| #97 | Isolate A2A_CARD declared-description fallback path in eval | S | Med | From #95 design review |
| engine#632 | Record behavioral compliance signals from eidos framework | M | Med | Engine dispatch path |
| engine#638 | Surface MatchDegree in AgentCandidate for dispatch-time ranking | S | Low | From #84 design review |
| engine#639 | Routing strategies rank on match quality | M | Med | From #84 design review |
| engine#645 | Observe delegation and escalation compliance dimensions | M | Med | From #87 |
| engine#647 | Integrate with eidos behavioral contracts framework | M | Med | Transferred from eidos#89 |

## References

- Design review: `~/adr/casehub-eidos/capability-desc-eval-profiles-20260705-021649/`
- Blog: `blog/2026-07-05-mdp02-the-eval-gap-nobody-tests.md`
- Spec: `specs/2026-07-05-eval-capability-description-profiles-design.md`
