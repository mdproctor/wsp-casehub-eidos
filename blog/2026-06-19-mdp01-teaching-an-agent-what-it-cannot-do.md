---
layout: post
title: "Teaching an Agent What It Cannot Do"
date: 2026-06-19
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [casehub, eidos, capability-health, specialization, design]
series: issue-55-capability-specialization-metadata
---

The probe was working backwards. Eidos could tell casehub-engine whether an agent was ready, degraded, overloaded, or epistemically weak in a domain. What it couldn't say was: *this agent has declined Rust security reviews three times and you should stop sending them*. That gap is what eidos#55 was about.

The design that came out of a long brainstorming session: two new things on `AgentCapability`. First, an `excludedDomains: Set<String>` — a declared negative specialization, what the admin knows at registration time ("not Rust, not Go"). Second, a `CapabilitySpecializationStore` SPI for the learned side — patterns from DECLINE attestations accumulating over time, managed by casehub-ledger/CBR once that integration exists. The probe order became: Degraded → Unavailable → Excluded(DECLARED) → Excluded(LEARNED) → EpistemicallyWeak → Ready. Exclusions take priority over epistemic weakness because an explicit exclusion is a stronger signal than a low-confidence score.

The threshold for converting accumulated declines into an exclusion needed a home. My initial instinct was `@ConfigProperty` — flat, deploy-wide. But that's wrong for a routing *policy*. A high-stakes tenancy should be able to set threshold=1 (never misroute); a general-purpose tenancy might tolerate threshold=5 for noise. That's exactly what `PreferenceProvider` exists for, with its ancestor-chain resolution per tenancyId. We wired `Instance<PreferenceProvider>` with an `isUnsatisfied()` fallback — no `@DefaultBean` in eidos, because two `@DefaultBean` implementations of the same platform-owned type collide at Quarkus augmentation. That collision was a specific gotcha we'd already hit before; avoiding it required `Instance<T>` rather than the simpler `@DefaultBean` approach.

The `InMemoryCapabilitySpecializationStore` uses a nested map structure: `ConcurrentHashMap<AgentCapKey, ConcurrentHashMap<String, ConcurrentLinkedQueue<Instant>>>`. Each DECLINE adds an expiry timestamp to the queue for that agent/tenancy/capability/domain combination. The TTL lives in the store — `@ConfigProperty decline-ttl-days=30` — not on the SPI signature. I thought about making it event-specific (the way `AgentStateStore.record()` takes an `Instant expiresAt` because rate-limit retry-after headers give you the exact window), but decline patterns aren't event-specific. How long a routing policy should remember them is eidos's concern, not the ledger's.

One thing we missed: `AgentDescriptorMapper.toCapability()` was migrated to the builder as part of the bulk call-site cleanup, but the protocols say JPA mappers must use the positional constructor — the compiler enforces field completeness at that boundary, and the builder silently defaults omitted fields to null. We caught it in the protocol sweep rather than in the code review. I filed eidos#63 to fix it. The immediate risk is managed — `excludedDomains` was explicitly mapped — but the structural problem stands.

The A2A_CARD renderer now includes `excludedDomains` in the capability JSON when populated. MARKDOWN and PROSE formats don't get it — those are for LLMs, not for dispatch routing. The A2A_CARD format is for machine consumption, and an engine making routing decisions needs to know what a capability explicitly cannot do, not just what it does well.

Three deferred issues came out of this: eidos#61 for `AgentQuery` domain filtering at the registry level (currently the engine calls `find()` then probes per candidate — efficient enough for now), eidos#62 for a JPA-backed `CapabilitySpecializationStore` when ephemeral in-memory isn't sufficient, and parent#281 for syncing PLATFORM.md once the learned exclusion story is complete.
