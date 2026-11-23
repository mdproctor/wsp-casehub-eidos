## D1: Architectural approach — standalone eidos SPI with snapshot input

**Choice:** New `CrossUnitEvidenceAssessor` SPI in `eidos-org-api` taking `List<Map<String, Object>>` (attestation relevance snapshots). Returns `CrossUnitEvidenceAssessment` record. Implementation in `org-runtime`. `@DefaultBean` NoOp returns UNKNOWN. Engine calls it after trust computation.
**Alternatives:**
- Extend `RelevanceComposer` with metadata channel — too invasive, changes ledger SPI and conflates per-attestation weighting with batch-level metadata
- Score-level annotation via ledger SPI — most invasive, changes ledger-api/core/persistence for a signal needed only at dispatch time
**Rationale:** Zero ledger changes. Follows existing eidos CDI priority ladder. Input is `List<Map<String, Object>>` — no dependency on ledger types. Reuses snapshot data written by `OrgDistanceAttestationEnricher`. Mass-preserving normalisation makes a relevance factor approach a no-op for uniform discounts — this is a fundamentally different kind of signal (evidence pool quality, not individual attestation weighting).
**Trade-offs:** Assessment computed on-demand by the engine, not automatically during trust computation. Engine must extract snapshots and call the assessor. Minimal coupling — engine already integrates trust scores with capability health.
**Sources:** `IndependenceAwareRelevanceFactor.java` (existing partition logic), `GraphDataSufficiency.java` (parallel pattern), `TrustScoreComputer.java` (mass-preserving normalisation constraint), `RelevanceComposer.java` (composition pipeline)
**Exploration:** deep-analysis
**Status:** captured
