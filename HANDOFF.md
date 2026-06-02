# CaseHub Eidos — Session Handover
**Date:** 2026-06-02

## Current State

eidos#23 closed and delivered to `casehubio/eidos` main (13 squashed commits):

- Design spec (v1–v5): sealed EvalCase interface rationale, ProximityJudge separation, three-stage personality preservation
- Foundation types: enums (SourceType, CoverageLoss, TraitPolarity, Attribution), records (AgentProfile, VariantPair, VariantIndex, ProximityResult, PairContrastResult, etc.)
- EvalCase → sealed interface (SyntheticEvalCase + ProfiledEvalCase); 20 call sites migrated
- ProximityReport + PersonalityPreservationReport with attribution logic (s1≥4 guard on all diagnoses beyond VOCABULARY_GAP)
- AgentProfileLoader (test scope) + RealWorldEvalDataset + Stage 0 variant pair validation
- Four judges: ProximityJudge, VocabularyExpressivenessJudge (Stage 1), TraitExpressionJudge (Stage 2 blind), PairContrastJudge (Stage 3 effect size)
- Phase 1: 8 real-world YAML profiles (O*NET + Anthropic + practitioner), Belbin/DISC grounded
- PromptEvalTest: evaluateRealWorldScenarios() + runReliabilityCheck()
- 81 tests; all green; **no real LLM results yet** (eval tests are @Tag("eval"), require credentials)

Branch `issue-023-real-world-profile-library` closed. Both repos on main.

## Immediate Next Step

Pick up eidos#25 — `JsonProcessingException` from `PromptJudge.parseResponse()` falls through to "LLM call failed". Same misclassification fixed in #24 for ISE, different cause. XS, Low — single-line fix.

## What's Left

- eidos#25 — JsonProcessingException misclassification in PromptJudge · XS · Low
- eidos#30 — minor eval cleanup: attribution guard clarity, duplicate Stage 0 test, emoji in preservationSummaryTable · XS · Low
- parent#141 — docs sync: casehub-eidos deep-dive needs eval module row, `delegation` field name fix, current state update · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #25 | JsonProcessingException misclassification in PromptJudge.parseResponse() | XS | Low | Same fix pattern as #24 ISE fix |
| #30 | Minor eval cleanup (post #23 review) | XS | Low | Batch with #25 or standalone |
| — | Phase 4: knowledge graph (descriptor, task, outcome, attestation nodes) | L | High | Next major milestone; real-world profiles (#23) should inform graph schema |
| #26 | Belbin/DISC/Big Five vocabulary module | L | High | Awaits #29 (framework mapping doc) |
| #29 | Docs: Mapping Personality and Role Frameworks to AgentDescriptor | M | Med | Prerequisite for #26; also gates #27/#28 |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-02-mdp01-grounding-agent-identity.md` |
| Design spec | `docs/specs/2026-06-01-real-world-profile-library-design.md` (project repo, v5) |
| Plan (archived) | `plans/attic/issue-023-real-world-profile-library/2026-06-01-real-world-profile-library.md` |
| Garden entries | GE-20260602-2cff5e (Jackson primitive boolean absent YAML key), GE-20260602-a4d290 (findAndRegisterModules required), GE-20260602-f2ca07 (blind judge payload isolation) |
| Protocols | PP-20260602-64fde8 (eval variant pair single-axis isolation), PP-20260602-3ecfdb (EvalDimension format-only gate) |
| Open issues | eidos#25, #26, #27, #28, #29, #30; parent#141 |
