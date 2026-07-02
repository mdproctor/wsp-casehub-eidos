# Behavioral Contracts Follow-ups — Design Spec

**Date:** 2026-07-02
**Issues:** #83, #86, #88, #90, #91, engine#631, parent#338
**Branch:** issue-88-behavioral-contracts-followups

---

## Scope

Seven of the follow-up issues from the eidos#85 behavioral contracts design review and code review, handled in one branch. Two require design (#88, #86); five are mechanical. Two sibling issues (#87, #89) are independent design efforts tracked separately — see Related Issues.

---

## #88 — Aggregate Behavioral Violation Threshold

### Problem

Per-dimension violation checking has a blind spot: an agent with 2 violations in each of 10 dimensions (20 total, all below per-dimension threshold of 3) appears fully compliant. This "death by a thousand cuts" pattern is undetectable with per-dimension-only checking.

### Design

**Total violation count** as a secondary safety valve. Sum all unexpired VIOLATED counts across all compliance dimensions; compare to an aggregate threshold.

The aggregate check only fires when per-dimension hasn't already caught a spike (per-dimension runs first in probe order). In that context, every dimension in the violations map has count < per-dimension threshold, so summing counts directly measures "total behavioral drift across all dimensions, below the per-dimension radar."

**Probe Step 6 order:**
1. Query `signalStore.learned()` for all VIOLATED dimension counts (existing)
2. Per-dimension check: filter to dimensions ≥ per-dimension threshold → `BehavioralViolation(PER_DIMENSION)` (existing logic)
3. If per-dimension didn't fire: sum all counts → compare to aggregate threshold → `BehavioralViolation(AGGREGATE)` (new)

### API Change — ViolationKind discriminator

`CapabilityStatus.BehavioralViolation` gains a `ViolationKind` field:

```java
record BehavioralViolation(Map<String, Integer> violations, ViolationKind kind) implements CapabilityStatus {
    enum ViolationKind { PER_DIMENSION, AGGREGATE }
}
```

- **PER_DIMENSION:** `violations` contains only dimensions ≥ per-dimension threshold (current behavior).
- **AGGREGATE:** `violations` contains ALL dimensions with any violations (full picture).

Breaking change to record constructor. No external consumers — engine doesn't reference `BehavioralViolation`.

**Downstream note:** Engine issue #89 contains routing guidance that predates ViolationKind: "The `violations` map provides all violated dimensions." This is now only accurate for `AGGREGATE`. For `PER_DIMENSION`, the map contains only dimensions exceeding the per-dimension threshold. Issue #89's implementation must handle both ViolationKind semantics when building dimension-aware routing fallback.

### Preference

New `PreferenceKey<AggregateViolationThresholdPreference>` in `EidosPreferenceKeys`:

```java
public static final PreferenceKey<AggregateViolationThresholdPreference> AGGREGATE_VIOLATION_THRESHOLD =
    new PreferenceKey<>("casehub.eidos", "behavioral.aggregate-violation-threshold",
                        new AggregateViolationThresholdPreference(5),
                        s -> new AggregateViolationThresholdPreference(Integer.parseInt(s)));
```

Default: 5. Resolved per-tenancy via `PreferenceProvider`.

**Default rationale:** The aggregate fires only when per-dimension hasn't — every dimension's count is below the per-dimension threshold (default 3, so max individual count = 2). With aggregate threshold 5, at least ⌈5/2⌉ = 3 dimensions must have violations to trigger. The invariant: **the aggregate detects broad behavioral drift across multiple dimensions, not spikes in any single dimension.**

**Co-tuning guidance:** When the aggregate fires, every dimension has at most `per_dimension − 1` violations. The minimum dimensions to trigger is `⌈aggregate / (per_dimension − 1)⌉`. To ensure at least K dimensions must contribute, set:

`aggregate ≥ (K − 1) × (per_dimension − 1) + 1`

For K = 3 (at least 3 dimensions): `aggregate ≥ 2 × (per_dimension − 1) + 1`. At the default per-dimension = 3, this gives aggregate ≥ 5.

| Per-dimension | Aggregate | Min dimensions | Formula | Character |
|---------------|-----------|----------------|---------|-----------|
| 3 | 5 (default) | ⌈5/2⌉ = 3 | 2(2)+1 | Standard: 3-dimension breadth |
| 3 | 8 | ⌈8/2⌉ = 4 | 3(2)+1 | Conservative: 4+ dimensions |
| 5 | 9 | ⌈9/4⌉ = 3 | 2(4)+1 | Higher per-dim, same 3-dim breadth |

`AggregateViolationThresholdPreference` record in `runtime/preferences/`, same pattern as `ComplianceViolationThresholdPreference`. Validation: value >= 1.

### Files Changed

| File | Change |
|------|--------|
| `api/.../CapabilityHealth.java` | Add `ViolationKind` enum to `BehavioralViolation` record |
| `runtime/.../DefaultCapabilityHealth.java` | Add aggregate check after per-dimension in Step 6; add `aggregateViolationThreshold()` method |
| `runtime/.../preferences/EidosPreferenceKeys.java` | Add `AGGREGATE_VIOLATION_THRESHOLD` key |
| `runtime/.../preferences/AggregateViolationThresholdPreference.java` | New record |
| Test files | Update all `BehavioralViolation` constructions to include `ViolationKind`; add aggregate-specific tests |

---

## #86 — Compliance Status in A2A_CARD: Decision is No

