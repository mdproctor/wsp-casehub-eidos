# Model Selection Schema Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #179 — Integrate model-selection schema from platform for task-level model requirements
**Issue group:** #179

**Goal:** Replace flat `modelTier`/`modelCapabilities` fields on `AgentCapability` with `String modelRef` + `ModelQuery model` from platform-api, enabling the full model-selection schema union type in YAML, annotations, and A2A_CARD.

**Architecture:** Two mutually exclusive fields on `AgentCapability` — `modelRef` for string shorthands (aliases, tier refs, model IDs) and `model` for inline `ModelQuery` constraint queries. This mirrors `AgentSessionConfig` on the platform side. YAML `model:` key dispatches to `modelRef` (string value) or `model` (object value). Annotations keep `modelTier`/`modelCapabilities` attributes for structured input, add `model` for string shorthand — recorder builds the appropriate field.

**Tech Stack:** Java 21, Quarkus 3.32.2, platform-api (`ModelQuery`, `ModelTier`, `ModelLocality`, `CostTier`, `ModelRef`), Jackson, JPA/Hibernate

## Global Constraints

- **Java version:** 21 (on Java 26 JVM)
- **Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- **Test command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- **Platform-api version:** `${project.version}` (0.2-SNAPSHOT, same reactor)
- **No existing installations** — schema changes go directly into V1 migration, no ALTER scripts
- **Pre-release** — breaking changes to `AgentCapability` record are acceptable
- **Use `mvn` not `./mvnw`** — maven wrapper not configured

---

## Batch 1: API Foundation

Everything downstream depends on `AgentCapability` compiling with the new fields.

### Task 1: Add platform-api dependency and update AgentCapability record

**Files:**
- Modify: `api/pom.xml`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentCapability.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java`

**Interfaces:**
- Produces: `AgentCapability.modelRef()` → `String` (nullable), `AgentCapability.model()` → `ModelQuery` (nullable), `AgentCapability.Builder.modelRef(String)`, `AgentCapability.Builder.model(ModelQuery)`

- [ ] **Step 1: Add platform-api dependency to api/pom.xml**

Add inside `<dependencies>`, before the test dependencies:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-api</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Write failing test for mutual exclusion**

In `AgentCapabilityTest.java`, add:

```java
@Test
void modelRefAndModelAreMutuallyExclusive() {
    assertThatThrownBy(() -> AgentCapability.builder()
            .name("test")
            .modelRef("reasoning-heavy")
            .model(ModelQuery.builder().tier(ModelTier.FLAGSHIP).build())
            .build())
        .isInstanceOf(AgentValidationException.class)
        .hasMessageContaining("mutually exclusive");
}

@Test
void modelRefAcceptsStringShorthand() {
    var cap = AgentCapability.builder()
            .name("test")
            .modelRef("reasoning-heavy")
            .build();
    assertThat(cap.modelRef()).isEqualTo("reasoning-heavy");
    assertThat(cap.model()).isNull();
}

