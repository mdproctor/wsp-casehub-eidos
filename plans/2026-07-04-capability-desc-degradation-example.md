# AgentCapability description + Degradation/Recovery Example

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `String description` to `AgentCapability` (#77) and create a degradation/recovery example (#79).

**Architecture:** `description` is a nullable human-readable field threaded through record → builder → entity → mapper → YAML config → renderer. Degradation example is a new `@QuarkusTest` exercising `AgentStateStore` and `CapabilityHealth` via SPI injection.

**Tech Stack:** Java 21, Quarkus 3.32.2, Flyway, JPA, AssertJ

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use `mvn` not `./mvnw`
- No existing installations — schema changes go directly into base migrations
- All commits reference an issue: `Refs #77` or `Refs #79`
- Protocol PP-20260611-228599: numeric routing signals (qualityHint, latencyHintP50Ms, costHint, epistemicDomains) render in A2A_CARD only; human-readable text renders in all formats
- Protocol PP-20260613-608684: fields rendered by `assembleA2aCard()` via direct descriptor access must appear in `buildDescriptorPayload()` A2A_CARD output

---

### Task 1: Add `description` field to `AgentCapability` record, builder, and validator

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentCapability.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java`

**Interfaces:**
- Consumes: nothing (foundational change)
- Produces: `AgentCapability.description()` — nullable `String`, validated ≤ 500 chars; `AgentCapability.Builder.description(String)` method; `AgentDescriptorValidator.MAX_DESCRIPTION = 500`

- [ ] **Step 1: Write failing tests for description validation and builder round-trip**

Add to `AgentCapabilityTest.java`:

```java
// ── description (optional) ────────────────────────────────────────────────

@Test
void description_null_is_allowed() {
    assertThatNoException().isThrownBy(() ->
        AgentCapability.builder().name("review").build());
}

@Test
void description_blank_throws() {
    assertThatThrownBy(() ->
        AgentCapability.builder().name("review").description("  ").build())
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("description"));
}

@Test
void description_exceeds_max_throws() {
    assertThatThrownBy(() ->
        AgentCapability.builder().name("review").description("d".repeat(501)).build())
        .isInstanceOf(AgentValidationException.class)
        .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName())
            .isEqualTo("description"));
}

@Test
void description_at_exactly_max_is_valid() {
    assertThatNoException().isThrownBy(() ->
        AgentCapability.builder().name("review").description("d".repeat(500)).build());
}

