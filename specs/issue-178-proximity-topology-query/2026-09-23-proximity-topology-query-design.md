# Proximity/Topology Query for Agent Discovery

**Issue:** casehubio/eidos#178
**Parent epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-23

## Problem

AgentQuery supports exact matching: by slot, capability name, goal, and domain. There is no way to ask "who's similar to me?", "who else is working on the same thing?", or "who am I collaborating with right now?"

These three questions require fundamentally different data sources:
- **Proximity** = capability hierarchy distance (descriptor matching)
- **Activity** = shared runtime task state (graph data)
- **Collaboration** = emergent team detection (engine stigmergy, exposed via SPI)

## Design

Three extensions, each in its natural SPI home. No unified discovery facade — consumers compose the queries they need.

### 1. Proximity Query — AgentQuery / AgentRegistry

Extend AgentQuery with a proximity mode that returns agents within a given depth in the capability vocabulary hierarchy.

**New types:**

```java
// AgentQuery — new factory method
public static AgentQuery byProximity(String capabilityName, int maxDepth, String tenancyId) {
    // maxDepth: maximum MatchDegree depth to include
    // Exact matches excluded (they ARE the queried capability, not neighbors)
}
```

**Semantics:**
- `maxDepth=1` → Plugin(1), Specialization(1) only
- `maxDepth=2` → Plugin(1), Plugin(2), Specialization(1), Specialization(2)
- `MatchDegree.Exact` is excluded — proximity means "nearby", not "identical"
- `MatchDegree.None` is always excluded
- Results ordered by match quality (best first, per existing OWLS-MX ordering)
- Only vocabulary-grounded capabilities participate — ungrounded capabilities can only match exactly, so they have no proximity concept
- `taskDomain` filtering still applies (excludedDomains honored)

**AgentQuery record changes:**

Add `maxDepth` field (nullable Integer). When non-null, switches `AgentRegistry.find()` into proximity mode:
- Instead of finding the single best-matching capability per agent, collect ALL capabilities within maxDepth
- Exclude Exact matches
- Each match produces an `AgentMatch` with the matching `ResolvedCapability`
- An agent with multiple capabilities within range appears once per matching capability

```java
public record AgentQuery(
    String slot,
    String capabilityName,
    String tenancyId,
    String taskDomain,
    String goalName,
    Integer maxDepth          // new — null = exact/subsumption (current behavior)
) {
    // existing factories unchanged (pass null for maxDepth)

    public static AgentQuery byProximity(String capabilityName, int maxDepth, String tenancyId) {
        if (maxDepth < 1) throw new IllegalArgumentException("maxDepth must be >= 1");
        return new AgentQuery(null, capabilityName, tenancyId, null, null, maxDepth);
    }

    public static AgentQuery byProximityAndDomain(
            String capabilityName, int maxDepth, String taskDomain, String tenancyId) {
        if (maxDepth < 1) throw new IllegalArgumentException("maxDepth must be >= 1");
        return new AgentQuery(null, capabilityName, tenancyId, taskDomain, null, maxDepth);
    }
}
```

**Implementation in JpaAgentRegistry and InMemoryAgentRegistry:**

Both implementations already call `CapabilityResolver.resolve()` which returns `MatchDegree` with depth. The change:
- When `maxDepth != null`: iterate ALL capabilities per agent (not just best match), collect those with depth ≤ maxDepth, exclude Exact, exclude None
- Sort results by MatchDegree (OWLS-MX ordering)

**CapabilityResolver changes:**

Add a static utility:

```java
public static List<ResolvedCapability> resolveWithinDepth(
        List<AgentCapability> capabilities,
        String capabilityTag,
        int maxDepth,
        VocabularyRegistry registry) {
    // Returns all capabilities matching within maxDepth, excluding Exact and None
    // Sorted by MatchDegree (best first)
}
```

### 2. Activity Query — AgentGraphQuery

Add a cross-agent activity query to the existing `AgentGraphQuery` SPI.

**New method:**

```java
public interface AgentGraphQuery {
    // ... existing methods ...

    /**
     * Returns agentIds with in-progress tasks sharing the given externalRef.
     * "In-progress" means endedAt is null.
     *
     * @param externalRef the shared external reference (e.g., case ID as string)
     * @param tenancyId tenant scope
     * @return agentIds of co-active agents, empty if none
     */
    List<String> coActiveAgents(String externalRef, String tenancyId);
}
```

**Return type:** `List<String>` — consistent with existing `topAgentsByOutcome()`. Callers compose with `AgentRegistry.findById()` when they need descriptors.

**JpaAgentGraphQuery implementation:**

```sql
SELECT DISTINCT t.agentId
FROM AgentTaskEntity t
WHERE t.externalRef = :ref
  AND t.tenancyId = :tn
  AND t.endedAt IS NULL
```

**NoOpAgentGraphQuery:** Returns `List.of()`.

**No schema change** — the query uses existing `agent_task` columns (externalRef, tenancyId, endedAt).

### 3. Runtime Collaboration Query — new SPI

New SPI in eidos-api for querying runtime collaboration topology. Follows the CapabilityHealth pattern.

**New types in eidos-api:**