@Test
void modelAcceptsModelQuery() {
    var query = ModelQuery.builder()
            .tier(ModelTier.FLAGSHIP)
            .requiredCapabilities(Set.of("text", "tool-use"))
            .build();
    var cap = AgentCapability.builder()
            .name("test")
            .model(query)
            .build();
    assertThat(cap.model()).isEqualTo(query);
    assertThat(cap.modelRef()).isNull();
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentCapabilityTest#modelRefAndModelAreMutuallyExclusive,AgentCapabilityTest#modelRefAcceptsStringShorthand,AgentCapabilityTest#modelAcceptsModelQuery`
Expected: compilation failure — `modelRef` and `model` fields don't exist yet

- [ ] **Step 4: Update AgentCapability record**

Use `ide_replace_member` to replace the record components. Remove `modelTier` and `modelCapabilities` fields, add `modelRef` and `model` in their place:

```java
public record AgentCapability(
        String name,
        String description,
        String capabilityVocabulary,
        Double qualityHint,
        Long latencyHintP50Ms,
        String costHint,
        String modelRef,
        io.casehub.platform.api.model.ModelQuery model,
        List<String> inputTypes,
        List<String> outputTypes,
        List<String> tags,
        Map<String, Double> epistemicDomains,
        Set<String> excludedDomains
) {
```

Update compact constructor — remove `modelTier`/`modelCapabilities` validation, add:

```java
if (modelRef != null && model != null) {
    throw new AgentValidationException("model",
        "modelRef and model are mutually exclusive");
}
AgentDescriptorValidator.validateOptional("modelRef", modelRef,
    AgentDescriptorValidator.MAX_CAPABILITY_STRING);
```

Update Builder — remove `modelTier(String)` and `modelCapabilities(Set<String>)`, add:

```java
private String modelRef;
private io.casehub.platform.api.model.ModelQuery model;

public Builder modelRef(String v) { this.modelRef = v; return this; }
public Builder model(io.casehub.platform.api.model.ModelQuery v) { this.model = v; return this; }
```

Update `Builder.build()` to pass `modelRef, model` instead of `modelTier, modelCapabilities`.

- [ ] **Step 5: Fix existing tests that use modelTier/modelCapabilities**

Update all existing test methods in `AgentCapabilityTest.java` that reference the removed fields. Migrate `.modelTier("flagship")` → `.model(ModelQuery.builder().tier(ModelTier.FLAGSHIP).build())` and `.modelCapabilities(Set.of("text"))` → fold into the same builder call.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/
git commit -m "feat(#179): replace modelTier/modelCapabilities with modelRef/model on AgentCapability

Add casehub-platform-api dependency to eidos-api. AgentCapability now
carries String modelRef (string shorthand) and ModelQuery model (inline
constraints), mutually exclusive. Removes flat modelTier and
modelCapabilities fields.

Refs #179"
```

### Task 2: Update CapabilityVocabularyValidator and AgentDescriptorComparator

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityVocabularyValidator.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java`
- Test: existing tests in `api/src/test/`

**Interfaces:**
- Consumes: `AgentCapability.modelRef()`, `AgentCapability.model()`, `ModelQuery.tier()`, `ModelRef.isTierRef(String)`, `ModelRef.parseTier(String)`

- [ ] **Step 1: Write failing test for tier validation via ModelQuery**

In the appropriate validator test, add a test that creates a capability with `model(ModelQuery.builder().tier(ModelTier.FLAGSHIP).build())` and verifies vocabulary validation passes. Add a second test with an invalid tier string to verify rejection.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: compilation failures — validator still references `cap.modelTier()`

- [ ] **Step 3: Update CapabilityVocabularyValidator**

Replace the `modelTier` validation block with:

```java
if (cap.model() != null && cap.model().tier() != null) {
    String tierValue = cap.model().tier().name().toLowerCase();
    validateAgainstVocabulary(tierValue, MODEL_TIER_URI, registry);
}
if (cap.modelRef() != null && ModelRef.isTierRef(cap.modelRef())) {
    String tierValue = ModelRef.parseTier(cap.modelRef()).name().toLowerCase();
    validateAgainstVocabulary(tierValue, MODEL_TIER_URI, registry);
}
```

Add import: `import io.casehub.platform.api.model.ModelRef;`

- [ ] **Step 4: Update AgentDescriptorComparator**

Replace `modelTier`/`modelCapabilities` comparison with `modelRef`/`model`. For `model` (ModelQuery), compare by tier name string (nullable-safe). For `modelRef`, compare as string.

- [ ] **Step 5: Fix any remaining compilation errors in api module**

Search for any remaining references to `modelTier()` or `modelCapabilities()` in the api module and fix them.

- [ ] **Step 6: Run full api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/
git commit -m "feat(#179): update validator and comparator for modelRef/model fields

CapabilityVocabularyValidator validates tier from both ModelQuery.tier()
and ModelRef tier refs. AgentDescriptorComparator updated for new fields.

Refs #179"
```

---

## Batch 2: Runtime Plumbing

YAML deserialization, JPA persistence, and mapper — all depend on Batch 1's API.

### Task 3: YAML deserializer union type

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializer.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializerTest.java`

**Interfaces:**
- Consumes: `AgentCapability.Builder.modelRef(String)`, `AgentCapability.Builder.model(ModelQuery)`, `ModelQuery.builder()`, `ModelTier`, `ModelLocality`, `CostTier`

- [ ] **Step 1: Write failing tests for YAML union type**

```java
@Test
void deserializesModelStringShorthand() {
    String yaml = """
        agents:
          - agentId: test
            name: Test
            tenancyId: t1
            capabilities:
              - name: review
                model: reasoning-heavy
        """;
    var descriptors = deserialize(yaml);
    var cap = descriptors.get(0).capabilities().get(0);
    assertThat(cap.modelRef()).isEqualTo("reasoning-heavy");
    assertThat(cap.model()).isNull();
}

@Test
void deserializesModelInlineConstraints() {
    String yaml = """
        agents:
          - agentId: test
            name: Test
            tenancyId: t1
            capabilities:
              - name: review
                model:
                  tier: FLAGSHIP
                  capabilities: [text, tool-use]
                  max-cost: HIGH
        """;
    var descriptors = deserialize(yaml);
    var cap = descriptors.get(0).capabilities().get(0);
    assertThat(cap.modelRef()).isNull();
    assertThat(cap.model()).isNotNull();
    assertThat(cap.model().tier()).isEqualTo(ModelTier.FLAGSHIP);
    assertThat(cap.model().requiredCapabilities()).containsExactlyInAnyOrder("text", "tool-use");
    assertThat(cap.model().maxCostTier()).isEqualTo(CostTier.HIGH);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentDescriptorDeserializerTest`
Expected: FAIL — deserializer still parses old `modelTier`/`modelCapabilities` keys

- [ ] **Step 3: Update AgentDescriptorDeserializer**

Replace the `modelTier`/`modelCapabilities` parsing block with:

```java
JsonNode modelNode = node.get("model");
if (modelNode != null) {
    if (modelNode.isTextual()) {
        b.modelRef(modelNode.asText());
    } else if (modelNode.isObject()) {
        var qb = ModelQuery.builder();
        if (modelNode.has("vendor")) qb.vendor(modelNode.get("vendor").asText());
        if (modelNode.has("family")) qb.family(modelNode.get("family").asText());
        if (modelNode.has("tier")) qb.tier(ModelTier.valueOf(modelNode.get("tier").asText()));
        if (modelNode.has("capabilities")) {
            qb.requiredCapabilities(new java.util.LinkedHashSet<>(stringList(modelNode.get("capabilities"))));
        }
        if (modelNode.has("locality")) qb.locality(ModelLocality.valueOf(modelNode.get("locality").asText()));
        if (modelNode.has("max-cost")) qb.maxCostTier(CostTier.valueOf(modelNode.get("max-cost").asText()));
        if (modelNode.has("min-context")) qb.minContextWindow(modelNode.get("min-context").asInt());
        if (modelNode.has("min-output")) qb.minMaxOutput(modelNode.get("min-output").asInt());
        if (modelNode.has("prefer-vendor")) qb.preferVendor(modelNode.get("prefer-vendor").asText());
        b.model(qb.build());
    }
}
```

Add imports: `ModelQuery`, `ModelTier`, `ModelLocality`, `CostTier` from `io.casehub.platform.api.model`.

Remove the old `modelTier`/`modelCapabilities` parsing lines.

- [ ] **Step 4: Fix existing deserializer tests that use modelTier/modelCapabilities YAML**

Update any existing tests that use `modelTier:` or `modelCapabilities:` YAML keys to use the new `model:` format.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentDescriptorDeserializerTest`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/
git commit -m "feat(#179): YAML union type — model: string|object in descriptor deserializer

String values → modelRef, object values → ModelQuery with kebab-case
properties matching model-selection.schema.json.

Refs #179"
```

### Task 4: JPA schema, entity, mapper, and converter

**Files:**
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/ModelQueryConverter.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registry/jpa/JpaAgentRegistryTest.java`

**Interfaces:**
- Consumes: `AgentCapability.modelRef()`, `AgentCapability.model()`, `ModelQuery` (Jackson-serializable record)

- [ ] **Step 1: Write failing test for JPA model round-trip**

In `JpaAgentRegistryTest.java`, add or update a test that registers a descriptor with `model(ModelQuery.builder().tier(ModelTier.FLAGSHIP).requiredCapabilities(Set.of("text")).build())`, retrieves it, and asserts the model query round-trips.

Add a second test for `modelRef("reasoning-heavy")` round-trip.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest`
Expected: compilation failure — entity still has old columns

- [ ] **Step 3: Update V1__initial_schema.sql**

Replace `model_tier VARCHAR(100)` and `model_capabilities TEXT` in the `agent_capability` table definition with:

```sql
model_ref VARCHAR(200),
model JSONB,
CONSTRAINT chk_capability_model_exclusion CHECK (model_ref IS NULL OR model IS NULL),
```

- [ ] **Step 4: Create ModelQueryConverter**

```java
package io.casehub.eidos.runtime.registry.jpa;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.platform.api.model.ModelQuery;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;

@Converter
public class ModelQueryConverter implements AttributeConverter<ModelQuery, String> {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(ModelQuery query) {
        if (query == null) return null;
        try {
            return MAPPER.writeValueAsString(query);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialize ModelQuery", e);
        }
    }

    @Override
    public ModelQuery convertToEntityAttribute(String json) {
        if (json == null || json.isBlank()) return null;
        try {
            return MAPPER.readValue(json, ModelQuery.class);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to deserialize ModelQuery", e);
        }
    }
}
```

- [ ] **Step 5: Update AgentCapabilityEntity**

Replace `modelTier` and `modelCapabilities` fields with:

```java
@Column(name = "model_ref", length = 200)
private String modelRef;

@Column(name = "model", columnDefinition = "JSONB")
@Convert(converter = ModelQueryConverter.class)
private ModelQuery model;
```

Update getters/setters accordingly.

- [ ] **Step 6: Update AgentDescriptorMapper**

Replace `modelTier`/`modelCapabilities` mapping in both `toEntity()` and `toRecord()` directions:

```java
// toEntity:
entity.setModelRef(cap.modelRef());
entity.setModel(cap.model());

// toRecord (Builder):
.modelRef(entity.getModelRef())
.model(entity.getModel())
```

- [ ] **Step 7: Fix existing JPA tests**

Update any tests referencing `modelTier`/`modelCapabilities` on entities or records.

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git add runtime/
git commit -m "feat(#179): JPA schema + ModelQueryConverter for modelRef/model persistence

V1 schema replaces model_tier/model_capabilities with model_ref/model
JSONB. ModelQueryConverter handles Jackson serialization. DB-level CHECK
constraint enforces mutual exclusion.

Refs #179"
```

---

## Batch 3: Rendering + Annotations

### Task 5: A2A_CARD rendering

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java` (or `EidosRenderPipeline.java` if that's the actual class name)
- Test: renderer tests

**Interfaces:**
- Consumes: `AgentCapability.modelRef()`, `AgentCapability.model()`, `ModelQuery.tier()`, `ModelQuery.requiredCapabilities()`, `ModelQuery.maxCostTier()`, etc.

- [ ] **Step 1: Write failing test for A2A_CARD model rendering**

Add tests for both forms:
- `modelRef("reasoning-heavy")` → renders as `"model": "reasoning-heavy"` in the capability JSON
- `model(ModelQuery.builder().tier(ModelTier.FLAGSHIP).requiredCapabilities(Set.of("text")).build())` → renders as `"model": {"tier": "FLAGSHIP", "capabilities": ["text"]}` in the capability JSON

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime` (or the module containing the renderer)
Expected: FAIL — renderer still outputs `modelTier`/`modelCapabilities`

- [ ] **Step 3: Update renderer**

Replace the `modelTier`/`modelCapabilities` rendering block in the A2A_CARD assembly with:

```java
// Model selection — string ref or inline query
if (cap.modelRef() != null) {
    capNode.put("model", cap.modelRef());
} else if (cap.model() != null) {
    ObjectNode modelNode = capNode.putObject("model");
    var q = cap.model();
    if (q.tier() != null) modelNode.put("tier", q.tier().name());
    if (!q.requiredCapabilities().isEmpty()) {
        ArrayNode caps = modelNode.putArray("capabilities");
        q.requiredCapabilities().forEach(caps::add);
    }
    if (q.locality() != null) modelNode.put("locality", q.locality().name());
    if (q.maxCostTier() != null) modelNode.put("max-cost", q.maxCostTier().name());
    if (q.minContextWindow() != null) modelNode.put("min-context", q.minContextWindow());
    if (q.minMaxOutput() != null) modelNode.put("min-output", q.minMaxOutput());
    if (q.vendor() != null) modelNode.put("vendor", q.vendor());
    if (q.family() != null) modelNode.put("family", q.family());
    if (q.preferVendor() != null) modelNode.put("prefer-vendor", q.preferVendor());
}
```

Update the A2A structural assembly hash to include the new `model` field instead of `modelTier`/`modelCapabilities`.

- [ ] **Step 4: Verify MARKDOWN/PROSE rendering**

Confirm `modelRef`/`model` are suppressed in MARKDOWN and PROSE formats (routing signals for machine consumption only). Remove any `modelTier`/`modelCapabilities` rendering in those paths.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/
git commit -m "feat(#179): A2A_CARD renders unified model field (string or constraints object)

modelRef renders as JSON string, ModelQuery renders as JSON object with
kebab-case keys. MARKDOWN/PROSE suppress model (routing signal only).
Hash payload updated.

Refs #179"
```

### Task 6: Annotation support

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilityDef.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedAgentConfig.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`
- Test: annotation integration tests

**Interfaces:**
- Consumes: `AgentCapability.Builder.modelRef(String)`, `AgentCapability.Builder.model(ModelQuery)`, `ModelQuery.builder()`, `ModelTier`
- Produces: `@AgentCapabilityDef(model = "...")` and `@AgentCapabilityDef(modelTier = "...", modelCapabilities = {...})`

- [ ] **Step 1: Write failing test**

Create a test class with two annotated agents — one using `model = "reasoning-heavy"` and one using `modelTier = "FLAGSHIP"`, `modelCapabilities = {"text"}`. Assert the resulting descriptors carry `modelRef` and `model` respectively.

Add a negative test: both `model` and `modelTier` set → build-time error.

- [ ] **Step 2: Run test to verify it fails**

Expected: compilation failure — `model` attribute doesn't exist on `@AgentCapabilityDef`

- [ ] **Step 3: Update @AgentCapabilityDef**

Add the `model` attribute:

```java
String model() default "";
```

Keep `modelTier()` and `modelCapabilities()` as-is.

- [ ] **Step 4: Update AnnotatedAgentConfig**

Add field:

```java
public String modelRef;
```

- [ ] **Step 5: Update EidosAnnotationsProcessor**

Add `model` extraction:

```java
cap.modelRef = stringValue(ann, "model");
```

Add mutual exclusion validation:

```java
if (notEmpty(cap.modelRef) && (notEmpty(cap.modelTier) || cap.modelCapabilities.length > 0)) {
    throw new IllegalArgumentException("@AgentCapabilityDef: 'model' and "
        + "'modelTier'/'modelCapabilities' are mutually exclusive on capability '"
        + cap.name + "'");
}
```

- [ ] **Step 6: Update EidosAnnotationsRecorder**

Replace the `modelTier`/`modelCapabilities` → builder code with:

```java
if (notEmpty(cap.modelRef)) {
    cb.modelRef(cap.modelRef);
} else if (notEmpty(cap.modelTier)) {
    var qb = ModelQuery.builder();
    qb.tier(ModelTier.valueOf(cap.modelTier.toUpperCase()));
    if (cap.modelCapabilities != null && cap.modelCapabilities.length > 0) {
        qb.requiredCapabilities(Set.of(cap.modelCapabilities));
    }
    cb.model(qb.build());
}
```

Add imports for `ModelQuery`, `ModelTier`.

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations/runtime,annotations/deployment`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add annotations/
git commit -m "feat(#179): @AgentCapabilityDef gains model attribute for string shorthand

model and modelTier/modelCapabilities are mutually exclusive (build-time
error). Recorder maps model → modelRef, modelTier → ModelQuery.

Refs #179"
```

---

## Batch 4: Consumer Updates

Mechanical updates to examples and eval profiles. No new logic.

### Task 7: Update example YAML descriptors and tests

**Files:**
- Modify: `examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml`
- Modify: `examples/model-selection-live/src/test/resources/META-INF/eidos/descriptors.yaml` (if exists)
- Modify: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/ModelSelectionScenarioTest.java`
- Modify: `examples/model-selection-live/src/test/java/io/casehub/eidos/examples/live/ModelSelectionLiveTest.java`

**Interfaces:**
- Consumes: `AgentCapability.modelRef()`, `AgentCapability.model()`, `ModelQuery`

- [ ] **Step 1: Update YAML descriptors**

Replace all `modelTier:` / `modelCapabilities:` entries with `model:` using the appropriate form:

```yaml
# Before:
capabilities:
  - name: code-review
    modelTier: flagship
    modelCapabilities: [text, tool-use]

# After:
capabilities:
  - name: code-review
    model:
      tier: FLAGSHIP
      capabilities: [text, tool-use]
```

For capabilities that only had `modelTier:`, use the simple form:

```yaml
# Before:
capabilities:
  - name: lint-check
    modelTier: fast

# After:
capabilities:
  - name: lint-check
    model:
      tier: FAST
```

- [ ] **Step 2: Update ModelSelectionScenarioTest**

Replace all `cap.modelTier()` → `cap.model().tier()` and `cap.modelCapabilities()` → `cap.model().requiredCapabilities()` assertions. Update any builder calls.

- [ ] **Step 3: Update ModelSelectionLiveTest**

Same mechanical migration as Step 2.

- [ ] **Step 4: Search for any remaining modelTier/modelCapabilities references**

Use `ide_search_text` to find any remaining references to `modelTier` or `modelCapabilities` across the entire project. Fix any stragglers.

- [ ] **Step 5: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and test

- [ ] **Step 6: Commit**

```bash
git add examples/
git commit -m "feat(#179): update example YAML and tests for model: union type

All modelTier/modelCapabilities references migrated to model: form.

Refs #179"
```

### Task 8: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update AgentCapability documentation**

Replace all `modelTier` / `modelCapabilities` references in the CLAUDE.md project guide with the new `modelRef` / `model` fields. Update the `AgentCapability` description, the "Two-layer capability model" section, and any YAML examples.

- [ ] **Step 2: Add platform-api to eidos-api dependency note**

Update the description of `casehub-eidos-api` to note the platform-api dependency.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#179): update CLAUDE.md for modelRef/model on AgentCapability

Refs #179"
```

---

## References

- `specs/issue-179-model-selection-schema/2026-09-16-model-selection-schema-design.md` — design spec this plan implements
- `specs/issue-179-model-selection-schema/decisions.md` — D1-D4 decisions
- `docs/specs/issue-172-model-selection-vocab/2026-09-14-model-selection-vocab-design.md` — prior design (#172)
- `api/src/main/java/io/casehub/eidos/api/AgentCapability.java` — record being modified
- `api/src/main/java/io/casehub/eidos/api/CapabilityVocabularyValidator.java` — tier validation
- `runtime/src/main/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializer.java` — YAML parsing
- `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java` — JPA entity
- `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql` — SQL schema
- `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilityDef.java` — annotation
- `agent-config-core/src/main/resources/schema/model-selection.schema.json` — upstream schema (casehubio/platform)
- `platform-api/.../ModelQuery.java` — upstream model query type
- `docs/protocols/renderer/capability-metadata-rendering.md` — A2A_CARD routing signal protocol
- GitHub #179 — focal issue
