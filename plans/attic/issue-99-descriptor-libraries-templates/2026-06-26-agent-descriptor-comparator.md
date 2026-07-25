# Agent Descriptor Content Comparator — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add field-by-field content comparison for `AgentDescriptor` to support drift detection in casehub-ops' desiredstate reconciliation.

**Architecture:** `AgentDescriptorComparator` in `casehub-eidos-api` (pure Java utility, zero deps) defines content equality for `AgentDescriptor` — all fields except identity keys `agentId` and `tenancyId`. Three structural coverage constants + reflection tests guard against silent field additions. In ops, `AgentNodeSpec.toDescriptor()` eliminates duplicated field mapping, and the enriched `AgentDriftChecker` calls the comparator for full-field drift detection with DEBUG logging.

**Tech Stack:** Java 21, JUnit 5, AssertJ (eidos tests), JUnit 5 with assertEquals (ops tests)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test a single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
- eidos repo: `/Users/mdproctor/claude/casehub/eidos`
- ops repo: `/Users/mdproctor/claude/casehub/ops`
- All commits reference `eidos#60`
- eidos-api is zero-dep — no framework imports, no CDI
- Tests in eidos use AssertJ (`assertThat`); tests in ops use JUnit `assertEquals`

---

### Task 1: AgentDescriptorComparator (eidos-api)

**Repo:** `casehub-eidos` (`/Users/mdproctor/claude/casehub/eidos`)

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java`
- Create: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java`

**Interfaces:**
- Consumes: `AgentDescriptor`, `AgentCapability`, `AgentDisposition`, `DispositionAxis` (all existing in `io.casehub.eidos.api`)
- Produces: `AgentDescriptorComparator.compare(AgentDescriptor desired, AgentDescriptor actual) → ComparisonResult`; `ComparisonResult.matches() → boolean`; `ComparisonResult.drifts() → List<FieldDrift>`; `FieldDrift(String field, String desiredValue, String actualValue)`; constants `COMPARED_FIELD_COUNT`, `COMPARED_CAPABILITY_FIELD_COUNT`, `COMPARED_DISPOSITION_FIELD_COUNT`

- [ ] **Step 1: Write structural sync tests and identical-descriptors test**

Create `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.*;

class AgentDescriptorComparatorTest {

    private static final AgentDisposition DISPOSITION = AgentDisposition.builder()
            .socialOrient("collaborative").ruleFollowing("principled")
            .riskAppetite("measured").autonomy("semi-autonomous")
            .conflictMode("compromising").delegation(false)
            .build();

    private static AgentDescriptor base() {
        return AgentDescriptor.builder()
                .agentId("agent-1").name("Alice").version("1.0")
                .provider("anthropic").modelFamily("claude").modelVersion("4.6")
                .weightsFingerprint("fp-abc")
                .domainVocabulary("urn:svo").slotVocabulary("urn:slot")
                .dispositionVocabulary("urn:disp")
                .axisVocabularies(Map.of(DispositionAxis.SOCIAL_ORIENTATION, "urn:svo"))
                .slot("reviewer")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("code-review")
                                .qualityHint(0.85)
                                .latencyHintP50Ms(2000L)
                                .costHint("medium")
                                .inputTypes(List.of("text/plain"))
                                .outputTypes(List.of("text/markdown"))
                                .tags(List.of("review"))
                                .epistemicDomains(Map.of("java", 0.95))
                                .excludedDomains(Set.of("cobol"))
                                .build()))
                .disposition(DISPOSITION)
                .jurisdiction("US")
                .dataHandlingPolicy("standard")
                .tenancyId("tenant-1")
                .briefing("Expert code reviewer")
                .build();
    }

    private static AgentDescriptor withField(java.util.function.UnaryOperator<AgentDescriptor.Builder> mutator) {
        var b = AgentDescriptor.builder()
                .agentId("agent-1").name("Alice").version("1.0")
                .provider("anthropic").modelFamily("claude").modelVersion("4.6")
                .weightsFingerprint("fp-abc")
                .domainVocabulary("urn:svo").slotVocabulary("urn:slot")
                .dispositionVocabulary("urn:disp")
                .axisVocabularies(Map.of(DispositionAxis.SOCIAL_ORIENTATION, "urn:svo"))
                .slot("reviewer")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("code-review")
                                .qualityHint(0.85)
                                .latencyHintP50Ms(2000L)
                                .costHint("medium")
                                .inputTypes(List.of("text/plain"))
                                .outputTypes(List.of("text/markdown"))
                                .tags(List.of("review"))
                                .epistemicDomains(Map.of("java", 0.95))
                                .excludedDomains(Set.of("cobol"))
                                .build()))
                .disposition(DISPOSITION)
                .jurisdiction("US")
                .dataHandlingPolicy("standard")
                .tenancyId("tenant-1")
                .briefing("Expert code reviewer");
        return mutator.apply(b).build();
    }

    // --- Structural sync tests ---

    @Test
    void comparatorCoversAllDescriptorComponents() {
        int total = AgentDescriptor.class.getRecordComponents().length;
        int skipped = 2; // agentId, tenancyId
        assertThat(AgentDescriptorComparator.COMPARED_FIELD_COUNT).isEqualTo(total - skipped);
    }

    @Test
    void comparatorCoversAllCapabilityComponents() {
        int total = AgentCapability.class.getRecordComponents().length;
        int matchKey = 1; // name
        assertThat(AgentDescriptorComparator.COMPARED_CAPABILITY_FIELD_COUNT).isEqualTo(total - matchKey);
    }

    @Test
    void comparatorCoversAllDispositionComponents() {
        assertThat(AgentDescriptorComparator.COMPARED_DISPOSITION_FIELD_COUNT)
                .isEqualTo(AgentDisposition.class.getRecordComponents().length);
    }

    // --- Identical descriptors ---

    @Test
    void identicalDescriptors_matches() {
        var result = AgentDescriptorComparator.compare(base(), base());
        assertThat(result.matches()).isTrue();
        assertThat(result.drifts()).isEmpty();
    }

    @Test
    void identicalDescriptors_differentIdentityKeys_matches() {
        var desired = base();
        var actual = withField(b -> b.agentId("agent-2").tenancyId("tenant-2"));
        var result = AgentDescriptorComparator.compare(desired, actual);
        assertThat(result.matches()).isTrue();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorComparatorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: Compilation failure — `AgentDescriptorComparator` does not exist yet.

- [ ] **Step 3: Create AgentDescriptorComparator skeleton**

Create `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java`:

```java
package io.casehub.eidos.api;

