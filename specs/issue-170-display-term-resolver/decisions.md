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

**Choice:** Four-arg primary method `resolveLabel(value, sourceVocabUri, targetVocabUri, axis)` with convenience overloads for common cases. Resolution chain: find source term → if target provided, cross-vocab via equivalentValues (axis-aware when axis is non-null, axis-unaware otherwise) → resolve target value to term → return label → fallback to raw value. Note: equivalentValues returns a target *value*, not a term — a second resolve() call is needed to get the label.
**Alternatives:**
- Two separate methods (direct + swap) — clearer intent per call but doubles the API surface for the same underlying logic
- Builder/Resolution pattern — over-engineered for what is fundamentally a label lookup
- Three-arg without axis — would silently fail cross-vocab swap for disposition terms (DISC, Thomas-Kilmann, Belbin) which use axisExactMatch exclusively
**Rationale:** The four-arg method handles all cases: direct resolution (targetVocabUri=null), cross-vocab swap (targetVocabUri=different), and axis-aware disposition swap (axis=non-null). Convenience overloads keep common cases clean. The "just works" swap behavior is: UI sets a target vocabulary, passes it on every call, cross-vocab mappings resolve automatically via existing exactMatch/axisExactMatch infrastructure.
**Trade-offs:** Nullable parameters (sourceVocabUri, targetVocabUri, axis) — null means "search all" / "use source label" / "axis-unaware" respectively. Four parameters is the maximum comfortable arity; further extension would need a context object.
**Sources:** VocabularyRegistry.java:35-36 (equivalentValues axis-aware + unaware), VocabularyTerm.java:27 (exactMatch), VocabularyTerm.java:44 (axisExactMatch)
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

## D5: Auto-discovery is best-effort; explicit sourceVocabUri preferred

**Choice:** Document auto-discovery (sourceVocabUri=null) as best-effort. Registration order is stable within a deployment but unspecified across deployments. Callers should provide sourceVocabUri when available — auto-discovery is the fallback, not the primary path.
**Alternatives:**
- Enforce deterministic ordering (alphabetical by URI) — adds complexity for an edge case
- Reject null sourceVocabUri entirely — breaks the "just works" convenience
**Rationale:** Decision review finding: registeredUris() returns a Set with unspecified iteration order. Same value in two vocabularies (unlikely) could yield different labels across runs. Documenting best-effort + preferring explicit sourceVocabUri handles this without over-engineering.
**Trade-offs:** Non-deterministic edge case remains. Acceptable: vocabulary values are unique in practice.
**Depends on:** D4 (auto-discovery)
**Sources:** Decision review finding #2
**Exploration:** quick (from review)
**Status:** captured