@Test
void description_with_bidi_control_throws() {
    assertThatThrownBy(() ->
        AgentCapability.builder().name("review").description("desc‮text").build())
        .isInstanceOf(AgentValidationException.class);
}
```

Update the existing `builder_round_trips_all_fields` test to include `description`:

```java
@Test
void builder_round_trips_all_fields() {
    var cap = AgentCapability.builder()
        .name("code-review")
        .description("Reviews code for quality and correctness")
        .qualityHint(0.9)
        .latencyHintP50Ms(100L)
        .costHint("low")
        .inputTypes(List.of("pull-request"))
        .outputTypes(List.of("review-comment"))
        .tags(List.of("java"))
        .epistemicDomains(Map.of("java", 0.95))
        .excludedDomains(java.util.Set.of("rust"))
        .build();
    assertThat(cap.name()).isEqualTo("code-review");
    assertThat(cap.description()).isEqualTo("Reviews code for quality and correctness");
    assertThat(cap.qualityHint()).isEqualTo(0.9);
    assertThat(cap.latencyHintP50Ms()).isEqualTo(100L);
    assertThat(cap.costHint()).isEqualTo("low");
    assertThat(cap.inputTypes()).containsExactly("pull-request");
    assertThat(cap.outputTypes()).containsExactly("review-comment");
    assertThat(cap.tags()).containsExactly("java");
    assertThat(cap.epistemicDomains()).containsEntry("java", 0.95);
    assertThat(cap.excludedDomains()).containsExactly("rust");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentCapabilityTest`
Expected: Compilation failure — `description()` method and `MAX_DESCRIPTION` constant do not exist.

- [ ] **Step 3: Add `MAX_DESCRIPTION` constant to `AgentDescriptorValidator`**

In `AgentDescriptorValidator.java`, add after line 24 (`MAX_BRIEFING`):

```java
static final int MAX_DESCRIPTION        = 500;
```

- [ ] **Step 4: Add `description` as second record component to `AgentCapability`**

Replace the record declaration and compact constructor in `AgentCapability.java`:

```java
public record AgentCapability(
        String name,
        String description,
        String capabilityVocabulary,
        Double qualityHint,
        Long latencyHintP50Ms,
        String costHint,
        List<String> inputTypes,
        List<String> outputTypes,
        List<String> tags,
        Map<String, Double> epistemicDomains,
        Set<String> excludedDomains
) {
    public AgentCapability {
        AgentDescriptorValidator.validateRequired("capability.name", name,
            AgentDescriptorValidator.MAX_CAPABILITY_NAME);
        AgentDescriptorValidator.validateOptional("description", description,
            AgentDescriptorValidator.MAX_DESCRIPTION);
        AgentDescriptorValidator.validateOptional("capabilityVocabulary", capabilityVocabulary,
            AgentDescriptorValidator.MAX_VOCABULARY_URI);
        // ... rest unchanged
```

Add `description` to the Builder:

```java
public static final class Builder {
    private String name;
    private String description;
    // ... rest unchanged

    public Builder description(String v)             { this.description = v; return this; }
    // ... rest unchanged

    public AgentCapability build() {
        return new AgentCapability(name, description, capabilityVocabulary, qualityHint,
            latencyHintP50Ms, costHint, inputTypes, outputTypes, tags,
            epistemicDomains, excludedDomains);
    }
}
```

- [ ] **Step 5: Fix the 3 direct constructor call sites**

**`AgentDescriptorMapper.toCapability()` (runtime module, line 59):**
```java
private AgentCapability toCapability(AgentCapabilityEntity c) {
    return new AgentCapability(
        c.name,
        c.description,
        c.capabilityVocabulary,
        // ... rest unchanged
```

**`ClasspathYamlDescriptorRegistrar.toDescriptor()` (runtime module, line 97):**
```java
new AgentCapability(c.name, c.description, c.capabilityVocabulary, c.qualityHint,
    c.latencyHintP50Ms, c.costHint, c.inputTypes, c.outputTypes, c.tags,
    c.epistemicDomains, c.excludedDomains)
```

**`AgentCapability.Builder.build()` (already updated in Step 4).**

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: All tests pass. (Runtime module won't compile yet — entity and YAML config don't have `description` field. That's fine, we're testing api module only.)

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/AgentCapability.java \
        api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java \
        api/src/test/java/io/casehub/eidos/api/AgentCapabilityTest.java
git commit -m "feat(eidos#77): add description field to AgentCapability record, builder, and validator

Refs #77"
```

---

### Task 2: Persistence — entity, mapper, migration, and YAML registrar

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`
- Create: `runtime/src/main/resources/db/eidos/migration/V6__capability_description.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`

**Interfaces:**
- Consumes: `AgentCapability.description()` from Task 1
- Produces: `description` persisted through JPA entity and mapped in both directions; `description` parsed from YAML `descriptors.yaml` files

- [ ] **Step 1: Add `description` to `AgentCapabilityEntity`**

In `AgentCapabilityEntity.java`, after the `name` field (line 29):

```java
@Column(nullable = false)
String name;

@Column(columnDefinition = "TEXT") String description;
```

- [ ] **Step 2: Add `description` to mapper — both directions**

In `AgentDescriptorMapper.java`:

`toCapability()` (line 59) — add `c.description` as second argument:
```java
private AgentCapability toCapability(AgentCapabilityEntity c) {
    return new AgentCapability(
        c.name,
        c.description,
        c.capabilityVocabulary,
        c.qualityHint,
        c.latencyHintP50Ms,
        c.costHint,
        readJson(c.inputTypes, new TypeReference<List<String>>() {}),
        readJson(c.outputTypes, new TypeReference<List<String>>() {}),
        readJson(c.tags, new TypeReference<List<String>>() {}),
        readJson(c.epistemicDomains, new TypeReference<Map<String, Double>>() {}),
        c.excludedDomains
    );
}
```

`toCapabilityEntity()` (line 73) — add `e.description = c.description();` after `e.name`:
```java
private AgentCapabilityEntity toCapabilityEntity(AgentCapability c, AgentDescriptorEntity parent) {
    var e = new AgentCapabilityEntity();
    e.descriptor = parent;
    e.agentId    = parent.agentId;
    e.tenancyId  = parent.tenancyId;
    e.name = c.name();
    e.description = c.description();
    e.capabilityVocabulary = c.capabilityVocabulary();
    // ... rest unchanged
```

- [ ] **Step 3: Create Flyway migration V6**

Create `runtime/src/main/resources/db/eidos/migration/V6__capability_description.sql`:

```sql
ALTER TABLE agent_capability ADD COLUMN description TEXT;
```

- [ ] **Step 4: Add `description` to YAML `CapabilityConfig` and `toDescriptor`**

In `ClasspathYamlDescriptorRegistrar.java`:

Add to `CapabilityConfig` (line 124):
```java
static class CapabilityConfig {
    public String name;
    public String description;
    public String capabilityVocabulary;
    // ... rest unchanged
}
```

The constructor call in `toDescriptor` (line 97) was already updated in Task 1 Step 5 to include `c.description`.

- [ ] **Step 5: Run full build to verify compilation and persistence**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All runtime tests pass. The Flyway migration runs against H2 in test profile.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java \
        runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java \
        runtime/src/main/resources/db/eidos/migration/V6__capability_description.sql \
        runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java
git commit -m "feat(eidos#77): persist description through entity, mapper, migration V6, and YAML registrar

Refs #77"
```

---

### Task 3: Renderer — description in all three format paths

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

**Interfaces:**
- Consumes: `AgentCapability.description()` from Task 1; `A2AEnrichment.CapabilityNarrative` (existing — enriched description wins when non-blank)
- Produces: `description` rendered in MARKDOWN (` — desc`), PROSE (`name (desc)`), A2A_CARD (declared fallback when enrichment absent/blank); `description` included in `buildDescriptorPayload` for all formats (hash coverage)

- [ ] **Step 1: Write failing tests for description rendering**

Add to `EidosRenderPipelineTest.java`:

```java
// ── capability description rendering ──────────────────────────────────

@Test
void descriptor_payload_includes_description_for_all_formats() {
    var desc = AgentDescriptor.builder()
        .agentId("a1").name("Agent").version("1.0").provider("p")
        .modelFamily("m").modelVersion("v").slot("s")
        .capabilities(List.of(AgentCapability.builder()
            .name("review").description("Reviews code for quality").build()))
        .disposition(AgentDisposition.builder()
            .socialOrient("independent").ruleFollowing("strict")
            .riskAppetite("conservative").autonomy("directed").build())
        .tenancyId("default").build();

    for (var format : new RenderFormat[]{MARKDOWN, PROSE, A2A_CARD}) {
        var payload = pipeline.buildDescriptorPayload(desc, format);
        var cap = payload.get("capabilities").get(0);
        assertThat(cap.has("description")).as("description present in " + format).isTrue();
        assertThat(cap.get("description").asText()).isEqualTo("Reviews code for quality");
    }
}

@Test
void descriptor_payload_omits_description_when_null() {
    var desc = fullDescriptor();
    var payload = pipeline.buildDescriptorPayload(desc, MARKDOWN);
    var cap = payload.get("capabilities").get(0);
    assertThat(cap.has("description")).isFalse();
}

@Test
void structural_markdown_includes_description_after_name() {
    var desc = AgentDescriptor.builder()
        .agentId("a1").name("Agent").version("1.0").provider("p")
        .modelFamily("m").modelVersion("v").slot("reviewer")
        .capabilities(List.of(AgentCapability.builder()
            .name("code-review").description("Reviews code for quality")
            .inputTypes(List.of("code")).outputTypes(List.of("review")).build()))
        .disposition(AgentDisposition.builder()
            .socialOrient("independent").ruleFollowing("strict")
            .riskAppetite("conservative").autonomy("directed").build())
        .tenancyId("default").build();
    var ctx = AgentPromptContext.forFormat(MARKDOWN);
    var result = pipeline.assemble(
        pipeline.buildStage1(desc, ctx), Optional.empty(), Optional.empty(), desc, ctx);
    assertThat(result.content()).contains("**code-review** — Reviews code for quality");
}

@Test
void structural_prose_includes_description_in_parentheses() {
    var desc = AgentDescriptor.builder()
        .agentId("a1").name("Agent").version("1.0").provider("p")
        .modelFamily("m").modelVersion("v").slot("reviewer")
        .capabilities(List.of(
            AgentCapability.builder().name("code-review")
                .description("Reviews code for quality").build(),
            AgentCapability.builder().name("test-writing").build()))
        .disposition(AgentDisposition.builder()
            .socialOrient("independent").ruleFollowing("strict")
            .riskAppetite("conservative").autonomy("directed").build())
        .tenancyId("default").build();
    var ctx = AgentPromptContext.forFormat(PROSE);
    var result = pipeline.assemble(
        pipeline.buildStage1(desc, ctx), Optional.empty(), Optional.empty(), desc, ctx);
    assertThat(result.content()).contains("code-review (Reviews code for quality)");
    assertThat(result.content()).contains("test-writing");
    assertThat(result.content()).doesNotContain("test-writing (");
}

@Test
void a2a_card_uses_declared_description_when_no_enrichment() {
    var desc = AgentDescriptor.builder()
        .agentId("a1").name("Agent").version("1.0").provider("p")
        .modelFamily("m").modelVersion("v").slot("reviewer")
        .capabilities(List.of(AgentCapability.builder()
            .name("review").description("Declared description").build()))
        .disposition(AgentDisposition.builder()
            .socialOrient("independent").ruleFollowing("strict")
            .riskAppetite("conservative").autonomy("directed").build())
        .tenancyId("default").build();
    var card = renderA2aCard(desc);
    assertThat(card.at("/capabilities/0/description").asText()).isEqualTo("Declared description");
}

@Test
void a2a_card_enriched_description_wins_over_declared() {
    var desc = AgentDescriptor.builder()
        .agentId("a1").name("Agent").version("1.0").provider("p")
        .modelFamily("m").modelVersion("v").slot("reviewer")
        .capabilities(List.of(AgentCapability.builder()
            .name("review").description("Declared description").build()))
        .disposition(AgentDisposition.builder()
            .socialOrient("independent").ruleFollowing("strict")
            .riskAppetite("conservative").autonomy("directed").build())
        .tenancyId("default").build();
    var enrichment = Optional.of(new A2AEnrichment(
        List.of(new A2AEnrichment.CapabilityNarrative("review", "Enriched description"))));
    var ctx = AgentPromptContext.forFormat(A2A_CARD);
    var result = pipeline.assemble(
        pipeline.buildStage1(desc, ctx), Optional.empty(), enrichment, desc, ctx);
    var card = MAPPER.readTree(result.content());
    assertThat(card.at("/capabilities/0/description").asText()).isEqualTo("Enriched description");
}

@Test
void a2a_card_blank_enriched_description_falls_through_to_declared() {
    var desc = AgentDescriptor.builder()
        .agentId("a1").name("Agent").version("1.0").provider("p")
        .modelFamily("m").modelVersion("v").slot("reviewer")
        .capabilities(List.of(AgentCapability.builder()
            .name("review").description("Declared description").build()))
        .disposition(AgentDisposition.builder()
            .socialOrient("independent").ruleFollowing("strict")
            .riskAppetite("conservative").autonomy("directed").build())
        .tenancyId("default").build();
    var enrichment = Optional.of(new A2AEnrichment(
        List.of(new A2AEnrichment.CapabilityNarrative("review", ""))));
    var ctx = AgentPromptContext.forFormat(A2A_CARD);
    var result = pipeline.assemble(
        pipeline.buildStage1(desc, ctx), Optional.empty(), enrichment, desc, ctx);
    var card = MAPPER.readTree(result.content());
    assertThat(card.at("/capabilities/0/description").asText()).isEqualTo("Declared description");
}
```

Note: the `a2a_card_enriched_description_wins_over_declared` and `a2a_card_blank_enriched_description_falls_through_to_declared` tests will need a `throws Exception` on the method signature for `MAPPER.readTree()`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest`
Expected: Compilation or assertion failures — description not yet in payload or structural output.

- [ ] **Step 3: Add `description` to `buildDescriptorPayload` capability node**

In `EidosRenderPipeline.java`, in the capability loop inside `buildDescriptorPayload()` (around line 188), after `capNode.put("name", cap.name());`:

```java
capNode.put("name", cap.name());
if (cap.description() != null) capNode.put("description", cap.description());
```

This is NOT gated behind `format == A2A_CARD` — description is human-readable text, included in all formats per protocol PP-20260611-228599.

- [ ] **Step 4: Add description to `assembleMarkdownCapabilities`**

In `EidosRenderPipeline.java`, method `assembleMarkdownCapabilities()` (around line 360):

```java
for (final AgentCapability cap : descriptor.capabilities()) {
    sb.append("- **").append(cap.name()).append("**");
    if (cap.description() != null)
        sb.append(" — ").append(cap.description());
    if (cap.inputTypes() != null && !cap.inputTypes().isEmpty())
        sb.append(": accepts ").append(String.join(", ", cap.inputTypes()));
    if (cap.outputTypes() != null && !cap.outputTypes().isEmpty())
        sb.append(" → ").append(String.join(", ", cap.outputTypes()));
    sb.append("\n");
}
```

- [ ] **Step 5: Add description to `assembleProse`**

In `EidosRenderPipeline.java`, method `assembleProse()` (around line 480-486). Replace the capability rendering block:

```java
if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
    sb.append("\nCapabilities: ");
    final var parts = descriptor.capabilities().stream()
            .map(cap -> cap.description() != null
                ? cap.name() + " (" + cap.description() + ")"
                : cap.name())
            .collect(Collectors.joining(", "));
    sb.append(parts).append(".\n");
}
```

- [ ] **Step 6: Update `assembleA2aCard` description fallback**

In `EidosRenderPipeline.java`, method `assembleA2aCard()` (around line 639). Replace the enriched description block:

```java
final String enrichedDesc = descriptionByName.get(cap.name());
if (enrichedDesc != null && !enrichedDesc.isBlank()) {
    capNode.put("description", enrichedDesc);
} else if (cap.description() != null) {
    capNode.put("description", cap.description());
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest`
Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
        runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java
git commit -m "feat(eidos#77): render capability description in MARKDOWN, PROSE, and A2A_CARD

Declared description renders in all formats. A2A_CARD: enriched wins when
non-blank, falls through to declared otherwise.

Refs #77"
```

---

### Task 4: Degradation and recovery example (#79)

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DegradationAndRecoveryTest.java`

**Interfaces:**
- Consumes: `AgentStateStore` SPI (api module), `CapabilityHealth` SPI (api module), `AgentRegistry` SPI (api module), `CapabilityHealth.ProbeContext`, `CapabilityHealth.CapabilityStatus.Degraded`, `CapabilityHealth.CapabilityStatus.Ready`, `DegradationReason`
- Produces: runnable example test demonstrating degradation lifecycle

- [ ] **Step 1: Write the example test**

Create `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DegradationAndRecoveryTest.java`:

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DegradationAndRecoveryTest {

    @Inject AgentRegistry registry;
    @Inject AgentStateStore stateStore;
    @Inject CapabilityHealth health;

    static final String TENANCY = "default";
    AgentDescriptor descriptor;
    ProbeContext ctx;

    @BeforeEach
    void setUp() {
        stateStore.clear("worker-1", TENANCY);
        registry.register(AgentDescriptor.builder()
            .agentId("worker-1")
            .name("Data Worker")
            .version("1.0")
            .provider("anthropic")
            .modelFamily("claude")
            .modelVersion("claude-3-7-sonnet")
            .slot("executor")
            .capabilities(List.of(AgentCapability.builder()
                .name("data-processing")
                .description("Processes and transforms structured data")
                .qualityHint(0.85)
                .latencyHintP50Ms(300L)
                .costHint("medium")
                .build()))
            .disposition(AgentDisposition.builder()
                .socialOrient("collaborative")
                .ruleFollowing("principled")
                .riskAppetite("measured")
                .autonomy("semi-autonomous")
                .build())
            .tenancyId(TENANCY)
            .build());
        descriptor = registry.findById("worker-1", TENANCY).orElseThrow();
        ctx = ProbeContext.of(null);
    }

    @Test
    void probe_returns_ready_when_no_degradation() {
        var status = health.probe(descriptor, "data-processing", ctx);
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void probe_returns_degraded_after_recording() {
        stateStore.record("worker-1", TENANCY, DegradationReason.RATE_LIMITED,
            Instant.now().plusSeconds(60));
        var status = health.probe(descriptor, "data-processing", ctx);
        assertThat(status).isInstanceOf(CapabilityStatus.Degraded.class);
        assertThat(((CapabilityStatus.Degraded) status).reason())
            .isEqualTo(DegradationReason.RATE_LIMITED);
    }

    @Test
    void probe_returns_ready_after_clear() {
        stateStore.record("worker-1", TENANCY, DegradationReason.RATE_LIMITED,
            Instant.now().plusSeconds(60));
        assertThat(health.probe(descriptor, "data-processing", ctx))
            .isInstanceOf(CapabilityStatus.Degraded.class);

        stateStore.clear("worker-1", TENANCY);
        assertThat(health.probe(descriptor, "data-processing", ctx))
            .isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void probe_returns_ready_after_ttl_expires() {
        stateStore.record("worker-1", TENANCY, DegradationReason.CONTEXT_EXHAUSTED,
            Instant.now().minusSeconds(1));
        var status = health.probe(descriptor, "data-processing", ctx);
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void degradation_reasons_are_distinguishable() {
        stateStore.record("worker-1", TENANCY, DegradationReason.OVERLOADED,
            Instant.now().plusSeconds(60));
        assertThat(((CapabilityStatus.Degraded) health.probe(descriptor, "data-processing", ctx))
            .reason()).isEqualTo(DegradationReason.OVERLOADED);

        stateStore.clear("worker-1", TENANCY);
        stateStore.record("worker-1", TENANCY, DegradationReason.DOMAIN_MISMATCH,
            Instant.now().plusSeconds(60));
        assertThat(((CapabilityStatus.Degraded) health.probe(descriptor, "data-processing", ctx))
            .reason()).isEqualTo(DegradationReason.DOMAIN_MISMATCH);
    }
}
```

- [ ] **Step 2: Run the test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=DegradationAndRecoveryTest`
Expected: All 5 tests pass.

- [ ] **Step 3: Commit**

```bash
git add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DegradationAndRecoveryTest.java
git commit -m "feat(eidos#79): degradation and recovery example — AgentStateStore TTL lifecycle

Demonstrates probe Ready → Degraded → Ready transitions via SPI injection.
Covers record, clear, TTL expiry, and reason discrimination.

Closes #79"
```

---

### Task 5: Full build verification and remaining call-site fixes

**Files:**
- Potentially modify: any file that constructs `AgentCapability` via builder and might need description for completeness (examples, eval profiles)

**Interfaces:**
- Consumes: all changes from Tasks 1–4
- Produces: green full build

- [ ] **Step 1: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

If any compilation errors remain from call sites using the direct `AgentCapability` constructor, fix them by adding `null` as the second argument (description). Most call sites use the builder and need no changes.

- [ ] **Step 2: Check for any test failures and fix**

Address any test failures — most likely from assertions on rendering output that now include description text, or from YAML parsing tests.

- [ ] **Step 3: Commit any remaining fixes**

```bash
git add -A
git commit -m "fix(eidos#77): fix remaining call sites for AgentCapability description field

Refs #77"
```