import java.util.*;
import java.util.stream.Collectors;

public final class AgentDescriptorComparator {

    static final int COMPARED_FIELD_COUNT = 16;
    static final int COMPARED_CAPABILITY_FIELD_COUNT = 8;
    static final int COMPARED_DISPOSITION_FIELD_COUNT = 6;

    public record ComparisonResult(List<FieldDrift> drifts) {
        public boolean matches() { return drifts.isEmpty(); }
    }

    public record FieldDrift(String field, String desiredValue, String actualValue) {}

    private AgentDescriptorComparator() {}

    public static ComparisonResult compare(AgentDescriptor desired, AgentDescriptor actual) {
        List<FieldDrift> drifts = new ArrayList<>();

        compareSimpleFields(drifts, desired, actual);
        compareAxisVocabularies(drifts, desired.axisVocabularies(), actual.axisVocabularies());
        compareDisposition(drifts, desired.disposition(), actual.disposition());
        compareCapabilities(drifts, desired.capabilities(), actual.capabilities());

        return new ComparisonResult(List.copyOf(drifts));
    }

    private static void compareSimpleFields(List<FieldDrift> drifts, AgentDescriptor desired, AgentDescriptor actual) {
        compareField(drifts, "name", desired.name(), actual.name());
        compareField(drifts, "slot", desired.slot(), actual.slot());
        compareField(drifts, "version", desired.version(), actual.version());
        compareField(drifts, "provider", desired.provider(), actual.provider());
        compareField(drifts, "modelFamily", desired.modelFamily(), actual.modelFamily());
        compareField(drifts, "modelVersion", desired.modelVersion(), actual.modelVersion());
        compareField(drifts, "weightsFingerprint", desired.weightsFingerprint(), actual.weightsFingerprint());
        compareField(drifts, "domainVocabulary", desired.domainVocabulary(), actual.domainVocabulary());
        compareField(drifts, "slotVocabulary", desired.slotVocabulary(), actual.slotVocabulary());
        compareField(drifts, "dispositionVocabulary", desired.dispositionVocabulary(), actual.dispositionVocabulary());
        compareField(drifts, "jurisdiction", desired.jurisdiction(), actual.jurisdiction());
        compareField(drifts, "dataHandlingPolicy", desired.dataHandlingPolicy(), actual.dataHandlingPolicy());
        compareField(drifts, "briefing", desired.briefing(), actual.briefing());
    }

    private static void compareAxisVocabularies(List<FieldDrift> drifts,
                                                 Map<DispositionAxis, String> desired,
                                                 Map<DispositionAxis, String> actual) {
        Map<DispositionAxis, String> d = desired != null ? desired : Map.of();
        Map<DispositionAxis, String> a = actual != null ? actual : Map.of();
        Set<DispositionAxis> allKeys = new TreeSet<>(Comparator.comparing(Enum::name));
        allKeys.addAll(d.keySet());
        allKeys.addAll(a.keySet());
        for (DispositionAxis key : allKeys) {
            compareField(drifts, "axisVocabularies[" + key.name() + "]", d.get(key), a.get(key));
        }
    }

