# Goals Query, API Enhancements, and Template Examples — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #101 — Goal-based querying, routing, and advanced goal use cases
**Issue group:** #101, #102, #103, #105

**Goal:** Add capability name uniqueness validation, constraint severity, goal-based querying with A2A awareness, and descriptor template examples.

**Architecture:** Pre-release platform — breaking changes cost nothing. All edits via IntelliJ MCP (`ide_edit_member`, `ide_replace_member`, `ide_insert_member`, `ide_create_file`). Flyway migrations rewrite base files (no deployed instances). TDD throughout.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, Hibernate 6, H2 (test), Jackson

## Global Constraints

- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` for all Maven commands
- Maven command: `/opt/homebrew/bin/mvn`
- IntelliJ MCP `project_path`: `/Users/mdproctor/claude/casehub/eidos`
- No deployed instances — schema rewrites to base migration files
- All commits reference issues: `Refs casehubio/eidos#N`
- Verify with `ide_diagnostics` after structural edits
- Run module tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f <module>/pom.xml`

---

### Task 1: ConstraintSeverity — API layer (#103)

This task must be first — it changes the `AgentConstraint` record signature, which breaks every construction site. All downstream tasks depend on this.

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/ConstraintSeverity.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentConstraint.java` — add `severity` field
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java` — severity in constraint comparison, increment field count
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentConstraintTest.java` — all 8 construction sites + new severity tests
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java` — 5 construction sites
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java` — 5 construction sites + severity drift test
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` — toConstraint() and toConstraintEntity()
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java` — ConstraintConfig + toDescriptor()
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java` — 2 construction sites
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java` — 2 construction sites

**Interfaces:**
- Produces: `ConstraintSeverity { HARD, SOFT }` enum used by all subsequent tasks
- Produces: `AgentConstraint(String name, String description, Visibility visibility, ConstraintSeverity severity)` — new 4-arg signature

- [ ] **Step 1: Write failing test — severity validation**

In `AgentConstraintTest.java`, add test for null severity rejection:

```java
@Test void rejects_null_severity() {
    assertThatThrownBy(() -> new AgentConstraint("c", "desc", Visibility.PUBLIC, null))
        .isInstanceOf(NullPointerException.class)
        .hasMessageContaining("constraint.severity");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f api/pom.xml -Dtest=AgentConstraintTest`
Expected: compilation error — `ConstraintSeverity` doesn't exist yet.

- [ ] **Step 3: Create ConstraintSeverity enum**

Use `ide_create_file`:

```java
package io.casehub.eidos.api;

public enum ConstraintSeverity { HARD, SOFT }
```

- [ ] **Step 4: Add severity field to AgentConstraint**

Use `ide_edit_member` on `AgentConstraint` record to add `severity` as 4th component and add `Objects.requireNonNull(severity, "constraint.severity must not be null")` to the compact constructor.

- [ ] **Step 5: Fix all 24 construction sites**

Every `new AgentConstraint(name, desc, visibility)` becomes `new AgentConstraint(name, desc, visibility, ConstraintSeverity.HARD)` (or `.SOFT` where semantically appropriate). Use `ide_search_text` to find them all, then `ide_replace_member` / `ide_edit_member` for each.

Files and line counts:
- `AgentConstraintTest.java` — 8 sites (lines 10, 18, 24, 29, 34, 40, 45, 50)
- `AgentDescriptorValidatorTest.java` — 5 sites (lines 308, 309, 330, 352, 353)
- `AgentDescriptorComparatorTest.java` — 5 sites (lines 464, 473, 474, 483, 484)
- `JpaAgentRegistryTest.java` — 2 sites (lines 639, 640)
- `EidosRenderPipelineTest.java` — 2 sites (lines 1013, 1014)
- `AgentDescriptorMapper.java:121` — toConstraint() method
- `ClasspathYamlDescriptorRegistrar.java:82` — toDescriptor()

For the mapper `toConstraint()`:
```java
return new AgentConstraint(c.name, c.description,
                           Visibility.valueOf(c.visibility),
                           ConstraintSeverity.valueOf(c.severity));
```

For the mapper `toConstraintEntity()`, add:
```java
e.severity = c.severity().name();
```

