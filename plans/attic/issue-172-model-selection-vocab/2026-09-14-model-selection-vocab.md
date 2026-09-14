# Model Selection Vocabulary Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #172 — feat: LLM model selection via eidos vocabulary — identity-scoped routing
**Issue group:** #172

**Goal:** Add `modelTier` and `modelCapabilities` fields to `AgentCapability`, create a `ModelTierTerm` vocabulary with linear subsumption, and wire rendering/validation/persistence/annotations.

**Architecture:** Two new optional fields on `AgentCapability` declare model requirements per capability. `modelTier` is vocabulary-grounded via `ModelTierTerm` (FLAGSHIP→STANDARD→FAST, EMBEDDING standalone). `modelCapabilities` is an open `Set<String>` of vendor feature flags. Platform reads these from the descriptor at dispatch time — eidos declares, platform resolves.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson, JPA/Hibernate, Jandex (build extension)

## Global Constraints

- No dependency on `platform-api`'s `ModelRegistry` — dependency flows one way (platform reads eidos)
- No `ModelSelector` SPI in eidos
- No cost ceiling or locality fields on descriptors
- `modelTier` vocabulary URI: `urn:casehub:vocab:model-tier`
- All schema changes go directly into base migration files (no existing installations)
- Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

---

## Batch 1: Foundation — API record + vocabulary

