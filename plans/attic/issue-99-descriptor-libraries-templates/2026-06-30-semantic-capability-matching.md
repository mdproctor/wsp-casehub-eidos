# Semantic Capability Matching Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add XKOS-style hierarchy to VocabularyTerm, graded subsumption matching to VocabularyRegistry, optional vocabulary grounding to AgentCapability, and subsumption-aware queries to AgentRegistry and CapabilityHealth.

**Architecture:** VocabularyTerm gains `specializes()` to declare DAG parent relationships between enum constants. CdiVocabularyRegistry precomputes ancestor/descendant indexes at registration time for O(1) subsumption lookups. AgentCapability gains optional `capabilityVocabulary` URI — when present, AgentRegistry.find() and CapabilityHealth.probe() use subsumption instead of exact string matching.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, AssertJ

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use `mvn` not `./mvnw`
- All commits reference eidos#71: `Refs #71`
- No deployed instances — schema changes go directly into `V1__initial_schema.sql`
- Spec: `specs/2026-06-30-semantic-capability-matching-design.md` (in workspace)

---

### Task 1: MatchDegree sealed interface + VocabularyTerm.specializes()

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/MatchDegree.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java`
- Create: `api/src/test/java/io/casehub/eidos/api/MatchDegreeTest.java`

**Interfaces:**
- Produces: `MatchDegree` sealed interface (Exact, Plugin, Specialization, None) — used by Tasks 2, 4, 5
- Produces: `VocabularyTerm.specializes()` default method — used by Task 2

- [ ] **Step 1: Write MatchDegree tests**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class MatchDegreeTest {

    @Test
    void exact_has_no_depth() {
        var exact = new MatchDegree.Exact();
        assertThat(exact).isInstanceOf(MatchDegree.class);
    }

    @Test
    void plugin_carries_depth() {
        var plugin = new MatchDegree.Plugin(2);
        assertThat(plugin.depth()).isEqualTo(2);
    }

    @Test
    void specialization_carries_depth() {
        var spec = new MatchDegree.Specialization(3);
        assertThat(spec.depth()).isEqualTo(3);
    }

    @Test
    void none_is_singleton_value() {
        assertThat(new MatchDegree.None()).isEqualTo(new MatchDegree.None());
    }

    @Test
    void exhaustive_switch_compiles() {
        MatchDegree degree = new MatchDegree.Plugin(1);
        String result = switch (degree) {
            case MatchDegree.Exact e -> "exact";
            case MatchDegree.Plugin p -> "plugin:" + p.depth();
            case MatchDegree.Specialization s -> "spec:" + s.depth();
            case MatchDegree.None n -> "none";
        };
        assertThat(result).isEqualTo("plugin:1");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=MatchDegreeTest`
Expected: FAIL — `MatchDegree` class not found

- [ ] **Step 3: Create MatchDegree sealed interface**

```java
package io.casehub.eidos.api;

public sealed interface MatchDegree
        permits MatchDegree.Exact, MatchDegree.Plugin,
                MatchDegree.Specialization, MatchDegree.None {

    record Exact() implements MatchDegree {}

    record Plugin(int depth) implements MatchDegree {}

    record Specialization(int depth) implements MatchDegree {}

    record None() implements MatchDegree {}
}
```

- [ ] **Step 4: Add `specializes()` default method to VocabularyTerm**

In `api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java`, add after the `aliases()` default method:

```java
default List<VocabularyTerm> specializes() { return List.of(); }
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: all api tests PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/MatchDegree.java api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java api/src/test/java/io/casehub/eidos/api/MatchDegreeTest.java
git commit -m "feat(eidos#71): MatchDegree sealed interface and VocabularyTerm.specializes()

Refs #71"
```

---

### Task 2: VocabularyRegistry API + CdiVocabularyRegistry DAG index

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java`

**Interfaces:**
- Consumes: `MatchDegree` from Task 1, `VocabularyTerm.specializes()` from Task 1
- Produces: `VocabularyRegistry.subsumes()`, `.match()`, `.ancestors()`, `.descendants()`, `.expandForMatchingByVocabulary()` — used by Tasks 4, 5

- [ ] **Step 1: Add 5 new methods to VocabularyRegistry interface**

In `api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java`, add a new section after the vocabulary-level metadata section:

```java
// --- Hierarchy / subsumption ---

