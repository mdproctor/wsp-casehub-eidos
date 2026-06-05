# CaseHub Eidos — Session Handover
**Date:** 2026-06-05

## Current State

eidos#29 closed — `docs/personality-frameworks.md` complete. 828 lines covering 11
frameworks, cross-reference table, 3 combination patterns, 7 anti-patterns, vocabulary
draft tables for Belbin (mechanical for #26) and DISC (blocked on eidos#40). ADR 0003
written: DISC as disposition vocabulary, not slot vocabulary. Protocols filed to parent
repo (parent#177, parent#178).

Key architectural decision: DISC types are disposition vocabulary, not slot vocabulary.
An agent holds both simultaneously — `slotVocabulary=belbin` for team role and
`dispositionVocabulary=disc` for behavioral style. The Belbin+DISC combination is
additive; blocked on eidos#40 for full implementation.

## Immediate Next Step

Start eidos#40 — design the axis-aware `equivalentValues()` extension. Run `/work eidos#40`.
Small: likely just an API change + ADR. Unblocks the DISC half of eidos#26.

## What's Left

- parent#177 — add DISC vocab protocol to parent repo protocols/casehub/ · XS · Low
- parent#178 — add delegation platform-semantic protocol · XS · Low
- parent#174 — update eidos deep-dive with personality-frameworks.md reference · XS · Low
- eidos#41 — three minor polish items in personality-frameworks.md · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| eidos#40 | equivalentValues() axis-aware extension — API design decision | S | Med | Unblocks DISC half of eidos#26 |
| eidos#26 | Belbin/DISC vocabulary module — BelbinVocabularyProducer | M | Low | Belbin half immediately startable; DISC half blocked on #40 |
| eidos#38 | conflictMode as 5th AgentDisposition axis | M | Med | Design decision; requires AgentDisposition API change |
| eidos#39 | AgentDisposition as Map<String,String> | L | High | Significant rearchitect; evaluate after #38 |
| eidos#27 | Theoretical framework grounding in AgentDescriptor + renderer | M | Med | Depends on #26 (Option B) |
| eidos#28 | casehub-engine: Belbin-based agent composition for phases | L | High | Cross-repo; depends on #26, #27 |

## References

| What | Path |
|------|------|
| Personality frameworks doc | `docs/personality-frameworks.md` (project) |
| Spec (3 rounds of review) | `docs/superpowers/specs/2026-06-05-personality-framework-mapping-design.md` |
| ADR 0003 | `docs/adr/0003-disc-vocabulary-disposition-not-slot.md` |
| Latest blog | `blog/2026-06-05-mdp01-two-vocabularies.md` |