```java
package io.casehub.eidos.api;

public enum CollaborationRelation {
    COACTIVE,          // working on the same case simultaneously
    SHARED_INTEREST,   // attending to the same context keys
    SHARED_SIGNAL,     // depositing/reading the same signals
    COMPLEMENTARY      // filling different roles in the same team
}
```

```java
package io.casehub.eidos.api;

public record Collaborator(
    String agentId,
    String tenancyId,
    Set<CollaborationRelation> relations,
    double affinity
) {
    public Collaborator {
        Objects.requireNonNull(agentId, "agentId");
        Objects.requireNonNull(tenancyId, "tenancyId");
        relations = Set.copyOf(relations);
        if (affinity < 0.0 || affinity > 1.0)
            throw new IllegalArgumentException("affinity must be in [0.0, 1.0]");
    }
}
```

```java
package io.casehub.eidos.api;

public interface RuntimeCollaborationQuery {
    /**
     * Returns agents currently collaborating with the given agent.
     * "Currently" means across all active cases/contexts.
     *
     * @param agentId the focal agent
     * @param tenancyId tenant scope
     * @return collaborators with relationship metadata, empty if none detected
     */
    List<Collaborator> collaborators(String agentId, String tenancyId);
}
```

**NoOp implementation in eidos-core:**

```java
package io.casehub.eidos.core.graph;

public class NoOpRuntimeCollaborationQuery implements RuntimeCollaborationQuery {
    @Override
    public List<Collaborator> collaborators(String agentId, String tenancyId) {
        return List.of();
    }
}
```

**CDI wiring:** `@DefaultBean` in runtime module (same pattern as `NoOpAgentStateStore`).

**Engine implementation (out of scope for this issue):** Engine would provide an `@Alternative @Priority(1)` implementation that aggregates `TeamDetector.getDetectedTeams()` across active cases, maps engine's `NeighborRelation` → eidos's `CollaborationRelation`, and computes per-agent collaborator lists.

### What's NOT in scope

| Concept | Why excluded | Where it lives |
|---------|-------------|---------------|
| Static org team membership | Already handled by OrgRegistry.unitsFor() + membersOf() | eidos-org |
| Channel presence (online/offline) | Orthogonal to discovery | qhorus PresenceTracker |
| Engine's TeamDetector implementation | Engine-side work, separate issue | engine#1104 epic |
| Unified DiscoveryService facade | Premature — consumers can compose the three queries | Future if needed |
| Result ordering by similarity score | Proximity already ordered by MatchDegree. Activity has no natural ordering. Collaboration has affinity. | Per-query semantics sufficient |

## Module Impact

| Module | Changes |
|--------|---------|
| **api** | Add `maxDepth` to AgentQuery, `byProximity`/`byProximityAndDomain` factories. Add `CapabilityResolver.resolveWithinDepth()`. Add `coActiveAgents()` to AgentGraphQuery. Add `CollaborationRelation`, `Collaborator`, `RuntimeCollaborationQuery`. |
| **runtime** | Update JpaAgentRegistry to handle proximity mode in `find()`. |
| **persistence-memory** | Update InMemoryAgentRegistry to handle proximity mode in `find()`. |
| **eidos-core** | Update NoOpAgentGraphQuery with `coActiveAgents()`. Add `NoOpRuntimeCollaborationQuery`. |
| **graph** | Update JpaAgentGraphQuery with `coActiveAgents()` JPA query. |
| **annotations/deployment** | No change — proximity is a query-time concept, not a descriptor declaration. |
| **vocab** | No change. |
| **routing** | No change — EngineAwareAgentSelector works with whatever AgentRegistry.find() returns. |

## Testing Strategy

**Unit tests (api):**
- AgentQuery.byProximity validation (maxDepth >= 1)
- CapabilityResolver.resolveWithinDepth — exact excluded, depth filtering, ordering
- Collaborator record validation (affinity bounds, immutable relations set)

**Integration tests (runtime, persistence-memory):**
- Proximity: register agents with hierarchical capabilities (CasehubCapabilityTerm), query with different maxDepth values, verify depth filtering and ordering
- Proximity + domain: verify excludedDomains still honored in proximity mode
- Proximity with ungrounded capabilities: verify they don't match in proximity mode

**Integration tests (graph):**
- coActiveAgents: record tasks with shared externalRef, verify co-active list; verify completed tasks (endedAt != null) excluded; verify tenancy isolation

**Unit tests (eidos-core):**
- NoOpRuntimeCollaborationQuery returns empty list
- NoOpAgentGraphQuery.coActiveAgents returns empty list

## Migration

No database migration needed. All new queries use existing columns.

## References

- AgentQuery.java — current query criteria record
- AgentRegistry.java:22 — find() method contract
- CapabilityResolver.java:35-52 — existing match() with MatchDegree depth
- AgentGraphQuery.java — existing graph query SPI
- AgentTask.java:11 — externalRef field
- CapabilityHealth.java — SPI pattern reference (eidos defines, engine implements)
- engine DetectedTeam.java — runtime team detection result type
- engine NeighborRelation.java — relationship types (COACTIVE, SHARED_INTEREST, SHARED_SIGNAL, COMPLEMENTARY)
- engine MetricsSpace.java:37 — detectedTeams() exposure (per-case, agent-scoped only)
- engine TeamDetector.java — Jaccard similarity team detection (internal)
- blocks CbrAgentRoutingStrategy.java — primary consumer of AgentGraphQuery
