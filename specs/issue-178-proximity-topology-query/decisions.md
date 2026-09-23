# Decisions — Issue #178: Proximity/Topology Query

## D1: Separate SPIs by concern

**Choice:** Three separate surfaces — proximity on AgentQuery/AgentRegistry, activity on AgentGraphQuery, collaboration topology on a new RuntimeCollaborationQuery SPI.
**Alternatives:**
- All on AgentQuery — simpler API but forces AgentRegistry to depend on graph and org stores, violating tier separation
- New DiscoveryService facade — higher-level service composing all three, but adds indirection without solving the dependency problem
**Rationale:** Each query type needs a different backing store. Proximity is descriptor matching (AgentRegistry). Activity is runtime task state (AgentGraphQuery/graph module). Collaboration topology is emergent from engine's stigmergy system (SPI pattern like CapabilityHealth). Keeping them separate preserves eidos's tier architecture.
**Trade-offs:** Consumers who want "find me everyone relevant" must compose three queries. No single discovery surface.
**Sources:** AgentQuery.java, AgentRegistry.java, AgentGraphQuery.java, OrgRegistry.java, engine TeamDetector (internal)
**Exploration:** quick
**Status:** captured

## D2: Depth-based threshold for proximity

**Choice:** Use MatchDegree's existing depth as the proximity threshold. `maxDepth=2` means Plugin(1), Plugin(2), Specialization(1), Specialization(2) match.
**Alternatives:**
- Normalized similarity score (0.0–1.0) — more flexible but introduces a scoring model that doesn't exist yet and needs calibration
- Epistemic domain overlap — orthogonal to capability hierarchy, would need a separate scoring metric
**Rationale:** CapabilityResolver already computes MatchDegree with depth. Depth is a natural, integer-valued distance metric in the vocabulary hierarchy. No new scoring model needed.
**Trade-offs:** Depth-based matching doesn't capture cross-vocabulary similarity or epistemic domain overlap. Those are future concerns if needed.
**Sources:** CapabilityResolver.java:35-52, MatchDegree.java (Exact/Plugin(depth)/Specialization(depth)/None), VocabularyRegistry.match()
**Exploration:** quick
**Status:** captured

## D3: Use externalRef for activity queries

**Choice:** Query by AgentTask.externalRef (opaque String). No typed caseId field.
**Alternatives:**
- Add typed UUID caseId to AgentTask — more explicit but couples eidos to the case concept, which is engine-owned
**Rationale:** externalRef is already the opaque reference field on AgentTask. Callers pass whatever identifier they used when recording the task. No schema change. eidos stays decoupled from engine's case concept.
**Trade-offs:** Less type safety — callers must know the externalRef convention. No validation that the ref actually identifies a case.
**Sources:** AgentTask.java:11 (externalRef field)
**Exploration:** quick
**Status:** captured

## D4: Activity queries extend AgentGraphQuery

**Choice:** Add `coActiveAgents(String externalRef, String tenancyId)` to AgentGraphQuery, returning `List<String>` (agentIds).
**Alternatives:**
- Extend OrgRegistry — treats runtime collaboration as an organizational concern, which it isn't
- New CollaborationTopology SPI — separate interface for a single method is over-abstraction
**Rationale:** AgentGraphQuery already owns the read-side of task data. The task table has externalRef and endedAt. The query is: find agents with in-progress tasks sharing the same externalRef. This is a graph query, not an org query. Returns `List<String>` consistent with existing `topAgentsByOutcome()` — graph queries return identifiers, callers compose with registry.
**Trade-offs:** Callers must look up descriptors separately via AgentRegistry if they need AgentMatch. Existing implementations (JPA, NoOp) must be updated.
**Sources:** AgentGraphQuery.java, JpaAgentGraphQuery.java, blocks CbrAgentRoutingStrategy.java (consumer)
**Exploration:** quick
**Depends on:** D1 (separate SPIs)
**Status:** revised (review: D6 interaction — AgentGraphQuery returns List<String>, not AgentMatch)

## D5: RuntimeCollaborationQuery SPI in eidos-api

**Choice:** New SPI interface in eidos-api for runtime collaboration topology. Engine's TeamDetector provides the implementation. Follows the CapabilityHealth pattern (SPI in eidos, engine implements). Define `CollaborationRelation` enum in eidos-api (not imported from engine-api).
**Alternatives:**
- Document existing coverage only — MetricsSpace.detectedTeams() exists but is per-case, agent-scoped, not a general query API
- Extend OrgRegistry with runtime teams — blurs static/dynamic boundary
- Import NeighborRelation from engine-api — wrong dependency direction (eidos-api cannot depend on engine-api)
**Rationale:** Engine's TeamDetector produces DetectedTeam(teamId, memberAgents, dominantRelations, avgAffinity, stabilityCount) with NeighborRelation { COACTIVE, SHARED_INTEREST, SHARED_SIGNAL, COMPLEMENTARY }. This is exposed via MetricsSpace.detectedTeams() — but only inside a worker runtime, per-case. No general query SPI exists for "what emergent teams has agent X participated in across cases?" The CapabilityHealth pattern is the proven approach: eidos-api defines the SPI, eidos-core provides NoOp @DefaultBean, engine provides the real implementation. NeighborRelation lives in engine-api (io.casehub.api.spi.observation) — eidos-api can't depend on engine-api, so eidos defines its own `CollaborationRelation` enum with the same semantics.
**Trade-offs:** New SPI surface area. Engine must implement it. CollaborationRelation duplicates NeighborRelation semantics (but preserves tier independence).
**Sources:** engine MetricsSpace.java:37 (detectedTeams()), DetectedTeam.java, NeighborRelation.java (io.casehub.api.spi.observation), TeamDetector.java (internal), CapabilityHealth.java (pattern reference)
**Exploration:** quick
**Status:** revised (review: NeighborRelation dependency direction resolved — define own enum)

## D6: Per-query result types matched to SPI tier

**Choice:** Result types follow each SPI's tier constraints:
- **Proximity** → `List<AgentMatch>` (natural — it IS a registry query via AgentRegistry.find())
- **Activity** → `List<String>` (agentIds) from AgentGraphQuery, consistent with existing `topAgentsByOutcome()` pattern
- **Collaboration** → `List<Collaborator>` — new record wrapping agentId + `Set<CollaborationRelation>` + affinity score. Callers compose with AgentRegistry for full descriptors.
**Alternatives:**
- All return List<AgentMatch> — forces AgentGraphQuery to depend on AgentRegistry, violating its current tier-1 purity
- All return List<String> — loses proximity's MatchDegree metadata and collaboration's relationship metadata
**Rationale:** Each SPI has different metadata to carry. Proximity carries MatchDegree (via ResolvedCapability on AgentMatch). Activity is a set membership question (who's on this ref?). Collaboration carries relationship type and affinity score. Forcing a single type loses information or creates wrong-tier dependencies.
**Trade-offs:** Three different result types. Consumers composing across all three must handle heterogeneous results.
**Sources:** AgentMatch.java, AgentGraphQuery.topAgentsByOutcome() (returns List<String>), DetectedTeam.java (carries memberAgents + dominantRelations + avgAffinity)
**Exploration:** quick
**Depends on:** D1 (separate SPIs), D4 (AgentGraphQuery returns List<String>), D5 (CollaborationRelation enum)
**Status:** revised (review: D4 interaction — tier boundaries require per-SPI result types)
