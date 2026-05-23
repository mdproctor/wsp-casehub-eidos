---
layout: post
title: "Closing the Loop on Dispatch"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [eidos]
tags: [agent-identity, capability-health, engine]
---

I wanted to understand how casehub-engine would call `CapabilityHealth.probe()`
at dispatch time. The design question is real: `WorkOrchestrator` needs an
`AgentDescriptor` to call probe, but the engine's `Worker` model carries no
such thing.

## Where the Descriptor Lives

Two options. Attach an optional `AgentDescriptor` directly to `Worker` — set
at construction, accessible via `worker.agentDescriptor()`, guarded by
`hasDescriptor()`. Or inject `AgentRegistry` into `WorkOrchestrator` and look
up by worker name at dispatch time.

The second falls apart quickly. Worker names aren't guaranteed unique across
tenancies. The lookup adds registry I/O on the dispatch path. And
`AgentDescriptor` already carries the `tenancyId` the probe needs — a registry
lookup would have to pass tenancy separately anyway. Option A is the right home
for the descriptor, and workers without one skip the probe entirely. Non-agent
workers are assumed capable by construction.

The `casehub-engine-api → casehub-eidos-api` dependency is optional, and the
engine provides a `NoOpCapabilityHealth @DefaultBean` — deployments without
eidos get no filtering.

## Two Strings, One Silent Failure

The probe signature is `probe(AgentDescriptor, capabilityTag, ProbeContext)`.
The two string parameters look equivalent but aren't. `capabilityTag` is the
capability being checked (`"code-review"`). `ProbeContext.taskDomain` is the
subject domain of the task within that capability (`"rust"`, `"contracts"`,
`"python"`).

Pass `capabilityTag` as `taskDomain` and `DefaultCapabilityHealth` calls
`epistemicDomains.get("code-review")`. The map keys are subject domains, not
capability names. `EpistemicallyWeak` never fires. No error, no warning.

The engine doesn't yet carry a subject domain concept — it knows which
capability it needs, not what the task is about. So epistemic filtering stays
dormant until the dispatch context gets richer. The probe design is right; the
calling layer isn't there yet.

One question still open: `EpistemicallyWeak` is currently treated as a hard
filter alongside `Unavailable`. If three workers all pass the capability match
but score low on domain confidence, all three get excluded and the orchestrator
has zero candidates. Probably wrong. Weak workers should be deprioritised, not
gated — ranked below confident ones but used as fallback when nothing better is
available.

PLATFORM.md was missing the cross-repo dependency row for the new engine
optional dep — the build-order "optionally" comment doesn't count as a tracked
dependency for propagation impact analysis. The `CapabilityHealth` ownership
entry said nothing about engine dispatch or the ProbeContext distinction. There
was no boundary rule, and no `docs/repos/casehub-eidos.md` deep-dive despite
CLAUDE.md already referencing the URL. We wrote the deep-dive and closed all
four gaps.
