## D1: Metric ownership model

**Choice:** SPI with structured snapshots in eidos
**Alternatives:**
- CDI events only — consumers re-implement aggregation
- Micrometer direct — couples library to metrics framework
**Rationale:** Consistent with existing SPI patterns (BehavioralSignalStore, DispositionSignalStore). Eidos owns the domain model; consumers decide how to surface it.
**Trade-offs:** More code in eidos than CDI events alone
**Sources:** BehavioralSignalStore.java, DispositionSignalStore.java
**Exploration:** quick
**Status:** captured

## D2: Signal granularity

**Choice:** Per-computation (one snapshot per trust score computation for a scored agent)
**Alternatives:**
- Per-attestation — maximum detail but very high volume
- Both tiered — per-computation for dashboards, per-attestation for drill-down
**Rationale:** The natural unit for operator questions: "how concentrated is agent X's evidence?" not "what happened to attestation Y?"
**Trade-offs:** Loses per-attestation forensic detail
**Sources:** RelevanceComposer.java, IndependenceAwareRelevanceFactor.java
**Exploration:** quick
**Status:** captured

## D3: Signal recording location

**Choice:** Factor-side recording — each factor records its own signal
**Alternatives:**
- Composer-side — requires ledger changes to instrument RelevanceComposer
- Decorator pattern — less access to internal state (same-unit share) that only the factor knows
**Rationale:** Factors already have the data. No ledger changes needed for recording. Factors opt in individually.
**Trade-offs:** Each factor must inject the collector — slight coupling
**Sources:** IndependenceAwareRelevanceFactor.java, CapabilityOverlapRelevanceFactor.java
**Exploration:** quick
**Status:** captured

## D4: Transient vs persisted signal architecture

**Choice:** Record transient relevance signals; query persisted credibility data
**Alternatives:**
- Unified real-time recording for all signal types — duplicates already-persisted credibility data, bloats union type
- CDI events for credibility — requires ledger changes to fire events from credibility policies
**Rationale:** Relevance signals (same-unit share, cap activation) are transient — computed and discarded during trust scoring. Credibility flags (COLLUSION_SUSPECT) are already persisted on attestations. Recording them again duplicates data. The dashboard aggregates both sources.
**Trade-offs:** Two data sources for the dashboard service to join. Credibility query needs a ledger-side service (separate issue).
**Sources:** AttestorCredibilityPolicy.java, CredibilityFlag.java, IndependenceAwareRelevanceFactor.java
**Exploration:** deep-analysis
**Status:** captured
