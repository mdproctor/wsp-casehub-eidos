# CaseHub Eidos — Session Handover
**Date:** 2026-05-26

## Current State

Phases 1–3 complete and merged. 133 tests, all green.

This session: closed #8 (EpistemicallyWeak already soft preference in
WorkOrchestrator — verified, no code change), closed #11 (agentId null guard
on findById across all four registry impls — symmetric with tenancyId), closed
#12 (@throws NullPointerException Javadoc on both SPI interfaces and AgentQuery).
AgentQuery null message normalized to bare field name. Branch
issue-011-registry-agentid-guard merged, closed, EPIC-CLOSED.md stamped.
Garden: GE-20260526-cacddb (UniAsserter Consumer<Throwable> overload for
message assertion in reactive tests).

## Immediate Next Step

Pick the next issue from What's Next — Phase 4 knowledge graph or reactive
parity (#7). Run `work-start` to begin.

## What's Left

Nothing trailing from closed issues.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation) | L | High | |
| #6 | Semantic rendering pipeline research | M | High | Research issue |
| #7 | Reactive parity: AgentStateStore + SystemPromptRenderer | S | Med | After JPA persistence |

## References

| What | Path |
|------|------|
| Blog entries | `blog/2026-05-26-mdp02-crash-or-silence.md` |
| ADR 0001 | `adr/0001-langchain4j-core-for-llm-semantic-pass.md` |
| Design journal | `design/JOURNAL.md` |
| DESIGN.md | `design/DESIGN.md` (initial — §AgentRegistry) |