    private static void compareDisposition(List<FieldDrift> drifts,
                                            AgentDisposition desired,
                                            AgentDisposition actual) {
        if (desired == null && actual == null) return;
        if (desired == null || actual == null) {
            drifts.add(new FieldDrift("disposition", String.valueOf(desired), String.valueOf(actual)));
            return;
        }
        compareField(drifts, "disposition.socialOrient", desired.socialOrient(), actual.socialOrient());
        compareField(drifts, "disposition.ruleFollowing", desired.ruleFollowing(), actual.ruleFollowing());
        compareField(drifts, "disposition.riskAppetite", desired.riskAppetite(), actual.riskAppetite());
        compareField(drifts, "disposition.autonomy", desired.autonomy(), actual.autonomy());
        compareField(drifts, "disposition.conflictMode", desired.conflictMode(), actual.conflictMode());
        if (desired.delegation() != actual.delegation()) {
            drifts.add(new FieldDrift("disposition.delegation",
                    String.valueOf(desired.delegation()), String.valueOf(actual.delegation())));
        }
    }

    private static void compareCapabilities(List<FieldDrift> drifts,
                                             List<AgentCapability> desired,
                                             List<AgentCapability> actual) {
        Map<String, AgentCapability> desiredByName = desired.stream()
                .collect(Collectors.toMap(AgentCapability::name, c -> c));
        Map<String, AgentCapability> actualByName = actual.stream()
                .collect(Collectors.toMap(AgentCapability::name, c -> c));

        for (String name : new TreeSet<>(desiredByName.keySet())) {
            if (!actualByName.containsKey(name)) {
                drifts.add(new FieldDrift("capabilities[" + name + "]", "(present)", "(absent)"));
            }
        }
        for (String name : new TreeSet<>(actualByName.keySet())) {
            if (!desiredByName.containsKey(name)) {
                drifts.add(new FieldDrift("capabilities[" + name + "]", "(absent)", "(present)"));
            }
        }
        for (var entry : new TreeMap<>(desiredByName).entrySet()) {
            AgentCapability actualCap = actualByName.get(entry.getKey());
            if (actualCap != null) {
                compareCapability(drifts, entry.getKey(), entry.getValue(), actualCap);
            }
        }
    }

    private static void compareCapability(List<FieldDrift> drifts, String capName,
                                           AgentCapability desired, AgentCapability actual) {
        String prefix = "capabilities[" + capName + "].";
        compareField(drifts, prefix + "qualityHint", desired.qualityHint(), actual.qualityHint());
        compareField(drifts, prefix + "latencyHintP50Ms", desired.latencyHintP50Ms(), actual.latencyHintP50Ms());
        compareField(drifts, prefix + "costHint", desired.costHint(), actual.costHint());
        compareField(drifts, prefix + "inputTypes", desired.inputTypes(), actual.inputTypes());
        compareField(drifts, prefix + "outputTypes", desired.outputTypes(), actual.outputTypes());
        compareField(drifts, prefix + "tags", desired.tags(), actual.tags());
        compareField(drifts, prefix + "epistemicDomains", desired.epistemicDomains(), actual.epistemicDomains());
        compareField(drifts, prefix + "excludedDomains", desired.excludedDomains(), actual.excludedDomains());
    }

