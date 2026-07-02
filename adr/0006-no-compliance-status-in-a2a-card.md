# ADR 0006: No Compliance Status in A2A_CARD Rendering

**Status:** Accepted
**Date:** 2026-07-02
**Issue:** casehubio/eidos#86

## Context

The A2A_CARD format surfaces capability routing signals (`qualityHint`, `latencyHintP50Ms`, `costHint`, `epistemicDomains`) for machine consumers. After eidos#85 introduced behavioral compliance checking (probe Step 6), the question arose whether compliance status should also appear in A2A_CARD.

## Decision

Do not surface compliance status in A2A_CARD rendering.

## Rationale

1. **Category mismatch.** The A2A_CARD represents what an agent *declares* — its capabilities, routing signals, and disposition. All current card content derives from `AgentDescriptor` (static, set at registration). Compliance status is *runtime state* about whether the agent is living up to those declarations. Mixing these conflates identity with performance.

2. **Caching architecture.** A2A_CARD uses fingerprint-based caching (`descriptorHash + contextHash + templateHash`). Adding dynamic compliance state would require re-rendering the card whenever compliance changes, defeating the caching design. Routing signals like `qualityHint` do not have this problem — they are static descriptor fields.

3. **Redundancy with probe().** The engine calls `CapabilityHealth.probe()` at dispatch time, which already checks compliance (Step 6). Adding compliance to the card duplicates information the consumer obtains through `probe()`.

4. **Renderer contract.** `assembleA2aCard()` receives only `AgentDescriptor` (plus optional A2A enrichment). Injecting `BehavioralSignalStore` into the renderer would break the boundary between descriptor rendering and runtime health checking.

5. **Architecture alignment.** CaseHub uses engine-coordinated dispatch — agents do not route to each other directly via A2A_CARD. Adding compliance to the card for hypothetical agent-to-agent routing would be premature design.

## Consequences

- Machine consumers needing compliance information alongside agent identity use `probe()` via the engine dispatch path.
- The engine's `AgentCandidate` (descriptor + probe result) remains the correct composite for dispatch decisions.
- If future work introduces direct agent-to-agent routing, this decision should be revisited.
- Protocol PP-20260611-228599 (capability-metadata-rendering) is unaffected — compliance is not a routing signal.
