# eidos Session Handover — 2026-07-02

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#85 — behavioral contracts and runtime validation. Generalized `CapabilitySpecializationStore` → `BehavioralSignalStore` with COMPLIANT/VIOLATED signals. Added probe Step 6 (behavioral compliance checking with configurable violation threshold). Created `ComplianceDimension` constants, `BehavioralExpectations` utility, `ComplianceAttestations` ledger bridge. Renamed `domain` → `qualifier` parameter. Design review ran (3 rounds, 17 issues, all resolved). CLAUDE.md updated.
- Filed eidos#86–91, engine#631–632, parent#338 as follow-up issues from the design review and code review.

## Immediate Next Step

Pick from the follow-up work. #83 (XS refactor — InMemoryAgentRegistry CapabilityResolver unification) is the cleanest standalone task. #91 (XS cleanup — inner class renames, FQ import) is quick hygiene. For the examples epic, #79 (degradation and recovery) remains the quickest win. Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #83 | Unify InMemoryAgentRegistry.matchesCapability() → CapabilityResolver.match() | XS | Low | Pure refactor, API ready from #76 |
| #91 | Minor cleanup from behavioral contracts review (inner class renames, FQ import, preference validation test) | XS | Low | Hygiene from #85 code review |
| #84 | AgentRegistry.find() returns match metadata (matched capability, match degree) | M | Med | Breaking API change, no external consumers |
| #87 | Design observation semantics for delegation and escalation compliance dimensions | M | Med | Requires observation window + vocabulary-aware interpretation |
| #88 | Aggregate behavioral violation threshold (cross-dimension safety valve) | S | Med | Layered on top of per-dimension checking |
| #89 | Engine integration with behavioral contracts framework | M | Med | CapabilityStatus handling, observation recording, PLATFORM.md |
| #90 | ARC42STORIES.MD update for behavioral contracts | S | Low | Layer updates for probe Step 6, BehavioralSignalStore |
| #86 | Consider surfacing compliance status in A2A_CARD rendering | S | Med | Design decision needed |
| #78 | Example: learned specialization lifecycle (SUCCESS/DECLINE signals) | M | Med | Epic #82 |
| #79 | Example: degradation and recovery (AgentStateStore TTL) | S | Low | Epic #82, quickest win |
| #80 | Example: full probe pipeline (all five decision steps) | M | Med | Epic #82 — now six steps |
| #81 | Example: cost-aware multi-agent routing | M | Med | Epic #82 |
| engine#631 | Update engine imports for BehavioralSignalStore rename | XS | Low | Mechanical |
| engine#632 | Record behavioral compliance signals from eidos framework | M | Med | Engine dispatch path |
| parent#338 | Sync PLATFORM.md and eidos deep-dive for BehavioralSignalStore rename | S | Low | Doc sync |
| — | Behavioral contracts / runtime validation | L | High | No issue filed; bridges eidos and ledger |

## References

- Design spec: `specs/2026-07-02-behavioral-contracts-design.md` (promoted to project `docs/specs/`)
- Blog: `blog/2026-07-02-mdp02-the-contract-was-already-there.md`
- Design review: `~/adr/casehub-eidos/behavioral-contracts-20260702-131301/`
- Implementation plan: `plans/2026-07-02-behavioral-contracts-plan.md`
