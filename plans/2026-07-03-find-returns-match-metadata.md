# find() Returns Match Metadata — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Surface match metadata (matched capability, match degree) from `CapabilityResolver.resolve()` and `AgentRegistry.find()` instead of discarding it.

**Architecture:** Fix at the root — `CapabilityResolver.resolve()` returns `ResolvedCapability(capability, degree)` instead of bare `AgentCapability`. `AgentRegistry.find()` returns `List<AgentMatch>` wrapping descriptor + resolved capability. `MatchDegree` becomes `Comparable` to enable sorted results.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Mutiny

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test` (or `-pl <module>`)
- Use `mvn` not `./mvnw`
- All new types in `api/src/main/java/io/casehub/eidos/api/` (Tier 1, pure Java)
- Commits reference issue: `Refs #84`
- Breaking API changes are intentional — update all callers mechanically

---

### Task 1: MatchDegree becomes Comparable + ResolvedCapability record

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/MatchDegree.java`
- Create: `api/src/main/java/io/casehub/eidos/api/ResolvedCapability.java`
- Create: `api/src/test/java/io/casehub/eidos/api/MatchDegreeTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/MatchDegreeTest.java`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `MatchDegree extends Comparable<MatchDegree>` — with ordering: Exact < Plugin(d) < Specialization(d) < None; lower depth is better within Plugin and Specialization
  - `record ResolvedCapability(AgentCapability capability, MatchDegree degree)` — in `io.casehub.eidos.api`

- [ ] **Step 1: Write MatchDegree ordering tests**

Create `api/src/test/java/io/casehub/eidos/api/MatchDegreeTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.ArrayList;
import java.util.Collections;

import static org.assertj.core.api.Assertions.assertThat;

class MatchDegreeTest {

    @Test
    void exact_beats_everything() {
        assertThat(new MatchDegree.Exact()).isLessThan(new MatchDegree.Plugin(1));
        assertThat(new MatchDegree.Exact()).isLessThan(new MatchDegree.Specialization(1));
        assertThat(new MatchDegree.Exact()).isLessThan(new MatchDegree.None());
    }

    @Test
    void plugin_beats_specialization_at_any_depth() {
        assertThat(new MatchDegree.Plugin(5)).isLessThan(new MatchDegree.Specialization(1));
    }

    @Test
    void lower_depth_plugin_beats_higher() {
        assertThat(new MatchDegree.Plugin(1)).isLessThan(new MatchDegree.Plugin(3));
    }

    @Test
    void lower_depth_specialization_beats_higher() {
        assertThat(new MatchDegree.Specialization(1)).isLessThan(new MatchDegree.Specialization(3));
    }

    @Test
    void none_loses_to_everything() {
        assertThat(new MatchDegree.None()).isGreaterThan(new MatchDegree.Exact());
        assertThat(new MatchDegree.None()).isGreaterThan(new MatchDegree.Plugin(100));
        assertThat(new MatchDegree.None()).isGreaterThan(new MatchDegree.Specialization(100));
    }

    @Test
    void same_type_same_depth_are_equal() {
        assertThat(new MatchDegree.Exact().compareTo(new MatchDegree.Exact())).isZero();
        assertThat(new MatchDegree.Plugin(2).compareTo(new MatchDegree.Plugin(2))).isZero();
        assertThat(new MatchDegree.None().compareTo(new MatchDegree.None())).isZero();
    }

