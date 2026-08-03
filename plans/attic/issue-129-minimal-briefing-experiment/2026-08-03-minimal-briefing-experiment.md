# Minimal Briefing Experiment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #129 — eval: Minimal briefing experiment — isolate framework contribution from briefing override
**Issue group:** #129

**Goal:** Run a 2×2 factorial experiment (framework × briefing) measuring function activation accuracy across 8 Jungian profiles to isolate whether the personality framework or the briefing text drives cognitive function activation.

**Architecture:** Load Jungian profiles and function scenarios from existing YAML resources. For each profile, construct 4 descriptor variants (baseline/jungian × minimal/rich briefing). Render each as MARKDOWN, run through FunctionActivationJudge, collect TAA scores per condition. Report as JSON + console table.

**Tech Stack:** Java 21, Quarkus, Jackson YAML, JUnit 5, AssertJ. Eval tag excludes from CI.

## Global Constraints

- All new files in `eval/src/test/java/io/casehub/eidos/eval/` (test scope)
- `@Tag("eval")` on test class — excluded from CI
- `@QuarkusTest @TestProfile(EvalProfile.class)` — same test profile as existing eval tests
- IntelliJ MCP for all source file operations
- Minimal briefing text uses `role` field (not `name`) to avoid MBTI type leakage

---