    private static void compareField(List<FieldDrift> drifts, String field, Object desired, Object actual) {
        if (!Objects.equals(desired, actual)) {
            drifts.add(new FieldDrift(field, String.valueOf(desired), String.valueOf(actual)));
        }
    }
}
```

- [ ] **Step 4: Run structural sync + identity tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorComparatorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: All 5 tests PASS.

- [ ] **Step 5: Add simple field drift tests**

Append to `AgentDescriptorComparatorTest.java`:

```java
    // --- Simple field drift ---

    @Test
    void nameDrifted() {
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.name("Bob")));
        assertThat(result.matches()).isFalse();
        assertThat(result.drifts()).hasSize(1);
        assertThat(result.drifts().get(0).field()).isEqualTo("name");
        assertThat(result.drifts().get(0).desiredValue()).isEqualTo("Alice");
        assertThat(result.drifts().get(0).actualValue()).isEqualTo("Bob");
    }

    @Test
    void slotDrifted() {
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.slot("planner")));
        assertThat(result.matches()).isFalse();
        assertThat(result.drifts()).extracting("field").containsExactly("slot");
    }

    @Test
    void versionDrifted() {
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.version("2.0")));
        assertThat(result.drifts()).extracting("field").containsExactly("version");
    }

    @Test
    void providerDrifted() {
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.provider("openai")));
        assertThat(result.drifts()).extracting("field").containsExactly("provider");
    }

    @Test
    void vocabularyFieldsDrifted() {
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.domainVocabulary("urn:other")));
        assertThat(result.drifts()).extracting("field").containsExactly("domainVocabulary");
    }

    @Test
    void briefingDrifted() {
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.briefing("Changed briefing")));
        assertThat(result.drifts()).extracting("field").containsExactly("briefing");
    }

    @Test
    void nullToNonNull_drifted() {
        var desired = withField(b -> b.weightsFingerprint(null));
        var actual = base();
        var result = AgentDescriptorComparator.compare(desired, actual);
        assertThat(result.drifts()).extracting("field").containsExactly("weightsFingerprint");
    }

    @Test
    void nonNullToNull_drifted() {
        var desired = base();
        var actual = withField(b -> b.weightsFingerprint(null));
        var result = AgentDescriptorComparator.compare(desired, actual);
        assertThat(result.drifts()).extracting("field").containsExactly("weightsFingerprint");
    }

    @Test
    void axisVocabularies_entryDrifted() {
        var actual = withField(b -> b.axisVocabularies(
                Map.of(DispositionAxis.SOCIAL_ORIENTATION, "urn:other")));
        var result = AgentDescriptorComparator.compare(base(), actual);
        assertThat(result.drifts()).extracting("field")
                .containsExactly("axisVocabularies[SOCIAL_ORIENTATION]");
    }

    @Test
    void axisVocabularies_entryAdded() {
        var actual = withField(b -> b.axisVocabularies(Map.of(
                DispositionAxis.SOCIAL_ORIENTATION, "urn:svo",
                DispositionAxis.AUTONOMY, "urn:new")));
        var result = AgentDescriptorComparator.compare(base(), actual);
        assertThat(result.drifts()).extracting("field")
                .containsExactly("axisVocabularies[AUTONOMY]");
    }
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorComparatorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: All tests PASS (implementation already handles simple fields).

- [ ] **Step 7: Add disposition drift tests**

Append to `AgentDescriptorComparatorTest.java`:

```java
    // --- Disposition drift ---

    @Test
    void disposition_axisDrifted() {
        var drifted = AgentDisposition.builder()
                .socialOrient("independent").ruleFollowing("principled")
                .riskAppetite("measured").autonomy("semi-autonomous")
                .conflictMode("compromising").delegation(false)
                .build();
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.disposition(drifted)));
        assertThat(result.drifts()).extracting("field").containsExactly("disposition.socialOrient");
        assertThat(result.drifts().get(0).desiredValue()).isEqualTo("collaborative");
        assertThat(result.drifts().get(0).actualValue()).isEqualTo("independent");
    }

    @Test
    void disposition_delegationDrifted() {
        var drifted = AgentDisposition.builder()
                .socialOrient("collaborative").ruleFollowing("principled")
                .riskAppetite("measured").autonomy("semi-autonomous")
                .conflictMode("compromising").delegation(true)
                .build();
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.disposition(drifted)));
        assertThat(result.drifts()).extracting("field").containsExactly("disposition.delegation");
    }

    @Test
    void disposition_nullDesired_nullActual_matches() {
        var desired = withField(b -> b.disposition(null));
        var actual = withField(b -> b.disposition(null));
        var result = AgentDescriptorComparator.compare(desired, actual);
        assertThat(result.drifts()).filteredOn(d -> d.field().startsWith("disposition")).isEmpty();
    }

    @Test
    void disposition_nullDesired_nonNullActual_drifted() {
        var desired = withField(b -> b.disposition(null));
        var result = AgentDescriptorComparator.compare(desired, base());
        assertThat(result.drifts()).extracting("field").contains("disposition");
    }

    @Test
    void disposition_multipleAxesDrifted() {
        var drifted = AgentDisposition.builder()
                .socialOrient("independent").ruleFollowing("flexible")
                .riskAppetite("measured").autonomy("semi-autonomous")
                .conflictMode("compromising").delegation(false)
                .build();
        var result = AgentDescriptorComparator.compare(base(), withField(b -> b.disposition(drifted)));
        assertThat(result.drifts()).extracting("field")
                .containsExactlyInAnyOrder("disposition.socialOrient", "disposition.ruleFollowing");
    }
```

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorComparatorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: All tests PASS.

- [ ] **Step 9: Add capability drift tests**

Append to `AgentDescriptorComparatorTest.java`:

