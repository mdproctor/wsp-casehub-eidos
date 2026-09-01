# Descriptor Jackson Module Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #147 — generate YAML config, JSON Schema, and annotation config from AgentDescriptor record
**Issue group:** #147

**Goal:** Replace hand-maintained DescriptorConfig intermediate classes with a Jackson Module
pattern (custom deserializers + CDI-injected VocabularyRegistry), following engine's
CaseDefinitionModule precedent.

**Architecture:** New `EidosDescriptorModule` (Jackson Module) registers custom deserializers
for `AgentDescriptor` and `AgentDisposition`. The deserializers use the existing builders to
construct records from YAML. `ClasspathYamlDescriptorRegistrar` simplifies to just classpath
scanning + ObjectMapper setup — all 7 inner config classes and the `toDescriptor()` method
are deleted.

**Tech Stack:** Java 21, Jackson YAML, Quarkus CDI (optional — tests run without it)

## Global Constraints

- Java 21 source, Java 26 JVM
- `api/` module must remain pure Java — no Jackson annotations on records
- YAML format is unchanged — all existing `descriptors.yaml` files must parse identically
- All 22 existing tests in `ClasspathYamlDescriptorRegistrarTest` must pass after migration
- `FAIL_ON_UNKNOWN_PROPERTIES = true` must be preserved

---

## Batch 1: Deserializers (pure additions, no existing code changes)

