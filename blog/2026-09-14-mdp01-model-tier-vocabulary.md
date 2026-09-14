---
layout: post
title: "Model tiers as vocabulary — where selection doesn't belong"
date: 2026-09-14
entry_type: note
subtype: diary
projects: [casehubio/eidos]
tags: [vocabulary, model-selection, architecture, boundary]
---

The platform's `ModelRegistry` shipped with `ModelTier` (FLAGSHIP, STANDARD, FAST, EMBEDDING), `CostTier`, `ModelLocality`, and a `ModelQuery` builder. Issue #172 asked: how does eidos integrate with this?

The tempting answer was a `ModelSelector` SPI in eidos that consumes `ModelRegistry` and applies agent requirements directly. I rejected it for three reasons. First, it creates a dependency cycle — `ModelRegistry` lives in `platform-api`, and eidos should never depend upstream on platform. Second, CBR integration (blocks#270) needs to inject historical outcome signals into model selection via `RoutingSignalAssembler`, which lives in the platform layer. If eidos owns selection, blocks has to reach into eidos to compose its signals. Third, the existing `RoutingAgentProvider` already dispatches by resolved model — the resolution step belongs there, not in a new SPI.

So eidos declares; platform resolves. The descriptor says "I need FLAGSHIP with vision." The platform figures out which actual model satisfies that, within the tenant's budget and locality constraints.

The interesting design question was where on the descriptor to put the requirements. `AgentCapability` already carries `qualityHint`, `latencyHintP50Ms`, `costHint`, and `epistemicDomains` — all per-capability operational metadata. Model tier is the same kind of signal. A code-review capability genuinely needs different compute than a lint check. Per-capability also gives CBR a clean grouping key: it scores by `(agentId, modelId, capabilityName)`, so per-capability requirements map directly to its learning surface.

Two new fields: `modelTier` (vocabulary-grounded, validated against `urn:casehub:vocab:model-tier`) and `modelCapabilities` (open `Set<String>` — vendor feature flags like "text", "vision", "tool-use"). The split mirrors a real asymmetry. Model tiers are stable abstractions — FLAGSHIP/STANDARD/FAST will outlive any vendor's naming. They earn a vocabulary with subsumption: FLAGSHIP specializes STANDARD specializes FAST, meaning a FLAGSHIP model satisfies a STANDARD requirement at `MatchDegree.Plugin(1)`. The platform router can use that match degree to implement fallback when the preferred tier is unavailable or over budget.

Model capabilities are different. They're vendor feature flags that change every quarter — text, vision, tool-use, code, audio, computer-use. Grounding them in a vocabulary would mean a new enum constant every time a vendor ships a modality. Set intersection (`descriptor.capabilities().containsAll(required)`) is what `ModelQuery` already does. No reason to duplicate that in eidos terms.

Cost ceiling and locality got the same treatment: not on the descriptor. Cost budgets are tenant policy — the agent author doesn't know what the tenant can afford. `costHint` already signals "this is expensive" for discovery. Locality (CLOUD vs LOCAL) is infrastructure policy driven by data sovereignty requirements. Both are applied by the platform router from tenant preferences, not declared by the agent.

The implementation touched every layer — API record, vocabulary enum, JPA persistence, A2A_CARD rendering, YAML deserialization, annotation processor — but each change followed an existing pattern exactly. The `AgentDescriptorComparator` has a reflective coverage test that caught the field count change immediately, which is the kind of safety net that makes mechanical changes across 19 files feel routine rather than risky.

`ModelTierTerm` is the thirteenth vocabulary in `casehub-eidos-vocab`, and it's the simplest — four terms, a linear chain, no cross-vocabulary mappings. EMBEDDING sits outside the hierarchy entirely because it's a different modality, not a tier level. An embedding model can't satisfy a text generation requirement regardless of how capable it is — `MatchDegree.None` is the correct answer, and the hierarchy encodes that structurally.

The boundary decision — eidos declares, platform resolves — will matter more as the model landscape fragments. When tenants run mixed fleets (cloud flagships for complex work, local models for data-sovereign tasks, fast models for high-throughput pipelines), the resolution logic gets arbitrarily complex. That complexity belongs in one place — the platform router — reading simple, stable requirements from the descriptor. Eidos stays focused on identity and discovery.