boolean subsumes(String vocabUri, String generalValue, String specificValue);

MatchDegree match(String vocabUri, String declaredValue, String requestedValue);

List<? extends VocabularyTerm> ancestors(String vocabUri, String value);

List<? extends VocabularyTerm> descendants(String vocabUri, String value);

Map<String, Set<String>> expandForMatchingByVocabulary(String value);
```

Add `import java.util.Map; import java.util.Set;` to the imports.

- [ ] **Step 2: Write hierarchy tests in CdiVocabularyRegistryTest**

Add a test vocabulary with hierarchy and tests. Add inside `CdiVocabularyRegistryTest`:

```java
@VocabularyMetadata(uri = "urn:test:hierarchy", name = "Hierarchy Vocab", version = "1.0")
enum HierarchyTerm implements VocabularyTerm {
    ROOT("root", "Root"),
    CHILD_A("child-a", "Child A") {
        @Override public List<VocabularyTerm> specializes() { return List.of(ROOT); }
    },
    CHILD_B("child-b", "Child B") {
        @Override public List<VocabularyTerm> specializes() { return List.of(ROOT); }
    },
    GRANDCHILD("grandchild", "Grandchild") {
        @Override public List<VocabularyTerm> specializes() { return List.of(CHILD_A); }
    },
    DAG_LEAF("dag-leaf", "DAG Leaf") {
        @Override public List<VocabularyTerm> specializes() { return List.of(CHILD_A, CHILD_B); }
    };

    final String value, label;
    HierarchyTerm(String v, String l) { value = v; label = l; }
    @Override public String value() { return value; }
    @Override public String label() { return label; }
}

// --- Hierarchy / subsumption tests ---

@Test
void subsumes_parent_subsumes_child() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.subsumes("urn:test:hierarchy", "root", "child-a")).isTrue();
}

@Test
void subsumes_grandparent_subsumes_grandchild() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.subsumes("urn:test:hierarchy", "root", "grandchild")).isTrue();
}

@Test
void subsumes_self() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.subsumes("urn:test:hierarchy", "root", "root")).isTrue();
}

@Test
void subsumes_child_does_not_subsume_parent() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.subsumes("urn:test:hierarchy", "child-a", "root")).isFalse();
}

@Test
void subsumes_siblings_do_not_subsume() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.subsumes("urn:test:hierarchy", "child-a", "child-b")).isFalse();
}

@Test
void subsumes_unknown_vocab_returns_false() {
    assertThat(registry.subsumes("urn:unknown", "a", "b")).isFalse();
}

@Test
void subsumes_unknown_term_returns_false() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.subsumes("urn:test:hierarchy", "root", "nonexistent")).isFalse();
}

@Test
void match_exact() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.match("urn:test:hierarchy", "child-a", "child-a"))
        .isEqualTo(new MatchDegree.Exact());
}

@Test
void match_plugin_immediate_parent() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.match("urn:test:hierarchy", "root", "child-a"))
        .isEqualTo(new MatchDegree.Plugin(1));
}

@Test
void match_plugin_grandparent() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.match("urn:test:hierarchy", "root", "grandchild"))
        .isEqualTo(new MatchDegree.Plugin(2));
}

@Test
void match_specialization_immediate_child() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.match("urn:test:hierarchy", "child-a", "root"))
        .isEqualTo(new MatchDegree.Specialization(1));
}

@Test
void match_none_for_siblings() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.match("urn:test:hierarchy", "child-a", "child-b"))
        .isEqualTo(new MatchDegree.None());
}

@Test
void match_unknown_vocab_returns_none() {
    assertThat(registry.match("urn:unknown", "a", "b"))
        .isEqualTo(new MatchDegree.None());
}

@Test
void ancestors_returns_ordered_by_depth() {
    registry.register(HierarchyTerm.class);
    var ancestors = registry.ancestors("urn:test:hierarchy", "grandchild");
    assertThat(ancestors).extracting(VocabularyTerm::value)
        .containsExactly("child-a", "root");
}

@Test
void ancestors_of_root_is_empty() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.ancestors("urn:test:hierarchy", "root")).isEmpty();
}