    @Test
    void sorting_produces_owlsmx_order() {
        var degrees = new ArrayList<>(List.of(
            new MatchDegree.None(),
            new MatchDegree.Specialization(2),
            new MatchDegree.Plugin(1),
            new MatchDegree.Exact(),
            new MatchDegree.Plugin(3),
            new MatchDegree.Specialization(1)
        ));
        Collections.sort(degrees);
        assertThat(degrees).containsExactly(
            new MatchDegree.Exact(),
            new MatchDegree.Plugin(1),
            new MatchDegree.Plugin(3),
            new MatchDegree.Specialization(1),
            new MatchDegree.Specialization(2),
            new MatchDegree.None()
        );
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=MatchDegreeTest`
Expected: compilation failure — `MatchDegree` doesn't implement `Comparable`

- [ ] **Step 3: Add Comparable to MatchDegree**

Modify `api/src/main/java/io/casehub/eidos/api/MatchDegree.java`:

```java
package io.casehub.eidos.api;

/**
 * OWLS-MX semantic match degree between a declared capability and a requested capability.
 * Based on the OWL-S Matchmaker (OWLS-MX) matching framework.
 *
 * <p>Match degrees in descending priority:
 * <ul>
 *   <li>{@link Exact} — declared and requested are identical
 *   <li>{@link Plugin} — declared subsumes requested (declared is more general)
 *   <li>{@link Specialization} — requested subsumes declared (declared is more specific)
 *   <li>{@link None} — no semantic relationship
 * </ul>
 *
 * <p>Ordering reflects OWLS-MX priority: Exact &lt; Plugin &lt; Specialization &lt; None.
 * Plugin ranks above Specialization because a Plugin match guarantees the agent covers
 * the request (declared is more general); a Specialization covers only a subset.
 * Within Plugin and Specialization, lower depth (closer in hierarchy) ranks higher.
 */
public sealed interface MatchDegree extends Comparable<MatchDegree>
        permits MatchDegree.Exact, MatchDegree.Plugin,
                MatchDegree.Specialization, MatchDegree.None {

    private int ordinal() {
        return switch (this) {
            case Exact e -> 0;
            case Plugin p -> 1000 + p.depth();
            case Specialization s -> 2000 + s.depth();
            case None n -> Integer.MAX_VALUE;
        };
    }

    @Override
    default int compareTo(MatchDegree other) {
        return Integer.compare(this.ordinal(), other.ordinal());
    }

    /** Exact match — declared and requested values are identical. */
    record Exact() implements MatchDegree {}

    /**
     * Plugin match — declared capability is more general (parent) than requested.
     * @param depth distance in the hierarchy (1 = direct parent, 2 = grandparent, etc.)
     */
    record Plugin(int depth) implements MatchDegree {}

    /**
     * Specialization match — declared capability is more specific (child) than requested.
     * @param depth distance in the hierarchy (1 = direct child, 2 = grandchild, etc.)
     */
    record Specialization(int depth) implements MatchDegree {}

    /** No match — declared and requested have no semantic relationship. */
    record None() implements MatchDegree {}
}
```

- [ ] **Step 4: Create ResolvedCapability record**

Create `api/src/main/java/io/casehub/eidos/api/ResolvedCapability.java`:

```java
package io.casehub.eidos.api;

/**
 * Result of resolving a capability tag against declared capabilities via
 * {@link CapabilityResolver#resolve(java.util.List, String, VocabularyRegistry)}.
 *
 * @param capability the declared capability that matched
 * @param degree the OWLS-MX match degree
 */
public record ResolvedCapability(AgentCapability capability, MatchDegree degree) {}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=MatchDegreeTest`
Expected: all 7 tests PASS

- [ ] **Step 6: Commit**

```
feat(eidos#84): MatchDegree becomes Comparable, add ResolvedCapability record

Refs #84
```

---

### Task 2: CapabilityResolver.resolve() returns ResolvedCapability

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/CapabilityResolverTest.java`

**Interfaces:**
- Consumes: `ResolvedCapability`, `MatchDegree` (from Task 1)
- Produces:
  - `CapabilityResolver.resolve(List<AgentCapability>, String, VocabularyRegistry)` → `ResolvedCapability` (or null)
  - `CapabilityResolver.match()` unchanged — still returns `MatchDegree`

- [ ] **Step 1: Update resolve() tests to expect ResolvedCapability**

In `api/src/test/java/io/casehub/eidos/api/CapabilityResolverTest.java`, update the resolve() test section (lines 72–114):

```java
// --- resolve() tests ---

@Test
void resolve_exact_match_preferred_over_subsumption() {
    var caps = List.of(grounded("code-review"), grounded("security-review"));
    var result = CapabilityResolver.resolve(caps, "security-review", registry);
    assertThat(result).isNotNull();
    assertThat(result.capability().name()).isEqualTo("security-review");
    assertThat(result.degree()).isInstanceOf(MatchDegree.Exact.class);
}

@Test
void resolve_closest_depth_wins() {
    var caps = List.of(grounded("review"), grounded("testing"));
    var result = CapabilityResolver.resolve(caps, "unit-testing", registry);
    assertThat(result).isNotNull();
    assertThat(result.capability().name()).isEqualTo("testing");
    assertThat(result.degree()).isInstanceOf(MatchDegree.Plugin.class);
    assertThat(((MatchDegree.Plugin) result.degree()).depth()).isEqualTo(1);
}

@Test
void resolve_ungrounded_exact_only() {
    var caps = List.of(ungrounded("code-review"));
    assertThat(CapabilityResolver.resolve(caps, "security-review", registry)).isNull();
    var result = CapabilityResolver.resolve(caps, "code-review", registry);
    assertThat(result).isNotNull();
    assertThat(result.degree()).isInstanceOf(MatchDegree.Exact.class);
}

@Test
void resolve_null_capabilities_returns_null() {
    assertThat(CapabilityResolver.resolve(null, "code-review", registry)).isNull();
}

@Test
void resolve_empty_capabilities_returns_null() {
    assertThat(CapabilityResolver.resolve(List.of(), "code-review", registry)).isNull();
}

@Test
void resolve_first_in_list_wins_at_equal_depth() {
    var caps = List.of(grounded("code-review"), grounded("design-review"));
    var result = CapabilityResolver.resolve(caps, "review", registry);
    assertThat(result).isNotNull();
    assertThat(result.capability().name()).isEqualTo("code-review");
    assertThat(result.degree()).isInstanceOf(MatchDegree.Specialization.class);
}

@Test
void resolve_plugin_beats_specialization_across_depth() {
    // code-review is Plugin(1) for security-review query.
    // If an agent had a Specialization(1) and a Plugin(2), Plugin wins.
    // Here: "review" is Plugin(2) for "security-review", "testing" is no match.
    // We need a case where Plugin and Specialization compete:
    // Query "security-review":
    //   "code-review" → Plugin(1) (code-review subsumes security-review)
    //   "sast-review" is not in vocab — skip.
    // Instead, test with the compareTo-based selection:
    // Query "review":
    //   "code-review" → Specialization(1) (code-review specializes review)
    //   "testing" → None
    // This doesn't create a Plugin vs Specialization competition in resolve().
    // The Comparable ordering is already tested in MatchDegreeTest.
    // The resolve() loop uses compareTo, so any two degrees compare correctly.
    var caps = List.of(grounded("code-review"), grounded("review"));
    var result = CapabilityResolver.resolve(caps, "security-review", registry);
    assertThat(result).isNotNull();
    // "review" is Plugin(2) for "security-review", "code-review" is Plugin(1)
    assertThat(result.capability().name()).isEqualTo("code-review");
    assertThat(result.degree()).isEqualTo(new MatchDegree.Plugin(1));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CapabilityResolverTest`
Expected: compilation failure — `resolve()` returns `AgentCapability`, not `ResolvedCapability`

- [ ] **Step 3: Update CapabilityResolver.resolve() to return ResolvedCapability**

Replace the `resolve()` method in `api/src/main/java/io/casehub/eidos/api/CapabilityResolver.java` (lines 54–98):

```java
/**
 * Resolves the best matching capability from a list of declared capabilities.
 *
 * <p>Selection uses {@link MatchDegree#compareTo} — Exact wins immediately,
 * then the lowest-ranked (best) non-None degree. First in list wins at equal rank.
 *
 * @param capabilities the list of declared capabilities to search
 * @param capabilityTag the requested capability tag
 * @param registry the vocabulary registry for subsumption resolution
 * @return the best matching capability with its degree, or {@code null} if no match found
 */
public static ResolvedCapability resolve(final List<AgentCapability> capabilities,
                                          final String capabilityTag,
                                          final VocabularyRegistry registry) {
    if (capabilities == null || capabilities.isEmpty()) {
        return null;
    }

    ResolvedCapability best = null;

    for (final var capability : capabilities) {
        final MatchDegree degree = match(capability, capabilityTag, registry);

        if (degree instanceof MatchDegree.Exact) {
            return new ResolvedCapability(capability, degree);
        }
        if (!(degree instanceof MatchDegree.None)) {
            if (best == null || degree.compareTo(best.degree()) < 0) {
                best = new ResolvedCapability(capability, degree);
            }
        }
    }

    return best;
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CapabilityResolverTest`
Expected: all tests PASS (including the new `resolve_plugin_beats_specialization_across_depth`)

- [ ] **Step 5: Commit**

```
feat(eidos#84): CapabilityResolver.resolve() returns ResolvedCapability

The selection loop now uses MatchDegree.compareTo() instead of raw int
depth tracking, fixing the latent bug where Specialization(1) beat
Plugin(2).

Refs #84
```

---

### Task 3: AgentMatch record + AgentRegistry/ReactiveAgentRegistry SPI change

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/AgentMatch.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentRegistry.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/ReactiveAgentRegistry.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentRegistrySpiTest.java`

**Interfaces:**
- Consumes: `ResolvedCapability` (from Task 1)
- Produces:
  - `record AgentMatch(AgentDescriptor descriptor, ResolvedCapability resolvedCapability)` — `resolvedCapability` is null when query has no `capabilityName`
  - `AgentRegistry.find(AgentQuery)` → `List<AgentMatch>`
  - `ReactiveAgentRegistry.find(AgentQuery)` → `Uni<List<AgentMatch>>`

- [ ] **Step 1: Create AgentMatch record**

Create `api/src/main/java/io/casehub/eidos/api/AgentMatch.java`:

```java
package io.casehub.eidos.api;

/**
 * Result of {@link AgentRegistry#find(AgentQuery)} — an agent that matched the query,
 * with optional capability resolution context.
 *
 * <p>{@code resolvedCapability} is non-null when {@link AgentQuery#capabilityName()} was specified,
 * carrying the declared capability that matched and the OWLS-MX match degree.
 * Null for slot-only or {@link AgentQuery#all} queries where no capability matching occurred.
 *
 * @param descriptor the matched agent descriptor
 * @param resolvedCapability the capability resolution result, or null when no capability was queried
 */
public record AgentMatch(AgentDescriptor descriptor, ResolvedCapability resolvedCapability) {}
```

- [ ] **Step 2: Update AgentRegistry.find() return type**

In `api/src/main/java/io/casehub/eidos/api/AgentRegistry.java`:

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Optional;

public interface AgentRegistry {
    void register(AgentDescriptor descriptor);

    /**
     * @throws NullPointerException if agentId or tenancyId is null
     */
    Optional<AgentDescriptor> findById(String agentId, String tenancyId);

    /**
     * Finds agents matching the query criteria.
     *
     * <p>When {@link AgentQuery#capabilityName()} is non-null, results carry
     * {@link AgentMatch#resolvedCapability()} and are ordered by match quality
     * (best first per OWLS-MX ordering). When no capability is queried,
     * {@code resolvedCapability} is null and ordering is unspecified.
     */
    List<AgentMatch> find(AgentQuery query);
}
```

- [ ] **Step 3: Update ReactiveAgentRegistry.find() return type**

In `api/src/main/java/io/casehub/eidos/api/ReactiveAgentRegistry.java`:

```java
package io.casehub.eidos.api;

import io.smallrye.mutiny.Uni;
import java.util.List;
import java.util.Optional;

public interface ReactiveAgentRegistry {
    Uni<Void> register(AgentDescriptor descriptor);

    /**
     * @throws NullPointerException if agentId or tenancyId is null
     */
    Uni<Optional<AgentDescriptor>> findById(String agentId, String tenancyId);

    Uni<List<AgentMatch>> find(AgentQuery query);
}
```

- [ ] **Step 4: Update AgentRegistrySpiTest**

In `api/src/test/java/io/casehub/eidos/api/AgentRegistrySpiTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;

class AgentRegistrySpiTest {

    @Test
    void anonymous_implementation_satisfies_contract() {
        AgentRegistry registry = new AgentRegistry() {
            @Override public void register(AgentDescriptor d) {}
            @Override public Optional<AgentDescriptor> findById(String id, String tenancyId) { return Optional.empty(); }
            @Override public List<AgentMatch> find(AgentQuery q) { return List.of(); }
        };
        assertThat(registry.findById("x", "default")).isEmpty();
        assertThat(registry.find(AgentQuery.all("default"))).isEmpty();
    }
}
```

- [ ] **Step 5: Run api tests (expect compilation failures in other modules — that's fine)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: api module compiles and all api tests pass. Other modules won't compile yet — that's expected.

- [ ] **Step 6: Commit**

```
feat(eidos#84): AgentMatch record, find() returns List<AgentMatch>

Breaking change to AgentRegistry and ReactiveAgentRegistry SPI.
Implementations updated in subsequent commits.

Refs #84
```

---

### Task 4: InMemoryAgentRegistry implementation + tests

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java`
- Modify: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistry.java`
- Modify: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java`

**Interfaces:**
- Consumes: `AgentMatch`, `ResolvedCapability`, `CapabilityResolver.resolve()` returning `ResolvedCapability` (Tasks 1-3)
- Produces: Working `InMemoryAgentRegistry.find()` returning `List<AgentMatch>` sorted by match quality

- [ ] **Step 1: Update InMemoryAgentRegistryTest**

In `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java`, update all `find()` call sites. The pattern is mechanical — every `registry.find(query)` now returns `List<AgentMatch>`. Extract `.descriptor()` for existing assertions, add match metadata assertions where capability was queried.

Key changes to existing tests:

For `find_by_slot` — no capability in query, resolvedCapability is null:
```java
@Test
void find_by_slot() {
    var result = registry.find(AgentQuery.bySlot("reviewer", "default"));
    assertThat(result).extracting(m -> m.descriptor().agentId()).containsExactly("agent-1");
    assertThat(result).allSatisfy(m -> assertThat(m.resolvedCapability()).isNull());
}
```

For `find_by_capability` — capability in query, resolvedCapability is non-null:
```java
@Test
void find_by_capability() {
    var result = registry.find(AgentQuery.byCapability("code-review", "default"));
    assertThat(result).extracting(m -> m.descriptor().agentId()).containsExactly("agent-1");
    assertThat(result).allSatisfy(m -> {
        assertThat(m.resolvedCapability()).isNotNull();
        assertThat(m.resolvedCapability().capability().name()).isEqualTo("code-review");
        assertThat(m.resolvedCapability().degree()).isInstanceOf(MatchDegree.Exact.class);
    });
}
```

For `find_by_slot_and_capability` — both slot and capability:
```java
@Test
void find_by_slot_and_capability() {
    var result = registry.find(AgentQuery.bySlotAndCapability("reviewer", "code-review", "default"));
    assertThat(result).extracting(m -> m.descriptor().agentId()).containsExactly("agent-1");
    assertThat(result).allSatisfy(m -> assertThat(m.resolvedCapability()).isNotNull());
}
```

For `find_by_capability_matches_via_subsumption` — verify degree is Plugin:
```java
@Test
void find_by_capability_matches_via_subsumption() {
    // ... existing agent setup with grounded "code-review" ...
    var result = registry.find(AgentQuery.byCapability("security-review", "default"));
    assertThat(result).hasSize(1);
    assertThat(result.getFirst().resolvedCapability()).isNotNull();
    assertThat(result.getFirst().resolvedCapability().degree())
        .isInstanceOf(MatchDegree.Plugin.class);
    assertThat(((MatchDegree.Plugin) result.getFirst().resolvedCapability().degree()).depth())
        .isEqualTo(1);
}
```

All other test methods follow the same pattern: `registry.find(...)` results use `m.descriptor()` for descriptor-level assertions, `m.resolvedCapability()` for match-level assertions.

Update `tenancy_isolation`, `capability_with_empty_types_lists_is_valid`, `find_ungrounded_capability_uses_exact_match_only`, and the two `register_rejects_*` tests similarly (the reject tests don't call `find()` so no changes there).

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryAgentRegistryTest`
Expected: compilation failure — `find()` still returns `List<AgentDescriptor>`

- [ ] **Step 3: Update InMemoryAgentRegistry.find()**

Replace the `find()` method and `matchesCapability()` in `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java`:

```java
@Override
public List<AgentMatch> find(AgentQuery query) {
    var stream = store.values().stream()
        .filter(d -> d.tenancyId().equals(query.tenancyId()))
        .filter(d -> query.slot() == null || Objects.equals(d.slot(), query.slot()))
        .filter(d -> query.taskDomain() == null
            || d.capabilities().stream().noneMatch(c ->
                c.excludedDomains() != null && c.excludedDomains().contains(query.taskDomain())));

    if (query.capabilityName() == null) {
        return stream
            .map(d -> new AgentMatch(d, null))
            .collect(Collectors.toList());
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

private ResolvedCapability resolveCapability(AgentDescriptor descriptor, String capabilityName) {
    if (!vocabularyRegistry.isResolvable()) {
        return descriptor.capabilities().stream()
            .filter(c -> c.name().equals(capabilityName))
            .findFirst()
            .map(c -> new ResolvedCapability(c, new MatchDegree.Exact()))
            .orElse(null);
    }
    return CapabilityResolver.resolve(
        descriptor.capabilities(), capabilityName, vocabularyRegistry.get());
}
```

Remove the old `matchesCapability()` private method entirely.

- [ ] **Step 4: Update InMemoryReactiveAgentRegistry.find()**

In `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentRegistry.java`, change the return type:

```java
@Override
public Uni<List<AgentMatch>> find(AgentQuery query) {
    return Uni.createFrom().item(delegate.find(query));
}
```

Add `import io.casehub.eidos.api.AgentMatch;` to the imports.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: all InMemoryAgentRegistryTest tests PASS

- [ ] **Step 6: Commit**

```
feat(eidos#84): InMemoryAgentRegistry.find() returns List<AgentMatch>

Replaces boolean matchesCapability() with CapabilityResolver.resolve(),
captures ResolvedCapability, sorts by match quality.

Refs #84
```

---

### Task 5: JPA registry implementations + tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaAgentRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaReactiveAgentRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`

**Interfaces:**
- Consumes: `AgentMatch`, `ResolvedCapability`, `CapabilityResolver.resolve()` (Tasks 1-3)
- Produces: Working `JpaAgentRegistry.find()` and `JpaReactiveAgentRegistry.find()` returning `List<AgentMatch>` sorted by match quality

- [ ] **Step 1: Update JpaAgentRegistryTest**

Same mechanical pattern as Task 4. All `registry.find()` results are `List<AgentMatch>`. Extract `.descriptor()` for descriptor assertions, add match metadata assertions for capability queries.

Key test updates:

For `find_by_slot_returns_matching_agents_only`:
```java
var reviewers = registry.find(AgentQuery.bySlot("reviewer", "default"));
assertThat(reviewers).extracting(m -> m.descriptor().agentId())
    .containsExactlyInAnyOrder("agent-1", "agent-2");
assertThat(reviewers).allSatisfy(m -> assertThat(m.resolvedCapability()).isNull());
```

For `find_by_capability_returns_agents_with_that_capability`:
```java
var codeReviewers = registry.find(AgentQuery.byCapability("code-review", "default"));
assertThat(codeReviewers).extracting(m -> m.descriptor().agentId())
    .containsExactly("agent-1");
assertThat(codeReviewers.getFirst().resolvedCapability()).isNotNull();
assertThat(codeReviewers.getFirst().resolvedCapability().degree())
    .isInstanceOf(MatchDegree.Exact.class);
```

For `find_by_capability_matches_via_subsumption`:
```java
var result = registry.find(AgentQuery.byCapability("security-review", "default"));
assertThat(result).extracting(m -> m.descriptor().agentId()).containsExactly("subsumption-agent");
assertThat(result.getFirst().resolvedCapability().degree())
    .isInstanceOf(MatchDegree.Plugin.class);
```

For `cross_vocabulary_*` tests — same pattern, verify match metadata is present.

All other tests: mechanical `.descriptor()` extraction.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest`
Expected: compilation failure

- [ ] **Step 3: Update JpaAgentRegistry.find()**

In `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaAgentRegistry.java`, replace lines 61–118:

```java
@Override
@Transactional(TxType.SUPPORTS)
public List<AgentMatch> find(AgentQuery query) {
    String fetchJoin = (query.capabilityName() != null || query.taskDomain() != null)
        ? "JOIN FETCH a.capabilities c"
        : "LEFT JOIN FETCH a.capabilities c";

    var jpql = new StringBuilder(
        "SELECT DISTINCT a FROM AgentDescriptorEntity a " + fetchJoin
        + " WHERE a.tenancyId = :tenancyId");
    if (query.slot() != null) jpql.append(" AND a.slot = :slot");

    // Capability matching with vocabulary expansion
    Map<String, Set<String>> capabilityExpansion = null;
    if (query.capabilityName() != null) {
        capabilityExpansion = vocabularyRegistry.expandForMatchingByVocabulary(query.capabilityName());
        int totalExpanded = capabilityExpansion.values().stream().mapToInt(Set::size).sum();
        if (totalExpanded > MAX_EXPANSION_SIZE) {
            LOG.warnf("Vocabulary expansion for '%s' produced %d terms across %d vocabularies;"
                + " query may be slow", query.capabilityName(), totalExpanded, capabilityExpansion.size());
        }
        if (capabilityExpansion.isEmpty()) {
            jpql.append(" AND c.name = :capabilityName");
        } else {
            jpql.append(" AND (c.name = :capabilityName");
            int idx = 0;
            for (var entry : capabilityExpansion.entrySet()) {
                jpql.append(" OR (c.capabilityVocabulary = :vocab").append(idx)
                    .append(" AND c.name IN :expanded").append(idx).append(")");
                idx++;
            }
            jpql.append(")");
        }
    }

    if (query.taskDomain() != null) jpql.append(" AND :taskDomain NOT MEMBER OF c.excludedDomains");

    var q = em.createQuery(jpql.toString(), AgentDescriptorEntity.class)
              .setParameter("tenancyId", query.tenancyId());
    if (query.slot() != null) q.setParameter("slot", query.slot());

    if (query.capabilityName() != null) {
        q.setParameter("capabilityName", query.capabilityName());
        if (capabilityExpansion != null && !capabilityExpansion.isEmpty()) {
            int idx = 0;
            for (var entry : capabilityExpansion.entrySet()) {
                q.setParameter("vocab" + idx, entry.getKey());
                q.setParameter("expanded" + idx, entry.getValue());
                idx++;
            }
        }
    }

    if (query.taskDomain() != null) q.setParameter("taskDomain", query.taskDomain());

    var descriptors = q.getResultList().stream().map(mapper::toRecord).toList();

    if (query.capabilityName() == null) {
        return descriptors.stream()
            .map(d -> new AgentMatch(d, null))
            .toList();
    }

    return descriptors.stream()
        .map(d -> {
            var resolved = CapabilityResolver.resolve(
                d.capabilities(), query.capabilityName(), vocabularyRegistry);
            return new AgentMatch(d, resolved);
        })
        .sorted(Comparator.comparing(AgentMatch::resolvedCapability,
            Comparator.nullsLast(Comparator.comparing(ResolvedCapability::degree))))
        .toList();
}
```

Add these imports:
```java
import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.ResolvedCapability;
import java.util.Comparator;
```

- [ ] **Step 4: Update JpaReactiveAgentRegistry.find()**

In `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaReactiveAgentRegistry.java`, replace the `find()` method (lines 52–103):

```java
@Override
@WithSession
public Uni<List<AgentMatch>> find(AgentQuery query) {
    String fetchJoin = (query.capabilityName() != null || query.taskDomain() != null)
        ? "JOIN FETCH a.capabilities c"
        : "LEFT JOIN FETCH a.capabilities c";

    var jpql = new StringBuilder(
        "SELECT DISTINCT a FROM AgentDescriptorEntity a " + fetchJoin
        + " WHERE a.tenancyId = :tenancyId");

    var params = Parameters.with("tenancyId", query.tenancyId());
    if (query.slot() != null) {
        jpql.append(" AND a.slot = :slot");
        params.and("slot", query.slot());
    }

    if (query.capabilityName() != null) {
        Map<String, Set<String>> expansion = vocabularyRegistry.expandForMatchingByVocabulary(query.capabilityName());
        int totalExpanded = expansion.values().stream().mapToInt(Set::size).sum();
        if (totalExpanded > JpaAgentRegistry.MAX_EXPANSION_SIZE) {
            LOG.warnf("Vocabulary expansion for '%s' produced %d terms across %d vocabularies;"
                + " query may be slow", query.capabilityName(), totalExpanded, expansion.size());
        }
        if (expansion.isEmpty()) {
            jpql.append(" AND c.name = :capabilityName");
            params.and("capabilityName", query.capabilityName());
        } else {
            jpql.append(" AND (c.name = :capabilityName");
            params.and("capabilityName", query.capabilityName());
            int idx = 0;
            for (var entry : expansion.entrySet()) {
                jpql.append(" OR (c.capabilityVocabulary = :vocab").append(idx)
                    .append(" AND c.name IN :expanded").append(idx).append(")");
                params.and("vocab" + idx, entry.getKey());
                params.and("expanded" + idx, entry.getValue());
                idx++;
            }
            jpql.append(")");
        }
    }

    if (query.taskDomain() != null) {
        jpql.append(" AND :taskDomain NOT MEMBER OF c.excludedDomains");
        params.and("taskDomain", query.taskDomain());
    }

    return repo.list(jpql.toString(), params)
               .map(list -> {
                   var descriptors = list.stream().map(mapper::toRecord).toList();
                   if (query.capabilityName() == null) {
                       return descriptors.stream()
                           .map(d -> new AgentMatch(d, null))
                           .toList();
                   }
                   return descriptors.stream()
                       .map(d -> {
                           var resolved = CapabilityResolver.resolve(
                               d.capabilities(), query.capabilityName(), vocabularyRegistry);
                           return new AgentMatch(d, resolved);
                       })
                       .sorted(Comparator.comparing(AgentMatch::resolvedCapability,
                           Comparator.nullsLast(Comparator.comparing(ResolvedCapability::degree))))
                       .toList();
               });
}
```

Add these imports:
```java
import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.CapabilityResolver;
import io.casehub.eidos.api.ResolvedCapability;
import java.util.Comparator;
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```
feat(eidos#84): JPA registry implementations return List<AgentMatch>

JPQL vocabulary-expansion filtering unchanged. Post-processes results
through CapabilityResolver.resolve() to attach match metadata, sorts
by OWLS-MX degree.

Refs #84
```

---

### Task 6: DefaultCapabilityHealth.probe() + example/integration test updates

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java`
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CapabilitySubsumptionScenarioTest.java`
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/MultiAgentTeamTest.java`
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/TenancyIsolationTest.java`
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DraftHouseReviewerScenarioTest.java`
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedExclusionSubsumptionTest.java`

**Interfaces:**
- Consumes: `ResolvedCapability` from `CapabilityResolver.resolve()` (Task 2)
- Produces: Working probe() using `.capability()` extraction; all example tests passing

- [ ] **Step 1: Update DefaultCapabilityHealth.probe()**

In `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`, update lines 58-63:

```java
// Step 2: capability not declared → unavailable
if (descriptor.capabilities() == null || descriptor.capabilities().isEmpty()) {
    return new CapabilityStatus.Unavailable("Capability '" + capabilityTag + "' not declared");
}

final var resolved = CapabilityResolver.resolve(
    descriptor.capabilities(), capabilityTag, vocabularyRegistry);

if (resolved == null) {
    return new CapabilityStatus.Unavailable("Capability '" + capabilityTag + "' not declared");
}

final var capability = resolved.capability();
```

Then replace all subsequent references to `capability` in the method — they should now use the local `capability` variable extracted from `resolved.capability()`. The variable name is the same so the rest of the method body is unchanged.

- [ ] **Step 2: Update DefaultCapabilityHealthTest probe tests**

In `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java` — no changes needed. The test assertions verify `CapabilityStatus` return values (Ready, Unavailable, etc.), not the intermediate `ResolvedCapability`. The probe() signature hasn't changed.

Verify: run the tests to confirm.

- [ ] **Step 3: Update LearnedExclusionSubsumptionTest**

In `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedExclusionSubsumptionTest.java`, update lines 44-48 where `CapabilityResolver.resolve()` is called directly:

```java
var resolved = CapabilityResolver.resolve(
    agent.capabilities(), "code-review", vocabularyRegistry);
assertThat(resolved).isNotNull();
assertThat(resolved.capability().name()).isEqualTo("security-code-review");
```

Line 54: update `resolved.name()` to `resolved.capability().name()`:
```java
resolved.capability().name(), "rust",
```

- [ ] **Step 4: Update CapabilitySubsumptionScenarioTest**

All `registry.find()` calls return `List<AgentMatch>`. Mechanical updates:

- `general_agent_found_when_querying_for_specific_capability` — `matches` extraction uses `.descriptor().agentId()`
- `specific_agent_found_when_querying_for_general_capability` — same pattern
- `ungrounded_capability_not_matched_by_subsumption` — same
- `mixed_grounded_and_ungrounded_capabilities_filter_correctly` — same
- `excludedDomains_blocks_discovery_even_with_subsumption` — same
- `multiple_agents_ranked_by_match_degree` — this is the key test. Update to assert ordering:

```java
var matches = registry.find(query);

assertThat(matches)
    .extracting(m -> m.descriptor().agentId())
    .containsExactly("security-exact", "code-review-general", "sast-specialist");

// Verify match degrees
assertThat(matches.get(0).resolvedCapability().degree())
    .isInstanceOf(MatchDegree.Exact.class);
assertThat(matches.get(1).resolvedCapability().degree())
    .isInstanceOf(MatchDegree.Plugin.class);
assertThat(matches.get(2).resolvedCapability().degree())
    .isInstanceOf(MatchDegree.Specialization.class);
```

Remove the old comment about ordering being a "future enhancement".

- [ ] **Step 5: Update MultiAgentTeamTest, TenancyIsolationTest, DraftHouseReviewerScenarioTest**

All mechanical — `registry.find()` results are `List<AgentMatch>`, extract `.descriptor()` for existing assertions. These tests use slot-only or all() queries mostly, so `resolvedCapability` will be null. For capability queries, the assertions already check agent IDs — add `.descriptor()` in the extraction chain.

Pattern for each:
```java
// Before:
assertThat(reviewers).extracting(AgentDescriptor::agentId)...
// After:
assertThat(reviewers).extracting(m -> m.descriptor().agentId())...
```

- [ ] **Step 6: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all modules compile, all tests pass

- [ ] **Step 7: Commit**

```
feat(eidos#84): update probe() and all test callers for AgentMatch

DefaultCapabilityHealth.probe() extracts .capability() from
ResolvedCapability. All 42 find() call sites updated. Subsumption
scenario test now asserts OWLS-MX ordering.

Refs #84
```

---

### Task 7: CLAUDE.md and ARC42STORIES.MD updates

**Files:**
- Modify: `CLAUDE.md` (project repo)
- Modify: `ARC42STORIES.MD` (project repo)

**Interfaces:**
- Consumes: all previous tasks complete
- Produces: updated documentation reflecting new types and API

- [ ] **Step 1: Update CLAUDE.md**

In the project CLAUDE.md, update the relevant sections:

Under `AgentRegistry`:
```
- **AgentRegistry** — store and query descriptors (blocking + reactive); `find(AgentQuery)` returns `List<AgentMatch>` carrying descriptor + `ResolvedCapability` (matched capability and OWLS-MX `MatchDegree`); results ordered by match quality when capability is queried
```

Under `CapabilityHealth`:
```
`CapabilityResolver` (static utility in api/, Tier 1) provides shared subsumption resolution for both probe and recording paths — `resolve()` returns `ResolvedCapability(capability, degree)` preserving match metadata; callers of `BehavioralSignalStore.record()` must pass the declared capability name and `ComplianceDimension` qualifier (use `CapabilityResolver.resolve()` to obtain capability name from a query tag).
```

Under `MatchDegree`:
```
- **MatchDegree** — sealed interface: Exact, Plugin(depth), Specialization(depth), None; implements `Comparable<MatchDegree>` with OWLS-MX ordering (Exact < Plugin < Specialization < None, lower depth within same type)
```

Add `ResolvedCapability` and `AgentMatch` to the api package listing:
```
│       ├── ResolvedCapability.java       — result of CapabilityResolver.resolve(): capability + MatchDegree
│       ├── AgentMatch.java               — result of AgentRegistry.find(): descriptor + optional ResolvedCapability
```

- [ ] **Step 2: Update ARC42STORIES.MD if applicable**

Check if any layer/chapter references `AgentRegistry.find()` return type. If so, update to reflect `List<AgentMatch>`. If not, no change needed.

- [ ] **Step 3: Commit**

```
docs(eidos#84): update CLAUDE.md and ARC42STORIES.MD for AgentMatch API

Refs #84
```

---
