# eidos Session Handover — 2026-07-02

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#76 — fixed learned exclusion tag mismatch: `probe()` now uses declared capability name for `CapabilitySpecializationStore.count()` lookups. Extracted `CapabilityResolver` utility (api module, Tier 1) with `match()` and `resolve()` static methods — shared subsumption resolution for probe and recording paths. Updated SPI Javadoc to specify declared-name contract. Updated ARC42STORIES L4 (probe order, variant count). CLAUDE.md synced.
- Filed eidos#78–82 (practical examples epic) and eidos#83–84 (deferred from #76 brainstorming).

## Immediate Next Step

Pick from the examples epic (#82) or the follow-up issues (#83, #84). #79 (degradation and recovery example) is the quickest win. #83 (InMemoryAgentRegistry CapabilityResolver unification) is a clean refactor.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #83 | Unify InMemoryAgentRegistry.matchesCapability() → CapabilityResolver.match() | XS | Low | Pure refactor, API ready from #76 |
| #84 | AgentRegistry.find() returns match metadata (matched capability, match degree) | M | Med | Breaking API change, no external consumers |
| #78 | Example: learned specialization lifecycle (SUCCESS/DECLINE signals) | M | Med | Epic #82 |
| #79 | Example: degradation and recovery (AgentStateStore TTL) | S | Low | Epic #82, quickest win |
| #80 | Example: full probe pipeline (all five decision steps) | M | Med | Epic #82 |
| #81 | Example: cost-aware multi-agent routing | M | Med | Epic #82 |
| engine#609 | AgentCandidateFactory subsumption matching | S | Low | Engine dispatch path |
| — | Behavioral contracts / runtime validation | L | High | No issue filed; bridges eidos and ledger |

## References

- Design spec: `specs/2026-07-02-learned-exclusion-tag-fix-design.md`
- Blog: `blog/2026-07-02-mdp01-the-tag-that-didnt-match.md`
- ADR review responses: `~/adr/casehub-eidos/learned-exclusion-tag-fix-20260702-010251/`