@Test
void ancestors_unknown_term_returns_empty() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.ancestors("urn:test:hierarchy", "nonexistent")).isEmpty();
}

@Test
void descendants_returns_ordered_by_depth() {
    registry.register(HierarchyTerm.class);
    var desc = registry.descendants("urn:test:hierarchy", "root");
    assertThat(desc).extracting(VocabularyTerm::value)
        .startsWith("child-a", "child-b");
    assertThat(desc).extracting(VocabularyTerm::value)
        .contains("grandchild", "dag-leaf");
}

@Test
void descendants_of_leaf_is_empty() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.descendants("urn:test:hierarchy", "grandchild")).isEmpty();
}

@Test
void dag_leaf_has_two_parents() {
    registry.register(HierarchyTerm.class);
    var ancestors = registry.ancestors("urn:test:hierarchy", "dag-leaf");
    assertThat(ancestors).extracting(VocabularyTerm::value)
        .contains("child-a", "child-b", "root");
}

@Test
void dag_leaf_min_depth_to_root_is_2() {
    registry.register(HierarchyTerm.class);
    assertThat(registry.match("urn:test:hierarchy", "root", "dag-leaf"))
        .isEqualTo(new MatchDegree.Plugin(2));
}

@Test
void expandForMatchingByVocabulary_returns_scoped_expansion() {
    registry.register(HierarchyTerm.class);
    var expansion = registry.expandForMatchingByVocabulary("child-a");
    assertThat(expansion).containsKey("urn:test:hierarchy");
    var names = expansion.get("urn:test:hierarchy");
    assertThat(names).contains("child-a", "root", "grandchild", "dag-leaf");
}

@Test
void expandForMatchingByVocabulary_unknown_term_returns_empty() {
    var expansion = registry.expandForMatchingByVocabulary("nonexistent");
    assertThat(expansion).isEmpty();
}

@Test
void register_cycle_throws() {
    @VocabularyMetadata(uri = "urn:test:cycle")
    enum CycleTerm implements VocabularyTerm {
        A("a", "A") {
            @Override public List<VocabularyTerm> specializes() { return List.of(B); }
        },
        B("b", "B") {
            @Override public List<VocabularyTerm> specializes() { return List.of(A); }
        };
        final String value, label;
        CycleTerm(String v, String l) { value = v; label = l; }
        @Override public String value() { return value; }
        @Override public String label() { return label; }
    }
    assertThatThrownBy(() -> registry.register(CycleTerm.class))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("Cycle");
}

