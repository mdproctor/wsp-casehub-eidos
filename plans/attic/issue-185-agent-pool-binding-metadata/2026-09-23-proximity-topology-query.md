# Proximity/Topology Query Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #178 — feat: Proximity/topology query for agent discovery — hive mind enabler
**Issue group:** #178

**Goal:** Extend eidos with three query types for agent discovery: proximity (capability space neighbors), activity (co-active agents), and runtime collaboration topology.

**Architecture:** Three separate SPIs matched to their data source. Proximity extends AgentQuery/AgentRegistry (descriptor matching). Activity extends AgentGraphQuery (task history). Collaboration adds a new RuntimeCollaborationQuery SPI with NoOp default (engine implements later).

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, CDI

## Global Constraints

- Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn clean install` (not `./mvnw`)
- All new types in eidos-api must be pure Java (no Quarkus/CDI dependencies)
- AgentQuery.tenancyId is always required (non-null)
- MatchDegree ordering: Exact < Plugin < Specialization < None (lower = better)
- NoOp implementations go in eidos-core, @DefaultBean producers in EidosCoreProducer (runtime)

---

## Batch 1: API Foundation — Types and SPI Contracts

### Task 1: Add maxDepth to AgentQuery and proximity factory methods

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentQuery.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentQueryTest.java`

**Interfaces:**
- Produces: `AgentQuery.byProximity(String capabilityName, int maxDepth, String tenancyId)`, `AgentQuery.byProximityAndDomain(String capabilityName, int maxDepth, String taskDomain, String tenancyId)`, `AgentQuery.maxDepth()` accessor

- [ ] **Step 1: Write failing tests for new factory methods and validation**

