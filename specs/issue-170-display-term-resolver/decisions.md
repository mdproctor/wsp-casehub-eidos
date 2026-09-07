# Decisions — Issue #170: DisplayTermResolver

## D1: Vocabulary context comes from domain objects, not tenancy

**Choice:** Caller provides vocabUri directly from the domain object (AgentDescriptor.vocabUriForSlot(), OrganizationalUnit.kindVocabulary(), AgentCapability.capabilityVocabulary()). No tenancy-based vocabulary derivation.
**Alternatives:**
- DisplayContext with tenancyId-derived vocabulary — conflates legal/customer data separation (tenancy) with terminology choice (vocabulary); requires new preference infrastructure
**Rationale:** Tenancy is a legal barrier separating customer data. Vocabulary choice is a per-descriptor/unit property already carried on the domain objects. Two agents in the same tenancy can use different vocabularies.
**Trade-offs:** No tenant-level "default vocabulary" — each resolve call must provide the vocabUri. Acceptable because callers have the domain object in hand.
**Sources:** AgentDescriptor.java:112-118 (vocabUriForSlot), VocabularyRegistry.java:30 (resolve)
**Exploration:** quick
**Status:** captured

## D2: API shape — three-arg method with cross-vocabulary swap

**Choice:** Single primary method `resolveLabel(value, sourceVocabUri, targetVocabUri)` with a two-arg convenience overload. Resolution chain: find source term → if target provided, cross-vocab via equivalentValues → return label → fallback to raw value.
**Alternatives:**
- Two separate methods (direct + swap) — clearer intent per call but doubles the API surface for the same underlying logic
- Builder/Resolution pattern — over-engineered for what is fundamentally a label lookup
**Rationale:** The three-arg method handles both cases: direct resolution (targetVocabUri=null) and cross-vocab swap (targetVocabUri=different). The convenience overload covers the common case. The "just works" swap behavior is: UI sets a target vocabulary, passes it on every call, cross-vocab mappings resolve automatically via existing exactMatch() infrastructure.
**Trade-offs:** Nullable parameters (sourceVocabUri, targetVocabUri) — null means "search all" / "use source label" respectively. Acceptable for a utility service.
**Sources:** VocabularyRegistry.java:35 (equivalentValues), VocabularyTerm.java:27 (exactMatch)
**Exploration:** quick
**Status:** captured

## D3: SPI in eidos-api, DefaultBean impl in eidos-runtime

**Choice:** Interface in eidos-api (Tier 1, pure Java). DefaultDisplayTermResolver @DefaultBean @ApplicationScoped in eidos-runtime injecting VocabularyRegistry.
**Alternatives:**
- Static utility class — no CDI override possible; when this moves to platform-api, consumers can't customize
- Concrete @ApplicationScoped (no SPI) — locks the implementation; breaks when platform needs a different strategy
**Rationale:** SPI now avoids a breaking change when the service moves to platform-api (casehubio/platform#283). @DefaultBean allows consumers to displace with @Alternative.
**Trade-offs:** SPI indirection for what is currently a simple delegation to VocabularyRegistry. Acceptable — the move to platform is planned.
**Sources:** casehubio/platform#283, DefaultCapabilityHealth pattern (@DefaultBean + Instance<>)
**Exploration:** quick
**Status:** captured

## D4: Auto-discovery when sourceVocabUri is null

**Choice:** When sourceVocabUri is null, search all registered vocabularies for the value. Return the first match's label (or cross-vocab to target if targetVocabUri is set).
**Alternatives:**
- Require sourceVocabUri always (no null) — simpler but breaks the "just works" experience when the caller doesn't know which vocab a value belongs to
- Return raw value immediately when sourceVocabUri is null — safe but defeats the purpose
**Rationale:** The caller may have a raw value from a relationship or org unit without knowing its vocabulary. Searching all registered vocabs (typically < 20) is cheap and makes resolution work without requiring the caller to track provenance.
**Trade-offs:** O(N) search across registered vocabularies — negligible for display-time use. Ambiguity if the same value exists in multiple vocabularies (first-match wins). Acceptable: vocabulary values are typically unique across a deployment.
**Sources:** VocabularyRegistry.java:25 (registeredUris), CdiVocabularyRegistry.java:418 (resolve)
**Exploration:** quick
**Status:** captured
