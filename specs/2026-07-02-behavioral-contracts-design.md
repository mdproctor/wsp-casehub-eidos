# Behavioral Contracts and Runtime Validation — Design Spec

**Issue:** casehubio/eidos#85
**Date:** 2026-07-02
**Status:** Draft

## Problem

The platform describes what an agent IS (descriptor), checks if it's AVAILABLE
(probe), and records what it DID (ledger). But nobody checks whether the agent's
actual behavior matches its declared identity.

The descriptor already contains behavioral promises — `latencyHintP50Ms`,
`qualityHint`, `excludedDomains`, `epistemicDomains`, `delegation`, disposition
axes. These are implicit behavioral expectations with no validation mechanism.

The gap is the feedback loop: observe behavior → compare against expectations →
record evidence → adjust routing.

## Core Insight

The descriptor IS the behavioral contract. It just needs a compliance checking
framework.

The `CapabilitySpecializationStore` already implements the right pattern —
accumulate signals over time, query at probe time, influence routing. Behavioral
compliance extends this pattern to all descriptor-implied expectations.

## Design

### 1. Generalized Behavioral Signal Store

Rename and extend the existing specialization store.

**API (eidos-api):**

```
SpecializationSignal     → BehavioralSignal { DECLINE, SUCCESS, COMPLIANT, VIOLATED }
CapabilitySpecializationStore → BehavioralSignalStore
```

The SPI method signatures retain the same four methods (`record`, `clear`,
`learned`, `count`) with the same parameter shape. The `domain` parameter is
renamed to `qualifier` — a free-text key whose meaning depends on signal type:
- Task domain for DECLINE/SUCCESS signals (existing semantics)
- Compliance dimension key for COMPLIANT/VIOLATED signals (new semantics)

The rename from `domain` to `qualifier` makes the dual-purpose nature explicit
in the API, Javadoc, entity, and database column. `ProbeContext.taskDomain()`
is unchanged — it describes the probe context; consumers pass its value as
the `qualifier` argument for DECLINE/SUCCESS signals.

**Implementation changes required in store implementations:**
- `JpaBehavioralSignalStore`: `ttlDaysFor()` expands from 2-way branch
  (DECLINE vs everything) to 4-way (one per signal type), with 4
  `@ConfigProperty` injections. `@IfBuildProperty` gating preserved.
- `InMemoryBehavioralSignalStore`: `StoreKey` record type parameter changes
  from `SpecializationSignal` to `BehavioralSignal`.
- `NoOpBehavioralSignalStore`: signature-only rename, no logic changes.

**Runtime:**

```
JpaCapabilitySpecializationStore  → JpaBehavioralSignalStore
NoOpCapabilitySpecializationStore → NoOpBehavioralSignalStore
CapabilitySpecializationEntity    → BehavioralSignalEntity
CapabilitySpecializationId        → BehavioralSignalId
capability_specialization (table) → behavioral_signal
```

**Memory module:**

```
InMemoryCapabilitySpecializationStore → InMemoryBehavioralSignalStore
```

**Config properties:**

```properties
casehub.eidos.behavioral-signal.decline-ttl-days=30
casehub.eidos.behavioral-signal.success-ttl-days=30
casehub.eidos.behavioral-signal.compliant-ttl-days=30
casehub.eidos.behavioral-signal.violated-ttl-days=90
```

Violations have longer default TTL — higher-stakes signals that shouldn't expire
quickly.

**Schema:** No deployed instances. Modify the existing V5 migration directly.
Table rename (`capability_specialization` → `behavioral_signal`), column
rename (`domain` → `qualifier`). The composite key structure and column types
are unchanged. `signal_type` column already stores enum names as strings; new
values work without DDL changes beyond the renames.

**Config property mapping (old → new):**

| Old | New |
|-----|-----|
| `casehub.eidos.specialization.decline-ttl-days` | `casehub.eidos.behavioral-signal.decline-ttl-days` |
| `casehub.eidos.specialization.success-ttl-days` | `casehub.eidos.behavioral-signal.success-ttl-days` |
| (new) | `casehub.eidos.behavioral-signal.compliant-ttl-days` |
| (new) | `casehub.eidos.behavioral-signal.violated-ttl-days` |

