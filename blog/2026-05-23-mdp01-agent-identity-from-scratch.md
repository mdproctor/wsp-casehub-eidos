---
layout: post
title: "Agent Identity from Scratch"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [eidos]
tags: [agent-identity, quarkus, jpa, vocabulary]
---

## The Four-Layer Agent

I wanted a structured way to describe an LLM agent — not a name and a model string, but everything a dispatch system needs to make a good routing decision. The descriptor has four layers: identity (who are you), slot (what role do you fill), capabilities (what can you do, and how well, in which domains), and disposition (how do you behave — strict or flexible, autonomous or directed).

The interesting design tension: slot is deliberately an open string. There's no `SlotType` enum. A devtown deployment might have "planner" and "reviewer" while a clinical deployment has "triage-nurse" and "attending-physician." The platform never constrains — that's what vocabulary registries are for.

## Normalise the Join Table

The first real design argument was how to persist capabilities. I proposed a hybrid — scalar columns for the queryable fields, JSON columns for the nested types like `epistemicDomains`. Claude pushed back when I pointed out that `findByCapability` would need a full table scan with Java-layer filtering. A normalised `agent_capability` table with an indexed `name` column gives us a proper JOIN query from day one.

I was initially resistant to the extra table but the argument is straightforward: the capability name is the query driver. If you're querying on it, it belongs in a column with an index. JSON columns in SQL are appealing until you realise they require DB-specific syntax to get any filtering benefit — PostgreSQL JSONB works, H2 doesn't, and suddenly your tests are testing different code paths than production.

## Tenancy: Always On, Never Optional

The `no-conditional-tenancy-filtering` protocol caught a design gap during the coherence review. The original `AgentRegistry` had `findBySlot(String slot)` — no filter hook for future tenancy or visibility policies. The fix collapsed three query methods into one `find(AgentQuery)` with a mandatory `tenancyId` field that `Objects.requireNonNull` enforces at construction.

Claude's final code review caught a deeper instance of the same problem: `findById(String agentId)` had no tenancy parameter at all. Any caller with a known agent ID could cross tenant boundaries. The fix was the same — add `tenancyId` to the signature, filter unconditionally. Breaking change, but that's the point: every caller is now forced to be explicit about which tenant they're operating in.

## The Probe as a Pure Function

Phase 2's `CapabilityHealth.probe()` originally took a `String agentId`. The implementation would need to look up the descriptor from the registry — which requires a `tenancyId`, which the probe doesn't carry. Adding `tenancyId` to the probe context would tangle a data-isolation concern into a health check.

The better design: pass the `AgentDescriptor` directly. The caller already has it from the registry query. The probe becomes a pure function — descriptor in, status out, no side effects, no registry dependency. It checks whether the agent declares the capability, then whether the epistemic domain confidence meets the configurable threshold. Ready, Unavailable, or EpistemicallyWeak. No runtime state probing yet — that needs infrastructure we don't have.

## Cross-Vocabulary Equivalence

The vocabulary system turned out to be more satisfying than I expected. Three well-known vocabularies — SVO (performer/evaluator/coordinator), CasehubSlot (planner/reviewer/executor/supervisor), and Conscientiousness (12 terms across four disposition axes) — all cross-referenced via `exactMatches` on each term. An agent described as an SVO "evaluator" can be discovered by querying for a CasehubSlot "reviewer" because the vocabulary registry traverses the equivalence links bidirectionally.

The `CdiVocabularyRegistry` discovers vocabulary producers at startup via CDI `Instance<Vocabulary>`, then allows programmatic registration at runtime. Clean and extensible — domain apps bring their own vocabularies as CDI beans, and cross-vocabulary discovery works automatically.

## Reactive Build Gating

I wanted to understand how the platform handles the reactive/blocking split before committing to a pattern. Digging into qhorus and engine revealed the answer: `@IfBuildProperty` on each reactive bean, backed by a `@ConfigRoot(phase=BUILD_TIME)` config interface in the deployment module. The existing protocol was slightly misleading — it mentioned `ExcludedTypeBuildItem` as the mechanism, but qhorus (the canonical implementation) uses the simpler `@IfBuildProperty` approach. We updated the protocol and PLATFORM.md to reflect what actually works.