Recorded as ADR 0006. Rationale summary:

1. **Category mismatch** — A2A_CARD is descriptor-derived (static). Compliance is runtime state. Different lifecycles.
2. **Caching breaks** — A2A_CARD uses fingerprint caching. Dynamic compliance state defeats it.
3. **Redundant** — `probe()` already returns compliance status at dispatch time.
4. **Separation of concerns** — Card describes what an agent IS. Compliance describes how an agent BEHAVES.
5. **Architecture alignment** — Engine coordinates agents; agents don't self-route via A2A_CARD.

---

## #83 — CapabilityResolver Unification

Replace `InMemoryAgentRegistry.matchesCapability()` body with a delegation to `CapabilityResolver.match()`.

The methods share the same algorithm (exact name match → vocabulary guard → `registry.match()`) but differ in two concrete ways:

1. **Vocabulary guard divergence:** `CapabilityResolver.match()` checks `capabilityVocabulary().isBlank()` (string content). `matchesCapability()` checks `!vocabularyRegistry.isResolvable()` (CDI bean availability). The CDI availability check moves to the call site as a pre-guard.

2. **CDI Instance resolution:** `InMemoryAgentRegistry` holds `Instance<VocabularyRegistry>` requiring `.isResolvable()` guard and `.get()` resolution before calling the static `CapabilityResolver.match()`.

**Latent bug fix:** `matchesCapability()` does not check `isBlank()` on the vocabulary string — a capability with a blank (non-null, whitespace-only) vocabulary passes through to `registry.match()` with a blank URI. `CapabilityResolver.match()` correctly returns `MatchDegree.None` for blank vocabularies. The unification fixes this.

**Replacement body (~4 lines):**

```java
private boolean matchesCapability(AgentCapability capability, String requestedName) {
    if (!vocabularyRegistry.isResolvable()) {
        return capability.name().equals(requestedName);
    }
    return !(CapabilityResolver.match(capability, requestedName,
                                      vocabularyRegistry.get()) instanceof MatchDegree.None);
}
```

Reactive registry delegates to blocking, inherits fix automatically.

---

## #91 — Minor Cleanup

1. Rename `StubSpecializationStore` → `StubBehavioralSignalStore` in `DefaultCapabilityHealthExclusionTest`
2. Rename `NoOpSpecializationStore` → `NoOpBehavioralSignalStore` in `DefaultCapabilityHealthDegradedTest`
3. Rename field `specializationStore` → `signalStore` in `DefaultCapabilityHealthExclusionTest`
4. Replace FQ `java.util.LinkedHashMap` with import in `DefaultCapabilityHealth.java:95`
5. Add validation tests for `ComplianceViolationThresholdPreference(0)` and `ExcludeThresholdPreference(0)` throwing
6. **Item 4 — SOUND evidence convention affirmed:** `ComplianceAttestations.compliance()` hardcodes `null` evidence for SOUND verdicts. This is the correct convention: SOUND means "normal operation observed" — `dimensionScore` captures the quantitative measurement. No evidence string is needed. If positive compliance observations need structured evidence in future, adding an optional `evidence` parameter is a mechanical API change, not a design concern.

---

## #90 — ARC42STORIES.MD Update

Document behavioral contracts additions: BehavioralSignal, BehavioralSignalStore, ComplianceDimension, BehavioralExpectations, ComplianceAttestations, CapabilityStatus.BehavioralViolation, ViolationKind. Update probe scenario to show Step 6. Update glossary.

---

## engine#631 — Close as Not Applicable

Neither `CapabilitySpecializationStore` nor `SpecializationSignal` exist in the engine codebase. The engine never imported these types. No-op.

---

## parent#338 — PLATFORM.md Sync

Update `docs/PLATFORM.md` and `docs/repos/casehub-eidos.md`:
- `CapabilitySpecializationStore` → `BehavioralSignalStore`
- `SpecializationSignal { DECLINE, SUCCESS }` → `BehavioralSignal { DECLINE, SUCCESS, COMPLIANT, VIOLATED }`
- Add capability entries: ComplianceDimension, BehavioralExpectations, ComplianceAttestations
- Update probe step description to include Step 6

---

## Implementation Order

1. **#83** — CapabilityResolver unification (no dependencies)
2. **#91** — Cleanup (no dependencies, but do after #83 since tests touch same area)
3. **#88** — Aggregate threshold (depends on clean test infrastructure from #91)
4. **#86** — ADR (no code changes, just write ADR)
5. **#90** — ARC42STORIES.MD (do last — documents everything above including #88)
6. **parent#338** — PLATFORM.md sync (cross-repo, after eidos work is complete)
7. **engine#631** — Close issue (no code changes)

## Related Issues

Open issues from the same eidos#85 review, not addressed by this spec:

- **#87** — Delegation and escalation compliance dimension design. Requires observation window semantics (delegation) and vocabulary-aware interpretation (escalation). Independent design effort.
- **#89** — Engine integration with behavioral contracts (CapabilityStatus handling, observation recording). Directly affected by the ViolationKind change in #88 — see §#88 API Change downstream note.

## Platform Coherence

- `ViolationKind` in api/ (Tier 1, pure Java) — correct
- `AggregateViolationThresholdPreference` in runtime/preferences — follows established pattern
- Breaking `BehavioralViolation` record change — zero downstream consumers
- ADR for #86 aligns with PP-20260611-228599
- No new deferred concerns — sibling issues #87 and #89 are pre-existing (see Related Issues)