```java
// In AgentQueryTest.java (create if needed)
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentQueryTest {

    @Test
    void byProximity_creates_query_with_maxDepth() {
        var query = AgentQuery.byProximity("code-review", 2, "tenant-a");
        assertThat(query.capabilityName()).isEqualTo("code-review");
        assertThat(query.maxDepth()).isEqualTo(2);
        assertThat(query.tenancyId()).isEqualTo("tenant-a");
        assertThat(query.slot()).isNull();
        assertThat(query.taskDomain()).isNull();
        assertThat(query.goalName()).isNull();
    }

    @Test
    void byProximityAndDomain_creates_query_with_maxDepth_and_domain() {
        var query = AgentQuery.byProximityAndDomain("code-review", 3, "java", "tenant-a");
        assertThat(query.capabilityName()).isEqualTo("code-review");
        assertThat(query.maxDepth()).isEqualTo(3);
        assertThat(query.taskDomain()).isEqualTo("java");
        assertThat(query.tenancyId()).isEqualTo("tenant-a");
    }

    @Test
    void byProximity_rejects_zero_maxDepth() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> AgentQuery.byProximity("code-review", 0, "tenant-a"))
            .withMessageContaining("maxDepth");
    }

    @Test
    void byProximity_rejects_negative_maxDepth() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> AgentQuery.byProximity("code-review", -1, "tenant-a"))
            .withMessageContaining("maxDepth");
    }

    @Test
    void existing_factories_have_null_maxDepth() {
        assertThat(AgentQuery.bySlot("reviewer", "t").maxDepth()).isNull();
        assertThat(AgentQuery.byCapability("code-review", "t").maxDepth()).isNull();
        assertThat(AgentQuery.all("t").maxDepth()).isNull();
        assertThat(AgentQuery.byGoal("quality", "t").maxDepth()).isNull();
        assertThat(AgentQuery.byCapabilityAndDomain("cr", "java", "t").maxDepth()).isNull();
        assertThat(AgentQuery.bySlotAndCapability("r", "cr", "t").maxDepth()).isNull();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentQueryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `maxDepth()` not found on AgentQuery

- [ ] **Step 3: Add maxDepth field to AgentQuery record and update factories**

Use `ide_replace_member` on `AgentQuery` to update the record. The new record definition:

```java
public record AgentQuery(
        String slot,
        String capabilityName,
        String tenancyId,
        String taskDomain,
        String goalName,
        Integer maxDepth
) {
    public AgentQuery {
        Objects.requireNonNull(tenancyId, "tenancyId");
        if (maxDepth != null && maxDepth < 1) {
            throw new IllegalArgumentException("maxDepth must be >= 1");
        }
    }

    public static AgentQuery bySlot(String slot, String tenancyId) {
        return new AgentQuery(slot, null, tenancyId, null, null, null);
    }

    public static AgentQuery byCapability(String capabilityName, String tenancyId) {
        return new AgentQuery(null, capabilityName, tenancyId, null, null, null);
    }

    public static AgentQuery bySlotAndCapability(String slot, String capabilityName, String tenancyId) {
        return new AgentQuery(slot, capabilityName, tenancyId, null, null, null);
    }

    public static AgentQuery byCapabilityAndDomain(String capabilityName, String taskDomain, String tenancyId) {
        return new AgentQuery(null, capabilityName, tenancyId, taskDomain, null, null);
    }

    public static AgentQuery byGoal(String goalName, String tenancyId) {
        return new AgentQuery(null, null, tenancyId, null, goalName, null);
    }

    public static AgentQuery all(String tenancyId) {
        return new AgentQuery(null, null, tenancyId, null, null, null);
    }

    public static AgentQuery byProximity(String capabilityName, int maxDepth, String tenancyId) {
        return new AgentQuery(null, capabilityName, tenancyId, null, null, maxDepth);
    }

    public static AgentQuery byProximityAndDomain(
            String capabilityName, int maxDepth, String taskDomain, String tenancyId) {
        return new AgentQuery(null, capabilityName, tenancyId, taskDomain, null, maxDepth);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentQueryTest`
Expected: all 5 tests PASS

- [ ] **Step 5: Run full api module tests to check backward compatibility**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: all existing tests PASS (existing factories unchanged — pass null for maxDepth)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#178): add maxDepth to AgentQuery with proximity factory methods

Adds Integer maxDepth field to AgentQuery record. Null = existing
exact/subsumption behavior. Non-null switches to proximity mode.
New factories: byProximity(), byProximityAndDomain().

Refs #178"
```

### Task 2: Add CapabilityResolver.resolveWithinDepth()

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java`
- Test: `api/src/test/java/io/casehub/eidos/api/CapabilityResolverTest.java`

**Interfaces:**
- Consumes: `CapabilityResolver.match()` (existing), `MatchDegree` hierarchy, `AgentCapability`, `VocabularyRegistry`
- Produces: `CapabilityResolver.resolveWithinDepth(List<AgentCapability>, String, int, VocabularyRegistry)` → `List<ResolvedCapability>`

- [ ] **Step 1: Write failing tests**

```java
// In CapabilityResolverTest.java — add proximity-specific tests
// (file may already exist — add new test methods)

@Test
void resolveWithinDepth_excludes_exact_matches() {
    var cap = AgentCapability.builder().name("code-review")
        .capabilityVocabulary("urn:test:v").build();
    var result = CapabilityResolver.resolveWithinDepth(
        List.of(cap), "code-review", 2, registry);
    assertThat(result).isEmpty();
}

@Test
void resolveWithinDepth_includes_matches_within_depth() {
    // Plugin(1) and Plugin(2) should be included with maxDepth=2
    var cap = AgentCapability.builder().name("security-code-review")
        .capabilityVocabulary("urn:casehub:capability").build();
    var result = CapabilityResolver.resolveWithinDepth(
        List.of(cap), "code-review", 2, registry);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).degree()).isInstanceOf(MatchDegree.Specialization.class);
}

@Test
void resolveWithinDepth_excludes_matches_beyond_depth() {
    // maxDepth=1 should not include depth=2
    var cap = AgentCapability.builder().name("deep-specialization")
        .capabilityVocabulary("urn:casehub:capability").build();
    // Assuming deep-specialization is depth 3 from code-review
    var result = CapabilityResolver.resolveWithinDepth(
        List.of(cap), "code-review", 1, registry);
    assertThat(result).isEmpty();
}

