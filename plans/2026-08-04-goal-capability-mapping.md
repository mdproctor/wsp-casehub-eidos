# Goal-Capability Mapping Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #135 — Add capabilities field to AgentGoal for goal-capability mapping
**Issue group:** #135

**Goal:** Add `List<String> capabilities` to `AgentGoal` so the engine can map capability failures to specific goals rather than blanket-declining all goals.

**Architecture:** Thread a new `List<String> capabilities` field through `AgentGoal` record, `AgentDescriptor` cross-validation, YAML parsing, JPA persistence, goal evolution, and A2A_CARD rendering. All changes follow existing patterns — no new abstractions.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn clean install` (not `./mvnw`)
- No deployed instances — schema changes go directly in base migration files
- Capability names in `goal.capabilities` reference declared `AgentCapability.name()` on the same descriptor (exact string match, no subsumption)
- Empty capabilities list = cross-cutting goal (affected by any capability failure)
- Null normalized to empty `List.of()` in compact constructor
- Use `mcp__intellij-index__*` tools for all code navigation and editing

---

### Task 1: Add `capabilities` field to `AgentGoal` record

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentGoal.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentGoalTest.java`

**Interfaces:**
- Produces: `AgentGoal(String name, String description, GoalPriority priority, Visibility visibility, List<String> capabilities)` — 5-arg record constructor

- [ ] **Step 1: Write failing tests for new field**

Add tests to `AgentGoalTest.java`:

