# Delegation and Escalation Observation Semantics — Design Spec

**Issue:** casehubio/eidos#87
**Date:** 2026-07-04
**Status:** Draft
**Depends on:** casehubio/eidos#85 (behavioral contracts framework — merged)

## Problem

The behavioral contracts framework (eidos#85) defined the compliance
infrastructure — `BehavioralSignalStore`, probe Step 6, `ComplianceDimension`,
`BehavioralExpectations` — and shipped two dimension constants: `LATENCY` and
`ATTESTATION_RATE`. Delegation and escalation were explicitly deferred because
they have two unsolved problems:

1. **Delegation** — detecting *absence* of behavior. An agent declares
   `delegation=true` but what constitutes "failing to delegate"? A single task
   without delegation isn't a violation — not every task warrants delegation.
   The issue frames this as an "observation window" problem.

2. **Escalation** — vocabulary-aware interpretation. The `autonomy` axis on
   `AgentDisposition` is an open String whose values come from arbitrary
   vocabularies. Determining whether an agent's autonomy value implies "should
   escalate when uncertain" requires resolving the value through the vocabulary
   system.

## Core Insight

The "observation window" problem is already solved. The existing
TTL + threshold model provides windowing:
- Per-signal TTL (default 90 days for VIOLATED) → time window
- Probe threshold (default 3 per-dimension, 5 aggregate) → evidence minimum

"3 delegation failures within 90 days → BehavioralViolation" — no new
windowing concept needed.

The real work is:
1. Two new `ComplianceDimension` constants
2. Vocabulary-aware `escalationExpected()` — which requires a new semantic
   method on `VocabularyTerm`
3. Observation contract documentation for engine

## Design

### 1. ComplianceDimension Constants

Two new constants in `ComplianceDimension` (eidos-api):

```java
public static final String DELEGATION = "delegation";
public static final String ESCALATION = "escalation";
```

Same pattern as existing `LATENCY` and `ATTESTATION_RATE` — string constants,
conventions not constraints. Probe Step 6 already counts ALL VIOLATED signals
regardless of dimension key, so these require zero probe changes.

### 2. VocabularyTerm.impliesSupervision()

New default method on `VocabularyTerm` (eidos-api):

```java
/**
 * Returns {@code true} if this term indicates the entity operates under
 * supervision and should escalate to a supervisor when encountering
 * uncertain or high-stakes decisions.
 *
 * <p>Primarily meaningful for {@link DispositionAxis#AUTONOMY AUTONOMY} axis
 * terms. Slot terms, capability terms, and other axis terms inherit the
 * default {@code false} — semantically correct and harmless if called in a
 * non-autonomy context.
 *
 * <p>Used by {@link BehavioralExpectations#escalationExpected} to determine
 * whether escalation compliance should be monitored for an agent.
 */
default boolean impliesSupervision() { return false; }
```

Vocabulary authors override for AUTONOMY axis terms that indicate supervised
behavior. The default `false` means no obligation for vocabularies that don't
participate in behavioral compliance.

**ConscientiousnessTerm overrides:**

| Term | `impliesSupervision()` | Rationale |
|------|------------------------|-----------|
| `DIRECTED` | `true` | "Follows explicit instructions" — operates under supervision |
| `SEMI_AUTONOMOUS` | `true` | "Acts within defined boundaries" — bounded supervision |
| `AUTONOMOUS` | `false` (default) | "Acts on own judgment" — no supervision implied |

Domain vocabulary authors who create AUTONOMY axis terms override this method
for terms that imply supervised behavior. The platform treats false as "no
escalation expectation" — fail open.

### 3. BehavioralExpectations.escalationExpected()

New static method in `BehavioralExpectations` (eidos-api):

```java
public static boolean escalationExpected(
        final AgentDisposition disposition,
        final String autonomyVocabUri,
        final VocabularyRegistry registry) {
    if (disposition == null || disposition.autonomy() == null) return false;
    if (autonomyVocabUri == null || registry == null) return false;

    return registry.resolve(autonomyVocabUri, disposition.autonomy())
            .map(VocabularyTerm::impliesSupervision)
            .orElse(false);
}
```

**Flow:**
1. Null guards — no disposition, no autonomy value, no vocabulary context →
   no expectation (fail open)
2. Resolve the autonomy string through `VocabularyRegistry` to get the
   `VocabularyTerm`
3. Query `impliesSupervision()` on the resolved term
4. Unresolvable value → false (no expectation)

**Caller provides `autonomyVocabUri`** — obtained from
`AgentDescriptor.vocabUriForAxis(DispositionAxis.AUTONOMY)` which does the
three-step resolution (axisVocabularies → dispositionVocabulary →
domainVocabulary). This keeps `BehavioralExpectations` decoupled from
`AgentDescriptor`'s vocabulary resolution logic.

**`delegationExpected(AgentDisposition)` is unchanged** — it already returns
the correct boolean from the disposition flag. "Reinstated with observation
context" is satisfied by connecting it to the compliance framework via the
new `ComplianceDimension.DELEGATION` constant and the observation contract
documented below.

### 4. Observation Contract

Eidos provides expectations. The engine observes and records. No new
abstractions — the existing `BehavioralSignalStore` API handles both
dimensions without modification.

**Delegation:**

| Aspect | Value |
|--------|-------|
| Precondition | `BehavioralExpectations.delegationExpected(disposition)` returns `true` |
| Observation unit | Per-task, engine determines eligibility |
| Eligibility criteria | Engine policy — task complexity, sub-task decomposability, multi-capability requirements |
| Violation signal | `record(agentId, tenancyId, capabilityName, DELEGATION, VIOLATED)` |
| Compliance signal | `record(agentId, tenancyId, capabilityName, DELEGATION, COMPLIANT)` |
| Window | TTL (90 days default) + threshold (3 per-dimension default) |

**Escalation:**

| Aspect | Value |
|--------|-------|
| Precondition | `BehavioralExpectations.escalationExpected(disposition, vocabUri, registry)` returns `true` |
| Observation unit | Per-task, engine determines escalation triggers |
| Trigger criteria | Engine policy — uncertainty detected, high-stakes decision, out-of-scope request |
| Violation signal | `record(agentId, tenancyId, capabilityName, ESCALATION, VIOLATED)` |
| Compliance signal | `record(agentId, tenancyId, capabilityName, ESCALATION, COMPLIANT)` |
| Window | TTL (90 days default) + threshold (3 per-dimension default) |

**What eidos does NOT define:**
- Delegation eligibility criteria — engine policy based on task characteristics
- Escalation trigger conditions — engine policy based on task context
- What platform events constitute delegation/escalation — casehub-work
  `DELEGATED`/`ESCALATED` statuses are one source, but engine may use others

The attestation bridge (`ComplianceAttestations`) already works for both
dimensions — `violation(...)` and `compliance(...)` accept any dimension
string. No changes needed.

### 5. Module Placement Summary

**eidos-api (Tier 1, pure Java):**
- `VocabularyTerm` — add `default boolean impliesSupervision() { return false; }`
- `ComplianceDimension` — add `DELEGATION` and `ESCALATION` constants
- `BehavioralExpectations` — add `escalationExpected(AgentDisposition, String, VocabularyRegistry)`

**casehub-eidos-vocab:**
- `ConscientiousnessTerm.DIRECTED` — override `impliesSupervision()` → `true`
- `ConscientiousnessTerm.SEMI_AUTONOMOUS` — override `impliesSupervision()` → `true`

**No changes to:**
- `BehavioralSignalStore` — existing API handles new dimensions
- `DefaultCapabilityHealth.probe()` — Step 6 already counts all VIOLATED signals
- `ComplianceAttestations` — already dimension-agnostic
- `BehavioralSignal` enum — COMPLIANT/VIOLATED cover both dimensions
- Schema/migrations — no persistence changes
- Probe pipeline order — unchanged

### 6. Test Plan

**Unit tests (eidos-api):**
- `BehavioralExpectationsTest` — `escalationExpected()`:
  - DIRECTED autonomy with registered vocab → `true`
  - SEMI_AUTONOMOUS autonomy with registered vocab → `true`
  - AUTONOMOUS autonomy with registered vocab → `false`
  - Null disposition → `false`
  - Null autonomy value → `false`
  - Null vocabUri → `false`
  - Null registry → `false`
  - Unresolvable autonomy value → `false`

**Unit tests (casehub-eidos-vocab):**
- `ConscientiousnessTermTest` — `impliesSupervision()` for all three
  AUTONOMY terms

**Integration test (runtime):**
- Probe pipeline with DELEGATION VIOLATED signals — verify Step 6 fires
  `BehavioralViolation` with delegation dimension
- Probe pipeline with ESCALATION VIOLATED signals — same verification
- Both use existing `InMemoryBehavioralSignalStore` — no new test
  infrastructure needed

### 7. Cross-Repo Impact

**Engine #89** already tracks compliance signal recording for latency and
attestation-rate. This design extends the scope to delegation and escalation —
update #89's description to include both new dimensions.

**No new issues needed.** The engine observation implementation is already
scoped to #89.

## What This Design Does NOT Do

- Does not define delegation eligibility criteria — engine policy
- Does not define escalation trigger conditions — engine policy
- Does not add new signal types to `BehavioralSignal` — COMPLIANT/VIOLATED
  are sufficient
- Does not modify the probe pipeline — Step 6 already handles all VIOLATED
  signals generically
- Does not add observation windowing concepts — TTL + threshold IS the window
- Does not require casehub-eidos-vocab to be present — `escalationExpected()`
  fails open when vocabulary context is unavailable

## Rationale

**Why `impliesSupervision()` on VocabularyTerm, not a separate interface:**
The vocabulary author knows what their terms mean. A default method on the
existing interface is the minimal extension — no new types, no new
registration, no cross-module coupling. Domain vocabulary authors override it
for supervision-implying terms; everyone else ignores it.

**Why not hardcode against ConscientiousnessTerm values:** eidos-api cannot
depend on casehub-eidos-vocab. More importantly, domain apps define their own
autonomy vocabularies — hardcoding against one vocabulary's values breaks the
open-string design principle.

**Why no new observation window concept:** The existing TTL + threshold model
already provides time-bounded, count-gated observation windowing. Adding
another layer would duplicate infrastructure and create confusion about which
window governs behavior.

**Why delegation eligibility is engine policy:** Eidos models agent identity
and behavioral contracts. It does not model task characteristics — complexity,
decomposability, and multi-capability requirements are engine/work-layer
concepts. Eidos says WHAT is expected; the engine decides WHEN to observe.

**Why fail open:** When vocabulary context is missing (no registry, no vocab
URI, unresolvable value), returning `false` means "no escalation expectation."
This is safer than false positives — an agent without vocabulary-grounded
disposition should not be penalised for not escalating.
