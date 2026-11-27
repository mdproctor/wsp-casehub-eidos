## D1: Collaboration data source

**Choice:** AgentGraphQuery task history with externalRef intersection
**Alternatives:**
- LedgerEntry subjectId intersection via LedgerEntryRepository — works without graph module but introduces new dependency pattern, subjectId is a rawer signal, heavier queries
**Rationale:** externalRef is the semantically correct "shared work item" concept; consistent with existing coActiveAgents(); no new dependency on LedgerEntryRepository (no eidos enricher currently uses it)
**Trade-offs:** Requires AgentGraphQuery implementation on classpath (not just NoOpAgentGraphQuery); degrades to NEUTRAL when graph module absent
**Sources:** AgentGraphQuery.java, AgentTask.java (externalRef field), OrgDistanceAttestationEnricher.java (sibling pattern), CapabilityOverlapAttestationEnricher.java (sibling pattern)
**Exploration:** quick
**Status:** captured

## D2: Decay function

**Choice:** Exponential decay with configurable half-life and baseline
**Alternatives:**
- Linear decay with floor — simpler but abrupt transition at the floor
- Step function with tiers — matches OrgProximity's weight-table pattern but loses smoothness
**Rationale:** Exponential decay is a natural fit for recency: recent collaboration is worth much more than old, with a smooth asymptotic approach to baseline. Configurable half-life (default 7 days) and baseline (default 0.5) via @ConfigProperty.
**Trade-offs:** Slightly more complex than linear/step, but the formula is a single line and the parameters are intuitive (half-life in days)
**Sources:** CapabilityOverlapRelevanceFactor.java (baseline pattern), OrgProximityRelevanceFactor.java (configurable weights pattern)
**Exploration:** quick
**Status:** captured

## D3: Snapshot data format

**Choice:** Days since last collaboration as a single double value
**Alternatives:**
- ISO timestamp of last collaboration — more precise if gap between enrichment and scoring, but couples factor to clock semantics and differs from the numeric-value pattern used by all other enrichers
**Rationale:** Consistent with org-distance (single numeric value in snapshot), simple for the factor to consume, and the enricher already knows the attestation timestamp so it can compute elapsed days at write time
**Trade-offs:** If scoring happens significantly later than enrichment, the elapsed days is slightly stale — but enrichment and scoring happen in the same pipeline so the gap is negligible
**Sources:** OrgDistanceUtil.java (SNAPSHOT_KEY pattern), CapabilityOverlapRelevanceFactor.java (SNAPSHOT_KEY pattern)
**Exploration:** quick
**Status:** captured

## D4: Module placement

**Choice:** Both enricher and factor in `runtime` module under `io.casehub.eidos.runtime.trust`
**Alternatives:**
- Enricher in runtime, factor in eidos-core — unusual split with no precedent
**Rationale:** Follows the CapabilityOverlap precedent (both in runtime). AgentGraphQuery is in eidos-api which runtime already depends on. No new module or dependency needed.
**Trade-offs:** None significant — this is the natural home
**Sources:** CapabilityOverlapAttestationEnricher.java (runtime module placement), CapabilityOverlapRelevanceFactor.java (runtime module placement)
**Exploration:** quick
**Status:** captured
