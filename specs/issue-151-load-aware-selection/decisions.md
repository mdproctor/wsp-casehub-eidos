# Decisions — Issue #151: Load-Aware Agent Selection

## D1: Capacity view wiring and probe chain architecture

**Choice:** CDI injection of `Instance<ActorCapacityView>` in `DefaultCapabilityHealth`, with new `Overloaded` probe step positioned after Degraded (step 2), before Unavailable.

**Alternatives:**
- `SelectionContext.capacityView` (design spec approach) — gives per-call caller control but breaks eidos-api tier boundary (Tier 1 pure Java would gain platform-api dependency); `ActorCapacityView` is a singleton infrastructure component, not per-call context, making `SelectionContext` the wrong carrier
- Thin eidos-local `@FunctionalInterface` on `SelectionContext` — avoids the dep but creates a parallel type hierarchy wrapping the canonical platform type; indirection for no gain
- Probe position after BehavioralViolation (design spec) — agent with both behavioral violations AND overloaded would be reported as BehavioralViolation (kept in pool); wrong because overloaded agents can't take work regardless of behavioral record

**Rationale:** Every other singleton infrastructure component in `DefaultCapabilityHealth` is CDI-injected (`AgentStateStore`, `BehavioralSignalStore`, `Instance<PreferenceProvider>`, `VocabularyRegistry`). `ActorCapacityView` is categorically identical — singleton, same answer for all callers at the same instant. The probe chain groups agent-level runtime checks (Degraded, Overloaded) before capability-level quality checks (Unavailable through BehavioralViolation). Deployment-level control via `Instance<>` (unsatisfied = no capacity check) handles backward compatibility without per-call toggling.

**Trade-offs:** No per-call capacity toggle — a caller cannot say "ignore capacity for this selection." No production use case exists; deployment-level control (CDI bean presence) handles all real scenarios. A future `ProbeOptions` mechanism could add per-call toggles if needed.

**Sources:**
- `DefaultCapabilityHealth.java:29-42` — existing CDI injection pattern (Instance<PreferenceProvider>)
- `SelectionContext.java` — current Tier 1 record in eidos-api, no platform-api dependency
- `CapabilityHealth.java:14-32` — sealed CapabilityStatus with 6 variants
- `ActorCapacityView.class` (platform-api JAR) — singleton interface: aggregatedPressure(actorId) → CapacitySignal
- Cross-platform capacity redistribution spec (qhorus#405) — Batch 2 scope definition
- Trust-ranked selection spec (#144) — established SelectionContext and AgentSelector patterns

**Exploration:** deep-analysis
**Status:** captured

### Sub-decisions within D1

**D1a: New `CapabilityStatus.Overloaded(double pressure, double threshold)` variant**
- Added to sealed interface in eidos-api (pure Java, no new dependency)
- Selectors filter it out: `SimpleAgentSelector` positive filter misses it (implicit exclude); `EngineAwareAgentSelector.mapHealth()` maps to `null` (explicit exclude)

**D1b: Probe position — after Degraded (step 2), before Unavailable**
- Agent-level runtime checks (Degraded, Overloaded) grouped before capability-level quality checks
- Matches the "agent can't work right now" vs "capability has quality concerns" distinction

**D1c: Global config `casehub.eidos.health.capacity-threshold` (default 0.8)**
- Follows `casehub.eidos.epistemic.weak-threshold` naming pattern (health probe parameter)
- No per-tenancy PreferenceProvider for now — capacity limits are infrastructure-level, not business-level
- Per-tenancy can be added later when a real use case emerges

**D1d: `Instance<ActorCapacityView>` injection pattern (not `@DefaultBean` no-op)**
- Follows `Instance<PreferenceProvider>` pattern, not `AgentStateStore` pattern
- `ActorCapacityView` is a platform SPI — eidos should not provide a `@DefaultBean` for it
- Unsatisfied Instance = no capacity check (natural "not deployed" state)

**D1e: `agentId` ↔ `actorId` identity assumption**
- `ActorCapacityView.aggregatedPressure(actorId)` called with `descriptor.agentId()`
- Established platform convention: agents are actors, same identifier space
- Documented assumption, not a design choice