### Task 1: Add modelTier and modelCapabilities to AgentCapability

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentCapability.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java`

**Interfaces:**
- Produces: `AgentCapability.modelTier()` (String), `AgentCapability.modelCapabilities()` (Set<String>), `AgentCapability.Builder.modelTier(String)`, `AgentCapability.Builder.modelCapabilities(Set<String>)`

- [ ] **Step 1: Write failing tests for the new fields**

```java
// In AgentCapabilityTest.java — create this file if it doesn't exist
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class AgentCapabilityTest {

    @Test
    void modelTierAndModelCapabilitiesStoredViaBuilder() {
        var cap = AgentCapability.builder()
            .name("code-review")
            .modelTier("flagship")
            .modelCapabilities(Set.of("text", "tool-use"))
            .build();

        assertEquals("flagship", cap.modelTier());
        assertEquals(Set.of("text", "tool-use"), cap.modelCapabilities());
    }

    @Test
    void modelTierNullWhenNotSet() {
        var cap = AgentCapability.builder().name("lint").build();
        assertNull(cap.modelTier());
        assertNull(cap.modelCapabilities());
    }

    @Test
    void modelCapabilitiesDefensivelyCopied() {
        var mutable = new java.util.HashSet<>(Set.of("text"));
        var cap = AgentCapability.builder()
            .name("test")
            .modelCapabilities(mutable)
            .build();
        mutable.add("vision");
        assertFalse(cap.modelCapabilities().contains("vision"));
    }

    @Test
    void modelTierValidatedForLength() {
        var longTier = "x".repeat(201);
        assertThrows(AgentValidationException.class, () ->
            AgentCapability.builder().name("test").modelTier(longTier).build());
    }

    @Test
    void modelCapabilitiesBlankStringRejected() {
        assertThrows(AgentValidationException.class, () ->
            AgentCapability.builder()
                .name("test")
                .modelCapabilities(Set.of("text", ""))
                .build());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentCapabilityTest`
Expected: compilation failure — `modelTier()` and `modelCapabilities()` don't exist yet

- [ ] **Step 3: Add modelTier and modelCapabilities to AgentCapability record**

In `api/src/main/java/io/casehub/eidos/api/AgentCapability.java`, add the two new fields to the record after `costHint`. Update the compact constructor with validation. Add Builder fields and methods.

Record parameter list becomes:
```java
public record AgentCapability(
        String name,
        String description,
        String capabilityVocabulary,
        Double qualityHint,
        Long latencyHintP50Ms,
        String costHint,
        String modelTier,
        Set<String> modelCapabilities,
        List<String> inputTypes,
        List<String> outputTypes,
        List<String> tags,
        Map<String, Double> epistemicDomains,
        Set<String> excludedDomains
) { ... }
```

Add to compact constructor (after `costHint` validation, before `inputTypes` validation):
```java
AgentDescriptorValidator.validateOptional("modelTier", modelTier,
    AgentDescriptorValidator.MAX_CAPABILITY_STRING);
if (modelCapabilities != null) {
    AgentDescriptorValidator.validateItems("modelCapabilities",
        modelCapabilities, AgentDescriptorValidator.MAX_CAPABILITY_STRING);
    modelCapabilities = Set.copyOf(modelCapabilities);
}
```

Add to Builder class:
```java
private String modelTier;
private Set<String> modelCapabilities;

public Builder modelTier(String v)              { this.modelTier = v; return this; }
public Builder modelCapabilities(Set<String> v) { this.modelCapabilities = v; return this; }
```

Update Builder.build() to pass the two new fields in the correct position (after `costHint`, before `inputTypes`).

- [ ] **Step 4: Fix all callers of the canonical constructor**

The record signature change breaks callers that use the positional constructor. Search for `new AgentCapability(` across the project and update each call to include `null, null` (or the appropriate values) for the two new parameters after `costHint`.

Key files to update:
- `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java:79-91` — `toCapability()` method
- Any test files using the canonical constructor

Use `ide_search_text` with query `new AgentCapability(` to find all callers.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentCapabilityTest`
Expected: all 5 tests PASS

- [ ] **Step 6: Verify full API module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#172): add modelTier and modelCapabilities to AgentCapability Refs casehubio/eidos#172"
```

### Task 2: Create ModelTierTerm vocabulary and registrar

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/ModelTierTerm.java`
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/ModelTierVocabRegistrar.java`
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/ModelTierTermTest.java`

**Interfaces:**
- Produces: `ModelTierTerm.FLAGSHIP`, `.STANDARD`, `.FAST`, `.EMBEDDING`, `ModelTierTerm.URI` = `"urn:casehub:vocab:model-tier"`

- [ ] **Step 1: Write failing test for ModelTierTerm**

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyTerm;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ModelTierTermTest {

    @Test
    void flagshipSpecializesStandard() {
        assertEquals(ModelTierTerm.STANDARD, ModelTierTerm.FLAGSHIP.specializes());
    }

    @Test
    void standardSpecializesFast() {
        assertEquals(ModelTierTerm.FAST, ModelTierTerm.STANDARD.specializes());
    }

    @Test
    void fastDoesNotSpecialize() {
        assertNull(ModelTierTerm.FAST.specializes());
    }

    @Test
    void embeddingDoesNotSpecialize() {
        assertNull(ModelTierTerm.EMBEDDING.specializes());
    }

    @Test
    void uriIsCorrect() {
        assertEquals("urn:casehub:vocab:model-tier", ModelTierTerm.URI);
    }

    @Test
    void allTermsHaveValues() {
        for (var term : ModelTierTerm.values()) {
            assertNotNull(term.value());
            assertNotNull(term.label());
            assertNotNull(term.description());
            assertFalse(term.value().isBlank());
        }
    }

    @Test
    void linearChainDepth() {
        VocabularyTerm current = ModelTierTerm.FLAGSHIP;
        int depth = 0;
        while (current.specializes() != null) {
            current = current.specializes();
            depth++;
        }
        assertEquals(2, depth);
        assertEquals(ModelTierTerm.FAST, current);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ModelTierTermTest`
Expected: compilation failure — `ModelTierTerm` doesn't exist

- [ ] **Step 3: Create ModelTierTerm enum**

Create `vocab/src/main/java/io/casehub/eidos/vocab/ModelTierTerm.java`:

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyMetadata;
import io.casehub.eidos.api.VocabularyTerm;

import java.util.List;

@VocabularyMetadata(uri = "urn:casehub:vocab:model-tier",
                    name = "Model Tier", version = "1.0",
                    description = "LLM model capability tiers for identity-scoped routing. Linear subsumption: FLAGSHIP can satisfy any STANDARD requirement. EMBEDDING is a separate modality.")
public enum ModelTierTerm implements VocabularyTerm {

    FLAGSHIP("flagship", "Flagship",
             "Highest-capability models with advanced reasoning, code generation, and complex instruction following"),
    STANDARD("standard", "Standard",
             "Balanced models suitable for most production tasks — good quality at moderate cost"),
    FAST("fast", "Fast",
         "Low-latency models optimized for speed and throughput over reasoning depth"),
    EMBEDDING("embedding", "Embedding",
              "Embedding models for vector representations — different modality, not a compute tier");

    public static final String URI = "urn:casehub:vocab:model-tier";

    private final String value, label, description;

    ModelTierTerm(String value, String label, String description) {
        this.value       = value;
        this.label       = label;
        this.description = description;
    }

    @Override public String value()       { return value; }
    @Override public String label()       { return label; }
    @Override public String description() { return description; }
    @Override public List<String> aliases() { return List.of(); }
    @Override public List<String> exactMatch() { return List.of(); }
    @Override public List<String> axisExactMatch() { return List.of(); }

    @Override
    public VocabularyTerm specializes() {
        return switch (this) {
            case FLAGSHIP  -> STANDARD;
            case STANDARD  -> FAST;
            case FAST      -> null;
            case EMBEDDING -> null;
        };
    }
}
```

- [ ] **Step 4: Create ModelTierVocabRegistrar**

Create `vocab/src/main/java/io/casehub/eidos/vocab/ModelTierVocabRegistrar.java`:

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.spi.VocabularyRegistrar;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ModelTierVocabRegistrar implements VocabularyRegistrar {
    @Override
    public Class<ModelTierTerm> vocabulary() {
        return ModelTierTerm.class;
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ModelTierTermTest`
Expected: all 7 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add vocab/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#172): add ModelTierTerm vocabulary with linear subsumption Refs casehubio/eidos#172"
```

---

## Batch 2: Persistence + Validation

### Task 3: JPA entity, mapper, and schema migration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`

**Interfaces:**
- Consumes: `AgentCapability.modelTier()`, `AgentCapability.modelCapabilities()`
- Produces: JPA round-trip persistence for `modelTier` and `modelCapabilities`

- [ ] **Step 1: Write failing test for JPA round-trip**

Add to `JpaAgentRegistryTest.java`:

```java
@Test
void modelTierAndCapabilitiesRoundTrip() {
    var descriptor = AgentDescriptor.builder()
        .agentId("model-test").name("Model Test").slot("tester")
        .tenancyId("test-tenant").modelFamily("claude")
        .capabilities(List.of(
            AgentCapability.builder()
                .name("code-review")
                .modelTier("flagship")
                .modelCapabilities(Set.of("text", "tool-use"))
                .qualityHint(0.95)
                .build()))
        .build();

    registry.register(descriptor);
    var found = registry.findById("model-test", "test-tenant");

    assertTrue(found.isPresent());
    var cap = found.get().capabilities().getFirst();
    assertEquals("flagship", cap.modelTier());
    assertEquals(Set.of("text", "tool-use"), cap.modelCapabilities());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest#modelTierAndCapabilitiesRoundTrip`
Expected: FAIL — fields not persisted

- [ ] **Step 3: Add columns to AgentCapabilityEntity**

Add to `AgentCapabilityEntity.java` after the `costHint` field:

```java
@Column(name = "model_tier")
String modelTier;

@Column(name = "model_capabilities", columnDefinition = "TEXT")
String modelCapabilities;
```

- [ ] **Step 4: Update AgentDescriptorMapper**

In `toCapability()` — add the two new fields to the constructor call (after `costHint`, before `inputTypes`):
```java
c.modelTier,
readJson(c.modelCapabilities, new TypeReference<Set<String>>() {}),
```

In `toCapabilityEntity()` — add after `e.costHint`:
```java
e.modelTier          = c.modelTier();
e.modelCapabilities  = writeJson(c.modelCapabilities());
```

- [ ] **Step 5: Update V1__initial_schema.sql**

Add to the `agent_capability` table definition, after `cost_hint`:
```sql
model_tier          VARCHAR(100),
model_capabilities  TEXT,
```

- [ ] **Step 6: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest#modelTierAndCapabilitiesRoundTrip`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#172): persist modelTier and modelCapabilities in JPA Refs casehubio/eidos#172"
```

### Task 4: Vocabulary validation for modelTier

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityVocabularyValidator.java`
- Test: `api/src/test/java/io/casehub/eidos/api/CapabilityVocabularyValidatorTest.java` (or existing test file)

**Interfaces:**
- Consumes: `AgentCapability.modelTier()`, `VocabularyRegistry.isRegistered(String)`, `VocabularyRegistry.resolve(String, String)`

- [ ] **Step 1: Write failing test for modelTier validation**

Find or create `CapabilityVocabularyValidatorTest.java`:

```java
@Test
void modelTierValidatedAgainstVocabulary() {
    var registry = createRegistryWithModelTierVocab();
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("Test").slot("s").tenancyId("t")
        .capabilities(List.of(
            AgentCapability.builder()
                .name("review")
                .modelTier("invalid-tier")
                .build()))
        .build();

    assertThrows(AgentValidationException.class, () ->
        CapabilityVocabularyValidator.validate(descriptor, registry));
}

@Test
void validModelTierPassesValidation() {
    var registry = createRegistryWithModelTierVocab();
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("Test").slot("s").tenancyId("t")
        .capabilities(List.of(
            AgentCapability.builder()
                .name("review")
                .modelTier("flagship")
                .build()))
        .build();

    assertDoesNotThrow(() ->
        CapabilityVocabularyValidator.validate(descriptor, registry));
}

@Test
void nullModelTierSkipsValidation() {
    var registry = createRegistryWithModelTierVocab();
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("Test").slot("s").tenancyId("t")
        .capabilities(List.of(
            AgentCapability.builder().name("review").build()))
        .build();

    assertDoesNotThrow(() ->
        CapabilityVocabularyValidator.validate(descriptor, registry));
}

@Test
void modelTierSkippedWhenVocabNotRegistered() {
    var emptyRegistry = createEmptyRegistry();
    var descriptor = AgentDescriptor.builder()
        .agentId("test").name("Test").slot("s").tenancyId("t")
        .capabilities(List.of(
            AgentCapability.builder()
                .name("review")
                .modelTier("flagship")
                .build()))
        .build();

    assertDoesNotThrow(() ->
        CapabilityVocabularyValidator.validate(descriptor, emptyRegistry));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CapabilityVocabularyValidatorTest`
Expected: FAIL — modelTier not validated yet

- [ ] **Step 3: Add modelTier validation to CapabilityVocabularyValidator**

In `CapabilityVocabularyValidator.validate()`, add after the existing `capabilityVocabulary` validation block (still inside the capabilities loop):

```java
if (cap.modelTier() != null) {
    String modelTierUri = "urn:casehub:vocab:model-tier";
    if (vocabularyRegistry.isRegistered(modelTierUri)) {
        if (vocabularyRegistry.resolve(modelTierUri, cap.modelTier()).isEmpty()) {
            throw new AgentValidationException("modelTier",
                "'" + cap.modelTier() + "' is not a valid term in vocabulary '" + modelTierUri + "'");
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CapabilityVocabularyValidatorTest`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#172): validate modelTier against model-tier vocabulary Refs casehubio/eidos#172"
```

---

## Batch 3: Rendering + YAML + Annotations

### Task 5: A2A_CARD rendering and hash coverage

**Files:**
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java`
- Test: existing render pipeline tests

**Interfaces:**
- Consumes: `AgentCapability.modelTier()`, `AgentCapability.modelCapabilities()`

- [ ] **Step 1: Write failing test for A2A_CARD rendering**

Add to the render pipeline test class:

```java
@Test
void a2aCardIncludesModelTierAndCapabilities() {
    var descriptor = descriptorWithCapability(
        AgentCapability.builder()
            .name("code-review")
            .modelTier("flagship")
            .modelCapabilities(Set.of("text", "tool-use"))
            .qualityHint(0.95)
            .build());

    var result = renderer.render(descriptor, contextFor(RenderFormat.A2A_CARD));
    var json = parseJson(result.text());

    var caps = json.get("capabilities");
    assertNotNull(caps);
    var cap = caps.get(0);
    assertEquals("flagship", cap.get("modelTier").asText());
    var modelCaps = new java.util.HashSet<String>();
    cap.get("modelCapabilities").forEach(n -> modelCaps.add(n.asText()));
    assertEquals(Set.of("text", "tool-use"), modelCaps);
}

@Test
void markdownDoesNotIncludeModelTier() {
    var descriptor = descriptorWithCapability(
        AgentCapability.builder()
            .name("code-review")
            .modelTier("flagship")
            .build());

    var result = renderer.render(descriptor, contextFor(RenderFormat.MARKDOWN));
    assertFalse(result.text().contains("flagship"));
    assertFalse(result.text().contains("modelTier"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run the render pipeline tests.
Expected: FAIL — `modelTier` not rendered in A2A_CARD

- [ ] **Step 3: Add modelTier and modelCapabilities to buildDescriptorPayload**

In `EidosRenderPipeline.buildDescriptorPayload()`, inside the `if (format == RenderFormat.A2A_CARD)` block (around line 311-318), add after the `epistemicDomains` block:

```java
if (cap.modelTier() != null) capNode.put("modelTier", cap.modelTier());
if (cap.modelCapabilities() != null && !cap.modelCapabilities().isEmpty()) {
    final ArrayNode mcArr = capNode.putArray("modelCapabilities");
    cap.modelCapabilities().forEach(mcArr::add);
}
```

- [ ] **Step 4: Add to assembleA2aCard**

In `assembleA2aCard()`, inside the capabilities loop (around line 991-1019), add after the `excludedDomains` block:

```java
if (cap.modelTier() != null) { capNode.put("modelTier", cap.modelTier()); }
if (cap.modelCapabilities() != null && !cap.modelCapabilities().isEmpty()) {
    final ArrayNode mcArr = capNode.putArray("modelCapabilities");
    cap.modelCapabilities().forEach(mcArr::add);
}
```

- [ ] **Step 5: Run tests**

Expected: all PASS

- [ ] **Step 6: Verify hash coverage**

Write a test confirming that changing `modelTier` on a descriptor produces a different hash in `buildDescriptorPayload(A2A_CARD)`:

```java
@Test
void modelTierChangeAffectsA2aHash() {
    var cap1 = AgentCapability.builder().name("review").modelTier("flagship").build();
    var cap2 = AgentCapability.builder().name("review").modelTier("standard").build();

    var d1 = descriptorWithCapability(cap1);
    var d2 = descriptorWithCapability(cap2);

    var hash1 = pipeline.buildDescriptorPayload(d1, RenderFormat.A2A_CARD).toString();
    var hash2 = pipeline.buildDescriptorPayload(d2, RenderFormat.A2A_CARD).toString();

    assertNotEquals(hash1, hash2);
}
```

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eidos-core/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#172): render modelTier and modelCapabilities in A2A_CARD Refs casehubio/eidos#172"
```

### Task 6: YAML deserialization support

**Files:**
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/yaml/AgentDescriptorDeserializer.java`
- Test: existing YAML descriptor test or create

**Interfaces:**
- Consumes: `AgentCapability.Builder.modelTier(String)`, `AgentCapability.Builder.modelCapabilities(Set<String>)`

- [ ] **Step 1: Write failing test**

```java
@Test
void yamlDeserializerReadsModelTierAndCapabilities() {
    String yaml = """
        agentId: test-agent
        name: Test Agent
        slot: tester
        tenancyId: test
        capabilities:
          - name: code-review
            modelTier: flagship
            modelCapabilities:
              - text
              - tool-use
        """;

    var descriptor = parseYaml(yaml);
    var cap = descriptor.capabilities().getFirst();
    assertEquals("flagship", cap.modelTier());
    assertEquals(Set.of("text", "tool-use"), cap.modelCapabilities());
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `modelTier` and `modelCapabilities` not read from YAML

- [ ] **Step 3: Add deserialization in AgentDescriptorDeserializer.deserializeCapability()**

In `deserializeCapability()` method (around line 104-122), add after the `excludedDomains` line:

```java
if (node.has("modelTier")) b.modelTier(node.get("modelTier").asText());
if (node.has("modelCapabilities")) b.modelCapabilities(new LinkedHashSet<>(stringList(node.get("modelCapabilities"))));
```

- [ ] **Step 4: Run test**

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eidos-core/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#172): deserialize modelTier and modelCapabilities from YAML Refs casehubio/eidos#172"
```

### Task 7: Annotation extension — @AgentCapabilityDef

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilityDef.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`
- Test: existing annotations processor test

**Interfaces:**
- Consumes: `AgentCapability.Builder.modelTier(String)`, `AgentCapability.Builder.modelCapabilities(Set<String>)`

- [ ] **Step 1: Write failing test**

Add to the annotations deployment test:

```java
@Test
void capabilityDefWithModelTierProcessed() {
    // Test that @AgentCapabilityDef(name = "review", modelTier = "flagship",
    //   modelCapabilities = {"text", "tool-use"}) produces a capability
    //   with the correct modelTier and modelCapabilities
    // Use the existing processor test infrastructure
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — annotation fields don't exist

- [ ] **Step 3: Add fields to @AgentCapabilityDef**

In `AgentCapabilityDef.java`, add after `excludedDomains()`:

```java
String modelTier() default "";
String[] modelCapabilities() default {};
```

- [ ] **Step 4: Update EidosAnnotationsProcessor**

Find where `AgentCapabilityDef` fields are extracted and capability builders are populated. Add extraction for the two new fields:

```java
// After existing capability field extraction
String modelTier = capAnnotation.value("modelTier").asString();
if (!modelTier.isEmpty()) {
    // set on config or builder
}
String[] modelCapabilities = capAnnotation.value("modelCapabilities").asStringArray();
if (modelCapabilities.length > 0) {
    // set on config or builder
}
```

The exact code depends on the processor's extraction pattern — follow the existing pattern for `costHint` and `excludedDomains`.

- [ ] **Step 5: Run test**

Expected: PASS

- [ ] **Step 6: Run full build to verify all modules compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#172): extend @AgentCapabilityDef with modelTier and modelCapabilities Refs casehubio/eidos#172"
```

---

## References

- [2026-09-14-model-selection-vocab-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/eidos/api/AgentCapability.java` — capability record being extended
- `api/src/main/java/io/casehub/eidos/api/CapabilityVocabularyValidator.java` — validation pattern to follow
- `vocab/src/main/java/io/casehub/eidos/vocab/BelbinTerm.java` — vocabulary enum pattern
- `vocab/src/main/java/io/casehub/eidos/vocab/BelbinVocabRegistrar.java` — registrar pattern
- `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java` — JPA entity
- `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` — JPA mapper
- `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql` — base schema
- `eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java:273-329` — buildDescriptorPayload
- `eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java:868-1054` — assembleA2aCard
- `eidos-core/src/main/java/io/casehub/eidos/core/yaml/AgentDescriptorDeserializer.java:104-122` — capability deserialization
- `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilityDef.java` — annotation definition
- `docs/protocols/renderer/capability-metadata-rendering.md` — routing signals A2A_CARD only
- `docs/protocols/renderer/a2a-structural-assembly-hash-coverage.md` — hash coverage rule
- casehubio/eidos#172 — focal issue
- casehubio/platform#285 — upstream ModelRegistry epic
