---
layout: post
title: "When the spec is wrong about wiring"
date: 2026-09-05
entry_type: note
subtype: diary
projects: [casehubio/eidos]
tags: [capability-health, capacity, cdi, architecture]
---

The cross-platform capacity redistribution spec puts `ActorCapacityView` on `SelectionContext` — the caller passes it per selection call, the selector forwards it to the probe. Clean on paper. Wrong in practice.

`SelectionContext` lives in eidos-api, which is Tier 1: pure Java, no platform-api dependency. `ActorCapacityView` lives in platform-api. Adding that field would be the first platform-api type to leak into the API module — every consumer of eidos-api would transitively inherit platform capacity types whether they use capacity or not.

The spec was written before the platform types were finalised. It assumed a level of coupling that the actual module boundaries don't permit. The fix was to ask a different question: is `ActorCapacityView` per-call context (like `taskDomain`), or is it singleton infrastructure (like `AgentStateStore`)?

It's obviously the latter. There's one aggregated capacity view per deployment. Every call to `aggregatedPressure("agent-1")` returns the same answer at the same instant regardless of who asked. That makes it a CDI bean, not a method parameter. `DefaultCapabilityHealth` already injects four infrastructure dependencies this way — `AgentStateStore`, `BehavioralSignalStore`, `Instance<PreferenceProvider>`, `VocabularyRegistry`. Adding `Instance<ActorCapacityView>` follows the pattern exactly. Unsatisfied Instance means no capacity check — backward compatible without a nullable constructor parameter.

The other spec departure was probe chain position. The spec positioned the Overloaded check after BehavioralViolation (step 6 of 7). The probe returns on first match, so an agent with both behavioral violations *and* capacity overload would be reported as BehavioralViolation and stay in the selection pool. That's wrong — an overloaded agent can't take more work regardless of its behavioural record.

The probe chain has a natural grouping that the original design missed:

```
Agent-level runtime checks:
  1. Degraded    (from AgentStateStore)
  2. Overloaded  (from ActorCapacityView)

Capability-level quality checks:
  3. Unavailable
  4–5. Excluded (declared / learned)
  6. EpistemicallyWeak
  7. BehavioralViolation
  8. Ready
```

Both Degraded and Overloaded answer the same question: can this agent work right now? One is pre-recorded state, the other is live measurement. They belong together, before any capability-specific analysis begins.

The implementation itself was small — a new `Overloaded(pressure, threshold)` sealed variant, seven lines of probe logic with three guards, one switch case in `EngineAwareAgentSelector`, and `SimpleAgentSelector` filtering the new status implicitly through its existing positive match. The design work was the interesting part: recognising that the spec's wiring choice violated a tier boundary that every other type in the module respects, and that the spec's probe position conflated agent-level operational state with capability-level quality assessment.

Batch 2 of the capacity redistribution architecture is done. Batch 3 (qhorus signal sources and redistribution executor) can now exclude overloaded agents from `role:X` routing without any eidos changes — the probe fires automatically when an `ActorCapacityView` bean is deployed.