For `ClasspathYamlDescriptorRegistrar`:
- Add `ConstraintSeverity severity;` to `ConstraintConfig` inner class
- Update the mapping: `new AgentConstraint(c.name, c.description, c.visibility, c.severity)`

- [ ] **Step 6: Add severity to AgentConstraintEntity**

Use `ide_insert_member` to add field after `visibility`:
```java
@Column(nullable = false) String severity;
```

- [ ] **Step 7: Update AgentDescriptorComparator**

Use `ide_edit_member` on `COMPARED_CONSTRAINT_FIELD_COUNT` to change from `2` to `3`.

Use `ide_replace_member` on `compareConstraints` to add severity comparison. In the per-constraint comparison block (after the `visibility` comparison), add:
```java
compareField(drifts, prefix + "severity", entry.getValue().severity(), actualC.severity());
```

- [ ] **Step 8: Write comparator drift test for severity**

In `AgentDescriptorComparatorTest.java`, add test:
```java
@Test void detects_constraint_severity_drift() {
    var c1 = new AgentConstraint("c", "d", Visibility.PUBLIC, ConstraintSeverity.HARD);
    var c2 = new AgentConstraint("c", "d", Visibility.PUBLIC, ConstraintSeverity.SOFT);
    // build two descriptors with c1 vs c2, compare, assert drift on "constraints[c].severity"
}
```