@Test
void register_cross_vocab_specializes_throws() {
    @VocabularyMetadata(uri = "urn:test:cross-ref")
    enum CrossRefTerm implements VocabularyTerm {
        X("x", "X") {
            @Override public List<VocabularyTerm> specializes() {
                return List.of(SourceTerm.ALPHA);
            }
        };
        final String value, label;
        CrossRefTerm(String v, String l) { value = v; label = l; }
        @Override public String value() { return value; }
        @Override public String label() { return label; }
    }
    registry.register(SourceTerm.class);
    assertThatThrownBy(() -> registry.register(CrossRefTerm.class))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test
void flat_vocab_has_no_hierarchy() {
    registry.register(SourceTerm.class);
    assertThat(registry.ancestors("urn:test:source", "alpha")).isEmpty();
    assertThat(registry.descendants("urn:test:source", "alpha")).isEmpty();
    assertThat(registry.subsumes("urn:test:source", "alpha", "beta")).isFalse();
    assertThat(registry.match("urn:test:source", "alpha", "beta"))
        .isEqualTo(new MatchDegree.None());
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiVocabularyRegistryTest`
Expected: FAIL — methods not found on VocabularyRegistry / CdiVocabularyRegistry

- [ ] **Step 4: Implement CdiVocabularyRegistry DAG index**

In `CdiVocabularyRegistry.java`, add the new data structures and implement all 5 methods. The key implementation details:

New fields:
```java
private final ConcurrentHashMap<String, Set<String>> valueToVocabs = new ConcurrentHashMap<>();
private final ConcurrentHashMap<String, Map<String, List<AncestorEntry>>> ancestorIndex = new ConcurrentHashMap<>();
private final ConcurrentHashMap<String, Map<String, List<DescendantEntry>>> descendantIndex = new ConcurrentHashMap<>();

private record AncestorEntry(VocabularyTerm term, int depth) {}
private record DescendantEntry(VocabularyTerm term, int depth) {}
```

Inside `register()`, after the existing map writes, add the hierarchy build:
1. Validate specializes() references (same enum class check)
2. Cycle detection via Kahn's algorithm (topological sort)
3. BFS from each term to compute ancestors with min depth
4. Invert edges, BFS for descendants with min depth
5. Populate valueToVocabs

Implement `subsumes()`, `match()`, `ancestors()`, `descendants()`, `expandForMatchingByVocabulary()` using the precomputed indexes.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiVocabularyRegistryTest`
Expected: all PASS

- [ ] **Step 6: Run full build to catch compile errors across modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS (all 774+ tests pass)

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java runtime/src/test/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistryTest.java
git commit -m "feat(eidos#71): VocabularyRegistry subsumption API and CdiVocabularyRegistry DAG index

Refs #71"
```

---

### Task 3: AgentCapability.capabilityVocabulary + entity + mapper + YAML + comparator

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentCapability.java` — add `capabilityVocabulary` field
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java` — add field to comparison, bump `COMPARED_CAPABILITY_FIELD_COUNT`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java` — add test for new field
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java` — add column
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` — map new field
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql` — add column
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java` — add field to YAML config and toDescriptor

**Interfaces:**
- Consumes: Nothing new from prior tasks
- Produces: `AgentCapability.capabilityVocabulary()` — used by Tasks 4, 5

- [ ] **Step 1: Write tests for the new field**

Add test in `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java` (or create if absent — check first):

```java
@Test
void capability_vocabulary_carried_through_builder() {
    var cap = AgentCapability.builder()
        .name("code-review")
        .capabilityVocabulary("urn:casehub:vocab:capability")
        .build();
    assertThat(cap.capabilityVocabulary()).isEqualTo("urn:casehub:vocab:capability");
}

@Test
void capability_vocabulary_null_is_valid() {
    var cap = AgentCapability.builder()
        .name("code-review")
        .build();
    assertThat(cap.capabilityVocabulary()).isNull();
}
```

Add comparator test in `AgentDescriptorComparatorTest.java`:

```java
@Test
void capability_vocabulary_drift_detected() {
    var desired = descriptorWith(AgentCapability.builder()
        .name("review").capabilityVocabulary("urn:vocab:a").build());
    var actual = descriptorWith(AgentCapability.builder()
        .name("review").capabilityVocabulary("urn:vocab:b").build());
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.matches()).isFalse();
    assertThat(result.drifts()).anyMatch(d ->
        d.field().contains("capabilityVocabulary"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: FAIL — `capabilityVocabulary` not found

- [ ] **Step 3: Add capabilityVocabulary to AgentCapability record**

In `AgentCapability.java`, add `String capabilityVocabulary` as the second field (after `name`). Update:
- Record declaration: `String name, String capabilityVocabulary, Double qualityHint, ...`
- Compact constructor: add `AgentDescriptorValidator.validateOptional("capabilityVocabulary", capabilityVocabulary, AgentDescriptorValidator.MAX_VOCABULARY_URI);`
- Builder: add field and setter `public Builder capabilityVocabulary(String v) { this.capabilityVocabulary = v; return this; }`
- Builder.build(): include `capabilityVocabulary` in constructor call

**All existing call sites** that construct `AgentCapability` directly (not via builder) will break — they need a new `null` argument for `capabilityVocabulary`. These are:
- `AgentDescriptorMapper.toCapability()` — add `null` for capabilityVocabulary (or `c.capabilityVocabulary` after entity update)
- `ClasspathYamlDescriptorRegistrar.toDescriptor()` — add `c.capabilityVocabulary`
- Test files with direct construction — add `null`

Use IntelliJ find-references on the AgentCapability constructor to find all call sites.

- [ ] **Step 4: Update AgentDescriptorComparator**

In `compareCapability()`, add: `compareField(drifts, prefix + "capabilityVocabulary", desired.capabilityVocabulary(), actual.capabilityVocabulary());`

Update `COMPARED_CAPABILITY_FIELD_COUNT` from `8` to `9`.

- [ ] **Step 5: Update JPA entity**

In `AgentCapabilityEntity.java`, add:
```java
@Column(name = "capability_vocabulary") String capabilityVocabulary;
```

- [ ] **Step 6: Update V1 schema**

In `V1__initial_schema.sql`, add to the `agent_capability` table after the `name` column:
```sql
capability_vocabulary TEXT,
```

- [ ] **Step 7: Update mapper**

In `AgentDescriptorMapper.toCapability()`, add `c.capabilityVocabulary` to the `AgentCapability` constructor call (second argument after `c.name`).

In `AgentDescriptorMapper.toCapabilityEntity()`, add: `e.capabilityVocabulary = c.capabilityVocabulary();`

- [ ] **Step 8: Update YAML registrar**

In `ClasspathYamlDescriptorRegistrar.CapabilityConfig`, add: `public String capabilityVocabulary;`

In `toDescriptor()`, update the `AgentCapability` construction to include `c.capabilityVocabulary`.

- [ ] **Step 9: Fix all remaining call sites**

Use IntelliJ find-references on the AgentCapability record constructor to find every direct construction call (tests, examples, eval). Add `null` for `capabilityVocabulary` where no vocabulary grounding is intended.

- [ ] **Step 10: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git commit -m "feat(eidos#71): AgentCapability.capabilityVocabulary — optional vocabulary grounding

Refs #71"
```

---

### Task 4: AgentRegistry.find() with subsumption + registration validation

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaAgentRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaReactiveAgentRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`
- Modify: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java`

**Interfaces:**
- Consumes: `VocabularyRegistry.match()`, `.expandForMatchingByVocabulary()` from Task 2; `AgentCapability.capabilityVocabulary()` from Task 3
- Produces: Subsumption-aware `AgentRegistry.find()` — used by consumers

- [ ] **Step 1: Write InMemory subsumption test**

In `InMemoryAgentRegistryTest.java`, add:

```java
@Test
void find_by_capability_matches_via_subsumption() {
    // Register a hierarchical vocabulary
    // Register an agent with a general capability grounded in that vocabulary
    // Query for a specific capability that the general one subsumes
    // Assert the agent is found
}

@Test
void find_ungrounded_capability_uses_exact_match_only() {
    // Register an agent with capability "code-review" (no vocabulary)
    // Query for "security-code-review"
    // Assert NOT found (no subsumption without vocabulary grounding)
}
```

The test requires the InMemoryAgentRegistry to inject VocabularyRegistry and the test vocabulary to be registered. The test class needs `@QuarkusTest` and a vocabulary setup.

- [ ] **Step 2: Write JPA subsumption test**

In `JpaAgentRegistryTest.java`, add matching tests. The JPA test is `@QuarkusTest` with a real database — the vocabulary must be registered before the agent registration for the validation to pass.

- [ ] **Step 3: Write registration validation tests**

Test that `register()` rejects capabilities with unknown vocabulary URIs and unknown terms:

```java
@Test
void register_rejects_unknown_capability_vocabulary() {
    var desc = descriptorWith(AgentCapability.builder()
        .name("review")
        .capabilityVocabulary("urn:nonexistent:vocab")
        .build());
    assertThatThrownBy(() -> registry.register(desc))
        .isInstanceOf(AgentValidationException.class);
}

@Test
void register_rejects_unknown_term_in_known_vocabulary() {
    // Register vocab, then try to register agent with a name not in it
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory,runtime`
Expected: FAIL

- [ ] **Step 5: Implement InMemoryAgentRegistry changes**

Inject `VocabularyRegistry`. Add `matchesCapability()` private method. Update `find()` filter. Add validation in `register()`.

```java
@Inject VocabularyRegistry vocabularyRegistry;

private boolean matchesCapability(AgentCapability capability, String requestedName) {
    if (capability.name().equals(requestedName)) return true;
    if (capability.capabilityVocabulary() == null) return false;
    MatchDegree degree = vocabularyRegistry.match(
        capability.capabilityVocabulary(), capability.name(), requestedName);
    return !(degree instanceof MatchDegree.None);
}
```

In `register()`, add vocabulary validation before `store.put()`:
```java
if (descriptor.capabilities() != null) {
    for (var cap : descriptor.capabilities()) {
        if (cap.capabilityVocabulary() != null) {
            if (!vocabularyRegistry.isRegistered(cap.capabilityVocabulary())) {
                throw new AgentValidationException("capabilityVocabulary",
                    "vocabulary '" + cap.capabilityVocabulary() + "' is not registered");
            }
            if (vocabularyRegistry.resolve(cap.capabilityVocabulary(), cap.name()).isEmpty()) {
                throw new AgentValidationException("capability.name",
                    "'" + cap.name() + "' is not a valid term in vocabulary '" + cap.capabilityVocabulary() + "'");
            }
        }
    }
}
```

- [ ] **Step 6: Implement JpaAgentRegistry changes**

Inject `VocabularyRegistry`. Update `find()` to use `expandForMatchingByVocabulary()` for SQL expansion. Add vocabulary validation in `register()`.

- [ ] **Step 7: Update JpaReactiveAgentRegistry**

Same JPQL expansion logic as JpaAgentRegistry. The reactive registry builds its own JPQL — apply the same vocabulary-scoped expansion pattern.

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git commit -m "feat(eidos#71): subsumption-aware AgentRegistry.find() with vocabulary validation

Refs #71"
```

---

### Task 5: CapabilityHealth.probe() subsumption integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java`

**Interfaces:**
- Consumes: `VocabularyRegistry.match()` from Task 2, `AgentCapability.capabilityVocabulary()` from Task 3
- Produces: Subsumption-aware `CapabilityHealth.probe()`

- [ ] **Step 1: Write probe subsumption tests**

```java
@Test
void probe_finds_capability_via_subsumption() {
    // Agent declares "code-review" grounded in vocab
    // Probe for "security-code-review" (child of code-review in the vocab)
    // Assert Ready (not Unavailable)
}

@Test
void probe_prefers_exact_match_over_subsumption() {
    // Agent declares both "code-review" and "security-code-review"
    // Probe for "security-code-review"
    // Assert the exact match is used (check epistemicDomains or other distinguishing field)
}

@Test
void probe_selects_closest_subsumption_match() {
    // Agent declares "code-review" (depth 2) and "security-code-review" (depth 1)
    // Probe for "sast-review" (child of security-code-review)
    // Assert security-code-review (closer) is the matched capability
}

@Test
void probe_ungrounded_capability_uses_exact_only() {
    // Agent declares "code-review" without vocabulary
    // Probe for "security-code-review"
    // Assert Unavailable
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultCapabilityHealthTest`
Expected: FAIL

- [ ] **Step 3: Implement subsumption-aware probe**

In `DefaultCapabilityHealth.java`:
- Inject `VocabularyRegistry`
- Extract `findCapability()` private method with exact-match-first, then best-subsumption-match logic (per spec §7)
- Replace the inline `capabilities().stream().filter(c -> c.name().equals(capabilityTag)).findFirst()` with the new method

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(eidos#71): subsumption-aware CapabilityHealth.probe()

Refs #71"
```

---

### Task 6: CasehubCapabilityTerm starter vocabulary

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/CasehubCapabilityTerm.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/CasehubCapabilityRegistrar.java`
- Create: `vocab/src/test/java/io/casehub/eidos/vocab/CasehubCapabilityTermTest.java`

**Interfaces:**
- Consumes: `VocabularyTerm.specializes()` from Task 1
- Produces: `CasehubCapabilityTerm` enum with URI `urn:casehub:vocab:capability`

- [ ] **Step 1: Write tests**

```java
@QuarkusTest
class CasehubCapabilityTermTest {

    @Inject VocabularyRegistry registry;

    @Test
    void vocabulary_is_registered() {
        assertThat(registry.isRegistered(CasehubCapabilityTerm.URI)).isTrue();
    }

    @Test
    void sast_review_specializes_security_code_review_and_static_analysis() {
        assertThat(CasehubCapabilityTerm.SAST_REVIEW.specializes())
            .containsExactlyInAnyOrder(
                CasehubCapabilityTerm.SECURITY_CODE_REVIEW,
                CasehubCapabilityTerm.STATIC_ANALYSIS);
    }

    @Test
    void code_review_subsumes_sast_review() {
        assertThat(registry.subsumes(CasehubCapabilityTerm.URI,
            "code-review", "sast-review")).isTrue();
    }

    @Test
    void analysis_subsumes_sast_review() {
        assertThat(registry.subsumes(CasehubCapabilityTerm.URI,
            "analysis", "sast-review")).isTrue();
    }

    @Test
    void root_terms_have_no_ancestors() {
        assertThat(registry.ancestors(CasehubCapabilityTerm.URI, "code-review")).isEmpty();
        assertThat(registry.ancestors(CasehubCapabilityTerm.URI, "testing")).isEmpty();
    }

    @Test
    void all_terms_present() {
        var terms = registry.allTerms(CasehubCapabilityTerm.URI);
        assertThat(terms).extracting(VocabularyTerm::value)
            .contains("code-review", "security-code-review", "performance-code-review",
                       "sast-review", "analysis", "static-analysis", "testing", "documentation");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=CasehubCapabilityTermTest`
Expected: FAIL — class not found

- [ ] **Step 3: Create CasehubCapabilityTerm enum**

Create `vocab/src/main/java/io/casehub/eidos/vocab/CasehubCapabilityTerm.java` with the hierarchy from spec §8:
- Roots: CODE_REVIEW, ANALYSIS, TESTING, DOCUMENTATION
- CODE_REVIEW children: SECURITY_CODE_REVIEW, PERFORMANCE_CODE_REVIEW
- SECURITY_CODE_REVIEW child: SAST_REVIEW (also child of STATIC_ANALYSIS)
- ANALYSIS child: STATIC_ANALYSIS

Follow the same enum pattern as `CasehubSlotTerm` and `BelbinTerm`.

- [ ] **Step 4: Create registrar bean**

```java
@ApplicationScoped
public class CasehubCapabilityRegistrar implements VocabularyRegistrar {
    @Override public Class<CasehubCapabilityTerm> vocabulary() {
        return CasehubCapabilityTerm.class;
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(eidos#71): CasehubCapabilityTerm — starter capability vocabulary with hierarchy

Refs #71"
```

---

### Task 7: Integration tests — end-to-end scenario

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CapabilitySubsumptionScenarioTest.java`

**Interfaces:**
- Consumes: Everything from Tasks 1-6

- [ ] **Step 1: Write end-to-end scenario test**

A `@QuarkusTest` that exercises the full pipeline:
1. Register CasehubCapabilityTerm vocabulary (auto-discovered)
2. Register agents with vocabulary-grounded capabilities at different hierarchy levels
3. Query via AgentRegistry.find() and verify subsumption matches
4. Probe via CapabilityHealth and verify subsumption-discovered agents pass health check
5. Verify match degrees via VocabularyRegistry.match()
6. Verify ungrounded capabilities still use exact match only

```java
@QuarkusTest
class CapabilitySubsumptionScenarioTest {

    @Inject AgentRegistry registry;
    @Inject VocabularyRegistry vocabularyRegistry;
    @Inject CapabilityHealth capabilityHealth;

    @Test
    void general_agent_found_when_querying_for_specific_capability() {
        // Agent declares "code-review" with capabilityVocabulary
        // Query for "security-code-review"
        // Assert agent is in results
    }

    @Test
    void specific_agent_found_when_querying_for_general_capability() {
        // Agent declares "sast-review" with capabilityVocabulary
        // Query for "code-review"
        // Assert agent is in results
    }

    @Test
    void match_degree_reflects_hierarchy_depth() {
        // Verify VocabularyRegistry.match() returns correct Plugin/Specialization depths
    }

    @Test
    void health_probe_works_for_subsumption_discovered_agent() {
        // Agent declares "code-review" grounded
        // Probe for "security-code-review"
        // Assert Ready (not Unavailable)
    }

    @Test
    void ungrounded_capability_not_matched_by_subsumption() {
        // Agent declares "code-review" without vocabulary
        // Query for "security-code-review"
        // Assert NOT found
    }
}
```

- [ ] **Step 2: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git commit -m "test(eidos#71): end-to-end capability subsumption scenario tests

Refs #71"
```