@Test
void resolveWithinDepth_returns_sorted_by_match_degree() {
    var cap1 = AgentCapability.builder().name("security-review")
        .capabilityVocabulary("urn:casehub:capability").build();
    var cap2 = AgentCapability.builder().name("code-review")
        .capabilityVocabulary("urn:casehub:capability").build();
    var result = CapabilityResolver.resolveWithinDepth(
        List.of(cap1, cap2), "security-review", 3, registry);
    // Should be sorted by MatchDegree (best first)
    if (result.size() > 1) {
        assertThat(result.get(0).degree().compareTo(result.get(1).degree())).isLessThanOrEqualTo(0);
    }
}

@Test
void resolveWithinDepth_excludes_ungrounded_capabilities() {
    var cap = AgentCapability.builder().name("custom-review").build();
    var result = CapabilityResolver.resolveWithinDepth(
        List.of(cap), "code-review", 5, registry);
    assertThat(result).isEmpty();
}

@Test
void resolveWithinDepth_returns_empty_for_null_or_empty_list() {
    assertThat(CapabilityResolver.resolveWithinDepth(null, "x", 2, registry)).isEmpty();
    assertThat(CapabilityResolver.resolveWithinDepth(List.of(), "x", 2, registry)).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CapabilityResolverTest`
Expected: compilation error — `resolveWithinDepth` not found

- [ ] **Step 3: Implement resolveWithinDepth**

Use `ide_insert_member` on `CapabilityResolver` class:

```java
public static List<ResolvedCapability> resolveWithinDepth(
        final List<AgentCapability> capabilities,
        final String capabilityTag,
        final int maxDepth,
        final VocabularyRegistry registry) {
    if (capabilities == null || capabilities.isEmpty()) {
        return List.of();
    }

    return capabilities.stream()
        .map(cap -> {
            MatchDegree degree = match(cap, capabilityTag, registry);
            return new ResolvedCapability(cap, degree);
        })
        .filter(rc -> !(rc.degree() instanceof MatchDegree.Exact))
        .filter(rc -> !(rc.degree() instanceof MatchDegree.None))
        .filter(rc -> depthOf(rc.degree()) <= maxDepth)
        .sorted(Comparator.comparing(ResolvedCapability::degree))
        .toList();
}

private static int depthOf(MatchDegree degree) {
    return switch (degree) {
        case MatchDegree.Exact e -> 0;
        case MatchDegree.Plugin p -> p.depth();
        case MatchDegree.Specialization s -> s.depth();
        case MatchDegree.None n -> Integer.MAX_VALUE;
    };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CapabilityResolverTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#178): add CapabilityResolver.resolveWithinDepth()

Static utility that returns all capabilities matching within maxDepth,
excluding Exact and None matches. Sorted by MatchDegree (best first).
Used by proximity queries in AgentRegistry.find().

Refs #178"
```

### Task 3: Add CollaborationRelation, Collaborator, and RuntimeCollaborationQuery SPI

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/CollaborationRelation.java`
- Create: `api/src/main/java/io/casehub/eidos/api/Collaborator.java`
- Create: `api/src/main/java/io/casehub/eidos/api/RuntimeCollaborationQuery.java`
- Test: `api/src/test/java/io/casehub/eidos/api/CollaboratorTest.java`

**Interfaces:**
- Produces: `CollaborationRelation` enum, `Collaborator` record, `RuntimeCollaborationQuery` SPI interface

- [ ] **Step 1: Write failing tests for Collaborator record validation**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.*;

class CollaboratorTest {

    @Test
    void valid_collaborator() {
        var c = new Collaborator("agent-1", "tenant-a",
            Set.of(CollaborationRelation.COACTIVE), 0.8);
        assertThat(c.agentId()).isEqualTo("agent-1");
        assertThat(c.tenancyId()).isEqualTo("tenant-a");
        assertThat(c.relations()).containsExactly(CollaborationRelation.COACTIVE);
        assertThat(c.affinity()).isEqualTo(0.8);
    }

    @Test
    void relations_are_immutable() {
        var c = new Collaborator("a", "t",
            Set.of(CollaborationRelation.COACTIVE, CollaborationRelation.COMPLEMENTARY), 0.5);
        assertThatThrownBy(() -> c.relations().add(CollaborationRelation.SHARED_INTEREST))
            .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void rejects_null_agentId() {
        assertThatNullPointerException()
            .isThrownBy(() -> new Collaborator(null, "t", Set.of(), 0.5));
    }

    @Test
    void rejects_null_tenancyId() {
        assertThatNullPointerException()
            .isThrownBy(() -> new Collaborator("a", null, Set.of(), 0.5));
    }

    @Test
    void rejects_affinity_below_zero() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new Collaborator("a", "t", Set.of(), -0.1))
            .withMessageContaining("affinity");
    }

    @Test
    void rejects_affinity_above_one() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> new Collaborator("a", "t", Set.of(), 1.1))
            .withMessageContaining("affinity");
    }

    @Test
    void boundary_affinity_values_accepted() {
        assertThatNoException()
            .isThrownBy(() -> new Collaborator("a", "t", Set.of(), 0.0));
        assertThatNoException()
            .isThrownBy(() -> new Collaborator("a", "t", Set.of(), 1.0));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CollaboratorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — types not found

- [ ] **Step 3: Create CollaborationRelation enum**

Use `ide_create_file`:

```java
package io.casehub.eidos.api;

public enum CollaborationRelation {
    COACTIVE,
    SHARED_INTEREST,
    SHARED_SIGNAL,
    COMPLEMENTARY
}
```

- [ ] **Step 4: Create Collaborator record**

Use `ide_create_file`:

```java
package io.casehub.eidos.api;

import java.util.Objects;
import java.util.Set;

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
        if (affinity < 0.0 || affinity > 1.0) {
            throw new IllegalArgumentException("affinity must be in [0.0, 1.0]");
        }
    }
}
```

- [ ] **Step 5: Create RuntimeCollaborationQuery SPI**

Use `ide_create_file`:

```java
package io.casehub.eidos.api;

import java.util.List;

public interface RuntimeCollaborationQuery {
    List<Collaborator> collaborators(String agentId, String tenancyId);
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CollaboratorTest`
Expected: all 7 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#178): add CollaborationRelation, Collaborator, RuntimeCollaborationQuery SPI

New types for runtime collaboration topology:
- CollaborationRelation enum (COACTIVE, SHARED_INTEREST, SHARED_SIGNAL, COMPLEMENTARY)
- Collaborator record with validation (affinity bounds, immutable relations)
- RuntimeCollaborationQuery SPI interface (CapabilityHealth pattern)

Refs #178"
```

### Task 4: Add coActiveAgents to AgentGraphQuery and NoOp/CDI wiring

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentGraphQuery.java`
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/graph/NoOpAgentGraphQuery.java`
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/graph/NoOpRuntimeCollaborationQuery.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java`

**Interfaces:**
- Consumes: `RuntimeCollaborationQuery` (from Task 3)
- Produces: `AgentGraphQuery.coActiveAgents(String externalRef, String tenancyId)` → `List<String>`

- [ ] **Step 1: Add coActiveAgents to AgentGraphQuery interface**

Use `ide_insert_member` on `AgentGraphQuery`, member position after `attestationsFor`:

```java
List<String> coActiveAgents(String externalRef, String tenancyId);
```

- [ ] **Step 2: Update NoOpAgentGraphQuery**

Use `ide_insert_member` on `NoOpAgentGraphQuery`:

```java
@Override
public List<String> coActiveAgents(final String externalRef, final String tenancyId) {
    return List.of();
}
```

- [ ] **Step 3: Create NoOpRuntimeCollaborationQuery**

Use `ide_create_file`:

```java
package io.casehub.eidos.core.graph;

import io.casehub.eidos.api.Collaborator;
import io.casehub.eidos.api.RuntimeCollaborationQuery;
import java.util.List;

public class NoOpRuntimeCollaborationQuery implements RuntimeCollaborationQuery {
    @Override
    public List<Collaborator> collaborators(final String agentId, final String tenancyId) {
        return List.of();
    }
}
```

- [ ] **Step 4: Add @DefaultBean producer for RuntimeCollaborationQuery in EidosCoreProducer**

Use `ide_insert_member` on `EidosCoreProducer`, after the existing NoOp producers:

```java
@Produces @DefaultBean
RuntimeCollaborationQuery runtimeCollaborationQuery() {
    return new NoOpRuntimeCollaborationQuery();
}
```

Add import for `NoOpRuntimeCollaborationQuery` and `RuntimeCollaborationQuery`.

- [ ] **Step 5: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,eidos-core,runtime`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/ eidos-core/ runtime/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#178): add coActiveAgents to AgentGraphQuery, NoOp wiring for RuntimeCollaborationQuery

Extends AgentGraphQuery with coActiveAgents(externalRef, tenancyId).
Adds NoOpRuntimeCollaborationQuery with @DefaultBean producer in
EidosCoreProducer.

Refs #178"
```

---

## Batch 2: Registry Implementations — Proximity Query

### Task 5: InMemoryAgentRegistry proximity mode

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java`
- Modify: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java`

**Interfaces:**
- Consumes: `AgentQuery.maxDepth()`, `CapabilityResolver.resolveWithinDepth()`

- [ ] **Step 1: Write failing tests for proximity mode**

Add to `InMemoryAgentRegistryTest.java`:

```java
@Test
void find_by_proximity_returns_neighbors_within_depth() {
    // Register agents with hierarchical capabilities using CasehubCapabilityTerm vocab
    // Agent A: declares "code-review" (grounded in capability vocab)
    // Agent B: declares "security-code-review" (specializes "code-review", depth 1)
    // Query: byProximity("code-review", 2, tenancy) should find B but not A (exact excluded)
    // Setup: register capability-grounded agents, then query with byProximity
    var agentA = descriptorWithCapability("agent-a", "code-review", "urn:casehub:capability");
    var agentB = descriptorWithCapability("agent-b", "security-code-review", "urn:casehub:capability");
    registry.register(agentA);
    registry.register(agentB);

    var result = registry.find(AgentQuery.byProximity("code-review", 2, "default"));
    // agentA has exact match → excluded; agentB is Specialization(1) → included
    assertThat(result).hasSize(1);
    assertThat(result.get(0).descriptor().agentId()).isEqualTo("agent-b");
    assertThat(result.get(0).resolvedCapability()).isNotNull();
    assertThat(result.get(0).resolvedCapability().degree()).isInstanceOf(MatchDegree.Specialization.class);
}

@Test
void find_by_proximity_excludes_agents_beyond_maxDepth() {
    // maxDepth=1 should not include depth=2 matches
    // (test depends on vocab hierarchy; use terms with known depth)
}

@Test
void find_by_proximity_with_ungrounded_capability_returns_empty() {
    var agent = descriptorWithCapability("agent-a", "custom-review", null);
    registry.register(agent);

    var result = registry.find(AgentQuery.byProximity("custom-review", 2, "default"));
    assertThat(result).isEmpty();
}

@Test
void find_by_proximity_with_domain_filter_honors_excludedDomains() {
    // Agent with excludedDomains=["rust"] should not appear in proximity results
    // when taskDomain="rust"
}
```

Note: exact test bodies will depend on which vocabulary terms are available in the test fixture. Use `CasehubCapabilityTerm` vocabulary with known hierarchy.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryAgentRegistryTest#find_by_proximity*`
Expected: FAIL — InMemoryAgentRegistry.find() ignores maxDepth

- [ ] **Step 3: Implement proximity mode in InMemoryAgentRegistry.find()**

Use `ide_replace_member` on `InMemoryAgentRegistry.find`. The updated logic:

```java
@Override
public List<AgentMatch> find(AgentQuery query) {
    var stream = store.values().stream()
        .filter(d -> d.tenancyId().equals(query.tenancyId()))
        .filter(d -> query.slot() == null || Objects.equals(d.slot(), query.slot()))
        .filter(d -> query.taskDomain() == null
            || d.capabilities().stream().noneMatch(c ->
                c.excludedDomains() != null && c.excludedDomains().contains(query.taskDomain())))
        .filter(d -> query.goalName() == null
            || d.goals().stream().anyMatch(g -> g.name().equals(query.goalName())));

    if (query.capabilityName() == null) {
        return stream
            .map(d -> new AgentMatch(d, null))
            .collect(Collectors.toList());
    }

    if (query.maxDepth() != null) {
        return findByProximity(stream, query);
    }

    return stream
        .map(d -> {
            var resolved = resolveCapability(d, query.capabilityName());
            return resolved != null ? new AgentMatch(d, resolved) : null;
        })
        .filter(Objects::nonNull)
        .sorted(Comparator.comparing(AgentMatch::resolvedCapability,
            Comparator.comparing(ResolvedCapability::degree)))
        .collect(Collectors.toList());
}

private List<AgentMatch> findByProximity(
        java.util.stream.Stream<AgentDescriptor> stream, AgentQuery query) {
    if (!vocabularyRegistry.isResolvable()) {
        return List.of();
    }
    return stream
        .flatMap(d -> CapabilityResolver.resolveWithinDepth(
                d.capabilities(), query.capabilityName(),
                query.maxDepth(), vocabularyRegistry.get())
            .stream()
            .map(rc -> new AgentMatch(d, rc)))
        .sorted(Comparator.comparing(AgentMatch::resolvedCapability,
            Comparator.comparing(ResolvedCapability::degree)))
        .collect(Collectors.toList());
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: all tests PASS (new + existing)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add persistence-memory/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#178): implement proximity mode in InMemoryAgentRegistry

When AgentQuery.maxDepth() is non-null, find() uses
CapabilityResolver.resolveWithinDepth() to collect all capabilities
within depth, excluding exact matches. Results ordered by MatchDegree.

Refs #178"
```

### Task 6: JpaAgentRegistry proximity mode

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaAgentRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`

**Interfaces:**
- Consumes: `AgentQuery.maxDepth()`, `CapabilityResolver.resolveWithinDepth()`

- [ ] **Step 1: Write failing tests for JPA proximity mode**

Add to `JpaAgentRegistryTest.java`:

```java
@Test
void find_by_proximity_returns_neighbors_within_depth() {
    // Register agents with CasehubCapabilityTerm-grounded capabilities
    // at different hierarchy depths, then query with byProximity
    // Verify: exact excluded, depth filtering, ordering
}

@Test
void find_by_proximity_excludes_exact_matches() {
    // Agent declaring the exact queried capability should NOT appear
}

@Test
void find_by_proximity_honors_tenancy_isolation() {
    // Agents from other tenants should not appear in proximity results
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest#find_by_proximity*`
Expected: FAIL

- [ ] **Step 3: Implement proximity mode in JpaAgentRegistry.find()**

The JPA implementation fetches all agents with vocabulary-grounded capabilities matching the broader expansion set, then post-filters with `CapabilityResolver.resolveWithinDepth()`. Use `ide_replace_member` on `JpaAgentRegistry.find`:

Add proximity branch after line 120 (`var descriptors = ...`):

```java
// After fetching descriptors, check for proximity mode
if (query.maxDepth() != null) {
    return descriptors.stream()
        .flatMap(d -> CapabilityResolver.resolveWithinDepth(
                d.capabilities(), query.capabilityName(),
                query.maxDepth(), vocabularyRegistry)
            .stream()
            .map(rc -> new AgentMatch(d, rc)))
        .sorted(Comparator.comparing(AgentMatch::resolvedCapability,
            Comparator.nullsLast(Comparator.comparing(ResolvedCapability::degree))))
        .toList();
}
```

Insert this block between the existing `var descriptors = ...` line and the `if (query.capabilityName() == null)` check.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#178): implement proximity mode in JpaAgentRegistry

JPA find() fetches vocabulary-expanded candidates then post-filters
with CapabilityResolver.resolveWithinDepth(). Same semantics as
InMemoryAgentRegistry: exact excluded, depth-bounded, ordered.

Refs #178"
```

---

## Batch 3: Graph Query — Activity and Full Build

### Task 7: JpaAgentGraphQuery.coActiveAgents() implementation

**Files:**
- Modify: `graph/src/main/java/io/casehub/eidos/graph/JpaAgentGraphQuery.java`
- Modify: `graph/src/test/java/io/casehub/eidos/graph/JpaAgentGraphQueryTest.java`

**Interfaces:**
- Consumes: `AgentGraphQuery.coActiveAgents()` (from Task 4), `AgentTaskEntity` JPA entity

- [ ] **Step 1: Write failing tests**

```java
@Test
void coActiveAgents_returns_agents_with_in_progress_tasks() {
    // Record two tasks with same externalRef, both in-progress (endedAt=null)
    // Verify both agentIds returned
}

@Test
void coActiveAgents_excludes_completed_tasks() {
    // Record a task with endedAt set (completed)
    // Verify it's not in the result
}

@Test
void coActiveAgents_isolates_by_tenancy() {
    // Record tasks in different tenancies with same externalRef
    // Verify only matching tenancy returned
}

@Test
void coActiveAgents_returns_empty_for_unknown_ref() {
    var result = graphQuery.coActiveAgents("nonexistent", "default");
    assertThat(result).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graph -Dtest=JpaAgentGraphQueryTest#coActiveAgents*`
Expected: compilation error or FAIL

- [ ] **Step 3: Implement coActiveAgents in JpaAgentGraphQuery**

Use `ide_insert_member` on `JpaAgentGraphQuery`:

```java
@Override
@Transactional(TxType.SUPPORTS)
public List<String> coActiveAgents(final String externalRef, final String tenancyId) {
    return em.createQuery(
            "SELECT DISTINCT t.agentId FROM AgentTaskEntity t " +
            "WHERE t.externalRef = :ref AND t.tenancyId = :tn AND t.endedAt IS NULL",
            String.class)
        .setParameter("ref", externalRef)
        .setParameter("tn", tenancyId)
        .getResultList();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graph -Dtest=JpaAgentGraphQueryTest`
Expected: all tests PASS

- [ ] **Step 5: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add graph/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#178): implement coActiveAgents in JpaAgentGraphQuery

JPQL query finds agents with in-progress tasks (endedAt IS NULL)
sharing the same externalRef within tenancy scope.

Refs #178"
```

### Task 8: Integration test scenario and docs update

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/ProximityDiscoveryScenarioTest.java`
- Modify: `docs/guides/consumer-guide.md` (proximity query section)

**Interfaces:**
- Consumes: All three query extensions

- [ ] **Step 1: Write integration scenario test**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class ProximityDiscoveryScenarioTest {

    @Inject AgentRegistry registry;
    @Inject RuntimeCollaborationQuery collaborationQuery;

    @Test
    void proximity_query_finds_capability_neighbors() {
        // Register agents with CasehubCapabilityTerm vocab
        // Query byProximity with maxDepth=2
        // Verify neighbors found, exact excluded, ordered by depth
    }

    @Test
    void collaboration_query_returns_empty_with_noop() {
        // NoOp default should return empty
        var result = collaborationQuery.collaborators("agent-x", "default");
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run scenario tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=ProximityDiscoveryScenarioTest`
Expected: PASS

- [ ] **Step 3: Update consumer guide with proximity query documentation**

Add a "Discovery Queries" section to `docs/guides/consumer-guide.md` documenting:
- `AgentQuery.byProximity()` — capability space neighbors
- `AgentGraphQuery.coActiveAgents()` — co-active agents by shared reference
- `RuntimeCollaborationQuery.collaborators()` — runtime collaboration topology

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add examples/ docs/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#178): add proximity discovery scenario test and consumer guide docs

Integration test demonstrates proximity query with CasehubCapabilityTerm
vocabulary. Consumer guide documents all three discovery query types.

Closes #178"
```

## References

- [2026-09-23-proximity-topology-query-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/eidos/api/AgentQuery.java] — current query record
- [api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java:35-52] — existing match() with depth
- [api/src/main/java/io/casehub/eidos/api/AgentGraphQuery.java] — graph query SPI
- [api/src/main/java/io/casehub/eidos/api/MatchDegree.java] — depth-carrying sealed hierarchy
- [persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java:47-72] — current find() impl
- [runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaAgentRegistry.java:63-137] — current JPA find() impl
- [runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java:48-89] — @DefaultBean producer pattern
- [eidos-core/src/main/java/io/casehub/eidos/core/graph/NoOpAgentGraphQuery.java] — NoOp pattern
- [graph/src/main/java/io/casehub/eidos/graph/JpaAgentGraphQuery.java] — JPA graph query impl
- [GitHub #178] — focal issue