```java
    // --- Capability drift ---

    @Test
    void capability_added() {
        var extra = AgentCapability.builder().name("planning").build();
        var actual = withField(b -> b.capabilities(List.of(
                AgentCapability.builder()
                        .name("code-review").qualityHint(0.85).latencyHintP50Ms(2000L)
                        .costHint("medium").inputTypes(List.of("text/plain"))
                        .outputTypes(List.of("text/markdown")).tags(List.of("review"))
                        .epistemicDomains(Map.of("java", 0.95)).excludedDomains(Set.of("cobol"))
                        .build(),
                extra)));
        var result = AgentDescriptorComparator.compare(base(), actual);
        assertThat(result.drifts()).extracting("field").containsExactly("capabilities[planning]");
        assertThat(result.drifts().get(0).desiredValue()).isEqualTo("(absent)");
        assertThat(result.drifts().get(0).actualValue()).isEqualTo("(present)");
    }

    @Test
    void capability_removed() {
        var actual = withField(b -> b.capabilities(List.of()));
        var result = AgentDescriptorComparator.compare(base(), actual);
        assertThat(result.drifts()).extracting("field").containsExactly("capabilities[code-review]");
        assertThat(result.drifts().get(0).desiredValue()).isEqualTo("(present)");
        assertThat(result.drifts().get(0).actualValue()).isEqualTo("(absent)");
    }

    @Test
    void capability_subFieldDrifted_qualityHint() {
        var driftedCap = AgentCapability.builder()
                .name("code-review").qualityHint(0.50).latencyHintP50Ms(2000L)
                .costHint("medium").inputTypes(List.of("text/plain"))
                .outputTypes(List.of("text/markdown")).tags(List.of("review"))
                .epistemicDomains(Map.of("java", 0.95)).excludedDomains(Set.of("cobol"))
                .build();
        var actual = withField(b -> b.capabilities(List.of(driftedCap)));
        var result = AgentDescriptorComparator.compare(base(), actual);
        assertThat(result.drifts()).extracting("field")
                .containsExactly("capabilities[code-review].qualityHint");
        assertThat(result.drifts().get(0).desiredValue()).isEqualTo("0.85");
        assertThat(result.drifts().get(0).actualValue()).isEqualTo("0.5");
    }

    @Test
    void capability_epistemicDomainsDrifted() {
        var driftedCap = AgentCapability.builder()
                .name("code-review").qualityHint(0.85).latencyHintP50Ms(2000L)
                .costHint("medium").inputTypes(List.of("text/plain"))
                .outputTypes(List.of("text/markdown")).tags(List.of("review"))
                .epistemicDomains(Map.of("java", 0.90)).excludedDomains(Set.of("cobol"))
                .build();
        var actual = withField(b -> b.capabilities(List.of(driftedCap)));
        var result = AgentDescriptorComparator.compare(base(), actual);
        assertThat(result.drifts()).extracting("field")
                .containsExactly("capabilities[code-review].epistemicDomains");
    }

    @Test
    void capability_excludedDomainsDrifted() {
        var driftedCap = AgentCapability.builder()
                .name("code-review").qualityHint(0.85).latencyHintP50Ms(2000L)
                .costHint("medium").inputTypes(List.of("text/plain"))
                .outputTypes(List.of("text/markdown")).tags(List.of("review"))
                .epistemicDomains(Map.of("java", 0.95)).excludedDomains(Set.of("fortran"))
                .build();
        var actual = withField(b -> b.capabilities(List.of(driftedCap)));
        var result = AgentDescriptorComparator.compare(base(), actual);
        assertThat(result.drifts()).extracting("field")
                .containsExactly("capabilities[code-review].excludedDomains");
    }

    @Test
    void capability_orderIndependent() {
        var capA = AgentCapability.builder().name("alpha").build();
        var capB = AgentCapability.builder().name("beta").build();
        var desired = withField(b -> b.capabilities(List.of(capA, capB)));
        var actual = withField(b -> b.capabilities(List.of(capB, capA)));
        var result = AgentDescriptorComparator.compare(desired, actual);
        assertThat(result.matches()).isTrue();
    }

    // --- Multiple simultaneous drifts ---

    @Test
    void multipleDrifts_allReported() {
        var driftedDisp = AgentDisposition.builder()
                .socialOrient("independent").ruleFollowing("principled")
                .riskAppetite("measured").autonomy("semi-autonomous")
                .conflictMode("compromising").delegation(false)
                .build();
        var actual = withField(b -> b.name("Bob").slot("planner").disposition(driftedDisp));
        var result = AgentDescriptorComparator.compare(base(), actual);
        assertThat(result.drifts()).extracting("field")
                .containsExactlyInAnyOrder("name", "slot", "disposition.socialOrient");
    }

    // --- Field path format ---

    @Test
    void fieldPaths_followConvention() {
        var driftedDisp = AgentDisposition.builder()
                .socialOrient("independent").ruleFollowing("principled")
                .riskAppetite("measured").autonomy("semi-autonomous")
                .conflictMode("compromising").delegation(true)
                .build();
        var extra = AgentCapability.builder().name("planning").build();
        var driftedCap = AgentCapability.builder()
                .name("code-review").qualityHint(0.50).latencyHintP50Ms(2000L)
                .costHint("medium").inputTypes(List.of("text/plain"))
                .outputTypes(List.of("text/markdown")).tags(List.of("review"))
                .epistemicDomains(Map.of("java", 0.95)).excludedDomains(Set.of("cobol"))
                .build();
        var actual = withField(b -> b
                .name("Bob")
                .disposition(driftedDisp)
                .axisVocabularies(Map.of(DispositionAxis.AUTONOMY, "urn:new"))
                .capabilities(List.of(driftedCap, extra)));

        var result = AgentDescriptorComparator.compare(base(), actual);
        var fields = result.drifts().stream().map(AgentDescriptorComparator.FieldDrift::field).toList();

        assertThat(fields).contains(
                "name",
                "disposition.socialOrient",
                "disposition.delegation",
                "axisVocabularies[AUTONOMY]",
                "axisVocabularies[SOCIAL_ORIENTATION]",
                "capabilities[code-review].qualityHint",
                "capabilities[planning]");
    }
```

