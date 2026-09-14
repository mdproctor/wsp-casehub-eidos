# Decisions — #172 Model Selection Vocabulary

## D1: Integration boundary — eidos declares, platform resolves

**Choice:** Eidos is the requirements declaration layer. Platform (RoutingAgentProvider) does the actual model selection.
**Alternatives:**
- Eidos owns a ModelSelector SPI consuming ModelRegistry — creates platform→eidos→platform dependency cycle, blocks CBR composition via RoutingSignalAssembler
**Rationale:** Follows existing boundary rules (platform#285 layer model). Dependency flows one way: platform reads eidos descriptors, resolves models from its own registry. CBR (blocks#270) composes via RoutingSignalAssembler at the platform layer without reaching into eidos.
**Trade-offs:** Eidos has no runtime visibility into which model was actually selected — it only declares requirements. Selection telemetry lives in the platform.
**Sources:** casehubio/platform#285 (layer model), casehubio/blocks#270 (CBR routing), AgentDescriptor.java (existing modelFamily/modelVersion fields), RoutingAgentProvider (platform)
**Exploration:** quick
**Status:** captured

## D2: Per-capability model requirements

**Choice:** Model requirements (tier, capabilities) live on AgentCapability, not on AgentDescriptor.
**Alternatives:**
- Per-descriptor model requirements — forces entire agent to one tier, doesn't reflect different computational demands per capability
- Both (descriptor default + per-capability override) — adds complexity without a real use case; capability IS the requirement
**Rationale:** AgentCapability is already the unit of operational metadata (qualityHint, latencyHintP50Ms, costHint, epistemicDomains). Model tier is another operational dimension. CBR feedback (blocks#270) scores by (agentId, modelId, capabilityName) — per-capability requirements give it a natural grouping key.
**Trade-offs:** An agent with 5 capabilities each needing FLAGSHIP must declare it 5 times. No descriptor-level default means no shorthand. Acceptable because explicit is better than implicit for routing-critical metadata.
**Sources:** AgentCapability.java (existing per-capability fields), casehubio/blocks#270 (CBR scoring key)
**Exploration:** quick
**Status:** captured

## D3: Linear subsumption hierarchy for ModelTierTerm

**Choice:** ModelTierTerm vocabulary with linear subsumption: FLAGSHIP → STANDARD → FAST. EMBEDDING outside the hierarchy (different modality).
**Alternatives:**
- Flat vocabulary, no subsumption — loses "FLAGSHIP can satisfy STANDARD" matching via VocabularyRegistry.match()
- Bidirectional with cost-awareness — cost is a separate dimension, not a vocabulary relationship
**Rationale:** The tier hierarchy IS a capability relationship — FLAGSHIP can do everything STANDARD can do, plus more. OWLS-MX matching in eidos already handles this: FLAGSHIP satisfying a STANDARD requirement is a Plugin match. Cost is a separate filter applied after matching, not encoded in vocabulary.
**Trade-offs:** EMBEDDING is an island — no subsumption path to/from the compute tiers. Acceptable because embedding models genuinely cannot satisfy text generation requirements and vice versa.
**Sources:** VocabularyTerm.java (specializes()), MatchDegree.java (Plugin/Specialization), ModelTier.java (platform enum)
**Exploration:** quick
**Status:** captured

## D4: Open strings for model capabilities, no vocabulary

**Choice:** modelCapabilities on AgentCapability is Set<String> — open strings that pass through to platform's ModelQuery.requiredCapabilities.
**Alternatives:**
- Vocabulary-grounded ModelCapabilityTerm — maintenance burden tracking vendor releases (text, vision, tool-use, code, audio, video, computer-use evolve quarterly)
**Rationale:** Model capabilities are vendor feature flags that change every quarter. Set intersection (ModelDescriptor.capabilities().containsAll(required)) is already what ModelRegistry.query() does. No reason to duplicate in eidos vocabulary terms.
**Trade-offs:** No subsumption matching for model capabilities (e.g., "multimodal" doesn't automatically match "vision"). Acceptable because the platform router can handle this if needed.
**Sources:** ModelDescriptor.java (capabilities: Set<String>), ModelQuery.java (requiredCapabilities: Set<String>)
**Exploration:** quick
**Status:** captured

## D5: No cost ceiling on descriptor — tenant policy

**Choice:** Cost budgets are tenant policy applied by the platform router. No maxCostTier field on AgentCapability.
**Alternatives:**
- Add maxCostTier to AgentCapability — conflates identity with operations, creates competing authority (agent author vs tenant budget)
**Rationale:** The agent author doesn't know what the tenant can afford. costHint already signals "this is expensive" for discovery. The platform router applies maxCostTier from tenant preferences at resolution time. Single authority — no conflict to resolve.
**Trade-offs:** An agent author cannot prevent their agent from being backed by an expensive model. Acceptable because cost governance is an organizational concern, not an agent identity concern.
**Sources:** AgentCapability.java (existing costHint field), ModelQuery.java (maxCostTier), PreferenceProvider (tenant preferences)
**Exploration:** quick
**Status:** captured

## D6: No locality preference on descriptor — infrastructure policy

**Choice:** Locality (CLOUD vs LOCAL) is infrastructure/tenant policy. No locality field on AgentCapability.
**Alternatives:**
- Add localityPreference to AgentCapability — conflates agent identity with deployment topology
**Rationale:** Same reasoning as D5. Whether a model runs locally or in a vendor's cloud is determined by tenant compliance/data-handling policy (e.g., data-sovereign tenants constrain to LOCAL). The agent declares what it needs (tier, capabilities); the deployment context determines where it runs.
**Trade-offs:** An agent cannot declare "I must run locally" — but that's the jurisdiction/dataHandlingPolicy field's job at the descriptor level, not model selection's.
**Sources:** ModelLocality.java (CLOUD/LOCAL), AgentDescriptor.java (jurisdiction, dataHandlingPolicy)
**Exploration:** quick
**Depends on:** D5 (same boundary principle)
**Status:** captured
