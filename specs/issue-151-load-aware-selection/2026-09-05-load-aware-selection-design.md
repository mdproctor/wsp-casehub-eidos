# Load-Aware Agent Selection — Design Spec

**Issue:** casehubio/eidos#151
**Date:** 2026-09-05
**Scope:** S — one new sealed variant, one probe step, one config property, exhaustive switch updates
**Part of:** Cross-platform capacity redistribution (qhorus#405, Batch 2)

---

## Problem

Eidos agent selection (RAS) picks agents by trust and behavioral health but is load-blind. An agent at 95% context pressure or maximum task count is selected just as eagerly as an idle one. The cross-platform capacity redistribution architecture (qhorus#405) established a shared vocabulary for actor load (`ActorCapacityView` in platform-api, landed via platform#268). Eidos needs to consume this signal in its health probe chain so overloaded agents are excluded from selection.

## Solution

Add an `Overloaded` probe step to `DefaultCapabilityHealth` via CDI injection of `Instance<ActorCapacityView>`. No changes to `SelectionContext` or `ProbeContext` — capacity is singleton infrastructure, not per-call context.

```
DefaultCapabilityHealth probe chain (updated):

  1. Degraded          (from AgentStateStore — recorded operational state)
  2. Overloaded        (from ActorCapacityView — live capacity signal)  ← NEW
  3. Unavailable       (capability not declared)
  4. Excluded-DECLARED (domain excluded)
  5. Excluded-LEARNED  (domain learned exclusion)
  6. EpistemicallyWeak (low domain confidence)
  7. BehavioralViolation (compliance violations)
  8. Ready
```

### Why CDI injection (not SelectionContext)

The cross-platform design spec puts `capacityView` on `SelectionContext`. This spec departs from that approach for three reasons:

1. **Tier boundary.** `SelectionContext` is in eidos-api (Tier 1, pure Java). `ActorCapacityView` is in platform-api. Adding it would be the first platform-api type in eidos-api, breaking the established tier purity.

2. **Architectural consistency.** Every singleton infrastructure component in `DefaultCapabilityHealth` is CDI-injected: `AgentStateStore`, `BehavioralSignalStore`, `Instance<PreferenceProvider>`, `VocabularyRegistry`. `ActorCapacityView` is categorically identical — singleton, same answer for all callers at the same instant. Passing it through parameters would be an anomaly.

3. **No per-call use case.** The nullable `capacityView` on `SelectionContext` enables per-call toggling. No production use case exists for ignoring capacity on specific calls. Deployment-level control via `Instance<>` (unsatisfied = no capacity check) handles all real scenarios.

The behavioral intent is unchanged: overloaded agents are excluded from selection.

### Why probe position after Degraded (not after BehavioralViolation)

The design spec positions Overloaded after BehavioralViolation (step 6). The probe returns on first match — an agent with BehavioralViolation AND overloaded would be reported as BehavioralViolation and kept in the selection pool. An overloaded agent can't take more work regardless of its behavioral record.

The correct grouping: agent-level runtime checks (Degraded, Overloaded) before capability-level quality checks (Unavailable through BehavioralViolation).

---

## API Changes — eidos-api

### CapabilityStatus — new Overloaded variant

```java
sealed interface CapabilityStatus permits
        CapabilityStatus.Degraded,
        CapabilityStatus.Overloaded,         // NEW — after Degraded in probe chain
        CapabilityStatus.Unavailable,
        CapabilityStatus.Excluded,
        CapabilityStatus.EpistemicallyWeak,
        CapabilityStatus.BehavioralViolation,
        CapabilityStatus.Ready {

    // ... existing variants unchanged ...

    record Overloaded(double pressure, double threshold) implements CapabilityStatus {}
}
```

- `pressure` — current aggregated pressure from `ActorCapacityView` (0.0–1.0)
- `threshold` — the configured threshold that was exceeded (diagnostic context)

No other eidos-api changes. `SelectionContext`, `ProbeContext`, `CapabilityHealth` interface, and eidos-api dependencies are all unchanged.

---

## Runtime Changes — eidos-runtime

### DefaultCapabilityHealth

```java
@DefaultBean
@ApplicationScoped
public class DefaultCapabilityHealth implements CapabilityHealth {

    private final double weakThreshold;
    private final double capacityThreshold;           // NEW
    private final AgentStateStore stateStore;
    private final BehavioralSignalStore signalStore;
    private final Instance<PreferenceProvider> preferenceProviderInstance;
    private final Instance<ActorCapacityView> capacityViewInstance;  // NEW
    private final VocabularyRegistry vocabularyRegistry;

    @Inject
    public DefaultCapabilityHealth(
            @ConfigProperty(name = "casehub.eidos.epistemic.weak-threshold",
                            defaultValue = "0.3")
            final double weakThreshold,
            @ConfigProperty(name = "casehub.eidos.health.capacity-threshold",
                            defaultValue = "0.8")
            final double capacityThreshold,            // NEW
            final AgentStateStore stateStore,
            final BehavioralSignalStore signalStore,
            final Instance<PreferenceProvider> preferenceProviderInstance,
            final Instance<ActorCapacityView> capacityViewInstance,  // NEW
            final VocabularyRegistry vocabularyRegistry) {
        // ...
    }
```

**Probe step 2 (new — after Degraded, before Unavailable):**

```java
// Step 2: capacity overload — live signal from ActorCapacityView
if (capacityViewInstance.isResolvable()) {
    var signal = capacityViewInstance.get()
        .aggregatedPressure(descriptor.agentId());
    if (signal != null && signal.pressure() >= capacityThreshold) {
        return new CapabilityStatus.Overloaded(
            signal.pressure(), capacityThreshold);
    }
}
```

**Guard chain:**
1. `capacityViewInstance.isResolvable()` — false when no `ActorCapacityView` bean is deployed (backward compat)
2. `signal != null` — null when no capacity signals exist for this agent (new agent, no data)
3. `signal.pressure() >= capacityThreshold` — only exclude when at or above threshold

All three guards must pass before returning `Overloaded`. Any guard failing → falls through to step 3 (Unavailable check).

**Identity assumption:** `descriptor.agentId()` is passed to `aggregatedPressure(actorId)`. This relies on the established platform convention that eidos agent IDs and platform actor IDs are the same identifier space.

### Test constructor

The test constructor gains `capacityThreshold` and `capacityViewInstance` parameters to match the CDI constructor.

---

## Selector Changes

### SimpleAgentSelector — no code change needed

`filterHealthy()` uses a positive filter:

```java
if (status instanceof CapabilityStatus.Ready
    || status instanceof CapabilityStatus.Degraded
    || status instanceof CapabilityStatus.EpistemicallyWeak
    || status instanceof CapabilityStatus.BehavioralViolation) {
    result.add(match);
}
```

`Overloaded` does not match any of these → implicitly filtered out.

### EngineAwareAgentSelector — exhaustive switch update

```java
private AgentHealth mapHealth(CapabilityStatus status) {
    return switch (status) {
        case CapabilityStatus.Ready r -> AgentHealth.READY;
        case CapabilityStatus.Degraded d -> AgentHealth.DEGRADED;
        case CapabilityStatus.EpistemicallyWeak ew -> AgentHealth.EPISTEMICALLY_WEAK;
        case CapabilityStatus.BehavioralViolation bv -> AgentHealth.BEHAVIORAL_VIOLATION;
        case CapabilityStatus.Overloaded o -> null;   // NEW — filter out
        case CapabilityStatus.Unavailable u -> null;
        case CapabilityStatus.Excluded ex -> null;
    };
}
```

---

## Configuration

| Property | Default | Module | Description |
|---|---|---|---|
| `casehub.eidos.health.capacity-threshold` | `0.8` | runtime | Aggregated pressure at or above this value → `Overloaded` |

Global only — no per-tenancy `PreferenceProvider` override. Capacity limits are infrastructure-level. Per-tenancy can be added when a real use case emerges.

---

## Testing

### DefaultCapabilityHealth — new probe step tests

| Test | Setup | Expected |
|---|---|---|
| No `ActorCapacityView` deployed | `Instance<>` unsatisfied | Falls through to next step (backward compat) |
| `ActorCapacityView` returns null signal | No data for this agent | Falls through to next step |
| Pressure below threshold (0.5 < 0.8) | Signal with pressure 0.5 | Falls through → `Ready` |
| Pressure at threshold (0.8 = 0.8) | Signal with pressure 0.8 | `Overloaded(0.8, 0.8)` |
| Pressure above threshold (0.95 > 0.8) | Signal with pressure 0.95 | `Overloaded(0.95, 0.8)` |
| Custom threshold (0.6) | Config override, pressure 0.7 | `Overloaded(0.7, 0.6)` |
| Degraded AND overloaded | AgentStateStore has entry + high pressure | `Degraded` (step 1 wins — probe short-circuits) |
| Overloaded AND capability unavailable | High pressure + missing capability | `Overloaded` (step 2 wins — excluded before capability check) |

Test infrastructure: stub `ActorCapacityView` returning predetermined `CapacitySignal` values. Use existing test patterns from `DefaultCapabilityHealthTest` and `DefaultCapabilityHealthDegradedTest`.

### EngineAwareAgentSelector — exhaustive switch

| Test | Expected |
|---|---|
| `Overloaded` status from probe | Candidate filtered out (mapHealth returns null) |

### SimpleAgentSelector — implicit filtering

| Test | Expected |
|---|---|
| All candidates overloaded | `NoneQualified("all N candidates unhealthy")` |
| Mix of healthy and overloaded | Overloaded filtered, healthy selected |

### Integration test (examples module)

End-to-end: register agents → inject stub `ActorCapacityView` → one agent overloaded → select → verify overloaded agent excluded, healthy agent selected.

---

## Module Changes Summary

| Module | Change |
|---|---|
| `api/` | `CapabilityStatus` sealed permits list gains `Overloaded`; new `Overloaded` record |
| `runtime/` | `DefaultCapabilityHealth` gains `Instance<ActorCapacityView>` + capacity threshold config + probe step 2 |
| `routing/` | `EngineAwareAgentSelector.mapHealth()` gains `Overloaded → null` case |
| `persistence-memory/` | No changes |
| `vocab/` | No changes |
| `deployment/` | No changes |
| `annotations/` | No changes |
| `examples/` | New integration test for capacity-aware selection |

---

## Flyway

No migrations needed. No schema changes.

---

## Not in Scope

- **`SelectionContext` changes** — capacity is CDI-injected, not caller-passed (see rationale above)
- **Per-tenancy capacity threshold** — global config only for Batch 2; PreferenceProvider override deferred to when a real use case emerges
- **Reactive parity** — `ReactiveCapabilityHealth` does not exist in the codebase; no reactive concern
- **Per-channel capacity threshold** — qhorus concern (Batch 3), not eidos
- **Redistribution execution** — qhorus/engine concern (Batches 3-4)
- **`DegradationReason.OVERLOADED` interaction** — existing enum value is for manually recorded degradation via `AgentStateStore`; the new `Overloaded` status is a separate concept from live capacity measurement, not a reuse of Degraded

---

## References

- `DefaultCapabilityHealth.java:21-144` — existing probe chain and CDI injection patterns
- `CapabilityHealth.java:1-33` — sealed CapabilityStatus interface
- `SelectionContext.java:1-21` — current Tier 1 record (unchanged by this spec)
- `SimpleAgentSelector.java:82-104` — positive health filter (implicit Overloaded exclusion)
- `EngineAwareAgentSelector.java:103-112` — exhaustive switch (requires new case)
- `ActorCapacityView.class` (platform-api 0.2-SNAPSHOT) — `aggregatedPressure(actorId)` → `CapacitySignal`
- `CapacitySignal.class` (platform-api 0.2-SNAPSHOT) — `(actorId, source, pressure[0.0-1.0], timestamp)`
- Cross-platform capacity redistribution spec (`qhorus/specs/cross-platform-capacity-redistribution/2026-09-02-capacity-redistribution-design.md`) — Batch 2 scope, Layer 1 architecture
- Trust-ranked selection spec (`eidos/docs/specs/issue-144-trust-ranked-selection/2026-08-26-trust-ranked-selection-design.md`) — established AgentSelector SPI and SelectionContext
- Phase 2 capability health spec (`eidos/docs/specs/2026-05-23-phase2-capability-health-design.md`) — original probe chain design
- D1 decision record (`decisions.md`) — CDI injection rationale and alternatives analysis