- [ ] **Step 10: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorComparatorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: All tests PASS.

- [ ] **Step 11: Run full api module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: All existing + new tests PASS. No regressions.

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#60): AgentDescriptorComparator — content equality for drift detection

Adds pure-Java comparator defining content equality for AgentDescriptor
(all fields except identity keys agentId and tenancyId). Three structural
coverage constants guard against silent field additions at every compared
type boundary (AgentDescriptor, AgentCapability, AgentDisposition).

Refs #60"
```

---

### Task 2: ops — AgentNodeSpec.toDescriptor() + AgentProvisionHandler refactor

**Repo:** `casehub-ops` (`/Users/mdproctor/claude/casehub/ops`)

**Prerequisites:** Task 1 must be installed to the local Maven repository first:
`JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

**Files:**
- Modify: `api/src/main/java/io/casehub/ops/api/deployment/AgentNodeSpec.java` (add `toDescriptor` method)
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/handler/AgentProvisionHandler.java` (use `toDescriptor`)
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/handler/AgentProvisionHandlerTest.java` (verify refactored handler)

**Interfaces:**
- Consumes: `AgentDescriptor` constructor (from `casehub-eidos-api`), `AgentNodeSpec` fields
- Produces: `AgentNodeSpec.toDescriptor(String tenancyId) → AgentDescriptor`

- [ ] **Step 1: Add toDescriptor method to AgentNodeSpec**

In `api/src/main/java/io/casehub/ops/api/deployment/AgentNodeSpec.java`, add this method inside the record body (after the `nodeType()` method):

```java
    public AgentDescriptor toDescriptor(String tenancyId) {
        return new AgentDescriptor(
                agentId, name, version, provider, modelFamily, modelVersion,
                weightsFingerprint, domainVocabulary, slotVocabulary,
                dispositionVocabulary, axisVocabularies, slot, capabilities,
                disposition, jurisdiction, dataHandlingPolicy, tenancyId, briefing);
    }
```

Add the import:

```java
import io.casehub.eidos.api.AgentDescriptor;
```

Note: `AgentDescriptor` is already imported transitively via `AgentCapability` and `AgentDisposition`, but adding the explicit import is correct practice. Check whether it's already imported — if so, skip.

- [ ] **Step 2: Refactor AgentProvisionHandler.provision()**

Replace the 18-field constructor call in `deployment/src/main/java/io/casehub/ops/deployment/handler/AgentProvisionHandler.java`:

Replace:
```java
    public ProvisionResult provision(AgentNodeSpec spec, ProvisionContext context) {
        AgentDescriptor descriptor = new AgentDescriptor(
                spec.agentId(),
                spec.name(),
                spec.version(),
                spec.provider(),
                spec.modelFamily(),
                spec.modelVersion(),
                spec.weightsFingerprint(),
                spec.domainVocabulary(),
                spec.slotVocabulary(),
                spec.dispositionVocabulary(),
                spec.axisVocabularies(),
                spec.slot(),
                spec.capabilities(),
                spec.disposition(),
                spec.jurisdiction(),
                spec.dataHandlingPolicy(),
                context.tenancyId(),
                spec.briefing()
        );
        agentRegistry.register(descriptor);
```

