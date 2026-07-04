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

**Convenience overload** for callers with a full descriptor:

```java
public static boolean escalationExpected(
        final AgentDescriptor descriptor,
        final VocabularyRegistry registry) {
    if (descriptor == null || descriptor.disposition() == null) return false;
    return descriptor.vocabUriForAxis(DispositionAxis.AUTONOMY)
            .map(uri -> escalationExpected(descriptor.disposition(), uri, registry))
            .orElse(false);
}
```

This internalizes the `vocabUriForAxis` call and Optional handling, matching
`delegationExpected(AgentDisposition)` in ergonomics. The 3-param overload
remains for callers with pre-resolved vocabulary URIs (engine observation
paths, unit tests).

**`delegationExpected(AgentDisposition)` is unchanged** — it already returns
the correct boolean from the disposition flag.

**Issue #87 acceptance criterion 2** — "reinstated with observation context":
The "observation context" is not a parameter on the method itself. It is the
compliance framework that `delegationExpected()` feeds into:
`ComplianceDimension.DELEGATION` provides the dimension key, the engine
records `VIOLATED`/`COMPLIANT` signals via `BehavioralSignalStore`, and probe
Step 6 evaluates those signals against the TTL + threshold window (90 days
default, 3 violations per-dimension). The method was never removed — it was
deferred from #85 because the surrounding observation infrastructure was not
yet designed. This spec completes that infrastructure.

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

**Signal consumption asymmetry:**
- **VIOLATED signals** drive probe Step 6 (capability health) — counted
  against per-dimension and aggregate thresholds to determine
  `BehavioralViolation` status
- **COMPLIANT signals** drive positive trust attestation via
  `ComplianceAttestations.compliance()` (ledger impact) — the trust system
  learns about consistent good behavior, not just failures
- Both MUST be recorded by the engine for the system to function correctly

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
- `BehavioralExpectations` — add `escalationExpected(AgentDisposition, String, VocabularyRegistry)` and convenience overload `escalationExpected(AgentDescriptor, VocabularyRegistry)`

**casehub-eidos-vocab:**
- `ConscientiousnessTerm.DIRECTED` — override `impliesSupervision()` → `true`
- `ConscientiousnessTerm.SEMI_AUTONOMOUS` — override `impliesSupervision()` → `true`
- `DiscTerm.STEADINESS` — override `impliesSupervision()` → `true` (AUTONOMY axis equivalent: DIRECTED)
- `DiscTerm.INFLUENCE` — override `impliesSupervision()` → `true` (AUTONOMY axis equivalent: SEMI_AUTONOMOUS)
- `DiscTerm.CONSCIENTIOUSNESS_DISC` — override `impliesSupervision()` → `true` (AUTONOMY axis equivalent: SEMI_AUTONOMOUS)

**ARC42STORIES.MD:**
- §1 description — add DELEGATION, ESCALATION to `ComplianceDimension` constant list
- L1 and L4 layer entries — add `escalationExpected()` to `BehavioralExpectations` method list
- `ComplianceDimension.java` file entry — add DELEGATION, ESCALATION constants
- `BehavioralExpectations.java` file entry — add `escalationExpected` signatures

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
- `ConscientiousnessTermTest` — `impliesSupervision()` for ALL 12 terms:
  - DIRECTED → `true`, SEMI_AUTONOMOUS → `true`, AUTONOMOUS → `false`
  - STRICT, PRINCIPLED, FLEXIBLE → `false` (RULE_FOLLOWING axis)
  - CONSERVATIVE, MEASURED, BOLD → `false` (RISK_APPETITE axis)
  - COLLABORATIVE, INDEPENDENT, FACILITATIVE → `false` (SOCIAL_ORIENTATION axis)
- `DiscTermTest` — `impliesSupervision()` for all four terms:
  - STEADINESS → `true`, INFLUENCE → `true`, CONSCIENTIOUSNESS_DISC → `true`
  - DOMINANCE → `false`
- Cross-vocabulary consistency test: for every DISC term that has an
  AUTONOMY `axisExactMatch` to a ConscientiousnessTerm, assert that both
  terms return the same value from `impliesSupervision()`. This catches
  future vocabulary additions where the override is forgotten.

**Cross-vocabulary escalation test (eidos-api):**
- `BehavioralExpectationsTest` — `escalationExpected()` with DISC autonomy
  vocabulary:
  - DISC `steadiness` with registered DISC vocab → `true`
  - DISC `dominance` with registered DISC vocab → `false`

**Integration test (runtime):**
- Probe pipeline with DELEGATION VIOLATED signals — verify Step 6 fires
  `BehavioralViolation` with delegation dimension
- Probe pipeline with ESCALATION VIOLATED signals — same verification
- Both use existing `InMemoryBehavioralSignalStore` — no new test
  infrastructure needed

### 7. Cross-Repo Impact

**Engine #89** tracks compliance signal recording for latency and
attestation-rate — measurement-based observations with straightforward
recording logic. Its acceptance criteria and scope are unchanged.

**Engine #92** (filed) covers delegation and escalation observation — these
are policy-based judgments (task complexity for delegation, uncertainty
detection for escalation) that require independent design work in the engine.
See acceptance criteria on the issue.

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