### Task 1: JungianProfile record and JungianProfileLoader

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/JungianProfile.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/JungianProfileLoader.java`
- Test: `eval/src/test/java/io/casehub/eidos/eval/JungianProfileLoaderTest.java`

**Interfaces:**
- Consumes: `jungian-profiles/index.yaml`, `jungian-profiles/*.yaml` (existing YAML resources)
- Produces: `JungianProfile` record, `JungianProfileLoader.load() → List<JungianProfile>`, `JungianProfileLoader.loadIndex() → List<String>`

- [ ] **Step 1: Write the failing test for JungianProfileLoader**

```java
package io.casehub.eidos.eval;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class JungianProfileLoaderTest {

    @Test
    void loadsAllProfilesFromIndex() {
        var profiles = new JungianProfileLoader().load();
        assertThat(profiles).hasSize(8);
        assertThat(profiles).extracting(JungianProfile::mbtiType)
            .containsExactlyInAnyOrder("INTP", "ENTJ", "INFP", "ENFJ", "ISTJ", "ESTP", "INTJ", "ENTP");
    }

    @Test
    void profileHasRequiredFields() {
        var profiles = new JungianProfileLoader().load();
        var intp = profiles.stream().filter(p -> "INTP".equals(p.mbtiType())).findFirst().orElseThrow();
        assertThat(intp.name()).isEqualTo("intp-analyst");
        assertThat(intp.role()).isEqualTo("Systems Analyst — Ti dominant");
        assertThat(intp.dominantFunction()).isEqualTo("ti");
        assertThat(intp.auxiliaryFunction()).isEqualTo("ne");
        assertThat(intp.descriptor()).isNotNull();
        assertThat(intp.descriptor().briefing()).isNotBlank();
        assertThat(intp.descriptor().disposition().dispositionProfile()).isNotEmpty();
        assertThat(intp.descriptor().dispositionVocabulary()).isEqualTo("urn:casehub:vocab:jungian");
    }

    @Test
    void allProfilesHaveDominantAndAuxiliary() {
        var profiles = new JungianProfileLoader().load();
        for (var p : profiles) {
            assertThat(p.dominantFunction()).as(p.name() + " dominant").isNotBlank();
            assertThat(p.auxiliaryFunction()).as(p.name() + " auxiliary").isNotBlank();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=JungianProfileLoaderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `JungianProfile` and `JungianProfileLoader` do not exist.

- [ ] **Step 3: Create JungianProfile record**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;

record JungianProfile(
    String name,
    String role,
    String domain,
    String sourceType,
    String mbtiType,
    String dominantFunction,
    String auxiliaryFunction,
    AgentDescriptor descriptor
) {}
```

- [ ] **Step 4: Create JungianProfileLoader**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

class JungianProfileLoader {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory())
        .findAndRegisterModules()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    List<JungianProfile> load() {
        final List<String> filenames = loadIndex();
        final List<JungianProfile> profiles = new ArrayList<>();
        for (final String filename : filenames) {
            profiles.add(loadProfile(filename));
        }
        return profiles;
    }

    List<String> loadIndex() {
        try (InputStream is = cl().getResourceAsStream("jungian-profiles/index.yaml")) {
            if (is == null) throw new IllegalStateException("jungian-profiles/index.yaml not found");
            @SuppressWarnings("unchecked")
            final Map<String, List<String>> index = YAML.readValue(is, Map.class);
            return index.get("profiles");
        } catch (final IOException e) {
            throw new IllegalStateException("Failed to load jungian-profiles/index.yaml", e);
        }
    }

    private JungianProfile loadProfile(final String filename) {
        final String path = "jungian-profiles/" + filename;
        try (InputStream is = cl().getResourceAsStream(path)) {
            if (is == null) throw new IllegalStateException("Profile not found: " + path);
            return YAML.readValue(is, JungianProfile.class);
        } catch (final IOException e) {
            throw new IllegalStateException("Failed to load profile: " + path, e);
        }
    }

    private ClassLoader cl() {
        return Thread.currentThread().getContextClassLoader();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=JungianProfileLoaderTest`
Expected: all 3 tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add eval/src/test/java/io/casehub/eidos/eval/JungianProfile.java eval/src/test/java/io/casehub/eidos/eval/JungianProfileLoader.java eval/src/test/java/io/casehub/eidos/eval/JungianProfileLoaderTest.java
git -C $PROJECT commit -m "feat(#129): add JungianProfile record and loader for eval profiles"
```

---

### Task 2: FunctionScenarioLoader

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/FunctionScenarioLoader.java`
- Test: `eval/src/test/java/io/casehub/eidos/eval/FunctionScenarioLoaderTest.java`

**Interfaces:**
- Consumes: `function-scenarios/scenarios.yaml` (existing YAML resource), `FunctionActivationJudge.FunctionScenario` (existing record)
- Produces: `FunctionScenarioLoader.load() → Map<String, List<FunctionScenario>>`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.eval.FunctionActivationJudge.FunctionScenario;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class FunctionScenarioLoaderTest {

    @Test
    void loadsAllEightFunctions() {
        Map<String, List<FunctionScenario>> scenarios = FunctionScenarioLoader.load();
        assertThat(scenarios).hasSize(8);
        assertThat(scenarios.keySet()).containsExactlyInAnyOrder(
            "ti", "te", "fi", "fe", "si", "se", "ni", "ne");
    }

    @Test
    void eachFunctionHasThreeScenarios() {
        Map<String, List<FunctionScenario>> scenarios = FunctionScenarioLoader.load();
        for (var entry : scenarios.entrySet()) {
            assertThat(entry.getValue())
                .as("scenarios for " + entry.getKey())
                .hasSize(3);
        }
    }

    @Test
    void scenarioTargetFunctionMatchesKey() {
        Map<String, List<FunctionScenario>> scenarios = FunctionScenarioLoader.load();
        for (var entry : scenarios.entrySet()) {
            for (var s : entry.getValue()) {
                assertThat(s.targetFunction()).isEqualTo(entry.getKey());
            }
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=FunctionScenarioLoaderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `FunctionScenarioLoader` does not exist.

- [ ] **Step 3: Create FunctionScenarioLoader**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.eidos.eval.FunctionActivationJudge.FunctionScenario;

import java.io.IOException;
import java.io.InputStream;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

final class FunctionScenarioLoader {

    private FunctionScenarioLoader() {}

    private static final ObjectMapper YAML =
        new ObjectMapper(new YAMLFactory()).findAndRegisterModules();

    @SuppressWarnings("unchecked")
    static Map<String, List<FunctionScenario>> load() {
        try (InputStream is = Thread.currentThread().getContextClassLoader()
                .getResourceAsStream("function-scenarios/scenarios.yaml")) {
            if (is == null) throw new IllegalStateException("function-scenarios/scenarios.yaml not found");
            final Map<String, Object> root = YAML.readValue(is, Map.class);
            final List<Map<String, Object>> rawScenarios = (List<Map<String, Object>>) root.get("scenarios");
            final Map<String, List<FunctionScenario>> result = new LinkedHashMap<>();
            for (final Map<String, Object> entry : rawScenarios) {
                final String fn = (String) entry.get("targetFunction");
                final List<String> prompts = (List<String>) entry.get("prompts");
                result.put(fn, prompts.stream()
                    .map(p -> new FunctionScenario(fn, p))
                    .toList());
            }
            return result;
        } catch (final IOException e) {
            throw new IllegalStateException("Failed to load function-scenarios/scenarios.yaml", e);
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=FunctionScenarioLoaderTest`
Expected: all 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add eval/src/test/java/io/casehub/eidos/eval/FunctionScenarioLoader.java eval/src/test/java/io/casehub/eidos/eval/FunctionScenarioLoaderTest.java
git -C $PROJECT commit -m "feat(#129): add FunctionScenarioLoader for eval scenarios"
```

---

### Task 3: BriefingCondition enum with descriptor variant construction

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/BriefingCondition.java`
- Test: `eval/src/test/java/io/casehub/eidos/eval/BriefingConditionTest.java`

**Interfaces:**
- Consumes: `JungianProfile` (Task 1), `AgentDescriptor`, `AgentDisposition`
- Produces: `BriefingCondition.apply(JungianProfile) → AgentDescriptor` — 4 enum values: `BASELINE_MINIMAL`, `BASELINE_RICH`, `JUNGIAN_MINIMAL`, `JUNGIAN_RICH`

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class BriefingConditionTest {

    static JungianProfile intp;

    @BeforeAll
    static void loadProfile() {
        intp = new JungianProfileLoader().load().stream()
            .filter(p -> "INTP".equals(p.mbtiType()))
            .findFirst().orElseThrow();
    }

    @Test
    void jungianRich_returnsOriginalDescriptor() {
        AgentDescriptor d = BriefingCondition.JUNGIAN_RICH.apply(intp);
        assertThat(d.briefing()).isEqualTo(intp.descriptor().briefing());
        assertThat(d.disposition().dispositionProfile()).isNotEmpty();
        assertThat(d.dispositionVocabulary()).isEqualTo("urn:casehub:vocab:jungian");
    }

    @Test
    void jungianMinimal_keepFramework_stripBriefing() {
        AgentDescriptor d = BriefingCondition.JUNGIAN_MINIMAL.apply(intp);
        assertThat(d.briefing()).startsWith("You are an agent named ");
        assertThat(d.briefing()).doesNotContain("INTP");
        assertThat(d.disposition().dispositionProfile()).isNotEmpty();
        assertThat(d.dispositionVocabulary()).isEqualTo("urn:casehub:vocab:jungian");
    }

    @Test
    void baselineRich_stripFramework_keepBriefing() {
        AgentDescriptor d = BriefingCondition.BASELINE_RICH.apply(intp);
        assertThat(d.briefing()).isEqualTo(intp.descriptor().briefing());
        assertThat(d.disposition().dispositionProfile()).isEmpty();
        assertThat(d.dispositionVocabulary()).isNull();
    }

    @Test
    void baselineMinimal_stripBoth() {
        AgentDescriptor d = BriefingCondition.BASELINE_MINIMAL.apply(intp);
        assertThat(d.briefing()).startsWith("You are an agent named ");
        assertThat(d.briefing()).doesNotContain("INTP");
        assertThat(d.disposition().dispositionProfile()).isEmpty();
        assertThat(d.dispositionVocabulary()).isNull();
    }

    @Test
    void allConditions_preserveNonVariedFields() {
        for (BriefingCondition condition : BriefingCondition.values()) {
            AgentDescriptor d = condition.apply(intp);
            AgentDescriptor orig = intp.descriptor();
            assertThat(d.agentId()).as(condition + " agentId").isEqualTo(orig.agentId());
            assertThat(d.name()).as(condition + " name").isEqualTo(orig.name());
            assertThat(d.slot()).as(condition + " slot").isEqualTo(orig.slot());
            assertThat(d.tenancyId()).as(condition + " tenancyId").isEqualTo(orig.tenancyId());
            assertThat(d.capabilities()).as(condition + " capabilities").isEqualTo(orig.capabilities());
        }
    }

    @Test
    void minimalBriefing_usesRoleNotName() {
        AgentDescriptor d = BriefingCondition.BASELINE_MINIMAL.apply(intp);
        assertThat(d.briefing()).contains("Systems Analyst");
        assertThat(d.briefing()).doesNotContain("(INTP)");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=BriefingConditionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `BriefingCondition` does not exist.

- [ ] **Step 3: Create BriefingCondition enum**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentDisposition;

enum BriefingCondition {

    BASELINE_MINIMAL(false, false),
    BASELINE_RICH(false, true),
    JUNGIAN_MINIMAL(true, false),
    JUNGIAN_RICH(true, true);

    private final boolean keepFramework;
    private final boolean keepBriefing;

    BriefingCondition(boolean keepFramework, boolean keepBriefing) {
        this.keepFramework = keepFramework;
        this.keepBriefing = keepBriefing;
    }

    AgentDescriptor apply(JungianProfile profile) {
        final AgentDescriptor orig = profile.descriptor();
        final String briefing = keepBriefing
            ? orig.briefing()
            : "You are an agent named " + extractRole(profile.role());

        final AgentDisposition disposition = keepFramework
            ? orig.disposition()
            : AgentDisposition.builder()
                .delegation(orig.disposition() != null && orig.disposition().delegation())
                .build();

        final String dispositionVocabulary = keepFramework
            ? orig.dispositionVocabulary()
            : null;

        return AgentDescriptor.builder()
            .agentId(orig.agentId())
            .name(orig.name())
            .version(orig.version())
            .provider(orig.provider())
            .modelFamily(orig.modelFamily())
            .modelVersion(orig.modelVersion())
            .weightsFingerprint(orig.weightsFingerprint())
            .domainVocabulary(orig.domainVocabulary())
            .slotVocabulary(orig.slotVocabulary())
            .dispositionVocabulary(dispositionVocabulary)
            .axisVocabularies(keepFramework ? orig.axisVocabularies() : null)
            .slot(orig.slot())
            .capabilities(orig.capabilities())
            .disposition(disposition)
            .jurisdiction(orig.jurisdiction())
            .dataHandlingPolicy(orig.dataHandlingPolicy())
            .tenancyId(orig.tenancyId())
            .briefing(briefing)
            .templates(orig.templates())
            .goals(orig.goals())
            .constraints(orig.constraints())
            .build();
    }

    private static String extractRole(String role) {
        final int dash = role.indexOf(" — ");
        return dash > 0 ? role.substring(0, dash) : role;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=BriefingConditionTest`
Expected: all 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add eval/src/test/java/io/casehub/eidos/eval/BriefingCondition.java eval/src/test/java/io/casehub/eidos/eval/BriefingConditionTest.java
git -C $PROJECT commit -m "feat(#129): add BriefingCondition enum for 2×2 factorial experiment"
```

---

### Task 4: MinimalBriefingEvalTest and report output

**Files:**
- Create: `eval/src/test/java/io/casehub/eidos/eval/BriefingExperimentReport.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/MinimalBriefingEvalTest.java`

**Interfaces:**
- Consumes: `JungianProfileLoader.load()` (Task 1), `FunctionScenarioLoader.load()` (Task 2), `BriefingCondition.apply()` (Task 3), `FunctionActivationJudge.evaluate()` (existing), `SystemPromptRenderer.render()` (existing), `DescriptorCollector.deriveDispositionAxes()` (existing), `VocabularyRegistry` (existing CDI bean), `EvalReportWriter.MAPPER` pattern (existing)
- Produces: `target/briefing-experiment-report.json`, console summary table

- [ ] **Step 1: Create BriefingExperimentReport record**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.eval.FunctionActivationJudge.FunctionActivationResult;

import java.time.Instant;
import java.util.List;
import java.util.Map;

record BriefingExperimentReport(
    Instant timestamp,
    String modelLabel,
    List<ProfileResult> profiles,
    Map<BriefingCondition, ConditionSummary> aggregated,
    double frameworkContribution,
    double briefingContribution
) {
    record ProfileResult(
        String name,
        String mbtiType,
        String dominantFunction,
        String auxiliaryFunction,
        Map<BriefingCondition, FunctionActivationResult> conditions
    ) {}

    record ConditionSummary(double meanTaa) {}
}
```

- [ ] **Step 2: Create MinimalBriefingEvalTest**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.eval.FunctionActivationJudge.FunctionActivationResult;
import io.casehub.eidos.eval.FunctionActivationJudge.FunctionScenario;
import io.casehub.eidos.runtime.registrar.DescriptorCollector;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.util.ArrayList;
import java.util.EnumMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(EvalProfile.class)
@Tag("eval")
class MinimalBriefingEvalTest {

    static List<JungianProfile> profiles;
    static Map<String, List<FunctionScenario>> scenariosByFunction;

    @BeforeAll
    static void loadData() {
        profiles = new JungianProfileLoader().load();
        scenariosByFunction = FunctionScenarioLoader.load();
    }

    @Inject SystemPromptRenderer renderer;
    @Inject VocabularyRegistry vocabRegistry;
    @Inject FunctionActivationJudge functionJudge;
    @Inject ObjectMapper mapper;

    @ConfigProperty(name = "casehub.eval.model.label", defaultValue = "claude")
    String modelLabel;

    @Test
    void compareBriefingContribution() throws Exception {
        final List<BriefingExperimentReport.ProfileResult> profileResults = new ArrayList<>();

        for (final JungianProfile profile : profiles) {
            final List<FunctionScenario> scenarios = Stream.concat(
                scenariosByFunction.getOrDefault(profile.dominantFunction(), List.of()).stream(),
                scenariosByFunction.getOrDefault(profile.auxiliaryFunction(), List.of()).stream()
            ).toList();

            final Map<BriefingCondition, FunctionActivationResult> conditions = new EnumMap<>(BriefingCondition.class);
            for (final BriefingCondition condition : BriefingCondition.values()) {
                final AgentDescriptor descriptor = condition.apply(profile);
                final AgentDescriptor derived = DescriptorCollector.deriveDispositionAxes(descriptor, vocabRegistry);
                final String prompt = renderer.render(derived,
                    AgentPromptContext.forFormat(RenderFormat.MARKDOWN)).content();
                final FunctionActivationResult result = functionJudge.evaluate(
                    prompt, profile.mbtiType(), scenarios);
                conditions.put(condition, result);
            }

            profileResults.add(new BriefingExperimentReport.ProfileResult(
                profile.name(), profile.mbtiType(),
                profile.dominantFunction(), profile.auxiliaryFunction(),
                conditions));
        }

        final Map<BriefingCondition, BriefingExperimentReport.ConditionSummary> aggregated =
            new EnumMap<>(BriefingCondition.class);
        for (final BriefingCondition c : BriefingCondition.values()) {
            final double meanTaa = profileResults.stream()
                .mapToDouble(p -> p.conditions().get(c).taa())
                .average().orElse(0.0);
            aggregated.put(c, new BriefingExperimentReport.ConditionSummary(meanTaa));
        }

        final double frameworkContribution =
            aggregated.get(BriefingCondition.JUNGIAN_MINIMAL).meanTaa()
            - aggregated.get(BriefingCondition.BASELINE_MINIMAL).meanTaa();
        final double briefingContribution =
            aggregated.get(BriefingCondition.BASELINE_RICH).meanTaa()
            - aggregated.get(BriefingCondition.BASELINE_MINIMAL).meanTaa();

        final BriefingExperimentReport report = new BriefingExperimentReport(
            Instant.now(), modelLabel, profileResults,
            aggregated, frameworkContribution, briefingContribution);

        Files.createDirectories(Path.of("target"));
        mapper.copy()
            .enable(SerializationFeature.INDENT_OUTPUT)
            .writeValue(Path.of("target/briefing-experiment-report.json").toFile(), report);

        System.out.println(summaryTable(report));

        assertThat(aggregated.get(BriefingCondition.JUNGIAN_RICH).meanTaa())
            .as("JUNGIAN_RICH mean TAA should exceed BASELINE_MINIMAL")
            .isGreaterThanOrEqualTo(aggregated.get(BriefingCondition.BASELINE_MINIMAL).meanTaa());
    }

    static String summaryTable(BriefingExperimentReport report) {
        final var sb = new StringBuilder();
        sb.append("\n=== Minimal Briefing Experiment ===\n\n");
        sb.append(String.format("%-20s | %8s | %9s | %8s | %9s%n",
            "Profile", "Base+Min", "Base+Rich", "Jung+Min", "Jung+Rich"));
        sb.append("─".repeat(20)).append("─┼──────────┼───────────┼──────────┼───────────\n");
        for (final var p : report.profiles()) {
            sb.append(String.format("%-20s | %8.2f | %9.2f | %8.2f | %9.2f%n",
                p.name(),
                p.conditions().get(BriefingCondition.BASELINE_MINIMAL).taa(),
                p.conditions().get(BriefingCondition.BASELINE_RICH).taa(),
                p.conditions().get(BriefingCondition.JUNGIAN_MINIMAL).taa(),
                p.conditions().get(BriefingCondition.JUNGIAN_RICH).taa()));
        }
        sb.append("─".repeat(20)).append("─┼──────────┼───────────┼──────────┼───────────\n");
        sb.append(String.format("%-20s | %8.2f | %9.2f | %8.2f | %9.2f%n",
            "Mean",
            report.aggregated().get(BriefingCondition.BASELINE_MINIMAL).meanTaa(),
            report.aggregated().get(BriefingCondition.BASELINE_RICH).meanTaa(),
            report.aggregated().get(BriefingCondition.JUNGIAN_MINIMAL).meanTaa(),
            report.aggregated().get(BriefingCondition.JUNGIAN_RICH).meanTaa()));
        sb.append(String.format("%nFramework contribution (Jung+Min − Base+Min): %+.2f%n", report.frameworkContribution()));
        sb.append(String.format("Briefing contribution (Base+Rich − Base+Min): %+.2f%n", report.briefingContribution()));
        return sb.toString();
    }
}
```

- [ ] **Step 3: Run compilation check**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl eval`
Expected: compilation succeeds. (The test itself requires `@Tag("eval")` profile to run — not in default surefire.)

- [ ] **Step 4: Run full eval module test suite to check no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval`
Expected: all existing tests PASS (MinimalBriefingEvalTest is excluded by default — `@Tag("eval")` not in default groups).

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add eval/src/test/java/io/casehub/eidos/eval/BriefingExperimentReport.java eval/src/test/java/io/casehub/eidos/eval/MinimalBriefingEvalTest.java
git -C $PROJECT commit -m "feat(#129): add MinimalBriefingEvalTest — 2×2 briefing factorial experiment"
```

- [ ] **Step 6: Run the experiment (manual — requires Claude CLI or Ollama)**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval \
  -Dtest=MinimalBriefingEvalTest#compareBriefingContribution
```

Expected: console table with TAA scores per profile per condition; `target/briefing-experiment-report.json` written. No pass/fail assertion (diagnostic only, except the sanity check).

---

### Task 5: Build verification and CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` (add MinimalBriefingEvalTest run command)

**Interfaces:**
- Consumes: all tasks above
- Produces: green build, updated documentation

- [ ] **Step 1: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and all default tests pass.

- [ ] **Step 2: Run ide_diagnostics on all new files**

Check each new file for errors:
- `eval/src/test/java/io/casehub/eidos/eval/JungianProfile.java`
- `eval/src/test/java/io/casehub/eidos/eval/JungianProfileLoader.java`
- `eval/src/test/java/io/casehub/eidos/eval/FunctionScenarioLoader.java`
- `eval/src/test/java/io/casehub/eidos/eval/BriefingCondition.java`
- `eval/src/test/java/io/casehub/eidos/eval/BriefingExperimentReport.java`
- `eval/src/test/java/io/casehub/eidos/eval/MinimalBriefingEvalTest.java`

Expected: no errors.

- [ ] **Step 3: Update CLAUDE.md eval section**

Add to the eval harness run commands section in CLAUDE.md:

```markdown
# Run minimal briefing experiment — isolates framework vs briefing contribution
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval \
  -Dtest=MinimalBriefingEvalTest#compareBriefingContribution
```

- [ ] **Step 4: Commit**

```bash
git -C $PROJECT add CLAUDE.md
git -C $PROJECT commit -m "docs(#129): add MinimalBriefingEvalTest run command to CLAUDE.md"
```