With:
```java
    public ProvisionResult provision(AgentNodeSpec spec, ProvisionContext context) {
        agentRegistry.register(spec.toDescriptor(context.tenancyId()));
```

Remove the now-unused `AgentDescriptor` import from `AgentProvisionHandler.java` if no other reference remains.

- [ ] **Step 3: Run existing provision handler tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl deployment -Dtest=AgentProvisionHandlerTest -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: All existing tests PASS — `toDescriptor` produces the same `AgentDescriptor` as the inline constructor call.

- [ ] **Step 4: Run full ops test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: All tests PASS. No regressions.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ops add api/src/main/java/io/casehub/ops/api/deployment/AgentNodeSpec.java deployment/src/main/java/io/casehub/ops/deployment/handler/AgentProvisionHandler.java
git -C /Users/mdproctor/claude/casehub/ops commit -m "refactor(eidos#60): AgentNodeSpec.toDescriptor() — single fix point for field mapping

Extracts the 18-field AgentNodeSpec→AgentDescriptor mapping into
toDescriptor(String tenancyId) on the record itself. AgentProvisionHandler
uses it directly. When AgentDescriptor adds a field, one method breaks
instead of two call sites.

Refs eidos#60"
```

---

### Task 3: ops — Enrich AgentDriftChecker

**Repo:** `casehub-ops` (`/Users/mdproctor/claude/casehub/ops`)

**Prerequisites:** Task 1 installed (`mvn install -pl api` in eidos), Task 2 committed.

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ops/deployment/drift/AgentDriftChecker.java`
- Modify: `deployment/src/test/java/io/casehub/ops/deployment/drift/AgentDriftCheckerTest.java`

**Interfaces:**
- Consumes: `AgentDescriptorComparator.compare(AgentDescriptor, AgentDescriptor) → ComparisonResult` (from Task 1); `AgentNodeSpec.toDescriptor(String)` (from Task 2)
- Produces: Enriched `AgentDriftChecker.check()` — returns `DRIFTED` for any field mismatch (not just capability names), logs drifted fields at DEBUG

- [ ] **Step 1: Write failing test — non-capability field drift**

Add to `deployment/src/test/java/io/casehub/ops/deployment/drift/AgentDriftCheckerTest.java`:

```java
    @Test
    void agentDrifted_dispositionMismatch() {
        var cap = new AgentCapability("cap-a", null, null, null, List.of(), List.of(), List.of(), Map.of(), null);
        var disp1 = new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", "compromising", false);
        var descriptor = new AgentDescriptor(
                "agent-1", "Agent", "1.0", "anthropic", "claude", "4.6", "fp1",
                "domain", "slot", "disp", Map.of(), "worker",
                List.of(cap), disp1, "US", "policy", TENANCY_ID, null);
        agentRegistry.register(descriptor);

        var disp2 = new AgentDisposition("independent", "principled", "measured", "semi-autonomous", "compromising", false);
        var spec = new AgentNodeSpec("agent-1", "Agent", "worker", "anthropic", "claude", "4.6",
                "1.0", "fp1", "domain", "slot", "disp", Map.of(), List.of(cap), disp2, "US", "policy", null, List.of());

        assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
    }

    @Test
    void agentDrifted_briefingMismatch() {
        var cap = new AgentCapability("cap-a", null, null, null, List.of(), List.of(), List.of(), Map.of(), null);
        var descriptor = new AgentDescriptor(
                "agent-1", "Agent", "1.0", "anthropic", "claude", "4.6", "fp1",
                "domain", "slot", "disp", Map.of(), "worker",
                List.of(cap), null, "US", "policy", TENANCY_ID, "Original briefing");
        agentRegistry.register(descriptor);

        var spec = new AgentNodeSpec("agent-1", "Agent", "worker", "anthropic", "claude", "4.6",
                "1.0", "fp1", "domain", "slot", "disp", Map.of(), List.of(cap), null, "US", "policy", "Changed briefing", List.of());

        assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
    }

    @Test
    void agentDrifted_capabilitySubFieldMismatch() {
        var capDesired = new AgentCapability("cap-a", 0.85, null, null, List.of(), List.of(), List.of(), Map.of(), null);
        var capActual = new AgentCapability("cap-a", 0.50, null, null, List.of(), List.of(), List.of(), Map.of(), null);
        var descriptor = new AgentDescriptor(
                "agent-1", "Agent", "1.0", "anthropic", "claude", "4.6", "fp1",
                "domain", "slot", "disp", Map.of(), "worker",
                List.of(capActual), null, "US", "policy", TENANCY_ID, null);
        agentRegistry.register(descriptor);

        var spec = new AgentNodeSpec("agent-1", "Agent", "worker", "anthropic", "claude", "4.6",
                "1.0", "fp1", "domain", "slot", "disp", Map.of(), List.of(capDesired), null, "US", "policy", null, List.of());

        assertEquals(NodeStatus.DRIFTED, checker.check(spec, TENANCY_ID));
    }

    @Test
    void agentPresent_allFieldsMatch() {
        var cap = new AgentCapability("cap-a", 0.85, 2000L, "medium", List.of("text"), List.of("text"), List.of("tag"), Map.of("java", 0.95), Set.of("cobol"));
        var disp = new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", "compromising", false);
        var descriptor = new AgentDescriptor(
                "agent-1", "Agent", "1.0", "anthropic", "claude", "4.6", "fp1",
                "domain", "slot", "disp", Map.of(), "worker",
                List.of(cap), disp, "US", "policy", TENANCY_ID, "briefing");
        agentRegistry.register(descriptor);

        var spec = new AgentNodeSpec("agent-1", "Agent", "worker", "anthropic", "claude", "4.6",
                "1.0", "fp1", "domain", "slot", "disp", Map.of(), List.of(cap), disp, "US", "policy", "briefing", List.of());

        assertEquals(NodeStatus.PRESENT, checker.check(spec, TENANCY_ID));
    }
```