```java
@Test void capabilities_defaults_to_empty_when_null() {
    var goal = new AgentGoal("g", "desc", GoalPriority.PRIMARY, Visibility.PUBLIC, null);
    assertThat(goal.capabilities()).isEmpty();
}

@Test void capabilities_preserved() {
    var goal = new AgentGoal("g", "desc", GoalPriority.PRIMARY, Visibility.PUBLIC,
        List.of("code-review", "testing"));
    assertThat(goal.capabilities()).containsExactly("code-review", "testing");
}

@Test void capabilities_immutable() {
    var caps = new java.util.ArrayList<>(List.of("a"));
    var goal = new AgentGoal("g", "desc", GoalPriority.PRIMARY, Visibility.PUBLIC, caps);
    assertThatThrownBy(() -> goal.capabilities().add("b"))
        .isInstanceOf(UnsupportedOperationException.class);
}

@Test void capabilities_null_elements_filtered() {
    var goal = new AgentGoal("g", "desc", GoalPriority.PRIMARY, Visibility.PUBLIC,
        java.util.Arrays.asList("a", null, "b"));
    assertThat(goal.capabilities()).containsExactly("a", "b");
}

@Test void capabilities_duplicate_throws() {
    assertThatThrownBy(() -> new AgentGoal("g", "desc", GoalPriority.PRIMARY, Visibility.PUBLIC,
        List.of("cap-a", "cap-a")))
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("goal.capabilities"));
}

@Test void capabilities_name_exceeds_max_throws() {
    assertThatThrownBy(() -> new AgentGoal("g", "desc", GoalPriority.PRIMARY, Visibility.PUBLIC,
        List.of("a".repeat(101))))
        .isInstanceOf(AgentValidationException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentGoalTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `AgentGoal` has 4 params, tests pass 5

- [ ] **Step 3: Add `capabilities` field to `AgentGoal`**

Use `ide_edit_member` to replace the entire `AgentGoal` record:

```java
public record AgentGoal(
        String name,
        String description,
        GoalPriority priority,
        Visibility visibility,
        List<String> capabilities
) {
    public AgentGoal {
        AgentDescriptorValidator.validateRequired("goal.name", name,
            AgentDescriptorValidator.MAX_GOAL_NAME);
        AgentDescriptorValidator.validateRequired("goal.description", description,
            AgentDescriptorValidator.MAX_GOAL_DESCRIPTION);
        Objects.requireNonNull(priority, "goal.priority must not be null");
        Objects.requireNonNull(visibility, "goal.visibility must not be null");
        capabilities = capabilities != null
            ? List.copyOf(capabilities.stream().filter(Objects::nonNull).toList())
            : List.of();
        AgentDescriptorValidator.validateItems("goal.capabilities", capabilities,
            AgentDescriptorValidator.MAX_CAPABILITY_NAME);
        if (capabilities.size() != capabilities.stream().distinct().count()) {
            throw new AgentValidationException("goal.capabilities",
                "duplicate capability name in goal '" + name + "'");
        }
    }
}
```

Add `import java.util.List;` and `import java.util.Objects;` to imports.

- [ ] **Step 4: Fix all existing `AgentGoal` construction sites to pass `List.of()` as 5th arg**

Every existing `new AgentGoal(name, desc, priority, visibility)` call must become `new AgentGoal(name, desc, priority, visibility, List.of())`. The full list of construction sites (verified via `ide_find_references`):

**api module:**
- `api/src/test/java/io/casehub/eidos/api/AgentGoalTest.java` — all existing tests (add `List.of()`)
- `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java:297,298,319,367,368`
- `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java:360`
- `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java:414,425,433,434,443,444,453,454`

**runtime module:**
- `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java:118`
- `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java:112`
- `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultGoalEvolution.java:98,101`

**persistence-memory module:**
- `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java:296,314`

**examples module:**
- `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CrossVocabularyAgentDesignTest.java:70,72`

**runtime tests:**
- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java:1008,1010,1070,1127,1139`
- `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java:637,638,667,685`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultGoalEvolutionTest.java:24,42,43,57,58,69,70,86,95,96`

Use `ide_replace_text_in_file` with regex on each file to add the 5th argument. Pattern: find `new AgentGoal("` 4-arg constructions and add `, List.of()` before the closing `)`.

- [ ] **Step 5: Run full API module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: PASS

- [ ] **Step 6: Commit**

```
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#135): add capabilities field to AgentGoal record

Refs casehubio/eidos#135"
```

---

### Task 2: Cross-validation in `AgentDescriptor` compact constructor

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java:30-96` (compact constructor)
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`

**Interfaces:**
- Consumes: `AgentGoal.capabilities()` from Task 1
- Produces: `AgentDescriptor` constructor throws `AgentValidationException` when goal references unknown capability

- [ ] **Step 1: Write failing test**

Add to `AgentDescriptorValidatorTest.java`:

```java
@Test void goal_referencing_unknown_capability_throws() {
    var caps = List.of(AgentCapability.builder().name("code-review").build());
    var goals = List.of(new AgentGoal("quality", "Ensure quality",
        GoalPriority.PRIMARY, Visibility.PUBLIC, List.of("unknown-cap")));
    assertThatThrownBy(() -> AgentDescriptor.builder()
        .agentId("a").name("A").slot("s").tenancyId("t")
        .capabilities(caps).goals(goals).build())
        .isInstanceOf(AgentValidationException.class)
        .hasMessageContaining("unknown capability")
        .hasMessageContaining("unknown-cap");
}

@Test void goal_referencing_declared_capability_succeeds() {
    var caps = List.of(AgentCapability.builder().name("code-review").build());
    var goals = List.of(new AgentGoal("quality", "Ensure quality",
        GoalPriority.PRIMARY, Visibility.PUBLIC, List.of("code-review")));
    assertThatNoException().isThrownBy(() -> AgentDescriptor.builder()
        .agentId("a").name("A").slot("s").tenancyId("t")
        .capabilities(caps).goals(goals).build());
}

@Test void goal_with_empty_capabilities_succeeds() {
    var caps = List.of(AgentCapability.builder().name("code-review").build());
    var goals = List.of(new AgentGoal("quality", "Ensure quality",
        GoalPriority.PRIMARY, Visibility.PUBLIC, List.of()));
    assertThatNoException().isThrownBy(() -> AgentDescriptor.builder()
        .agentId("a").name("A").slot("s").tenancyId("t")
        .capabilities(caps).goals(goals).build());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest`
Expected: `goal_referencing_unknown_capability_throws` FAILS (no validation yet)

- [ ] **Step 3: Add cross-validation to `AgentDescriptor` compact constructor**

In `AgentDescriptor.java`, after the existing goals duplicate-name check block (after line ~85), add:

```java
if (!goals.isEmpty() && !capabilities.isEmpty()) {
    var capabilityNames = capabilities.stream()
        .map(AgentCapability::name).collect(java.util.stream.Collectors.toSet());
    for (var goal : goals) {
        for (var capName : goal.capabilities()) {
            if (!capabilityNames.contains(capName)) {
                throw new AgentValidationException("goals",
                    "goal '" + goal.name() + "' references unknown capability '" + capName + "'");
            }
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: PASS

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#135): cross-validate goal capabilities against declared capabilities

Refs casehubio/eidos#135"
```

---

### Task 3: YAML parsing, JPA persistence, goal evolution, and remaining construction sites

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentGoalEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultGoalEvolution.java:98,101`
- Modify: `runtime/src/main/resources/db/eidos/migration/V8__goals_constraints.sql`
- Modify: all runtime/persistence-memory/examples test files with `new AgentGoal(...)` calls
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultGoalEvolutionTest.java`

**Interfaces:**
- Consumes: `AgentGoal` 5-arg constructor from Task 1
- Produces: YAML `capabilities:` list on goals, JPA round-trip, evolution pass-through

- [ ] **Step 1: Write failing test for YAML parsing**

Add to `ClasspathYamlDescriptorRegistrarTest.java` (find the existing test class):

```java
@Test void yaml_goal_capabilities_parsed() {
    var yaml = """
        descriptors:
          - agentId: test-agent
            name: Test
            slot: tester
            tenancyId: test-tenant
            capabilities:
              - name: code-review
              - name: testing
            goals:
              - name: quality
                description: Ensure quality
                priority: PRIMARY
                visibility: PUBLIC
                capabilities:
                  - code-review
                  - testing
        """;
    var registrar = new ClasspathYamlDescriptorRegistrar();
    var descriptors = registrar.loadFrom(new java.io.ByteArrayInputStream(yaml.getBytes()));
    assertThat(descriptors).hasSize(1);
    assertThat(descriptors.get(0).goals().get(0).capabilities())
        .containsExactly("code-review", "testing");
}

@Test void yaml_goal_without_capabilities_has_empty_list() {
    var yaml = """
        descriptors:
          - agentId: test-agent
            name: Test
            slot: tester
            tenancyId: test-tenant
            goals:
              - name: quality
                description: Ensure quality
                priority: PRIMARY
                visibility: PUBLIC
        """;
    var registrar = new ClasspathYamlDescriptorRegistrar();
    var descriptors = registrar.loadFrom(new java.io.ByteArrayInputStream(yaml.getBytes()));
    assertThat(descriptors.get(0).goals().get(0).capabilities()).isEmpty();
}
```

- [ ] **Step 2: Run to verify failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlDescriptorRegistrarTest`
Expected: FAIL — `GoalConfig` doesn't have `capabilities` field

- [ ] **Step 3: Update YAML parsing**

In `ClasspathYamlDescriptorRegistrar.java`:

Add `public List<String> capabilities;` to `GoalConfig` inner class.

Update `toDescriptor` goal mapping (line ~118):
```java
new AgentGoal(g.name, g.description, g.priority, g.visibility, g.capabilities)
```

- [ ] **Step 4: Update JPA entity and mapper**

In `AgentGoalEntity.java`, add:
```java
@Column(name = "capabilities") String capabilities;
```

In `AgentDescriptorMapper.java`, update `toGoal` (line ~112):
```java
private AgentGoal toGoal(AgentGoalEntity g) {
    return new AgentGoal(g.name, g.description,
                         GoalPriority.valueOf(g.priority),
                         Visibility.valueOf(g.visibility),
                         readJson(g.capabilities, new TypeReference<List<String>>() {}));
}
```

Update `toGoalEntity` (line ~117), add after `e.visibility`:
```java
e.capabilities = writeJson(g.capabilities());
```

- [ ] **Step 5: Update Flyway migration**

In `V8__goals_constraints.sql`, add `capabilities TEXT,` column to `agent_goal` table after the `visibility` line:
```sql
capabilities   TEXT,
```

- [ ] **Step 6: Update `DefaultGoalEvolution` construction sites**

Both promotion (line 98) and demotion (line 101):
```java
// line 98 — promotion
return new AgentGoal(g.name(), g.description(), GoalPriority.PRIMARY, g.visibility(), g.capabilities());
// line 101 — demotion
return new AgentGoal(g.name(), g.description(), GoalPriority.SECONDARY, g.visibility(), g.capabilities());
```

- [ ] **Step 7: Fix all remaining `new AgentGoal(...)` construction sites in runtime/persistence-memory/examples tests**

Add `List.of()` as 5th argument to every remaining 4-arg `AgentGoal` constructor call. Files:

- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultGoalEvolutionTest.java`
- `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java`
- `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CrossVocabularyAgentDesignTest.java`

Use `ide_replace_text_in_file` with regex per file.

- [ ] **Step 8: Write JPA round-trip test**

Add to `JpaAgentRegistryTest.java`:

```java
@Test void goal_capabilities_persist_and_retrieve() {
    var caps = List.of(AgentCapability.builder().name("code-review").build());
    var goals = List.of(new AgentGoal("quality", "Ensure quality",
        GoalPriority.PRIMARY, Visibility.PUBLIC, List.of("code-review")));
    var descriptor = AgentDescriptor.builder()
        .agentId("cap-goal-test").name("Test").slot("s").tenancyId("test-tenant")
        .capabilities(caps).goals(goals).build();
    registry.register(descriptor);
    var retrieved = registry.findById("cap-goal-test", "test-tenant").orElseThrow();
    assertThat(retrieved.goals().get(0).capabilities()).containsExactly("code-review");
}
```

- [ ] **Step 9: Write goal evolution pass-through test**

Add to `DefaultGoalEvolutionTest.java`:

```java
@Test void evolution_preserves_capabilities() {
    var descriptor = descriptorWithGoals(
        new AgentGoal("primary", "Primary", GoalPriority.PRIMARY, Visibility.PUBLIC, List.of("cap-a")),
        new AgentGoal("rising", "Rising", GoalPriority.SECONDARY, Visibility.PUBLIC, List.of("cap-b")));
    var counts = Map.of(
        "rising", new GoalOutcomeCounts(20, 0));
    var result = evolution.evaluate(descriptor, counts);
    assertThat(result).isInstanceOf(GoalEvolutionResult.Evolved.class);
    var evolved = (GoalEvolutionResult.Evolved) result;
    var risingGoal = evolved.newGoals().stream()
        .filter(g -> g.name().equals("rising")).findFirst().orElseThrow();
    assertThat(risingGoal.priority()).isEqualTo(GoalPriority.PRIMARY);
    assertThat(risingGoal.capabilities()).containsExactly("cap-b");
}
```

- [ ] **Step 10: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 11: Commit**

```
git -C /Users/mdproctor/claude/casehub/eidos add runtime/ persistence-memory/ examples/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#135): YAML parsing, JPA persistence, goal evolution for capabilities

Refs casehubio/eidos#135"
```

---

### Task 4: Update `AgentDescriptorComparator` and engine construction sites

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java:127-145`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java`

**Interfaces:**
- Consumes: `AgentGoal.capabilities()` from Task 1

The comparator compares goals between desired and actual descriptors. It currently compares name, description, priority, visibility. It must now also compare `capabilities`. The test at line 399 (`comparatorCoversAllGoalComponents`) counts record components and will fail if not updated.

- [ ] **Step 1: Verify `AgentDescriptorComparator` needs updating**

Use `ide_read_file` to read `AgentDescriptorComparator.java` lines 127-155. The `compareGoals` method compares goal fields — `capabilities` must be added.

- [ ] **Step 2: Update comparator**

Add capabilities comparison alongside the existing priority/visibility/description comparisons in `compareGoals`.

- [ ] **Step 3: Run comparator tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorComparatorTest`
Expected: PASS (the component count test at line 399 must pass — it asserts the comparator covers all record components)

- [ ] **Step 4: Commit**

```
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#135): update AgentDescriptorComparator for capabilities field

Refs casehubio/eidos#135"
```

---

### Task 5: Final full build and engine cross-repo construction sites

**Files:**
- No eidos files created or modified
- Engine construction sites are out of scope (engine#860) but must not break

- [ ] **Step 1: Full clean build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 2: Check engine compilation**

The engine workspace has `AgentGoal` construction sites that will now fail compilation (4-arg → 5-arg). These are in:
- `engine/runtime/src/main/java/io/casehub/engine/internal/routing/GoalAbandonmentEvaluator.java`
- `engine/runtime/src/test/java/.../GoalFailureRecorderTest.java`
- `engine/runtime/src/test/java/.../GoalAbandonmentEvaluatorTest.java`
- `engine/runtime/src/test/java/.../GoalSignalProviderTest.java`
- `engine/runtime/src/test/java/.../AgentGoalCompletionMarkerTest.java`

These must be updated in the engine repo as part of engine#860 or as a compatibility fix. File an issue or note in HANDOFF.md.

- [ ] **Step 3: Commit (if any final fixes needed)**

Only if Step 1 revealed issues. Otherwise skip.
