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

**Choice:** Add `coActiveAgents(String externalRef, String tenancyId)` to AgentGraphQuery.
**Alternatives:**
- Extend OrgRegistry — treats runtime collaboration as an organizational concern, which it isn't
- New CollaborationTopology SPI — separate interface for a single method is over-abstraction
**Rationale:** AgentGraphQuery already owns the read-side of task data. The task table has externalRef and endedAt. The query is: find agents with in-progress tasks sharing the same externalRef. This is a graph query, not an org query.
**Trade-offs:** AgentGraphQuery now needs AgentRegistry injected to produce AgentMatch results. Existing implementations (JPA, NoOp) must be updated.
**Sources:** AgentGraphQuery.java, JpaAgentGraphQuery.java, blocks CbrAgentRoutingStrategy.java (consumer)
**Exploration:** quick
**Depends on:** D1 (separate SPIs)
**Status:** captured

## D5: RuntimeCollaborationQuery SPI in eidos-api

**Choice:** New SPI interface in eidos-api for runtime collaboration topology. Engine's TeamDetector provides the implementation. Follows the CapabilityHealth pattern (SPI in eidos, engine implements).
**Alternatives:**
- Document existing coverage only — MetricsSpace.detectedTeams() exists but is per-case, agent-scoped, not a general query API
- Extend OrgRegistry with runtime teams — blurs static/dynamic boundary
**Rationale:** Engine's TeamDetector produces DetectedTeam(teamId, memberAgents, dominantRelations, avgAffinity, stabilityCount) with NeighborRelation { COACTIVE, SHARED_INTEREST, SHARED_SIGNAL, COMPLEMENTARY }. This is exposed via MetricsSpace.detectedTeams() — but only inside a worker runtime, per-case. No general query SPI exists for "what emergent teams has agent X participated in across cases?" The CapabilityHealth pattern is the proven approach: eidos-api defines the SPI, eidos-core provides NoOp @DefaultBean, engine provides the real implementation.
**Trade-offs:** New SPI surface area. Engine must implement it, creating a cross-repo dependency on this issue. Relationship types (NeighborRelation) either need to be defined in eidos-api or imported from engine-api.
**Sources:** engine MetricsSpace.java:37 (detectedTeams()), DetectedTeam.java, NeighborRelation.java, TeamDetector.java (internal), CapabilityHealth.java (pattern reference)
**Exploration:** quick
**Status:** captured

## D6: Reuse AgentMatch as result type

**Choice:** All three query types return List<AgentMatch> — consistent with AgentRegistry.find().
**Alternatives:**
- New lightweight result (List<String> agentIds) — lighter, no registry dependency, but callers must look up descriptors separately
- Per-query result types — proximity returns AgentMatch, activity returns agentIds, topology returns CollaboratorMatch with relationship metadata
**Rationale:** Consistent result type across all discovery queries. Callers always get descriptor + capability resolution context. AgentMatch is the established currency for "found agent" results.
**Trade-offs:** AgentGraphQuery and RuntimeCollaborationQuery need AgentRegistry access to produce AgentMatch. Collaboration query loses relationship metadata (NeighborRelation, affinity) — may need an extended result type.
**Sources:** AgentMatch.java, AgentRegistry.find() return type
**Exploration:** quick
**Depends on:** D1 (separate SPIs)
**Status:** captured
