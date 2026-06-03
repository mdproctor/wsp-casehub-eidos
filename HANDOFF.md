# CaseHub Eidos — Session Handover
**Date:** 2026-06-03

## Current State

Both eidos#25/#30 (eval cleanup) and the Phase 4 knowledge graph (eidos#32/#33/#34/#35) are complete and delivered to `casehubio/eidos` main (squashed to 6 commits):

- `feat(api): add tenancyId to AgentStateStore — SPI, all impls, contract test` (Closes #33)
- `fix: agent_descriptor surrogate key — BIGSERIAL PK, UNIQUE (agent_id, tenancy_id)` (Closes #34)
- `feat(examples): Phase 4a PoC — InMemoryAgentGraph + V1/V2 scenarios` (Refs #35)
- `feat: knowledge graph API types, SPIs, and NoOp runtime defaults` (Refs #35)
- `feat(graph): casehub-eidos-graph — JPA impl, Wilson ranking, ArchUnit` (Closes #35)
- `docs: promote knowledge graph design spec from workspace` (Refs #32)

Design spec at `docs/specs/2026-06-02-knowledge-graph-design.md`.

## Immediate Next Step

Pick up eidos#36 — `AgentOutcome` is missing `observedAt`. The entity stamps `Instant.now()` at persistence time rather than when the outcome occurred. Needed before ledger backfill produces accurate timestamps. XS, Low — add field to record, update entity mapping.

## What's Left

- eidos#36 — AgentOutcome missing observedAt field · XS · Low
- eidos#37 — ReactiveAgentGraphQuery missing historyByCapability + attestationsFor · S · Low
- parent#149 — PLATFORM.md docs sync: Capability Ownership table (AgentGraphStore/Query/Backfill/TaskSemanticEnricher), cross-repo dep map (eidos-api ← engine write path), casehub-eidos deep-dive update · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #36 | AgentOutcome.observedAt — add to record and entity | XS | Low | Batch with #37 |
| #37 | ReactiveAgentGraphQuery — add historyByCapability + attestationsFor | XS | Low | Batch with #36 |
| parent#149 | PLATFORM.md docs sync | S | Low | Update Capability Ownership, cross-repo dep map, eidos deep-dive |
| — | casehub-engine: WorkOrchestrator write-path integration (AgentGraphStore.recordTask/recordOutcome) | M | Med | Requires engine issue; passes descriptor.tenancyId() |
| #29 | Docs: Mapping Personality and Role Frameworks to AgentDescriptor | M | Med | Gates #26 (Belbin/DISC vocab) |
| #26 | Belbin/DISC/Big Five vocabulary module | L | High | Gates on #29 |

## References

| What | Path |
|------|------|
| Knowledge graph design spec | `docs/specs/2026-06-02-knowledge-graph-design.md` (project repo) |
| Implementation plan (archived) | `plans/attic/issue-032-knowledge-graph/2026-06-03-phase4-knowledge-graph.md` |
| Latest blog | `blog/2026-06-03-mdp01-eidos-gets-a-memory.md` |
| Garden entries | GE-20260603-86f2a9 (H2 MODE=PostgreSQL: ON CONFLICT not supported) |
| Protocols | PP-20260603-9918e6 (eidos SPI agent scope → tenancyId), PP-20260603-ba301d (PoC gate before JPA) |
| Open issues | eidos#36, #37, #29, #26; parent#149; eidos#27, #28 |
