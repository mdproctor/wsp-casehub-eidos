# Behavioral Contracts Follow-ups — Design Spec

**Date:** 2026-07-02
**Issues:** #83, #86, #88, #90, #91, engine#631, parent#338
**Branch:** issue-88-behavioral-contracts-followups

---

## Scope

Seven issues from the eidos#85 behavioral contracts design review and code review, handled in one branch. Two require design (#88, #86); five are mechanical.

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

### Preference

New `PreferenceKey<AggregateViolationThresholdPreference>` in `EidosPreferenceKeys`:

```java
public static final PreferenceKey<AggregateViolationThresholdPreference> AGGREGATE_VIOLATION_THRESHOLD =
    new PreferenceKey<>("casehub.eidos", "behavioral.aggregate-violation-threshold",
                        new AggregateViolationThresholdPreference(5),
                        s -> new AggregateViolationThresholdPreference(Integer.parseInt(s)));
```

Default: 5. Resolved per-tenancy via `PreferenceProvider`.

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

Replace `InMemoryAgentRegistry.matchesCapability()` body with `CapabilityResolver.match()`. The method duplicates identical logic: exact name check → vocabulary grounding check → `registry.match()`. Single-line change.

Reactive registry delegates to blocking, inherits fix automatically.

---

## #91 — Minor Cleanup

1. Rename `StubSpecializationStore` → `StubBehavioralSignalStore` in `DefaultCapabilityHealthExclusionTest`
2. Rename `NoOpSpecializationStore` → `NoOpBehavioralSignalStore` in `DefaultCapabilityHealthDegradedTest`
3. Rename field `specializationStore` → `signalStore` in `DefaultCapabilityHealthExclusionTest`
4. Replace FQ `java.util.LinkedHashMap` with import in `DefaultCapabilityHealth.java:95`
5. Add validation tests for `ComplianceViolationThresholdPreference(0)` and `ExcludeThresholdPreference(0)` throwing

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

## Platform Coherence

- `ViolationKind` in api/ (Tier 1, pure Java) — correct
- `AggregateViolationThresholdPreference` in runtime/preferences — follows established pattern
- Breaking `BehavioralViolation` record change — zero downstream consumers
- ADR for #86 aligns with PP-20260611-228599
- No deferred concerns requiring new issues