### Task 1: DispositionDeserializer

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/DispositionDeserializer.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/DispositionDeserializerTest.java`

**Interfaces:**
- Consumes: `AgentDisposition.Builder` (existing, in api/)
- Consumes: `VocabularyRegistry` (existing SPI, in api/) — nullable for non-CDI contexts
- Produces: `DispositionDeserializer` — `JsonDeserializer<AgentDisposition>` that later tasks
  register in the Jackson Module

- [ ] **Step 1: Write failing tests for string axis adaptation**

```java
package io.casehub.eidos.runtime.yaml;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionAxis;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DispositionDeserializerTest {

    private final ObjectMapper mapper = createMapper(null);

    private static ObjectMapper createMapper(io.casehub.eidos.api.VocabularyRegistry registry) {
        var module = new SimpleModule();
        module.addDeserializer(AgentDisposition.class, new DispositionDeserializer(registry));
        return new ObjectMapper(new YAMLFactory()).registerModule(module);
    }

    @Test
    void stringAxis_convertsToDispositionValueList() throws Exception {
        var yaml = """
            socialOrient: collaborative
            ruleFollowing: strict
            riskAppetite: bold
            autonomy: autonomous
            conflictMode: competing
            delegation: true
            """;
        var disp = mapper.readValue(yaml, AgentDisposition.class);
        assertThat(disp.primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)).isEqualTo("collaborative");
        assertThat(disp.primaryTerm(DispositionAxis.RULE_FOLLOWING)).isEqualTo("strict");
        assertThat(disp.primaryTerm(DispositionAxis.RISK_APPETITE)).isEqualTo("bold");
        assertThat(disp.primaryTerm(DispositionAxis.AUTONOMY)).isEqualTo("autonomous");
        assertThat(disp.primaryTerm(DispositionAxis.CONFLICT_MODE)).isEqualTo("competing");
        assertThat(disp.delegation()).isTrue();
    }

    @Test
    void dispositionProfile_parsesWeightedValues() throws Exception {
        var yaml = """
            dispositionProfile:
              - term: te
                weight: 0.35
              - term: ni
                weight: 0.20
            """;
        var disp = mapper.readValue(yaml, AgentDisposition.class);
        assertThat(disp.dispositionProfile()).hasSize(2);
        assertThat(disp.dispositionProfile().get(0).term()).isEqualTo("te");
        assertThat(disp.dispositionProfile().get(0).weight()).isEqualTo(0.35);
    }

    @Test
    void styleProfile_parsesWeightedValues() throws Exception {
        var yaml = """
            styleProfile:
              - term: concise
                weight: 0.60
              - term: formal
                weight: 0.40
            """;
        var disp = mapper.readValue(yaml, AgentDisposition.class);
        assertThat(disp.styleProfile()).hasSize(2);
        assertThat(disp.styleProfile().get(0).term()).isEqualTo("concise");
    }

    @Test
    void emptyDisposition_returnsDefaults() throws Exception {
        var yaml = "delegation: false";
        var disp = mapper.readValue(yaml, AgentDisposition.class);
        assertThat(disp.socialOrient()).isEmpty();
        assertThat(disp.delegation()).isFalse();
    }

    @Test
    void mbtiType_resolvesProfile_whenRegistryAvailable() throws Exception {
        var registry = testVocabRegistry();
        var mapperWithVocab = createMapper(registry);
        var yaml = "mbtiType: ENTJ";
        var disp = mapperWithVocab.readValue(yaml, AgentDisposition.class);
        assertThat(disp.dispositionProfile()).hasSize(8);
        assertThat(disp.dispositionProfile().get(0).term()).isEqualTo("te");
    }

    @Test
    void mbtiType_caseInsensitive() throws Exception {
        var registry = testVocabRegistry();
        var mapperWithVocab = createMapper(registry);
        var yaml = "mbtiType: entj";
        var disp = mapperWithVocab.readValue(yaml, AgentDisposition.class);
        assertThat(disp.dispositionProfile()).hasSize(8);
    }

    @Test
    void explicitProfile_winsOverMbtiType() throws Exception {
        var registry = testVocabRegistry();
        var mapperWithVocab = createMapper(registry);
        var yaml = """
            mbtiType: ENTJ
            dispositionProfile:
              - term: ti
                weight: 0.50
            """;
        var disp = mapperWithVocab.readValue(yaml, AgentDisposition.class);
        assertThat(disp.dispositionProfile()).hasSize(1);
        assertThat(disp.dispositionProfile().get(0).term()).isEqualTo("ti");
    }

    @Test
    void enneagramType_projectsAxes() throws Exception {
        var registry = testVocabRegistry();
        var mapperWithVocab = createMapper(registry);
        var yaml = "enneagramType: challenger";
        var disp = mapperWithVocab.readValue(yaml, AgentDisposition.class);
        assertThat(disp.primaryTerm(DispositionAxis.RULE_FOLLOWING)).isEqualTo("flexible");
        assertThat(disp.primaryTerm(DispositionAxis.RISK_APPETITE)).isEqualTo("bold");
        assertThat(disp.primaryTerm(DispositionAxis.CONFLICT_MODE)).isEqualTo("competing");
    }

    @Test
    void enneagramType_doesNotOverwriteExplicitAxes() throws Exception {
        var registry = testVocabRegistry();
        var mapperWithVocab = createMapper(registry);
        var yaml = """
            enneagramType: challenger
            socialOrient: collaborative
            """;
        var disp = mapperWithVocab.readValue(yaml, AgentDisposition.class);
        assertThat(disp.primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)).isEqualTo("collaborative");
    }

    @Test
    void mbtiType_ignoredWithoutRegistry() throws Exception {
        var yaml = "mbtiType: ENTJ";
        var disp = mapper.readValue(yaml, AgentDisposition.class);
        assertThat(disp.dispositionProfile()).isEmpty();
    }

    private static io.casehub.eidos.api.VocabularyRegistry testVocabRegistry() {
        var registry = new io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry();
        registry.register(io.casehub.eidos.vocab.JungianFunctionTerm.class);
        registry.register(io.casehub.eidos.vocab.MbtiTypeTerm.class);
        registry.register(io.casehub.eidos.vocab.EnneagramTerm.class);
        registry.register(io.casehub.eidos.vocab.ConscientiousnessTerm.class);
        registry.register(io.casehub.eidos.vocab.ThomasKilmannTerm.class);
        return registry;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DispositionDeserializerTest -DfailIfNoTests=false`
Expected: compilation failure — `DispositionDeserializer` does not exist

- [ ] **Step 3: Implement DispositionDeserializer**

Create `runtime/src/main/java/io/casehub/eidos/runtime/yaml/DispositionDeserializer.java`.

Port the disposition mapping logic from `ClasspathYamlDescriptorRegistrar.toDescriptor()`
lines 57-108. The deserializer:

1. Reads the YAML node tree via `p.readValueAsTree()`
2. Builds `AgentDisposition` via the builder
3. String axis values → `builder.socialOrient(stringValue)` (builder already handles
   `String → List.of(DispositionValue.of(s))`)
4. `dispositionProfile` / `styleProfile` → parse array of `{term, weight}` objects
5. `mbtiType` → if registry available, resolve via `registry.resolve("urn:casehub:vocab:mbti", ...)`,
   call `term.defaultProfile()`, set on builder. Only if explicit `dispositionProfile` not present.
6. `enneagramType` → if registry available, cross-vocabulary axis projection via
   `registry.equivalentValues()` for each axis. Only fills axes not explicitly set.
7. `delegation` → `builder.delegation(boolean)`
8. Unknown fields (`mbtiType`, `enneagramType`) must not cause `FAIL_ON_UNKNOWN_PROPERTIES`
   errors — use `p.readValueAsTree()` to read all fields, process known ones, ignore
   convenience fields silently

```java
package io.casehub.eidos.runtime.yaml;

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonDeserializer;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.DispositionValue;
import io.casehub.eidos.api.VocabularyRegistry;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;

public class DispositionDeserializer extends JsonDeserializer<AgentDisposition> {

    private final VocabularyRegistry vocabRegistry;

    public DispositionDeserializer(VocabularyRegistry vocabRegistry) {
        this.vocabRegistry = vocabRegistry;
    }

    @Override
    public AgentDisposition deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        ObjectNode root = p.readValueAsTree();
        var builder = AgentDisposition.builder();

        // Track which axes are explicitly set (for enneagram non-overwrite)
        String explicitSocialOrient = stringField(root, "socialOrient");
        String explicitRuleFollowing = stringField(root, "ruleFollowing");
        String explicitRiskAppetite = stringField(root, "riskAppetite");
        String explicitAutonomy = stringField(root, "autonomy");
        String explicitConflictMode = stringField(root, "conflictMode");

        if (explicitSocialOrient != null) builder.socialOrient(explicitSocialOrient);
        if (explicitRuleFollowing != null) builder.ruleFollowing(explicitRuleFollowing);
        if (explicitRiskAppetite != null) builder.riskAppetite(explicitRiskAppetite);
        if (explicitAutonomy != null) builder.autonomy(explicitAutonomy);
        if (explicitConflictMode != null) builder.conflictMode(explicitConflictMode);

        if (root.has("delegation")) builder.delegation(root.get("delegation").asBoolean());

        // Weighted profiles
        List<DispositionValue> explicitProfile = parseWeightedList(root, "dispositionProfile");
        if (explicitProfile != null) builder.dispositionProfile(explicitProfile);

        List<DispositionValue> styleProfile = parseWeightedList(root, "styleProfile");
        if (styleProfile != null) builder.styleProfile(styleProfile);

        // Convenience: mbtiType → dispositionProfile (only if no explicit profile)
        if (root.has("mbtiType") && explicitProfile == null && vocabRegistry != null) {
            String mbtiType = root.get("mbtiType").asText().toLowerCase(Locale.ROOT);
            vocabRegistry.resolve("urn:casehub:vocab:mbti", mbtiType)
                .ifPresent(term -> builder.dispositionProfile(term.defaultProfile()));
        }

        // Convenience: enneagramType → axis values (only fills unset axes)
        if (root.has("enneagramType") && vocabRegistry != null) {
            String enneaValue = root.get("enneagramType").asText().toLowerCase(Locale.ROOT);
            if (vocabRegistry.resolve("urn:casehub:vocab:enneagram", enneaValue).isPresent()) {
                for (var axis : DispositionAxis.values()) {
                    if (axis == DispositionAxis.CONFLICT_MODE) {
                        if (explicitConflictMode == null) {
                            vocabRegistry.equivalentValues(
                                "urn:casehub:vocab:enneagram", enneaValue,
                                "urn:casehub:vocab:thomas-kilmann", axis)
                                .ifPresent(builder::conflictMode);
                        }
                    } else {
                        String explicit = switch (axis) {
                            case SOCIAL_ORIENTATION -> explicitSocialOrient;
                            case RULE_FOLLOWING -> explicitRuleFollowing;
                            case RISK_APPETITE -> explicitRiskAppetite;
                            case AUTONOMY -> explicitAutonomy;
                            default -> null;
                        };
                        if (explicit == null) {
                            vocabRegistry.equivalentValues(
                                "urn:casehub:vocab:enneagram", enneaValue,
                                "urn:casehub:vocab:conscientiousness", axis)
                                .ifPresent(val -> {
                                    switch (axis) {
                                        case SOCIAL_ORIENTATION -> builder.socialOrient(val);
                                        case RULE_FOLLOWING -> builder.ruleFollowing(val);
                                        case RISK_APPETITE -> builder.riskAppetite(val);
                                        case AUTONOMY -> builder.autonomy(val);
                                        default -> {}
                                    }
                                });
                        }
                    }
                }
            }
        }

        return builder.build();
    }

    private static String stringField(ObjectNode node, String field) {
        return node.has(field) && !node.get(field).isNull() ? node.get(field).asText() : null;
    }

    private static List<DispositionValue> parseWeightedList(ObjectNode node, String field) {
        if (!node.has(field) || !node.get(field).isArray()) return null;
        var values = new ArrayList<DispositionValue>();
        for (JsonNode item : node.get(field)) {
            values.add(new DispositionValue(item.get("term").asText(), item.get("weight").asDouble()));
        }
        return values;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DispositionDeserializerTest`
Expected: all 10 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/yaml/DispositionDeserializer.java runtime/src/test/java/io/casehub/eidos/runtime/yaml/DispositionDeserializerTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#147): add DispositionDeserializer — String→DispositionValue, mbtiType, enneagramType Refs #147"
```

### Task 2: AgentDescriptorDeserializer + EidosDescriptorModule

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializer.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/EidosDescriptorModule.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializerTest.java`

**Interfaces:**
- Consumes: `DispositionDeserializer` (from Task 1)
- Consumes: `AgentDescriptor.Builder`, `AgentCapability`, `AgentGoal`, `AgentConstraint`,
  `TemplateRef` (existing records in api/)
- Produces: `EidosDescriptorModule` — Jackson `SimpleModule` that registers both deserializers.
  Constructor: `EidosDescriptorModule(VocabularyRegistry registry)` (nullable).
  Task 3 uses this to replace config classes.

- [ ] **Step 1: Write failing tests for full descriptor deserialization**

```java
package io.casehub.eidos.runtime.yaml;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AgentDescriptorDeserializerTest {

    private final ObjectMapper mapper = EidosDescriptorModule.createMapper(null);

    @Test
    void fullDescriptor_allFieldsDeserialize() throws Exception {
        var yaml = """
            agentId: test-1
            name: Test Agent
            slot: reviewer
            tenancyId: default
            version: "1.0"
            provider: acme
            modelFamily: gpt-4
            modelVersion: "2026-01"
            weightsFingerprint: sha256:abc
            domainVocabulary: urn:domain
            slotVocabulary: urn:slot
            dispositionVocabulary: urn:disp
            styleVocabulary: urn:style
            jurisdiction: EU
            dataHandlingPolicy: GDPR
            briefing: You are a test agent.
            disposition:
              socialOrient: collaborative
              delegation: true
            capabilities:
              - name: review
                description: Code review capability
                qualityHint: 0.95
                latencyHintP50Ms: 5000
                costHint: medium
                inputTypes: [code]
                outputTypes: [review]
                tags: [quality]
                epistemicDomains:
                  java: 0.95
                  rust: 0.3
                excludedDomains: [cobol]
            goals:
              - name: ensure-quality
                description: Ensure code quality
                priority: PRIMARY
                visibility: PUBLIC
                capabilities: [review]
            constraints:
              - name: no-pii
                description: Never process PII
                visibility: PUBLIC
                severity: HARD
            templates:
              - ref: safety-preamble
                args:
                  domain: healthcare
            """;
        var d = mapper.readValue(yaml, AgentDescriptor.class);
        assertThat(d.agentId()).isEqualTo("test-1");
        assertThat(d.name()).isEqualTo("Test Agent");
        assertThat(d.slot()).isEqualTo("reviewer");
        assertThat(d.tenancyId()).isEqualTo("default");
        assertThat(d.version()).isEqualTo("1.0");
        assertThat(d.provider()).isEqualTo("acme");
        assertThat(d.modelFamily()).isEqualTo("gpt-4");
        assertThat(d.modelVersion()).isEqualTo("2026-01");
        assertThat(d.weightsFingerprint()).isEqualTo("sha256:abc");
        assertThat(d.domainVocabulary()).isEqualTo("urn:domain");
        assertThat(d.jurisdiction()).isEqualTo("EU");
        assertThat(d.briefing()).isEqualTo("You are a test agent.");
        assertThat(d.disposition().primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)).isEqualTo("collaborative");
        assertThat(d.disposition().delegation()).isTrue();
        assertThat(d.capabilities()).hasSize(1);
        assertThat(d.capabilities().get(0).name()).isEqualTo("review");
        assertThat(d.capabilities().get(0).description()).isEqualTo("Code review capability");
        assertThat(d.capabilities().get(0).qualityHint()).isEqualTo(0.95);
        assertThat(d.capabilities().get(0).latencyHintP50Ms()).isEqualTo(5000L);
        assertThat(d.capabilities().get(0).epistemicDomains()).containsEntry("java", 0.95);
        assertThat(d.capabilities().get(0).excludedDomains()).containsExactly("cobol");
        assertThat(d.goals()).hasSize(1);
        assertThat(d.goals().get(0).name()).isEqualTo("ensure-quality");
        assertThat(d.constraints()).hasSize(1);
        assertThat(d.constraints().get(0).severity()).isEqualTo(ConstraintSeverity.HARD);
        assertThat(d.templates()).hasSize(1);
        assertThat(d.templates().get(0).templateId()).isEqualTo("safety-preamble");
    }

    @Test
    void minimalDescriptor_nullOptionalFields() throws Exception {
        var yaml = """
            agentId: minimal
            name: Minimal
            slot: s
            tenancyId: t
            """;
        var d = mapper.readValue(yaml, AgentDescriptor.class);
        assertThat(d.version()).isNull();
        assertThat(d.disposition()).isNull();
        assertThat(d.capabilities()).isEmpty();
        assertThat(d.goals()).isEmpty();
    }

    @Test
    void axisVocabularies_deserializeToEnumKeys() throws Exception {
        var yaml = """
            agentId: vocab-test
            name: N
            slot: s
            tenancyId: t
            axisVocabularies:
              CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
              RULE_FOLLOWING: urn:casehub:vocab:conscientiousness
            """;
        var d = mapper.readValue(yaml, AgentDescriptor.class);
        assertThat(d.axisVocabularies())
            .containsEntry(DispositionAxis.CONFLICT_MODE, "urn:casehub:vocab:thomas-kilmann")
            .containsEntry(DispositionAxis.RULE_FOLLOWING, "urn:casehub:vocab:conscientiousness");
    }

    @Test
    void invalidAxisVocabularyKey_throws() {
        var yaml = """
            agentId: bad
            name: N
            slot: s
            tenancyId: t
            axisVocabularies:
              INVALID_AXIS: urn:foo
            """;
        assertThatThrownBy(() -> mapper.readValue(yaml, AgentDescriptor.class))
            .hasRootCauseInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void missingRequiredField_throwsValidationException() {
        var yaml = """
            name: No ID
            slot: s
            tenancyId: t
            """;
        assertThatThrownBy(() -> mapper.readValue(yaml, AgentDescriptor.class))
            .hasRootCauseInstanceOf(AgentValidationException.class)
            .hasMessageContaining("agentId");
    }

    @Test
    void tenancyId_defaultsWhenAbsent() throws Exception {
        var yaml = """
            agentId: no-tenancy
            name: N
            slot: s
            """;
        var d = mapper.readValue(yaml, AgentDescriptor.class);
        assertThat(d.tenancyId()).isEqualTo("default");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentDescriptorDeserializerTest -DfailIfNoTests=false`
Expected: compilation failure

- [ ] **Step 3: Implement EidosDescriptorModule + AgentDescriptorDeserializer**

Create `EidosDescriptorModule.java` — a Jackson `SimpleModule` with a static factory for
creating a pre-configured ObjectMapper:

```java
package io.casehub.eidos.runtime.yaml;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.VocabularyRegistry;

public class EidosDescriptorModule extends SimpleModule {

    public EidosDescriptorModule(VocabularyRegistry vocabRegistry) {
        addDeserializer(AgentDescriptor.class, new AgentDescriptorDeserializer());
        addDeserializer(AgentDisposition.class, new DispositionDeserializer(vocabRegistry));
    }

    public static ObjectMapper createMapper(VocabularyRegistry vocabRegistry) {
        return new ObjectMapper(new YAMLFactory())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, true)
            .registerModule(new EidosDescriptorModule(vocabRegistry));
    }
}
```

Create `AgentDescriptorDeserializer.java` — reads all fields from the YAML tree, constructs
via builder. `axisVocabularies` converted via `DispositionAxis.valueOf()`. `tenancyId` defaults
to `"default"` when absent. Capabilities, goals, constraints, templates parsed from their
respective JSON array nodes.

The deserializer reads the full tree (`p.readValueAsTree()`), then processes each known field.
For `disposition`, it delegates to Jackson's `ctxt.readTreeAsValue(node, AgentDisposition.class)`
which routes through `DispositionDeserializer`. For `capabilities`, `goals`, `constraints`,
`templates`, it uses `ctxt.readTreeAsValue()` with the appropriate record types — Jackson handles
records with public constructors natively for these simple types.

```java
package io.casehub.eidos.runtime.yaml;

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonDeserializer;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.eidos.api.*;

import java.io.IOException;
import java.util.*;

public class AgentDescriptorDeserializer extends JsonDeserializer<AgentDescriptor> {

    @Override
    public AgentDescriptor deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        ObjectNode root = p.readValueAsTree();
        var builder = AgentDescriptor.builder();

        // Required fields
        builder.agentId(stringField(root, "agentId"));
        builder.name(stringField(root, "name"));
        builder.slot(stringField(root, "slot"));
        builder.tenancyId(root.has("tenancyId") ? root.get("tenancyId").asText() : "default");

        // Optional string fields
        ifString(root, "version", builder::version);
        ifString(root, "provider", builder::provider);
        ifString(root, "modelFamily", builder::modelFamily);
        ifString(root, "modelVersion", builder::modelVersion);
        ifString(root, "weightsFingerprint", builder::weightsFingerprint);
        ifString(root, "domainVocabulary", builder::domainVocabulary);
        ifString(root, "slotVocabulary", builder::slotVocabulary);
        ifString(root, "dispositionVocabulary", builder::dispositionVocabulary);
        ifString(root, "styleVocabulary", builder::styleVocabulary);
        ifString(root, "jurisdiction", builder::jurisdiction);
        ifString(root, "dataHandlingPolicy", builder::dataHandlingPolicy);
        ifString(root, "briefing", builder::briefing);

        // axisVocabularies: Map<String,String> → Map<DispositionAxis,String>
        if (root.has("axisVocabularies")) {
            var axisMap = new LinkedHashMap<DispositionAxis, String>();
            root.get("axisVocabularies").fields().forEachRemaining(e ->
                axisMap.put(DispositionAxis.valueOf(e.getKey()), e.getValue().asText()));
            builder.axisVocabularies(axisMap);
        }

        // Disposition (delegates to DispositionDeserializer via Jackson)
        if (root.has("disposition")) {
            builder.disposition(ctxt.readTreeAsValue(root.get("disposition"), AgentDisposition.class));
        }

        // Capabilities
        if (root.has("capabilities") && root.get("capabilities").isArray()) {
            var caps = new ArrayList<AgentCapability>();
            for (JsonNode capNode : root.get("capabilities")) {
                caps.add(deserializeCapability(capNode));
            }
            builder.capabilities(caps);
        }

        // Goals
        if (root.has("goals") && root.get("goals").isArray()) {
            var goals = new ArrayList<AgentGoal>();
            for (JsonNode goalNode : root.get("goals")) {
                goals.add(deserializeGoal(goalNode));
            }
            builder.goals(goals);
        }

        // Constraints
        if (root.has("constraints") && root.get("constraints").isArray()) {
            var constraints = new ArrayList<AgentConstraint>();
            for (JsonNode cNode : root.get("constraints")) {
                constraints.add(deserializeConstraint(cNode));
            }
            builder.constraints(constraints);
        }

        // Templates
        if (root.has("templates") && root.get("templates").isArray()) {
            var templates = new ArrayList<TemplateRef>();
            for (JsonNode tNode : root.get("templates")) {
                String ref = tNode.get("ref").asText();
                Map<String, String> args = new LinkedHashMap<>();
                if (tNode.has("args")) {
                    tNode.get("args").fields().forEachRemaining(e ->
                        args.put(e.getKey(), e.getValue().asText()));
                }
                templates.add(new TemplateRef(ref, args));
            }
            builder.templates(templates);
        }

        return builder.build();
    }

    private AgentCapability deserializeCapability(JsonNode node) {
        var b = new AgentCapability.Builder().name(node.get("name").asText());
        if (node.has("description")) b.description(node.get("description").asText());
        if (node.has("capabilityVocabulary")) b.capabilityVocabulary(node.get("capabilityVocabulary").asText());
        if (node.has("qualityHint")) b.qualityHint(node.get("qualityHint").asDouble());
        if (node.has("latencyHintP50Ms")) b.latencyHintP50Ms(node.get("latencyHintP50Ms").asLong());
        if (node.has("costHint")) b.costHint(node.get("costHint").asText());
        if (node.has("inputTypes")) b.inputTypes(stringList(node.get("inputTypes")));
        if (node.has("outputTypes")) b.outputTypes(stringList(node.get("outputTypes")));
        if (node.has("tags")) b.tags(stringList(node.get("tags")));
        if (node.has("epistemicDomains")) {
            var map = new LinkedHashMap<String, Double>();
            node.get("epistemicDomains").fields().forEachRemaining(e ->
                map.put(e.getKey(), e.getValue().asDouble()));
            b.epistemicDomains(map);
        }
        if (node.has("excludedDomains")) b.excludedDomains(new LinkedHashSet<>(stringList(node.get("excludedDomains"))));
        return b.build();
    }

    private AgentGoal deserializeGoal(JsonNode node) {
        return new AgentGoal(
            node.get("name").asText(),
            node.get("description").asText(),
            GoalPriority.valueOf(node.get("priority").asText()),
            Visibility.valueOf(node.get("visibility").asText()),
            node.has("capabilities") ? stringList(node.get("capabilities")) : List.of(),
            null);
    }

    private AgentConstraint deserializeConstraint(JsonNode node) {
        return new AgentConstraint(
            node.get("name").asText(),
            node.get("description").asText(),
            Visibility.valueOf(node.get("visibility").asText()),
            ConstraintSeverity.valueOf(node.get("severity").asText()));
    }

    private static String stringField(ObjectNode node, String field) {
        return node.has(field) && !node.get(field).isNull() ? node.get(field).asText() : null;
    }

    private static void ifString(ObjectNode node, String field, java.util.function.Consumer<String> setter) {
        if (node.has(field) && !node.get(field).isNull()) setter.accept(node.get(field).asText());
    }

    private static List<String> stringList(JsonNode arrayNode) {
        var list = new ArrayList<String>();
        for (JsonNode item : arrayNode) list.add(item.asText());
        return list;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentDescriptorDeserializerTest`
Expected: all 6 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/yaml/ runtime/src/test/java/io/casehub/eidos/runtime/yaml/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#147): add AgentDescriptorDeserializer + EidosDescriptorModule Refs #147"
```

---

## Batch 2: Migration (swap to new deserializers, delete config classes)

### Task 3: Migrate ClasspathYamlDescriptorRegistrar

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`
  — delete 7 inner classes + `toDescriptor()`, use `EidosDescriptorModule`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java`
  — tests should pass unchanged (validates backward compatibility)

**Interfaces:**
- Consumes: `EidosDescriptorModule.createMapper(VocabularyRegistry)` (from Task 2)
- Produces: unchanged public API — `loadFrom(InputStream)`, `loadFrom(InputStream, VocabularyRegistry)`, `descriptors()`

- [ ] **Step 1: Run existing tests to confirm they pass before changes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlDescriptorRegistrarTest`
Expected: all 22 tests PASS

- [ ] **Step 2: Rewrite ClasspathYamlDescriptorRegistrar to use EidosDescriptorModule**

Replace the entire body of `ClasspathYamlDescriptorRegistrar`. Keep:
- Class declaration and `@ApplicationScoped`
- `RESOURCE_PATH` constant
- `descriptors()` method (classpath scanning)
- `loadFrom(InputStream)` and `loadFrom(InputStream, VocabularyRegistry)` methods

Delete:
- `YAML_MAPPER` static field
- `toDescriptor()` method
- `DescriptorFile` inner class → replace with simple static inner class
- `DescriptorConfig` and all 6 inner config classes

New body:

```java
package io.casehub.eidos.runtime.registrar;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.api.spi.AgentDescriptorRegistrar;
import io.casehub.eidos.runtime.yaml.EidosDescriptorModule;
import jakarta.enterprise.context.ApplicationScoped;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.List;

@ApplicationScoped
public class ClasspathYamlDescriptorRegistrar implements AgentDescriptorRegistrar {

    private static final String RESOURCE_PATH = "META-INF/eidos/descriptors.yaml";

    @Override
    public List<AgentDescriptor> descriptors() {
        final Enumeration<URL> urls;
        try {
            urls = Thread.currentThread().getContextClassLoader().getResources(RESOURCE_PATH);
        } catch (final IOException e) {
            throw new IllegalStateException("Failed to scan classpath for " + RESOURCE_PATH, e);
        }

        final var all = new ArrayList<AgentDescriptor>();
        while (urls.hasMoreElements()) {
            final var url = urls.nextElement();
            try (final var stream = url.openStream()) {
                all.addAll(loadFrom(stream));
            } catch (final Exception e) {
                throw new IllegalStateException(
                    "Failed to load descriptors from " + url + ": " + e.getMessage(), e);
            }
        }
        return List.copyOf(all);
    }

    public List<AgentDescriptor> loadFrom(final InputStream yaml) {
        return loadFrom(yaml, null);
    }

    public List<AgentDescriptor> loadFrom(final InputStream yaml, final VocabularyRegistry vocabRegistry) {
        if (yaml == null) return List.of();
        try {
            var mapper = EidosDescriptorModule.createMapper(vocabRegistry);
            var file = mapper.readValue(yaml, DescriptorFile.class);
            if (file.descriptors == null || file.descriptors.isEmpty()) return List.of();
            return List.copyOf(file.descriptors);
        } catch (final IOException e) {
            throw new IllegalStateException("Failed to parse YAML: " + e.getMessage(), e);
        }
    }

    static class DescriptorFile {
        public List<AgentDescriptor> descriptors;
    }
}
```

- [ ] **Step 3: Run existing tests to verify backward compatibility**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlDescriptorRegistrarTest`
Expected: all 22 tests PASS

- [ ] **Step 4: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all tests PASS

- [ ] **Step 5: Run full project tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: BUILD SUCCESS — all modules pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(#147): migrate ClasspathYamlDescriptorRegistrar to EidosDescriptorModule — delete 7 config classes Refs #147"
```

---

## References

- [2026-08-29-descriptor-codegen-design.md] — design spec this plan implements
- [ClasspathYamlDescriptorRegistrar.java] — current registrar with config classes (247 lines)
- [ClasspathYamlDescriptorRegistrarTest.java] — 22 existing tests (542 lines)
- [io.casehub.api.model.converter.CaseDefinitionModule] — engine pattern precedent
- [io.casehub.model.marshaller.WorkerMarshaller] — engine custom deserializer precedent
- [decisions.md D2, D5] — eliminate DescriptorConfig, convenience fields in Jackson Module
- [decision-review.md] — tenancyId, axisVocabularies, non-CDI path flags
- [spec-review.md] — DescriptorFile as POJO, static ObjectMapper lifecycle
- [GitHub #147] — focal issue
- [GitHub #146] — annotation parity gaps (deferred)
- [casehubio/parent#432] — shared schema generator (deferred)