`EidosPreferenceKeys.EXCLUDE_THRESHOLD` retains its key
`specialization.exclude-threshold` — it governs the learned exclusion threshold
for DECLINE signals (Step 4), which is semantically distinct from the new
compliance violation threshold. Renaming it would conflate two independent
thresholds.

### 2. Probe Integration — Step 6: Behavioral Compliance

The probe pipeline gains a sixth step after all existing checks.

**Updated probe order:**

```
Step 1: Degraded              → AgentStateStore (TTL-based operational degradation)
Step 2: Unavailable            → capability not declared / not resolved
Step 3: Excluded/DECLARED      → excludedDomains on AgentCapability
Step 4: Excluded/LEARNED       → BehavioralSignalStore DECLINE signals
Step 5: EpistemicallyWeak      → epistemic domain confidence below threshold
Step 6: BehavioralViolation    → BehavioralSignalStore VIOLATED signals  ← NEW
Step 7: Ready
```

Step 6 runs after existing checks because degraded, unavailable, or excluded
agents shouldn't reach compliance checking.

**Severity ordering:** The probe order encodes severity from hardest to
softest: Degraded (operational failure) → Unavailable (not declared) →
Excluded (hard domain block) → EpistemicallyWeak (domain confidence) →
BehavioralViolation (behavioral pattern) → Ready. `BehavioralViolation`
is intentionally the softest non-Ready status — an agent with violations
may still be the best available option. Routing strategies should prefer
Ready agents but may fall back to BehavioralViolation agents before trying
EpistemicallyWeak or Excluded agents, depending on the consumer's tolerance.
This guidance is included in the engine integration scope (eidos#89).

**New CapabilityStatus variant:**

```java
record BehavioralViolation(
    Map<String, Integer> violations
) implements CapabilityStatus {}
```

The map contains all dimensions whose violation count exceeds the threshold.
This preserves the cross-dimensional nature of Step 6 — which queries ALL
dimensions by design — rather than collapsing to a single dimension.

**Logic:** Query `BehavioralSignalStore.learned()` for VIOLATED signals using
the resolved `capability.name()` (not the raw `capabilityTag`), consistent
with Step 4's use of the resolved capability name for learned exclusion.
Collect all dimensions whose count exceeds the compliance violation threshold.
If any exist → return `BehavioralViolation` with the complete map.

**Sealed hierarchy impact:** Adding `BehavioralViolation` to the sealed
`CapabilityStatus` interface causes compile-time breakage in all consumers
with exhaustive `switch` or pattern match over the type — casehub-engine,
claudony, and application repos. This is intentional: it forces every call
site to explicitly handle the new variant. Consumer updates are tracked in
eidos#89.

**Reactive path:** `DefaultReactiveCapabilityHealth` delegates to
`DefaultCapabilityHealth` via `runSubscriptionOn(workerPool)`. Step 6 is
inherited automatically. The blocking JPA store query is safe because the
delegation already runs on the worker pool. This invariant must be
preserved if the reactive path is ever refactored to a native reactive
implementation.

**Threshold configuration:**

```properties
casehub.eidos.behavioral.compliance-violation-threshold=3
```

Per-tenancy override via `PreferenceProvider` using a new
`COMPLIANCE_VIOLATION_THRESHOLD` preference key in `EidosPreferenceKeys`,
following the existing `ExcludeThresholdPreference` pattern.

**Scope:** Step 6 queries ALL dimensions for the agent/capability, not just
the current task domain. Behavioral compliance is about overall contract
adherence — a latency violation pattern affects routing regardless of what
domain the current task is in.

### 3. Expectations and Observation Recording

**Expectation sources (resolved at observation time, not stored):**

1. Descriptor-implied — derived from existing `AgentCapability` and
   `AgentDisposition` fields
2. Platform-level — global rules from configuration
3. Tenant-level — per-tenant rules via `PreferenceProvider`

Consumers (engine, application repos) resolve expectations at the moment they
observe behavior, compare, and record COMPLIANT or VIOLATED into the
`BehavioralSignalStore`.

**Standard compliance dimension constants (eidos-api):**

```java
public final class ComplianceDimension {
    public static final String LATENCY = "latency";
    public static final String ATTESTATION_RATE = "attestation-rate";

    // Convention constants for attestation bridge (§4)
    public static final String ATTESTOR_ID = "eidos:compliance";
    public static final String TRUST_DIMENSION_PREFIX = "behavioral:";

    // Latency violation policy
    public static final double LATENCY_VIOLATION_MULTIPLIER = 2.0;

    private ComplianceDimension() {}
}
```

Conventions, not constraints. Consumers can use custom dimension keys. Probe()
counts all VIOLATED signals regardless of dimension key.

Delegation and escalation dimensions are deferred to eidos#87 — they require
observation window semantics (delegation) and vocabulary-aware interpretation
(escalation) that need independent design work.

**Descriptor-derived expectation utility (eidos-api):**

```java
public final class BehavioralExpectations {
    public static OptionalLong latencyBound(AgentCapability capability) { ... }
}
```

Pure static methods. Extracts what the descriptor already declares:
- `latencyBound()` returns the raw `latencyHintP50Ms` value from the
  descriptor. The violation multiplier (`ComplianceDimension.LATENCY_VIOLATION_MULTIPLIER`,
  default 2.0) is applied by the observer, not by this utility — separating
  the declared expectation from the violation policy.

`qualityHint` is explicitly transitional — its own Javadoc identifies
`ActorTrustScore` in casehub-ledger as the evidence-backed replacement. A
quality compliance dimension should be defined against trust scores when they
mature, not against a self-declared prior. No `qualityFloor()` method until
that foundation exists.

Consumers compare observed values against the bound, applying the multiplier
as appropriate, and decide COMPLIANT vs VIOLATED.

**Observation responsibility:**

| Observer | Measures | Dimension | Signal logic |
|----------|----------|-----------|-------------|
| Engine (dispatch) | Response latency vs `latencyBound()` | `latency` | VIOLATED if actual > `latencyBound() * LATENCY_VIOLATION_MULTIPLIER` |
| Engine (attestation) | Attestation verdict rate | `attestation-rate` | VIOLATED if FLAGGED; COMPLIANT if SOUND |
| Application repos | Domain-specific rules | Custom keys | Application decides |

Delegation and escalation dimensions are deferred (eidos#87) — they require
observation window design and vocabulary-aware interpretation respectively.

Eidos provides the framework. Consumers provide the observations.

**Aggregate gap:** The per-dimension threshold design is intentional for v1 —
per-dimension checking is specific and actionable. The aggregate "death by a
thousand cuts" pattern (many dimensions with sub-threshold violations) is a
known limitation tracked in eidos#88.

### 4. Attestation Bridge — Ledger Evidence

Compliance violations can optionally generate `LedgerAttestation` records for
trust scoring impact. Two independent feedback paths from a single observation:

1. **Fast path:** BehavioralSignalStore → probe() → immediate routing exclusion
2. **Trust path:** LedgerAttestation → trust scoring → gradual deprioritization

**Attestation shape for compliance violations:**

- `attestorId = ComplianceDimension.ATTESTOR_ID` (`"eidos:compliance"`)
- `attestorType = ActorType.SYSTEM`
- `verdict = FLAGGED` (violation) or `SOUND` (compliance)
- `confidence = 1.0` — measured, not subjective
- `trustDimension = ComplianceDimension.TRUST_DIMENSION_PREFIX + dimension`
  (`"behavioral:<dimension>"`)
- `dimensionScore` — 0.0 for hard violations, continuous for soft

Convention constants (`ATTESTOR_ID`, `TRUST_DIMENSION_PREFIX`) live in
`ComplianceDimension` in eidos-api — string constants with no dependency on
`casehub-ledger-api`. This ensures API-only consumers (casehub-engine) can
construct `LedgerAttestation` records using the correct conventions via their
own `casehub-ledger-api` dependency, without pulling eidos runtime.

**Convenience utility (eidos runtime):**

```java
public final class ComplianceAttestations {
    public static LedgerAttestation violation(
        UUID ledgerEntryId, UUID subjectId,
        String capabilityTag, String dimension,
        String evidence, double dimensionScore) { ... }

    public static LedgerAttestation compliance(
        UUID ledgerEntryId, UUID subjectId,
        String capabilityTag, String dimension,
        double dimensionScore) { ... }
}
```

Lives in runtime (depends on `casehub-ledger-api`), not api. Constructs
attestation with standard fields using the `ComplianceDimension` constants.
Consumer saves via `LedgerAttestationRepository` — eidos doesn't own the save.

Consumers with only an eidos-api dependency (e.g. casehub-engine) use the
`ComplianceDimension` constants to construct `LedgerAttestation` directly —
the construction is trivial field assignment.

### 5. Module Placement Summary

**eidos-api (Tier 1, pure Java):**
- `BehavioralSignal` — enum (replaces `SpecializationSignal`)
- `BehavioralSignalStore` — SPI (replaces `CapabilitySpecializationStore`)
- `ComplianceDimension` — dimension key constants + attestation convention
  constants (`ATTESTOR_ID`, `TRUST_DIMENSION_PREFIX`, `LATENCY_VIOLATION_MULTIPLIER`)
- `BehavioralExpectations` — static utility (`latencyBound`)
- `CapabilityStatus.BehavioralViolation` — new sealed variant

**eidos runtime:**
- `JpaBehavioralSignalStore` — JPA implementation
- `NoOpBehavioralSignalStore` — @DefaultBean
- `BehavioralSignalEntity`, `BehavioralSignalId` — JPA entities
- `ComplianceAttestations` — attestation construction utility
- `DefaultCapabilityHealth` — Step 6 added
- `EidosPreferenceKeys` — `COMPLIANCE_VIOLATION_THRESHOLD` added

**eidos-memory:**
- `InMemoryBehavioralSignalStore` — @Alternative @Priority(1)

### 6. Cross-Repo Impact (separate issues)

Engine import updates and compliance observation recording are follow-up issues.
This issue delivers the eidos framework; engine adopts independently.

**Follow-up issues:**
- eidos#87 — Delegation and escalation compliance dimension design
- eidos#88 — Aggregate behavioral violation threshold
- eidos#89 — Engine integration (CapabilityStatus handling, observation
  recording, PLATFORM.md update)
- eidos#90 — ARC42STORIES.MD update

## What This Design Does NOT Do

- Does not observe agent behavior — consumers do
- Does not enforce behavior at execution time — that's engine's responsibility
- Does not require LLM judges — observations are platform-measurable
- Does not add fields to AgentDescriptor — the descriptor IS the contract
- Does not create new ledger entry types — uses existing LedgerAttestation
- Does not replace trust scoring — feeds into it via the attestation bridge
- Does not require trust scoring to function — the fast path (store → probe)
  works independently

## Rationale

**Why rename, not add a parallel store:** The underlying data model is
identical — (agent, capability, qualifier, signal type) → count with TTL.
Two stores with the same shape is unnecessary duplication. The rename makes
the actual purpose explicit: the store was always about behavioral signals.

**Why `qualifier` not `domain`:** The parameter serves two distinct semantic
roles: task domain (for DECLINE/SUCCESS) and compliance dimension key (for
COMPLIANT/VIOLATED). `domain` only fits the first. `qualifier` honestly
communicates the dual-purpose nature. `ProbeContext.taskDomain()` is unchanged
— it describes the probe context, not the store parameter.

**Why VIOLATED signals, not continuous scores:** Probe() needs a threshold
decision: "has this agent violated enough to affect routing?" A count of
discrete violations is cleaner than aggregating continuous scores. Continuous
quality data belongs in the trust scoring layer via attestations.

**Why compliance is Step 6 (last before Ready):** Behavioral violations are
a softer signal than degradation or unavailability. An agent with violations
might still be the best option when no other agent is available. Checking
compliance last lets consumers distinguish "this agent has behavioral issues
but is available" from "this agent is down."

**Why expectations are resolved, not stored:** Three stakeholders set
expectations (agent developer, platform operator, tenant) and they change at
different rates. Storing them would create synchronization problems. Resolving
at observation time always uses the current expectations.
