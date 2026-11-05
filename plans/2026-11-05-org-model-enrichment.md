# OrganizationalUnit Model Enrichment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #229 — OrganizationalUnit model enrichment — normative roles, dissolution policy, topology governance
**Issue group:** #229, #230

**Goal:** Add normative obligations/permissions to RoleSpec, dissolution drain timeout to GroupPolicy, and topology family classification to org-api.

**Architecture:** Three additive model changes to org-api. RoleSpec gains `obligations` and `permissions` (List<String> norm identifiers). GroupPolicy gains `dissolutionDrainTimeoutMinutes` (long, default 30). New `TopologyFamily` enum and `TopologyFamilyClassifier` SPI for classifying topology strings into families. A default classifier implementation goes in org-runtime.

**Tech Stack:** Java 21, Quarkus 3.32.2, AssertJ, Jackson YAML

## Global Constraints

- Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn clean install` (not `./mvnw`)
- Records use compact constructor validation
- New fields on existing records must be null-safe (null → sensible default)
- All lists are defensively copied via `List.copyOf()`
- No backwards-compatibility shims — update all callers

---

## Batch 1: Core Model Changes (org-api)

### Task 1: RoleSpec — add obligations and permissions

**Files:**
- Modify: `org-api/src/main/java/io/casehub/eidos/org/api/RoleSpec.java`
- Modify: `org-api/src/test/java/io/casehub/eidos/org/api/RoleSpecTest.java`
- Modify: `org-api/src/test/java/io/casehub/eidos/org/api/OrganizationalUnitTest.java`
- Modify: `org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/runtime/EidosOrgAnnotationsRecorder.java:190`

**Interfaces:**
- Produces: `RoleSpec(roleName, roleVocabulary, minCardinality, maxCardinality, incompatibleWith, policies, obligations, permissions)` — 8-param record. `obligations()` and `permissions()` return `List<String>`.

- [ ] **Step 1: Write failing tests for obligations and permissions**

Add to `RoleSpecTest.java`:

```java
@Test void withObligations() {
    var spec = new RoleSpec("coordinator", null, 0, -1, Set.of(), List.of(),
            List.of("review-output", "approve-plan"), List.of());
    assertThat(spec.obligations()).containsExactly("review-output", "approve-plan");
    assertThat(spec.permissions()).isEmpty();
}

@Test void withPermissions() {
    var spec = new RoleSpec("coordinator", null, 0, -1, Set.of(), List.of(),
            List.of(), List.of("provision-agent", "change-topology"));
    assertThat(spec.permissions()).containsExactly("provision-agent", "change-topology");
}

@Test void obligationsDefaultToEmpty() {
    var spec = new RoleSpec("r", null, 0, -1, null, null, null, null);
    assertThat(spec.obligations()).isEmpty();
    assertThat(spec.permissions()).isEmpty();
}

