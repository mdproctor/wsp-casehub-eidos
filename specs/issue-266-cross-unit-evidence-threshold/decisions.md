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

## D2: Diversity level enum

**Choice:** Three values: `DIVERSE` (>= threshold), `HOMOGENEOUS` (< threshold, attestations exist), `UNKNOWN` (no snapshots or no attestations)
**Alternatives:**
- Four tiers with THIN middle tier — adds complexity without value since the threshold is configurable
- Two tiers (DIVERSE/NOT_DIVERSE) — loses the "no data" vs "homogeneous data" distinction
**Rationale:** Parallels `GraphDataSufficiency`'s three-tier model. UNKNOWN is essential for backward compatibility when eidos org module is absent.
**Trade-offs:** None significant.
**Sources:** `GraphDataSufficiency.java` (SufficiencyLevel enum pattern)
**Exploration:** quick
**Status:** captured

## D3: Default minimum cross-unit threshold

**Choice:** 2 cross-unit attestations for DIVERSE. Configurable via `@ConfigProperty(name = "casehub.eidos.trust.diversity.min-cross-unit", defaultValue = "2")`.
**Alternatives:**
- Threshold 1 — a single cross-unit attestation is noise, not validation
- Threshold 3 — too conservative for agents with limited cross-unit exposure
**Rationale:** Two is the minimum for a cross-unit pattern. One is an outlier; two suggests consistent cross-unit signal. Configurable so consumers set their own bar.
**Trade-offs:** Low threshold means the DIVERSE label is easy to earn. Acceptable because the raw counts are always available for stricter consumers.
**Sources:** `IndependenceAwareRelevanceFactor.java` (same-unit/cross-unit partition logic)
**Exploration:** quick
**Status:** captured

## D4: Assessment scope — per-capability

**Choice:** Per-capability assessment. Each capability's attestation batch is assessed independently.
**Alternatives:**
- Global-only — masks per-capability diversity gaps
- Both global and per-capability — over-engineering for the initial implementation
**Rationale:** An agent may have diverse evidence for "code-review" but homogeneous for "planning". Per-capability matches how `CapabilityHealth.probe()` and `TrustScoreSource` already operate — per-capability is the primary query axis.
**Trade-offs:** Consumers wanting a global diversity signal must aggregate per-capability assessments themselves. Acceptable — global is derivable from per-capability, not vice versa.
**Sources:** `CapabilityHealth.java` (per-capability probe pattern), `TrustScoreSource.java` (per-capability score queries)
**Exploration:** quick
**Status:** captured