Add the import for `AgentDisposition` and `Set` at the top of the test file:

```java
import io.casehub.eidos.api.AgentDisposition;
import java.util.Set;
```

- [ ] **Step 2: Run new tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl deployment -Dtest=AgentDriftCheckerTest -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: `agentDrifted_dispositionMismatch` and `agentDrifted_briefingMismatch` FAIL (current checker only compares capability names — disposition and briefing drift is not detected). `agentDrifted_capabilitySubFieldMismatch` also FAILS (current checker compares capability names, not sub-fields). `agentPresent_allFieldsMatch` should PASS (names match, so current checker returns PRESENT).

- [ ] **Step 3: Enrich AgentDriftChecker.check()**

Replace the entire `AgentDriftChecker.java`:

```java
package io.casehub.ops.deployment.drift;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeStatus;
import io.casehub.eidos.api.AgentDescriptorComparator;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.ops.api.deployment.AgentNodeSpec;
import io.casehub.ops.api.deployment.NodeDriftChecker;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class AgentDriftChecker implements NodeDriftChecker {

    private static final Logger LOG = Logger.getLogger(AgentDriftChecker.class);

    private final AgentRegistry agentRegistry;

    @Inject
    public AgentDriftChecker(AgentRegistry agentRegistry) {
        this.agentRegistry = agentRegistry;
    }

    @Override
    public String nodeType() {
        return "agent";
    }

    @Override
    public NodeStatus check(NodeSpec spec, String tenancyId) {
        if (!(spec instanceof AgentNodeSpec agentSpec)) {
            return NodeStatus.UNKNOWN;
        }

        var actual = agentRegistry.findById(agentSpec.agentId(), tenancyId);
        if (actual.isEmpty()) {
            return NodeStatus.ABSENT;
        }

        var desired = agentSpec.toDescriptor(tenancyId);
        var result = AgentDescriptorComparator.compare(desired, actual.get());

        if (!result.matches()) {
            for (var drift : result.drifts()) {
                LOG.debugf("agent %s: %s drifted [%s → %s]",
                        agentSpec.agentId(), drift.field(), drift.desiredValue(), drift.actualValue());
            }
            return NodeStatus.DRIFTED;
        }

        return NodeStatus.PRESENT;
    }
}
```

- [ ] **Step 4: Run all drift checker tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl deployment -Dtest=AgentDriftCheckerTest -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: All tests PASS — existing tests (nodeType, agentPresent, agentAbsent, agentDrifted_capabilitiesMismatch, unknownSpecType) + new tests (dispositionMismatch, briefingMismatch, capabilitySubFieldMismatch, allFieldsMatch).

- [ ] **Step 5: Run full ops test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/ops/pom.xml`

Expected: All tests PASS. No regressions.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ops add deployment/src/main/java/io/casehub/ops/deployment/drift/AgentDriftChecker.java deployment/src/test/java/io/casehub/ops/deployment/drift/AgentDriftCheckerTest.java
git -C /Users/mdproctor/claude/casehub/ops commit -m "feat(eidos#60): full field-by-field agent drift detection

AgentDriftChecker now uses AgentDescriptorComparator from eidos-api for
content equality comparison across all 16 descriptor fields, 8 capability
sub-fields, and 6 disposition axes. Drifted fields are logged at DEBUG
with field path and values.

Previously compared sorted capability names only — missed drift in
disposition, vocabularies, briefing, provider/model, and capability
sub-fields (qualityHint, epistemicDomains, excludedDomains, etc.).

Refs eidos#60"
```
