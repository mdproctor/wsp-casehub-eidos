# CaseHub Eidos — Session Handover
**Date:** 2026-06-04
*Updated: parent#149 closed since last session — removed from backlog.*

## Current State

eidos#36 and #37 are complete and delivered to `casehubio/eidos` main (squashed to 2 commits):

- `feat(graph): add observedAt to AgentOutcome — business time not persistence time (Closes #36)`
- `feat(graph): ReactiveAgentGraphQuery parity — historyByCapability + attestationsFor (Closes #37)`

Key design points captured in `design/DESIGN.md`: `observedAt` semantic (business time not persistence time), NaN IEEE 754 guard on confidence, CDI no-arg + test constructor pair for field-injected beans.

## Immediate Next Step

Pick up eidos#31 — complete ARC42STORIES.MD Foundation tier. Rich source material in workspace `DESIGN.md` and `blog/`. §3–§13 are stubs. Run `/work` to start.

## What's Left

- parent#167 — eidos deep-dive: ReactiveAgentGraphQuery missing from Reactive Build Gating section; AgentOutcome.observedAt not noted in Current State · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #31 | Complete ARC42STORIES.MD — Foundation tier §3–§13 | M | Med | Rich source: DESIGN.md + blog/ |
| parent#167 | eidos deep-dive: ReactiveAgentGraphQuery reactive build gating | XS | Low | Field-injected @DefaultBean bridge, not build-gated |
| #29 | Docs: Mapping Personality and Role Frameworks to AgentDescriptor | M | Med | Gates #26 (Belbin/DISC vocab) |
| #26 | Belbin/DISC/Big Five vocabulary module | L | High | Gates on #29 |
| — | casehub-engine: WorkOrchestrator write-path integration (AgentGraphStore.recordTask/recordOutcome) | M | Med | Requires engine issue; passes descriptor.tenancyId() |

## References

| What | Path |
|------|------|
| Knowledge graph design spec | `docs/specs/2026-06-02-knowledge-graph-design.md` (project repo) |
| observedAt + parity spec | `docs/specs/2026-06-04-agentoutcome-observedat-reactive-parity.md` (project repo) |
| Latest blog | `blog/2026-06-04-mdp01-timestamps-and-gaps.md` |
| Garden entries | GE-20260604-043617 (NaN IEEE 754 guard), GE-20260604-4bfd2c (CDI field-injection no-arg loss) |
| Open issues | eidos#31, #29, #26, #27, #28; parent#149, #167 |
