---
layout: post
title: "Wiring the Disposition Layer"
date: 2026-07-28
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [personality, jungian, disposition, spi, health-probing, auto-derivation]
series: issue-107-jungian-personality-framework
---

*Part of a series on [#107 — Jungian personality framework](https://github.com/casehubio/eidos/issues/107). Previous: [Personality from Cognitive Functions, Not Labels](2026-07-28-mdp01-personality-from-cognitive-functions.md).*

## The Signal Store That Isn't BehavioralSignalStore

The design review caught something I'd have missed: `BehavioralSignalStore` requires a `capabilityName` parameter on every call. Disposition function activations aren't capability-scoped — they track which cognitive functions an agent is actually using, independent of what capability it's performing. The lifecycle semantics differ too. Capability signals expire via TTL (windowed compliance monitoring). Activation counts accumulate until explicitly decayed or cleared after an evolution transition.

`DispositionSignalStore` is four methods: `recordActivation`, `activationCounts`, `decay`, `clear`. The decay operation multiplies all counts by `(1 - decayFactor)` with integer truncation — a single activation decays to zero, preventing noise from persisting indefinitely. The full CDI ladder follows: `NoOpDispositionSignalStore` as `@DefaultBean`, `InMemoryDispositionSignalStore` with `AtomicInteger` for thread safety, `JpaDispositionSignalStore` backed by a V9 migration.

## Probing Disposition Without Knowing the Vocabulary

The interesting design problem in `DefaultDispositionHealth` was shadow resolution. The probe needs to detect when a function's shadow has grown strong enough to threaten the dominant — that's DOMINANT_REPLACEMENT in JPAF terms. But `JungianFunctionTerm.shadow()` lives in the vocab module, and runtime can't depend on vocab without making optional vocabularies mandatory for every consumer.

The fix was a default method on `VocabularyTerm`: `opposite()` returns `Optional.empty()`. `JungianFunctionTerm` overrides it to delegate to `shadow()`. The probe resolves terms through the registry, calls `opposite()` generically, and never imports a single vocab class. Vocabularies without structural opposites inherit the no-op — they simply don't trigger shadow-based evolution conditions.

The evolution types themselves use the same pattern. The probe defines its constants as lambdas — `() -> "DOMINANT_AUXILIARY_SWAP"` — matching `JungianEvolutionType` enum names by convention. No compile-time coupling, name-based matching at the consumer.

## Auto-Derivation: Profile to Axes

When an agent declares a Jungian disposition profile but leaves the axis fields empty, `DescriptorCollector` now projects the profile onto the five disposition axes automatically. For each function in the profile, it calls `equivalentValues()` against every registered vocabulary for every axis, weights the result by the function's weight, aggregates per axis, and normalizes.

An INTP profile (Ti at 0.35, Ne at 0.20, the rest distributed) produces `socialOrient: [("independent", 0.65), ("collaborative", 0.31), ("facilitative", 0.04)]`. The projection bridges frameworks — queries by axis ("find agents near independent") work against Jungian-profiled agents through the derived values.

This required adding `registeredUris()` to `VocabularyRegistry` — the derivation iterates all registered vocabularies to discover cross-vocabulary mappings without hardcoding target URIs.

Explicit axis values always win. An agent can declare a Jungian profile for the cognitive model and override specific axes where the automatic projection doesn't capture the intended nuance.