@Test void obligationsDefensivelyCopied() {
    var mutable = new java.util.ArrayList<>(List.of("x"));
    var spec = new RoleSpec("r", null, 0, -1, null, null, mutable, null);
    mutable.add("y");
    assertThat(spec.obligations()).containsExactly("x");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-api -Dtest=RoleSpecTest`
Expected: compilation failure — RoleSpec constructor expects 6 params, got 8.

- [ ] **Step 3: Add obligations and permissions to RoleSpec record**

Update `RoleSpec.java` — add two new parameters after `policies`:

```java
public record RoleSpec(
    String roleName,
    String roleVocabulary,
    int minCardinality,
    int maxCardinality,
    Set<String> incompatibleWith,
    List<RolePolicy> policies,
    List<String> obligations,
    List<String> permissions
) {
    public RoleSpec {
        Objects.requireNonNull(roleName, "roleName");
        if (roleName.isBlank()) throw new OrgValidationException("roleName", "must not be blank");
        if (minCardinality < 0) throw new OrgValidationException("minCardinality", "must be >= 0 (was " + minCardinality + ")");
        if (maxCardinality != -1 && maxCardinality < minCardinality)
            throw new OrgValidationException("maxCardinality", "must be >= minCardinality or -1 (was " + maxCardinality + ", min " + minCardinality + ")");
        incompatibleWith = incompatibleWith != null ? Set.copyOf(incompatibleWith) : Set.of();
        policies = policies != null ? List.copyOf(policies) : List.of();
        obligations = obligations != null ? List.copyOf(obligations) : List.of();
        permissions = permissions != null ? List.copyOf(permissions) : List.of();
    }
}
```

- [ ] **Step 4: Update all existing RoleSpec callers to pass null, null for new params**

Every `new RoleSpec(name, vocab, min, max, incompat, policies)` becomes `new RoleSpec(name, vocab, min, max, incompat, policies, null, null)`.

Construction sites (19 total):
- `RoleSpecTest.java`: lines 16, 25, 30, 36, 42, 48, 53, 58, 64, 71 (11 sites)
- `OrganizationalUnitTest.java`: lines 84, 128, 129, 139, 149, 150, 162, 163 (8 sites — note line 163 has `Set.of("auditor")`)
- `EidosOrgAnnotationsRecorder.java`: line 190 (1 site)

For the recorder (line 190), change:
```java
result.add(new RoleSpec(rs.roleName, notEmpty(rs.roleVocabulary) ? rs.roleVocabulary : null,
    rs.minCardinality, rs.maxCardinality != 0 ? rs.maxCardinality : -1,
    incompatible, policies, null, null));
```
(The `null, null` will be replaced with actual annotation values in Task 6.)

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-api`
Expected: all RoleSpecTest and OrganizationalUnitTest tests pass.

- [ ] **Step 6: Commit**

```bash
git add org-api/src org-annotations/runtime/src
git commit -m "feat(org-api): add obligations and permissions to RoleSpec

Norm identifiers for agency-core's NormDecision<String> framework.
Obligations define what a role holder must do; permissions define what
they may do. Both are opaque strings resolved at enforcement time.

Refs #229"
```

### Task 2: GroupPolicy — add dissolutionDrainTimeoutMinutes

**Files:**
- Modify: `org-api/src/main/java/io/casehub/eidos/org/api/GroupPolicy.java`
- Modify: `org-api/src/test/java/io/casehub/eidos/org/api/OrganizationalUnitTest.java:119`
- Modify: `org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/runtime/EidosOrgAnnotationsRecorder.java:171`
- Create: `org-api/src/test/java/io/casehub/eidos/org/api/GroupPolicyTest.java`

**Interfaces:**
- Produces: `GroupPolicy(formationRules, normativePriority, normSpecificity, dissolutionDrainTimeoutMinutes)` — 4-param record. `dissolutionDrainTimeoutMinutes()` returns `long`. `DEFAULT_DISSOLUTION_DRAIN_TIMEOUT_MINUTES = 30`.

- [ ] **Step 1: Write failing tests for dissolutionDrainTimeoutMinutes**

Create `GroupPolicyTest.java`:

```java
package io.casehub.eidos.org.api;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GroupPolicyTest {

    @Test void defaultDrainTimeout() {
        var policy = new GroupPolicy(null, 0, null, 0);
        assertThat(policy.dissolutionDrainTimeoutMinutes())
            .isEqualTo(GroupPolicy.DEFAULT_DISSOLUTION_DRAIN_TIMEOUT_MINUTES);
    }

    @Test void customDrainTimeout() {
        var policy = new GroupPolicy(null, 0, null, 60);
        assertThat(policy.dissolutionDrainTimeoutMinutes()).isEqualTo(60);
    }

    @Test void rejectsNegativeDrainTimeout() {
        assertThatThrownBy(() -> new GroupPolicy(null, 0, null, -5))
            .isInstanceOf(OrgValidationException.class)
            .hasMessageContaining("dissolutionDrainTimeoutMinutes");
    }

    @Test void rejectsNegativeNormativePriority() {
        assertThatThrownBy(() -> new GroupPolicy(null, -1, null, 0))
            .isInstanceOf(OrgValidationException.class)
            .hasMessageContaining("normativePriority");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-api -Dtest=GroupPolicyTest`
Expected: compilation failure — GroupPolicy constructor expects 3 params, got 4.

- [ ] **Step 3: Add dissolutionDrainTimeoutMinutes to GroupPolicy record**

Update `GroupPolicy.java`:

```java
package io.casehub.eidos.org.api;

public record GroupPolicy(
    FormationRules formationRules,
    int normativePriority,
    NormSpecificity normSpecificity,
    long dissolutionDrainTimeoutMinutes
) {
    public static final long DEFAULT_DISSOLUTION_DRAIN_TIMEOUT_MINUTES = 30;

    public GroupPolicy {
        if (normativePriority < 0) throw new OrgValidationException("normativePriority", "must be >= 0 (was " + normativePriority + ")");
        if (dissolutionDrainTimeoutMinutes == 0) dissolutionDrainTimeoutMinutes = DEFAULT_DISSOLUTION_DRAIN_TIMEOUT_MINUTES;
        if (dissolutionDrainTimeoutMinutes < 0) throw new OrgValidationException("dissolutionDrainTimeoutMinutes", "must be > 0 (was " + dissolutionDrainTimeoutMinutes + ")");
    }
}
```

- [ ] **Step 4: Update existing GroupPolicy callers**

Construction sites (2 total):
- `OrganizationalUnitTest.java` line 119: `new GroupPolicy(rules, 0, null)` → `new GroupPolicy(rules, 0, null, 0)`
- `EidosOrgAnnotationsRecorder.java` line 171: `new GroupPolicy(fr, pc.normativePriority, ns)` → `new GroupPolicy(fr, pc.normativePriority, ns, 0)`
  (The `0` will default to 30 in the compact constructor. Will be replaced with actual annotation value in Task 6.)

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-api`
Expected: all GroupPolicyTest and OrganizationalUnitTest tests pass.

- [ ] **Step 6: Commit**

```bash
git add org-api/src org-annotations/runtime/src
git commit -m "feat(org-api): add dissolutionDrainTimeoutMinutes to GroupPolicy

Configurable drain timeout for PENDING_DRAIN during dissolution.
Defaults to 30 minutes. Post-timeout strategy is CBR-informed
at runtime, not modeled here.

Refs #229"
```

### Task 3: TopologyFamily enum + TopologyFamilyClassifier SPI

**Files:**
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/TopologyFamily.java`
- Create: `org-api/src/main/java/io/casehub/eidos/org/api/TopologyFamilyClassifier.java`
- Create: `org-api/src/test/java/io/casehub/eidos/org/api/TopologyFamilyTest.java`

**Interfaces:**
- Produces: `TopologyFamily { DISTRIBUTED, CENTRALISED, INDEPENDENT }` enum and `TopologyFamilyClassifier` functional interface: `Optional<TopologyFamily> classify(String topology)`.

- [ ] **Step 1: Write test for TopologyFamily enum values**

Create `TopologyFamilyTest.java`:

```java
package io.casehub.eidos.org.api;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TopologyFamilyTest {

    @Test void hasExpectedValues() {
        assertThat(TopologyFamily.values()).containsExactly(
            TopologyFamily.DISTRIBUTED,
            TopologyFamily.CENTRALISED,
            TopologyFamily.INDEPENDENT
        );
    }

    @Test void classifierContractReturnsOptional() {
        TopologyFamilyClassifier classifier = t -> java.util.Optional.of(TopologyFamily.DISTRIBUTED);
        assertThat(classifier.classify("swarm")).contains(TopologyFamily.DISTRIBUTED);
    }

    @Test void classifierCanReturnEmpty() {
        TopologyFamilyClassifier classifier = t -> java.util.Optional.empty();
        assertThat(classifier.classify("custom-topo")).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-api -Dtest=TopologyFamilyTest`
Expected: compilation failure — TopologyFamily and TopologyFamilyClassifier don't exist.

- [ ] **Step 3: Create TopologyFamily enum**

Create `TopologyFamily.java`:

```java
package io.casehub.eidos.org.api;

public enum TopologyFamily {
    DISTRIBUTED,
    CENTRALISED,
    INDEPENDENT
}
```

- [ ] **Step 4: Create TopologyFamilyClassifier SPI**

Create `TopologyFamilyClassifier.java`:

```java
package io.casehub.eidos.org.api;

import java.util.Optional;

@FunctionalInterface
public interface TopologyFamilyClassifier {
    Optional<TopologyFamily> classify(String topology);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-api -Dtest=TopologyFamilyTest`
Expected: PASS — all 3 tests green.

- [ ] **Step 6: Commit**

```bash
git add org-api/src
git commit -m "feat(org-api): add TopologyFamily enum and TopologyFamilyClassifier SPI

Classification for topology transition governance. DISTRIBUTED
(swarm, mesh), CENTRALISED (hive), INDEPENDENT (many). SPI allows
custom implementations for domain-specific topologies.

Refs #229"
```

---

## Batch 2: Runtime Implementation + Annotations

### Task 4: WellKnownTopologyFamilyClassifier @DefaultBean

**Files:**
- Create: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/topology/WellKnownTopologyFamilyClassifier.java`
- Create: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/topology/WellKnownTopologyFamilyClassifierTest.java`

**Interfaces:**
- Consumes: `TopologyFamily` enum, `TopologyFamilyClassifier` interface from Task 3
- Produces: `WellKnownTopologyFamilyClassifier` @DefaultBean — maps swarm/mesh→DISTRIBUTED, hive→CENTRALISED, many→INDEPENDENT, anything else→empty

- [ ] **Step 1: Write failing tests for the classifier**

Create `WellKnownTopologyFamilyClassifierTest.java`:

```java
package io.casehub.eidos.org.runtime.topology;

import io.casehub.eidos.org.api.TopologyFamily;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.assertj.core.api.Assertions.assertThat;

class WellKnownTopologyFamilyClassifierTest {

    private final WellKnownTopologyFamilyClassifier classifier = new WellKnownTopologyFamilyClassifier();

    @ParameterizedTest
    @ValueSource(strings = {"swarm", "mesh"})
    void distributedTopologies(String topology) {
        assertThat(classifier.classify(topology)).contains(TopologyFamily.DISTRIBUTED);
    }

    @Test void hiveIsCentralised() {
        assertThat(classifier.classify("hive")).contains(TopologyFamily.CENTRALISED);
    }

    @Test void manyIsIndependent() {
        assertThat(classifier.classify("many")).contains(TopologyFamily.INDEPENDENT);
    }

    @ParameterizedTest
    @ValueSource(strings = {"custom-topo", "unknown", ""})
    void unknownTopologiesReturnEmpty(String topology) {
        assertThat(classifier.classify(topology)).isEmpty();
    }

    @Test void nullTopologyReturnsEmpty() {
        assertThat(classifier.classify(null)).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WellKnownTopologyFamilyClassifierTest`
Expected: compilation failure — class doesn't exist.

- [ ] **Step 3: Create WellKnownTopologyFamilyClassifier**

Create `WellKnownTopologyFamilyClassifier.java`:

```java
package io.casehub.eidos.org.runtime.topology;

import io.casehub.eidos.org.api.TopologyFamily;
import io.casehub.eidos.org.api.TopologyFamilyClassifier;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Map;
import java.util.Optional;

@ApplicationScoped
@DefaultBean
public class WellKnownTopologyFamilyClassifier implements TopologyFamilyClassifier {

    private static final Map<String, TopologyFamily> FAMILIES = Map.of(
        "swarm", TopologyFamily.DISTRIBUTED,
        "mesh", TopologyFamily.DISTRIBUTED,
        "hive", TopologyFamily.CENTRALISED,
        "many", TopologyFamily.INDEPENDENT
    );

    @Override
    public Optional<TopologyFamily> classify(String topology) {
        if (topology == null) return Optional.empty();
        return Optional.ofNullable(FAMILIES.get(topology));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WellKnownTopologyFamilyClassifierTest`
Expected: PASS — all 5 tests green.

- [ ] **Step 5: Commit**

```bash
git add org-runtime/src
git commit -m "feat(org-runtime): add WellKnownTopologyFamilyClassifier @DefaultBean

Maps swarm/mesh → DISTRIBUTED, hive → CENTRALISED, many → INDEPENDENT.
Unknown topologies return empty → treated as cross-family at runtime.

Refs #229"
```

### Task 5: Annotation plumbing — RoleSpecDef, GroupPolicyDef, config, processor, recorder

**Files:**
- Modify: `org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/RoleSpecDef.java`
- Modify: `org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/GroupPolicyDef.java`
- Modify: `org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/runtime/AnnotatedOrgConfig.java`
- Modify: `org-annotations/deployment/src/main/java/io/casehub/eidos/org/annotations/deployment/EidosOrgAnnotationsProcessor.java`
- Modify: `org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/runtime/EidosOrgAnnotationsRecorder.java`
- Modify: `org-annotations/deployment/src/test/java/io/casehub/eidos/org/annotations/deployment/test/SocietyOrgUnit.java` (or appropriate test fixture)

**Interfaces:**
- Consumes: `RoleSpec` with obligations/permissions from Task 1, `GroupPolicy` with dissolutionDrainTimeoutMinutes from Task 2

- [ ] **Step 1: Add obligations() and permissions() to RoleSpecDef annotation**

Update `RoleSpecDef.java`:

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface RoleSpecDef {
    String roleName();
    String roleVocabulary() default "";
    int minCardinality() default 0;
    int maxCardinality() default 0;
    String[] incompatibleWith() default {};
    RolePolicyDef[] policies() default {};
    String[] obligations() default {};
    String[] permissions() default {};
}
```

- [ ] **Step 2: Add dissolutionDrainTimeoutMinutes() to GroupPolicyDef annotation**

Update `GroupPolicyDef.java`:

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface GroupPolicyDef {
    int normativePriority() default -1;
    String normSpecificity() default "";
    FormationRulesDef formationRules() default @FormationRulesDef;
    long dissolutionDrainTimeoutMinutes() default 0;
}
```

- [ ] **Step 3: Add fields to AnnotatedOrgConfig**

Add to `RoleSpecConfig`:
```java
public String[] obligations;
public String[] permissions;
```

Add to `GroupPolicyConfig`:
```java
public long dissolutionDrainTimeoutMinutes;
```

- [ ] **Step 4: Update EidosOrgAnnotationsProcessor.extractRoleSpecs() to extract new fields**

After the `incompatibleWith` extraction (around line 331), add:

```java
var ob = defs[i].value("obligations");
rsc.obligations = ob != null ? ob.asStringArray() : new String[0];
var pm = defs[i].value("permissions");
rsc.permissions = pm != null ? pm.asStringArray() : new String[0];
```

In `extractPolicy()`, after the formation rules extraction (around line 312), add:

```java
var ddt = policyAnn.value("dissolutionDrainTimeoutMinutes");
pc.dissolutionDrainTimeoutMinutes = ddt != null ? ddt.asLong() : 0;
```

- [ ] **Step 5: Update EidosOrgAnnotationsRecorder.buildRoleSpecs() to map new fields**

Change line 190 from:
```java
result.add(new RoleSpec(rs.roleName, notEmpty(rs.roleVocabulary) ? rs.roleVocabulary : null,
    rs.minCardinality, rs.maxCardinality != 0 ? rs.maxCardinality : -1,
    incompatible, policies, null, null));
```
to:
```java
List<String> obligations = rs.obligations != null && rs.obligations.length > 0 ? List.of(rs.obligations) : null;
List<String> permissions = rs.permissions != null && rs.permissions.length > 0 ? List.of(rs.permissions) : null;
result.add(new RoleSpec(rs.roleName, notEmpty(rs.roleVocabulary) ? rs.roleVocabulary : null,
    rs.minCardinality, rs.maxCardinality != 0 ? rs.maxCardinality : -1,
    incompatible, policies, obligations, permissions));
```

Update `buildPolicy()` line 171 from:
```java
return new GroupPolicy(fr, pc.normativePriority, ns, 0);
```
to:
```java
return new GroupPolicy(fr, pc.normativePriority, ns, pc.dissolutionDrainTimeoutMinutes);
```

- [ ] **Step 6: Add a test fixture with obligations and permissions**

In an existing test class (e.g. `SocietyOrgUnit.java`), add obligations and permissions to a role spec:

```java
@RoleSpecDef(
    roleName = "coordinator",
    obligations = {"review-output", "approve-plan"},
    permissions = {"provision-agent", "change-topology"}
)
```

- [ ] **Step 7: Run annotation tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-annotations/deployment`
Expected: PASS — all annotation processor tests green.

- [ ] **Step 8: Commit**

```bash
git add org-annotations/
git commit -m "feat(org-annotations): wire obligations, permissions, and drainTimeout

RoleSpecDef gains obligations() and permissions() attributes.
GroupPolicyDef gains dissolutionDrainTimeoutMinutes(). Processor
extracts values from Jandex; recorder maps to model records.

Refs #229"
```

### Task 6: YAML fixtures + documentation

**Files:**
- Modify: `org-runtime/src/test/resources/society-org.yaml`
- Modify: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/yaml/SocietyYamlDeserializerTest.java`
- Modify: `docs/guides/consumer-guide.md`
- Modify: `docs/guides/contributor-guide.md`

**Interfaces:**
- Consumes: Updated RoleSpec and GroupPolicy records from Tasks 1-2

- [ ] **Step 1: Add obligations and permissions to society-org.yaml**

In `society-org.yaml`, update the `lead` role spec (around line 29):

```yaml
      roleSpecs:
        - roleName: lead
          roleVocabulary: urn:casehub:vocab:roles
          minCardinality: 1
          maxCardinality: 1
          incompatibleWith: []
          obligations:
            - review-output
            - approve-plan
            - report-status
          permissions:
            - provision-agent
            - change-topology
            - create-subgroup
          policies:
            - capabilityName: code-review
              kind: OBLIGATION
              expectations:
                turnaround: "24h"
            - capabilityName: merge-authority
              kind: PERMISSION
        - roleName: contributor
          minCardinality: 2
          maxCardinality: -1
          obligations:
            - process-assigned-work
          permissions:
            - signal-consensus
            - request-collaboration
          policies:
            - capabilityName: code-review
              kind: ELIGIBILITY
```

Add `dissolutionDrainTimeoutMinutes: 15` to the dev-guild policy (around line 12):

```yaml
      policy:
        formationRules:
          allowedPaths: [BOTTOM_UP, EMERGENCE]
          minGroupSize: 3
          maxGroupSize: 10
          allowSubGroups: true
          proposalQuorum: 0.6
        normativePriority: 2
        normSpecificity: DOMAIN
        dissolutionDrainTimeoutMinutes: 15
```

- [ ] **Step 2: Add YAML round-trip assertions to SocietyYamlDeserializerTest**

Add assertions to the existing test method that reads `society-org.yaml`:

```java
// Verify obligations and permissions on lead role
var leadRole = unit.roleSpecs().stream()
    .filter(r -> r.roleName().equals("lead"))
    .findFirst().orElseThrow();
assertThat(leadRole.obligations()).containsExactly("review-output", "approve-plan", "report-status");
assertThat(leadRole.permissions()).containsExactly("provision-agent", "change-topology", "create-subgroup");

// Verify contributor role obligations
var contribRole = unit.roleSpecs().stream()
    .filter(r -> r.roleName().equals("contributor"))
    .findFirst().orElseThrow();
assertThat(contribRole.obligations()).containsExactly("process-assigned-work");

// Verify dissolution drain timeout
assertThat(unit.policy().dissolutionDrainTimeoutMinutes()).isEqualTo(15);
```

- [ ] **Step 3: Run YAML deserializer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=SocietyYamlDeserializerTest`
Expected: PASS — new fields round-trip correctly via Jackson.

- [ ] **Step 4: Update consumer-guide.md**

Find the society types description (line ~296) and add obligations/permissions and dissolution timeout:

Update the `GroupPolicy` description to mention `dissolutionDrainTimeoutMinutes`.
Update the `RoleSpec` description to mention `obligations` and `permissions`.
Add `TopologyFamily` and `TopologyFamilyClassifier` to the SPI listing.

- [ ] **Step 5: Update contributor-guide.md**

Update the org-api module description (line ~22) to include `TopologyFamily`, `TopologyFamilyClassifier` in the type listing.

- [ ] **Step 6: Run full build to verify everything is green**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and pass.

- [ ] **Step 7: Commit**

```bash
git add org-runtime/src/test/resources docs/guides
git commit -m "feat: YAML fixtures with normative roles + update guides

society-org.yaml demonstrates obligations, permissions, and
dissolutionDrainTimeoutMinutes. Consumer and contributor guides
updated with new types and fields.

Refs #229"
```

---

## References

- [2026-11-05-org-model-enrichment-design.md] — design spec this plan implements
- [org-api/src/main/java/io/casehub/eidos/org/api/RoleSpec.java] — current RoleSpec record
- [org-api/src/main/java/io/casehub/eidos/org/api/GroupPolicy.java] — current GroupPolicy record
- [org-api/src/main/java/io/casehub/eidos/org/api/OrganizationalUnit.java] — topology fields
- [org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/RoleSpecDef.java] — annotation
- [org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/GroupPolicyDef.java] — annotation
- [org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/runtime/AnnotatedOrgConfig.java] — config transfer
- [org-annotations/deployment/src/main/java/io/casehub/eidos/org/annotations/deployment/EidosOrgAnnotationsProcessor.java] — Jandex extraction
- [org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/runtime/EidosOrgAnnotationsRecorder.java] — build→runtime mapping
- [org-runtime/src/test/resources/society-org.yaml] — YAML fixture
- [GitHub #229] — OrganizationalUnit model enrichment
