# CaseHub Eidos — Session Handover
**Date:** 2026-05-22

## Current State

Bootstrap complete. Repo exists at casehubio/eidos and mdproctor/eidos.
Maven skeleton: api/ (pure Java types + SPIs), runtime/, persistence-memory/,
deployment/, vocab/. All core types defined in casehub-eidos-api. No implementations
yet — that is Phase 1.

Parent workflows updated: dashboard.yml and full-stack-build.yml include eidos.
Platform protocol added: platform-api-scope.md (garden) — casehub-platform-api
is for platform primitives only, not domain types.

## Immediate Next Step

Start Phase 1 implementation — `work-start` to create a branch and issue, then:
- `JpaAgentRegistry` in runtime/
- `CdiVocabularyRegistry` in runtime/
- `InMemoryAgentRegistry` in persistence-memory/
- Basic Flyway migration for `agent_descriptor` table
- Wire `casehub-eidos-vocab` SVO + Conscientiousness + CasehubSlot vocabularies

## What's Next

| Phase | Description | Scale | Complexity |
|-------|-------------|-------|-----------|
| Phase 1 | Descriptor + registry (JPA + in-memory) + vocab | M | Med |
| Phase 2 | Discovery queries + CapabilityHealth + engine dispatch integration | M | Med |
| Phase 3 | SystemPromptRenderer + ClaudeMarkdownRenderer + goal injection | M | High |
| Phase 4 | Knowledge graph — (descriptor, task, outcome, attestation) store | L | High |

## References

| What | Path |
|------|------|
| Eidos spec | `research/eidos.md` |
| Research notes | `research/agent-description-ontology.md` |
| casehub-platform-api scope protocol | garden: `docs/protocols/casehub/platform-api-scope.md` |
| Previous session handover (ledger) | `/Users/mdproctor/claude/public/casehub/ledger/HANDOFF.md` |
