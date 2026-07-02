---
layout: post
title: "The Tag That Didn't Match"
date: 2026-07-02
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [capability-health, subsumption, bug-fix]
---

## The Tag That Didn't Match

The bug was a one-line fix. The interesting part is what it exposed about where resolution logic belongs in a subsumption-aware system.

`DefaultCapabilityHealth.probe()` resolves a query tag to a declared capability via subsumption matching — `findCapability()` walks the vocabulary hierarchy and returns the agent's actual `AgentCapability`. That resolution worked correctly. But the very next operation used the *query tag* instead of the resolved name:

```java
specializationStore.count(descriptor.agentId(), descriptor.tenancyId(),
    capabilityTag,  // ← the query term, not the declared name
    context.taskDomain(), SpecializationSignal.DECLINE);
```

An agent declaring `clinical-documentation-review` matched via subsumption when you queried `documentation`. The resolution found the right capability. Then the DECLINE lookup checked `documentation` — and every signal recorded against `clinical-documentation-review` was invisible.

This was already in the codebase before cross-vocabulary subsumption, but cross-vocab made it operationally significant. Agents routinely get matched against capability tags from other vocabularies now. A bug that was theoretically wrong became practically dangerous.

### Resolution as a shared concern

The fix on line 72 — `capability.name()` instead of `capabilityTag` — took about thirty seconds. The design question took longer: the same resolution logic was private to `DefaultCapabilityHealth`, and the recording side (callers of `CapabilitySpecializationStore.record()`) needs it too. There are no production callers yet — engine will be the first — but we're defining the contract now, not after someone gets it wrong.

I initially proposed adding a `recordSignal()` method to `CapabilityHealth` — probe and recording share the same resolution invariant, so putting them together felt natural. The design review pushed back on the PLATFORM.md justification I used (I'd over-generalised from one example into a universal "domain SPIs separate read from write" convention), but the single-responsibility argument held on its own: CapabilityHealth is a probe SPI. Its name and interface express "check health status", not "manage health lifecycle."

The answer was a static utility. `CapabilityResolver` lives in the api module (Tier 1, pure Java, no CDI) and exposes two methods: `match()` for a single capability, `resolve()` for best-match-from-a-list. `DefaultCapabilityHealth` now delegates to it. Engine, when it integrates, will use the same utility before calling `store.record()`. One resolution path, shared.

### The duplication that remains

The extraction revealed that subsumption resolution was implemented three independent ways: `DefaultCapabilityHealth.findCapability()` (now replaced), `InMemoryAgentRegistry.matchesCapability()` (still inline), and `JpaAgentRegistry.find()` (SQL-level expansion via `expandForMatchingByVocabulary()` — necessarily different). The in-memory registry can delegate to `CapabilityResolver.match()` — that's #83. The JPA path stays separate; SQL IN-clause expansion is a fundamentally different approach.

There's a deeper gap behind all three: `AgentRegistry.find()` returns `List<AgentDescriptor>` with no match metadata. The caller doesn't know which declared capability matched or at what depth. Engine currently has to re-resolve after querying, which is why the resolution utility matters. That's #84 — return match metadata from `find()` so callers don't need to re-derive it.