- [ ] **Step 9: Run all api and runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f api/pom.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f runtime/pom.xml`
Expected: all pass.

- [ ] **Step 10: Commit**

```
feat(#103): ConstraintSeverity — HARD/SOFT on AgentConstraint

Add ConstraintSeverity enum (HARD, SOFT) as 4th field on AgentConstraint.
Update comparator (3 constraint fields), mapper, YAML registrar, entity.
All construction sites updated across api/, runtime/, tests.

Refs casehubio/eidos#103
```

---

### Task 2: Capability name uniqueness — API + Flyway (#102)

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` — add `MAX_CAPABILITIES`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — capability count + uniqueness checks
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql` — UNIQUE constraint
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java` — @UniqueConstraint annotation
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`

**Interfaces:**
- Consumes: nothing new
- Produces: validation constraint — duplicate capability names throw `AgentValidationException`

- [ ] **Step 1: Write failing test — duplicate capability names**

In `AgentDescriptorValidatorTest.java`, add test:

```java
@Test void rejects_duplicate_capability_names() {
    var cap1 = AgentCapability.builder().name("code-review").build();
    var cap2 = AgentCapability.builder().name("code-review").build();
    assertThatThrownBy(() -> AgentDescriptor.builder()
            .agentId("a").name("A").slot("s").tenancyId("t")
            .capabilities(List.of(cap1, cap2))
            .build())
        .isInstanceOf(AgentValidationException.class)
        .hasMessageContaining("duplicate capability name: code-review");
}
```

- [ ] **Step 2: Write failing test — capability count limit**

```java
@Test void rejects_capabilities_exceeding_max() {
    var caps = java.util.stream.IntStream.range(0, 21)
        .mapToObj(i -> AgentCapability.builder().name("cap-" + i).build())
        .toList();
    assertThatThrownBy(() -> AgentDescriptor.builder()
            .agentId("a").name("A").slot("s").tenancyId("t")
            .capabilities(caps)
            .build())
        .isInstanceOf(AgentValidationException.class)
        .hasMessageContaining("exceeds maximum count 20");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f api/pom.xml -Dtest=AgentDescriptorValidatorTest#rejects_duplicate_capability_names+rejects_capabilities_exceeding_max`
Expected: FAIL — no validation exists yet.

- [ ] **Step 4: Add MAX_CAPABILITIES to AgentDescriptorValidator**

Use `ide_insert_member` to add after `MAX_CONSTRAINTS`:
```java
static final int MAX_CAPABILITIES = 20;
```

- [ ] **Step 5: Add capability validation to AgentDescriptor compact constructor**

Use `ide_edit_member` on the `AgentDescriptor` compact constructor. Add after the `capabilities = capabilities != null ? List.copyOf(capabilities) : List.of();` line, before the `axisVocabularies` block:

```java
if (capabilities.size() > AgentDescriptorValidator.MAX_CAPABILITIES) {
    throw new AgentValidationException("capabilities",
        "exceeds maximum count " + AgentDescriptorValidator.MAX_CAPABILITIES + " (was " + capabilities.size() + ")");
}
if (capabilities.size() > 1) {
    long distinctNames = capabilities.stream().map(AgentCapability::name).distinct().count();
    if (distinctNames < capabilities.size()) {
        String dup = capabilities.stream().map(AgentCapability::name)
            .collect(java.util.stream.Collectors.groupingBy(n -> n, java.util.stream.Collectors.counting()))
            .entrySet().stream().filter(e -> e.getValue() > 1).map(java.util.Map.Entry::getKey)
            .findFirst().orElse("?");
        throw new AgentValidationException("capabilities", "duplicate capability name: " + dup);
    }
}
```

- [ ] **Step 6: Add UNIQUE constraint to V1 migration**

Use `Edit` tool (non-Java file) to add to `V1__initial_schema.sql` agent_capability CREATE TABLE:
```sql
UNIQUE (descriptor_id, name)
```
after the `epistemic_domains TEXT` line.

- [ ] **Step 7: Add @UniqueConstraint to AgentCapabilityEntity**

Use `ide_edit_member` on the `@Table` annotation:
```java
@Table(name = "agent_capability",
       uniqueConstraints = @UniqueConstraint(columnNames = {"descriptor_id", "name"}))
```

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f api/pom.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn clean test -f runtime/pom.xml`
Expected: all pass.

- [ ] **Step 9: Commit**

```
feat(#102): capability name uniqueness — constructor validation + DDL constraint

Add MAX_CAPABILITIES = 20 count check and duplicate name validation to
AgentDescriptor compact constructor. Add UNIQUE (descriptor_id, name) to
agent_capability in V1 base migration and JPA @UniqueConstraint.

Refs casehubio/eidos#102
```

---

### Task 3: Flyway V8 — add severity column to agent_constraint (#103)

**Files:**
- Modify: `runtime/src/main/resources/db/eidos/migration/V8__goals_constraints.sql` — add `severity` column

**Interfaces:**
- Consumes: `ConstraintSeverity` from Task 1
- Produces: `severity VARCHAR(20) NOT NULL` in DDL

- [ ] **Step 1: Add severity column to V8 migration**

Use `Edit` tool (SQL file) to add `severity VARCHAR(20) NOT NULL,` after the `visibility` line in the `agent_constraint` CREATE TABLE.

- [ ] **Step 2: Run runtime tests to verify schema works**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn clean test -f runtime/pom.xml`
Expected: all pass — Flyway creates the table with severity, entities map it.

- [ ] **Step 3: Commit**

```
feat(#103): add severity column to agent_constraint schema

Base-migration rewrite — severity VARCHAR(20) NOT NULL in V8 CREATE TABLE.
No deployed instances.

Refs casehubio/eidos#103
```

---

### Task 4: ConstraintSeverity — renderer (#103)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java` — 4 methods
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

**Interfaces:**
- Consumes: `AgentConstraint.severity()`, `ConstraintSeverity { HARD, SOFT }`
- Produces: severity-discriminated rendering in MARKDOWN, PROSE, A2A_CARD; severity in hash payload

- [ ] **Step 1: Write failing tests — MARKDOWN severity rendering**

In `EidosRenderPipelineTest.java`, add test:

```java
@Test void markdown_renders_constraint_severity_labels() {
    var desc = descriptorWithConstraints(
        new AgentConstraint("no-pii", "Never expose PII", Visibility.PUBLIC, ConstraintSeverity.HARD),
        new AgentConstraint("be-polite", "Use polite language", Visibility.PUBLIC, ConstraintSeverity.SOFT));
    var rendered = renderMarkdown(desc);
    assertThat(rendered.content()).contains("**[HARD]**");
    assertThat(rendered.content()).contains("**[SOFT]**");
    // HARD before SOFT in output
    assertThat(rendered.content().indexOf("[HARD]")).isLessThan(rendered.content().indexOf("[SOFT]"));
}
```

- [ ] **Step 2: Write failing tests — PROSE severity rendering**

```java
@Test void prose_renders_constraints_grouped_by_severity() {
    var desc = descriptorWithConstraints(
        new AgentConstraint("no-pii", "Never expose PII", Visibility.PUBLIC, ConstraintSeverity.HARD),
        new AgentConstraint("be-polite", "Use polite language", Visibility.PUBLIC, ConstraintSeverity.SOFT));
    var rendered = renderProse(desc);
    assertThat(rendered.content()).contains("Hard constraints:");
    assertThat(rendered.content()).contains("Also:");
}
```

- [ ] **Step 3: Write failing test — A2A_CARD severity field**

```java
@Test void a2a_card_includes_constraint_severity() {
    var desc = descriptorWithConstraints(
        new AgentConstraint("no-pii", "Never expose PII", Visibility.PUBLIC, ConstraintSeverity.HARD));
    var rendered = renderA2aCard(desc);
    assertThat(rendered.content()).contains("\"severity\":\"HARD\"");
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f runtime/pom.xml -Dtest=EidosRenderPipelineTest`
Expected: FAIL — current rendering has no severity labels.

- [ ] **Step 5: Update assembleMarkdownConstraints**

Use `ide_replace_member` on `assembleMarkdownConstraints`:

```java
private void assembleMarkdownConstraints(final StringBuilder sb, final AgentDescriptor descriptor) {
    if (descriptor.constraints().isEmpty()) {return;}
    sb.append("\n## Constraints\n");
    descriptor.constraints().stream()
              .sorted(java.util.Comparator.comparing(AgentConstraint::severity)
                                          .thenComparing(AgentConstraint::name))
              .forEach(c -> sb.append("- **[").append(c.severity().name()).append("]** ")
                              .append(c.description()).append("\n"));
}
```

- [ ] **Step 6: Update assembleProse constraint section**

In `assembleProse`, replace the constraint block (lines ~614-619) with severity-grouped rendering:

```java
if (!descriptor.constraints().isEmpty()) {
    var sorted = descriptor.constraints().stream()
                           .sorted(java.util.Comparator.comparing(AgentConstraint::severity)
                                                       .thenComparing(AgentConstraint::name))
                           .toList();
    var hard = sorted.stream().filter(c -> c.severity() == ConstraintSeverity.HARD).toList();
    var soft = sorted.stream().filter(c -> c.severity() == ConstraintSeverity.SOFT).toList();
    if (!hard.isEmpty()) {
        sb.append("\nHard constraints: ");
        sb.append(hard.stream().map(AgentConstraint::description).collect(Collectors.joining(". ")));
        sb.append(".");
    }
    if (!soft.isEmpty()) {
        sb.append(hard.isEmpty() ? "\nConstraints: " : " Also: ");
        sb.append(soft.stream().map(AgentConstraint::description).collect(Collectors.joining(". ")));
        sb.append(".");
    }
    sb.append("\n");
}
```

- [ ] **Step 7: Update assembleA2aCard constraint section**

In `assembleA2aCard`, add `severity` to the constraint node. After `cNode.put("description", c.description());` add:
```java
cNode.put("severity", c.severity().name());
```

Also update sorting to severity-then-name:
```java
.sorted(java.util.Comparator.comparing(AgentConstraint::severity)
                             .thenComparing(AgentConstraint::name))
```

- [ ] **Step 8: Update buildDescriptorPayload constraint section**

In `buildDescriptorPayload`, add `severity` to constraint hash objects. After `cNode.put("visibility", c.visibility().name());` add:
```java
cNode.put("severity", c.severity().name());
```

- [ ] **Step 9: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f runtime/pom.xml`
Expected: all pass.

- [ ] **Step 10: Commit**

```
feat(#103): severity-discriminated constraint rendering

MARKDOWN: [HARD]/[SOFT] label prefix, sorted severity-then-name.
PROSE: grouped by severity (hard first, soft with "Also:").
A2A_CARD: severity field on constraint objects.
Hash payload: severity included for cache invalidation.

Refs casehubio/eidos#103
```

---

### Task 5: Goal-based querying — API layer (#101)

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentQuery.java` — add `goalName` field + factory
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — `hasGoal()`, `hasConstraint()`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentQueryTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java`

**Interfaces:**
- Produces: `AgentQuery(String slot, String capabilityName, String tenancyId, String taskDomain, String goalName)` — 5-arg record
- Produces: `AgentQuery.byGoal(String goalName, String tenancyId)` — factory method
- Produces: `AgentDescriptor.hasGoal(String name)` → boolean
- Produces: `AgentDescriptor.hasConstraint(String name)` → boolean

- [ ] **Step 1: Write failing tests — AgentQuery.byGoal()**

In `AgentQueryTest.java`:

```java
@Test void byGoal_sets_goalName_and_tenancyId() {
    var q = AgentQuery.byGoal("quality-review", "t1");
    assertThat(q.goalName()).isEqualTo("quality-review");
    assertThat(q.tenancyId()).isEqualTo("t1");
    assertThat(q.slot()).isNull();
    assertThat(q.capabilityName()).isNull();
    assertThat(q.taskDomain()).isNull();
}
```

- [ ] **Step 2: Write failing tests — hasGoal/hasConstraint**

In `AgentDescriptorTest.java`:

```java
@Test void hasGoal_returns_true_for_existing_goal() {
    var desc = AgentDescriptor.builder()
        .agentId("a").name("A").slot("s").tenancyId("t")
        .goals(List.of(new AgentGoal("quality", "Ensure quality", GoalPriority.PRIMARY, Visibility.PUBLIC)))
        .build();
    assertThat(desc.hasGoal("quality")).isTrue();
    assertThat(desc.hasGoal("nonexistent")).isFalse();
}

@Test void hasConstraint_returns_true_for_existing_constraint() {
    var desc = AgentDescriptor.builder()
        .agentId("a").name("A").slot("s").tenancyId("t")
        .constraints(List.of(new AgentConstraint("no-pii", "Never expose PII", Visibility.PUBLIC, ConstraintSeverity.HARD)))
        .build();
    assertThat(desc.hasConstraint("no-pii")).isTrue();
    assertThat(desc.hasConstraint("nonexistent")).isFalse();
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f api/pom.xml -Dtest=AgentQueryTest+AgentDescriptorTest`
Expected: compilation error — `goalName` and `hasGoal` don't exist.

- [ ] **Step 4: Add goalName to AgentQuery**

Use `ide_edit_member` on the `AgentQuery` record to add `goalName` as 5th component. Update compact constructor (no new validation — goalName is optional like slot). Fix existing factory methods to pass `null` as 5th arg.

Add factory method:
```java
public static AgentQuery byGoal(String goalName, String tenancyId) {
    return new AgentQuery(null, null, tenancyId, null, goalName);
}
```

- [ ] **Step 5: Add hasGoal and hasConstraint to AgentDescriptor**

Use `ide_insert_member` after `publicConstraints()`:

```java
public boolean hasGoal(String name) {
    return goals.stream().anyMatch(g -> g.name().equals(name));
}

public boolean hasConstraint(String name) {
    return constraints.stream().anyMatch(c -> c.name().equals(name));
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f api/pom.xml`
Expected: all pass.

- [ ] **Step 7: Commit**

```
feat(#101): goalName on AgentQuery + hasGoal/hasConstraint on AgentDescriptor

Add goalName field to AgentQuery with byGoal() factory method.
Add hasGoal(String) and hasConstraint(String) convenience methods
on AgentDescriptor.

Refs casehubio/eidos#101
```

---

### Task 6: Goal-based querying — registries (#101)

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java` — goal filter
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/JpaAgentRegistry.java` — EXISTS subquery
- Test: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentRegistryTest.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`

**Interfaces:**
- Consumes: `AgentQuery.goalName()`, `AgentQuery.byGoal()`
- Produces: goal-filtered results from `AgentRegistry.find()`

- [ ] **Step 1: Write failing test — InMemoryAgentRegistry goal filter**

In `InMemoryAgentRegistryTest.java`:

```java
@Test void find_by_goal_returns_matching_agents() {
    var desc1 = AgentDescriptor.builder()
        .agentId("a1").name("A1").slot("s").tenancyId("t")
        .goals(List.of(new AgentGoal("quality-review", "Ensure quality", GoalPriority.PRIMARY, Visibility.PUBLIC)))
        .build();
    var desc2 = AgentDescriptor.builder()
        .agentId("a2").name("A2").slot("s").tenancyId("t")
        .build();
    registry.register(desc1);
    registry.register(desc2);

    var results = registry.find(AgentQuery.byGoal("quality-review", "t"));
    assertThat(results).hasSize(1);
    assertThat(results.get(0).descriptor().agentId()).isEqualTo("a1");
    assertThat(results.get(0).resolvedCapability()).isNull();
}

@Test void find_by_goal_returns_empty_when_no_match() {
    var desc = AgentDescriptor.builder()
        .agentId("a1").name("A1").slot("s").tenancyId("t")
        .goals(List.of(new AgentGoal("quality", "Q", GoalPriority.PRIMARY, Visibility.PUBLIC)))
        .build();
    registry.register(desc);

    var results = registry.find(AgentQuery.byGoal("nonexistent", "t"));
    assertThat(results).isEmpty();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f persistence-memory/pom.xml -Dtest=InMemoryAgentRegistryTest#find_by_goal_returns_matching_agents`
Expected: FAIL — no goal filtering in `find()`.

- [ ] **Step 3: Add goal filter to InMemoryAgentRegistry**

Use `ide_replace_member` on `find` method. Add filter after the `taskDomain` filter:

```java
.filter(d -> query.goalName() == null
    || d.goals().stream().anyMatch(g -> g.name().equals(query.goalName())))
```

- [ ] **Step 4: Run InMemoryAgentRegistry tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -f persistence-memory/pom.xml`
Expected: all pass.

- [ ] **Step 5: Write failing test — JpaAgentRegistry goal filter**

In `JpaAgentRegistryTest.java`:

```java
@Test void find_by_goal_returns_matching_agents() {
    var desc1 = AgentDescriptor.builder()
        .agentId("g1").name("G1").slot("s").tenancyId("t")
        .goals(List.of(new AgentGoal("quality-review", "Q", GoalPriority.PRIMARY, Visibility.PUBLIC)))
        .build();
    var desc2 = AgentDescriptor.builder()
        .agentId("g2").name("G2").slot("s").tenancyId("t")
        .build();
    registry.register(desc1);
    registry.register(desc2);

    var results = registry.find(AgentQuery.byGoal("quality-review", "t"));
    assertThat(results).hasSize(1);
    assertThat(results.get(0).descriptor().agentId()).isEqualTo("g1");
}
```

- [ ] **Step 6: Add EXISTS subquery to JpaAgentRegistry.find()**

Use `ide_replace_member` on `find` method. After the `taskDomain` filter block, add:

```java
if (query.goalName() != null) {
    jpql.append(" AND EXISTS (SELECT 1 FROM AgentGoalEntity g WHERE g.descriptor = a AND g.name = :goalName)");
}
```

And set the parameter after the existing parameter-setting block:
```java
if (query.goalName() != null) q.setParameter("goalName", query.goalName());
```

- [ ] **Step 7: Run JPA registry tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn clean test -f runtime/pom.xml -Dtest=JpaAgentRegistryTest`
Expected: all pass.

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn clean test`
Expected: BUILD SUCCESS.

- [ ] **Step 9: Commit**

```
feat(#101): goal-based querying in registries

InMemoryAgentRegistry: stream filter on goals by name.
JpaAgentRegistry: EXISTS subquery avoids MultipleBagFetchException.
Goal query returns unranked AgentMatch with null resolvedCapability.

Refs casehubio/eidos#101
```

---

### Task 7: Template examples (#105)

**Files:**
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DescriptorTemplateScenarioTest.java`
- Possibly modify: `examples/agent-scenarios/src/main/resources/META-INF/eidos/descriptors.yaml` and `templates.yaml`

**Interfaces:**
- Consumes: `TemplateRegistry`, `AgentRegistry`, `SystemPromptRenderer` (existing CDI beans)

- [ ] **Step 1: Write test — multi-template ordering**

```java
@Test void template_order_preserved_in_rendered_output() {
    var desc = registry.findById("drafthouse-structural-reviewer", "drafthouse").orElseThrow();
    assertThat(desc.templates()).hasSizeGreaterThanOrEqualTo(2);
    var ctx = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
    var rendered = renderer.render(desc, ctx);
    var t1Content = templateRegistry.resolve(desc.templates().get(0).templateId()).orElseThrow().content();
    var t2Content = templateRegistry.resolve(desc.templates().get(1).templateId()).orElseThrow().content();
    // Extract a distinctive substring from each template's rendered content
    // and verify ordering is preserved
    String firstKey = extractKeyPhrase(t1Content);
    String secondKey = extractKeyPhrase(t2Content);
    assertThat(rendered.content().indexOf(firstKey))
        .isLessThan(rendered.content().indexOf(secondKey));
}
```

Note: The exact key phrases depend on the template content already registered. Use `specific line references` (from document-review-conventions) and `professional register` (from communication-style) as the phrases — these are already asserted in existing tests.

- [ ] **Step 2: Write test — shared templates with different args**

Register a second descriptor that references the same `communication-style` template with different args. Verify both render correctly with their own arg values.

```java
@Test void shared_template_different_args_no_cross_contamination() {
    var casual = registry.findById("drafthouse-structural-reviewer", "drafthouse").orElseThrow();
    // The existing descriptor uses formality=professional, feedback_approach=collaborative
    var casualCtx = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
    var casualRendered = renderer.render(casual, casualCtx);
    assertThat(casualRendered.content()).contains("professional register");
    assertThat(casualRendered.content()).contains("collaborative approach");
    assertThat(casualRendered.content()).doesNotContain("${formality}");
}
```

If a second descriptor with different args is not already registered, create one programmatically in the test using InMemoryAgentRegistry or verify the existing test coverage is sufficient (the `parameterized_template_substituted` test already validates no raw `${var}` leaks).

- [ ] **Step 3: Write test — YAML template registration**

```java
@Test void yaml_registered_template_resolves_and_renders() {
    // document-review-conventions is loaded from META-INF/eidos/templates.yaml
    var template = templateRegistry.resolve("document-review-conventions").orElseThrow();
    assertThat(template.name()).isNotBlank();
    assertThat(template.content()).contains("specific line references");
    // Verify it renders via a descriptor that references it
    var desc = registry.findById("drafthouse-structural-reviewer", "drafthouse").orElseThrow();
    var rendered = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
    assertThat(rendered.content()).contains("specific line references");
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn clean test -f examples/agent-scenarios/pom.xml -Dtest=DescriptorTemplateScenarioTest`
Expected: all pass.

- [ ] **Step 5: Commit**

```
feat(#105): template composition examples — ordering, sharing, YAML

Add three scenarios to DescriptorTemplateScenarioTest:
- Multi-template ordering preserved in rendered output
- Shared templates with different args produce correct substitutions
- YAML-registered templates resolve and render correctly

Refs casehubio/eidos#105
```

---

### Task 8: Engine follow-on issues + full build

**Files:** None (GitHub issues only)

- [ ] **Step 1: Create engine issue — goal-based routing**

```bash
gh issue create --repo casehubio/engine --title "feat: Goal-aware agent routing using eidos goal query primitives" --body "## Context

eidos#101 added goalName to AgentQuery and hasGoal() to AgentDescriptor.
Engine routing strategies can now query for agents with specific declared goals.

## Scope

- Use descriptor.goals() in routing strategy candidate evaluation
- Consider AgentQuery.byGoal() for goal-aware candidate discovery
- Goals are standing objectives (identity-level), not per-invocation context

## Depends on

casehubio/eidos#101 (landed)"
```

- [ ] **Step 2: Create engine issue — goal-based termination**

```bash
gh issue create --repo casehubio/engine --title "feat: Goal-based termination — compare agent goals against task outcomes" --body "## Context

eidos#101 added structured goals to AgentDescriptor. Engine already has
GoalExpression/GoalBasedCompletion for case-level completion evaluation.

## Scope

- Agent standing goals (eidos) inform behavior via system prompt
- Engine completion evaluation can compare declared goals against outcomes
- Consider GoalKind extension for agent-goal-based completion criteria

## Depends on

casehubio/eidos#101 (landed)"
```

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn clean test`
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 4: Commit (no code — just verify green)**

No commit needed — this task only creates issues and verifies the full build.
