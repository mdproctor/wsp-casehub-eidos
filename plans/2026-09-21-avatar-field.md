# Avatar Field Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #181 — Add avatar field to agent YAML — archetype-resolved identity + external override
**Issue group:** #181

**Goal:** Add optional `avatar` field to `AgentDescriptor` that carries a compact visual identity code (collection preset, custom, or external URL), with auto-derivation from archetype at registration time.

**Architecture:** Single `String avatar` field on `AgentDescriptor` following the open-String pattern (slot, archetype). `AvatarCodec` in vocab/ computes the canonical archetype index and generates default preset codes. `ArchetypeDeriver` in runtime/ derives avatar alongside archetype at bootstrap. A2A_CARD surfaces the code for frontend consumption.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-eidos-api, casehub-eidos-vocab, casehub-eidos runtime

## Global Constraints

- Java 21 source (Java 26 JVM)
- All new public API goes in `casehub-eidos-api` (Tier 1, pure Java — no CDI, no Quarkus)
- Vocabulary utilities go in `casehub-eidos-vocab`
- No existing installations — schema changes go directly in base migrations
- `avatar` is an open String — eidos never interprets beyond URL prefix discrimination
- Default collection name: `mythic` (constant in `AvatarCodec`)

---

## Batch 1: Avatar Codec + Descriptor Field

### Task 1: AvatarCodec — canonical index and default code generation

**Files:**
- Create: `vocab/src/main/java/io/casehub/eidos/vocab/AvatarCodec.java`
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/AvatarCodecTest.java`

**Interfaces:**
- Consumes: `ArchetypeTerm.values()`, `ArchetypeTerm.family()`, `ArchetypeTerm.value()`
- Produces: `AvatarCodec.archetypeIndex(String value) → int`, `AvatarCodec.defaultCode(String collection, String archetypeValue) → String`, `AvatarCodec.isExternalUrl(String avatar) → boolean`, `AvatarCodec.DEFAULT_COLLECTION = "mythic"`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.vocab;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class AvatarCodecTest {

    @Test
    void detectiveIndexIsStable() {
        int idx = AvatarCodec.archetypeIndex("detective");
        assertThat(idx).isGreaterThanOrEqualTo(0);
        assertThat(idx).isLessThan(48);
    }

    @Test
    void indexIsSortedByFamilyThenValue() {
        int angel = AvatarCodec.archetypeIndex("angel");
        int guardian = AvatarCodec.archetypeIndex("guardian");
        int artist = AvatarCodec.archetypeIndex("artist");
        assertThat(angel).isLessThan(guardian);
        assertThat(guardian).isLessThan(artist);
    }

    @Test
    void defaultCodeFormatsAsPreset() {
        String code = AvatarCodec.defaultCode("mythic", "detective");
        assertThat(code).startsWith("mythic:P");
        assertThat(code).hasSize("mythic:P".length() + 1 + (AvatarCodec.archetypeIndex("detective") >= 36 ? 1 : 0));
    }

    @Test
    void defaultCodeForUnknownArchetypeReturnsNull() {
        assertThat(AvatarCodec.defaultCode("mythic", "nonexistent")).isNull();
    }

    @Test
    void isExternalUrlDetectsHttps() {
        assertThat(AvatarCodec.isExternalUrl("https://example.com/img.png")).isTrue();
        assertThat(AvatarCodec.isExternalUrl("http://example.com/img.png")).isTrue();
    }

    @Test
    void isExternalUrlRejectsCodes() {
        assertThat(AvatarCodec.isExternalUrl("mythic:P1B")).isFalse();
        assertThat(AvatarCodec.isExternalUrl(null)).isFalse();
    }

    @Test
    void allArchetypesHaveUniqueIndices() {
        var indices = new java.util.HashSet<Integer>();
        for (var term : ArchetypeTerm.values()) {
            int idx = AvatarCodec.archetypeIndex(term.value());
            assertThat(indices.add(idx))
                .as("duplicate index for " + term.value())
                .isTrue();
        }
        assertThat(indices).hasSize(48);
    }

    @Test
    void defaultCollectionIsMythic() {
        assertThat(AvatarCodec.DEFAULT_COLLECTION).isEqualTo("mythic");
    }
}
```

- [ ] **Step 2: Run tests — expect compilation failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=AvatarCodecTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — AvatarCodec not found

- [ ] **Step 3: Implement AvatarCodec**

```java
package io.casehub.eidos.vocab;

import java.util.Arrays;
import java.util.Comparator;
import java.util.List;
import java.util.Locale;

public final class AvatarCodec {

    public static final String DEFAULT_COLLECTION = "mythic";

    private static final List<String> CANONICAL_ORDER;

    static {
        CANONICAL_ORDER = Arrays.stream(ArchetypeTerm.values())
            .sorted(Comparator.comparing((ArchetypeTerm t) -> t.family().name())
                               .thenComparing(ArchetypeTerm::value))
            .map(ArchetypeTerm::value)
            .toList();
    }

    private AvatarCodec() {}

    public static int archetypeIndex(String archetypeValue) {
        return CANONICAL_ORDER.indexOf(archetypeValue.toLowerCase(Locale.ROOT));
    }

    public static String defaultCode(String collection, String archetypeValue) {
        int idx = archetypeIndex(archetypeValue);
        if (idx < 0) return null;
        return collection + ":P" + Integer.toString(idx, 36).toUpperCase();
    }

    public static boolean isExternalUrl(String avatar) {
        if (avatar == null) return false;
        String lower = avatar.toLowerCase(Locale.ROOT);
        return lower.startsWith("https://") || lower.startsWith("http://");
    }
}
```

- [ ] **Step 4: Run tests — expect all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=AvatarCodecTest`

- [ ] **Step 5: Commit**

```bash
git add vocab/src/main/java/io/casehub/eidos/vocab/AvatarCodec.java vocab/src/test/java/io/casehub/eidos/vocab/AvatarCodecTest.java
git commit -m "feat(vocab): add AvatarCodec — canonical archetype index and preset code generation

Sorted {Family}/{SubArchetype} canonical list for stable index computation.
Base36 encoding for compact preset codes (mythic:P0 through mythic:P1B).
URL discrimination for external avatar overrides.

Refs #181"
```

### Task 2: Add avatar field to AgentDescriptor + YAML + JPA

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — add `String avatar` field
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` — add `MAX_AVATAR`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java` — add avatar comparison
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/yaml/AgentDescriptorDeserializer.java` — parse avatar
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java` — add column
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` — map avatar
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql` — add column
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorAvatarTest.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializerTest.java` — add avatar tests

**Interfaces:**
- Consumes: `AgentDescriptorValidator`, `AgentDescriptorComparator`
- Produces: `AgentDescriptor.avatar() → String`, `AgentDescriptor.Builder.avatar(String)`

- [ ] **Step 1: Write failing test for AgentDescriptor avatar field**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AgentDescriptorAvatarTest {

    private static AgentDescriptor.Builder base() {
        return AgentDescriptor.builder()
            .agentId("test").name("Test").slot("analyst").tenancyId("t1");
    }

    @Test
    void avatarDefaultsToNull() {
        var d = base().build();
        assertThat(d.avatar()).isNull();
    }

    @Test
    void avatarCanBeSetToCollectionCode() {
        var d = base().avatar("mythic:P1B").build();
        assertThat(d.avatar()).isEqualTo("mythic:P1B");
    }

    @Test
    void avatarCanBeSetToExternalUrl() {
        var d = base().avatar("https://example.com/img.png").build();
        assertThat(d.avatar()).isEqualTo("https://example.com/img.png");
    }

    @Test
    void avatarExceedingMaxLengthThrows() {
        assertThatThrownBy(() -> base().avatar("x".repeat(501)).build())
            .isInstanceOf(AgentValidationException.class)
            .hasMessageContaining("avatar");
    }

    @Test
    void toBuilderPreservesAvatar() {
        var d = base().avatar("mythic:P1B").build();
        var copy = d.toBuilder().build();
        assertThat(copy.avatar()).isEqualTo("mythic:P1B");
    }
}
```

- [ ] **Step 2: Run test — expect compilation failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorAvatarTest`

- [ ] **Step 3: Add avatar field to AgentDescriptor**

Add `String avatar` record component after `archetypeAdjectives` (line 22). Use `ide_replace_text_in_file`:
- Record declaration: add `String avatar,` after `List<String> archetypeAdjectives,`
- Compact constructor: add `AgentDescriptorValidator.validateOptional("avatar", avatar, AgentDescriptorValidator.MAX_AVATAR);`
- Builder: add `private String avatar;` field and `public Builder avatar(String v)` method
- `toBuilder()`: add `.avatar(this.avatar)`
- `build()`: add `avatar,` in constructor call after `archetypeAdjectives,`

Add `MAX_AVATAR = 500` to `AgentDescriptorValidator`.

Update `AgentDescriptorComparator`: add `compareField(drifts, "avatar", ...)`, bump `COMPARED_FIELD_COUNT` to 24.

- [ ] **Step 4: Update YAML deserializer**

Add to `AgentDescriptorDeserializer`: `ifString(root, "avatar", builder::avatar);`

- [ ] **Step 5: Update JPA entity + mapper + schema**

Entity: add `@Column(name = "avatar") String avatar;` after archetypeAdjectives.
Mapper toRecord: add `e.avatar,` in constructor call.
Mapper toEntity: add `e.avatar = d.avatar();`.
Schema V1: add `avatar VARCHAR(500),` after `archetype_adjectives`.

- [ ] **Step 6: Add YAML deserialization tests**

Add to `AgentDescriptorDeserializerTest`:

```java
@Test
void avatarCollectionCode_deserializes() throws Exception {
    var yaml = """
        agentId: av-test
        name: Avatar Test
        slot: analyst
        tenancyId: t1
        avatar: mythic:P1B
        """;
    var d = mapper.readValue(yaml, AgentDescriptor.class);
    assertThat(d.avatar()).isEqualTo("mythic:P1B");
}

@Test
void avatarExternalUrl_deserializes() throws Exception {
    var yaml = """
        agentId: av-url
        name: URL
        slot: analyst
        tenancyId: t1
        avatar: https://example.com/img.png
        """;
    var d = mapper.readValue(yaml, AgentDescriptor.class);
    assertThat(d.avatar()).isEqualTo("https://example.com/img.png");
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime`

- [ ] **Step 8: Commit**

```bash
git add api/ eidos-core/ runtime/
git commit -m "feat: add avatar field to AgentDescriptor with YAML and JPA support

Optional String field following the open-String pattern. Supports
collection codes (mythic:P1B) and external URLs (https://...).
MAX_AVATAR = 500. Schema, entity, mapper, YAML deserializer updated.

Refs #181"
```

---

## Batch 2: Auto-Derivation + Rendering

### Task 3: Avatar auto-derivation in ArchetypeDeriver

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ArchetypeDeriver.java` — add avatar derivation
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/ArchetypeDeriverTest.java` — add avatar tests

**Interfaces:**
- Consumes: `AvatarCodec.defaultCode()`, `AvatarCodec.DEFAULT_COLLECTION`, `AgentDescriptor.archetype()`, `AgentDescriptor.avatar()`
- Produces: `AgentDescriptor` with avatar set when archetype is present and avatar was null

- [ ] **Step 1: Write failing tests**

Add to `ArchetypeDeriverTest`:

```java
@Test
void avatarDerivedWhenArchetypeSetAndAvatarNull() {
    var d = base()
        .archetype("detective")
        .build();
    var result = ArchetypeDeriver.deriveArchetype(d);
    assertThat(result.avatar()).isNotNull();
    assertThat(result.avatar()).startsWith("mythic:P");
}

@Test
void avatarNotOverriddenWhenExplicitlySet() {
    var d = base()
        .archetype("detective")
        .avatar("https://example.com/custom.png")
        .build();
    var result = ArchetypeDeriver.deriveArchetype(d);
    assertThat(result.avatar()).isEqualTo("https://example.com/custom.png");
}

@Test
void avatarNullWhenNoArchetype() {
    var d = base().build();
    var result = ArchetypeDeriver.deriveArchetype(d);
    assertThat(result.avatar()).isNull();
}

@Test
void avatarCodeMatchesCanonicalIndex() {
    var d = base().archetype("detective").build();
    var result = ArchetypeDeriver.deriveArchetype(d);
    int expectedIndex = io.casehub.eidos.vocab.AvatarCodec.archetypeIndex("detective");
    String expectedCode = io.casehub.eidos.vocab.AvatarCodec.defaultCode("mythic", "detective");
    assertThat(result.avatar()).isEqualTo(expectedCode);
}
```

- [ ] **Step 2: Run tests — expect failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ArchetypeDeriverTest`

- [ ] **Step 3: Extend ArchetypeDeriver**

Add avatar derivation after the archetype derivation block in `deriveArchetype()`. After the existing return statements that set archetype, and at the end of the method, add avatar derivation for the case where archetype is already set:

```java
// At the end of deriveArchetype(), before the final return:
// Derive avatar when archetype is set and avatar is null
if (descriptor.archetype() != null && descriptor.avatar() == null) {
    String code = AvatarCodec.defaultCode(AvatarCodec.DEFAULT_COLLECTION, descriptor.archetype());
    if (code != null) {
        descriptor = descriptor.toBuilder().avatar(code).build();
    }
}
```

Import `AvatarCodec`.

Note: the method may have already called `toBuilder()` for archetype derivation. The avatar derivation should apply to whichever descriptor is current at the end of the method — it works on the `descriptor` variable which may have been rebuilt.

Restructure the method to accumulate both derivations into a single `toBuilder()` call when both apply, or apply avatar as a second pass. The simplest approach: add avatar derivation as the last step before return, operating on whatever descriptor the method has at that point.

- [ ] **Step 4: Run tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ArchetypeDeriverTest`

- [ ] **Step 5: Commit**

```bash
git add runtime/
git commit -m "feat: auto-derive avatar code from archetype at registration time

ArchetypeDeriver now generates mythic:P{base36(index)} when archetype
is set and avatar is null. Uses AvatarCodec canonical index.

Refs #181"
```

### Task 4: A2A_CARD rendering + descriptor payload

**Files:**
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java` — add avatar to A2A_CARD + payload
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java` — add avatar tests

**Interfaces:**
- Consumes: `AgentDescriptor.avatar()`
- Produces: `avatar` field in A2A_CARD JSON, `avatar` in descriptor payload

- [ ] **Step 1: Write failing tests**

Add to `EidosRenderPipelineTest`:

```java
@Test
void a2aCardIncludesAvatarWhenSet() {
    var desc = AgentDescriptor.builder()
        .agentId("av").name("Av").slot("s").tenancyId("t1")
        .avatar("mythic:P1B")
        .build();
    var card = renderA2aCard(desc);
    assertThat(card.has("avatar")).isTrue();
    assertThat(card.get("avatar").asText()).isEqualTo("mythic:P1B");
}

@Test
void a2aCardOmitsAvatarWhenNull() {
    var desc = AgentDescriptor.builder()
        .agentId("no-av").name("No Av").slot("s").tenancyId("t1")
        .build();
    var card = renderA2aCard(desc);
    assertThat(card.has("avatar")).isFalse();
}

@Test
void descriptorPayloadIncludesAvatar() {
    var desc = AgentDescriptor.builder()
        .agentId("av").name("Av").slot("s").tenancyId("t1")
        .avatar("https://example.com/img.png")
        .build();
    var node = pipeline.buildDescriptorPayload(desc, MARKDOWN);
    assertThat(node.has("avatar")).isTrue();
    assertThat(node.get("avatar").asText()).isEqualTo("https://example.com/img.png");
}
```

- [ ] **Step 2: Run tests — expect failure**

- [ ] **Step 3: Add avatar to render pipeline**

In `buildDescriptorPayload()`: add `addIfPresent(node, "avatar", descriptor.avatar());` after the archetype line.

In `assembleA2aCard()`: add after the archetype block:

```java
addIfPresent(card, "avatar", descriptor.avatar());
```

- [ ] **Step 4: Run tests — expect pass**

- [ ] **Step 5: Commit**

```bash
git add eidos-core/ runtime/
git commit -m "feat(renderer): add avatar to A2A_CARD and descriptor payload

A2A_CARD includes avatar as a top-level string when non-null.
Descriptor payload includes avatar for cache key consistency.
Not rendered in MARKDOWN/PROSE — visual concern only.

Refs #181"
```

---

## Batch 3: Integration Tests + Docs

### Task 5: Integration test and consumer guide

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/AvatarScenarioTest.java`
- Modify: `docs/guides/consumer-guide.md` — add avatar field
- Modify: `CLAUDE.md` — add avatar to descriptor listing

**Interfaces:**
- Consumes: all previous tasks

- [ ] **Step 1: Write integration test**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.runtime.registrar.ArchetypeDeriver;
import io.casehub.eidos.vocab.AvatarCodec;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class AvatarScenarioTest {

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;

    static final String TENANCY = "avatar-examples";

    @Test
    void explicitAvatarPreserved() {
        var d = AgentDescriptor.builder()
            .agentId("explicit-av").name("Explicit").slot("s").tenancyId(TENANCY)
            .archetype("detective")
            .avatar("https://example.com/custom.png")
            .build();
        var derived = ArchetypeDeriver.deriveArchetype(d);
        assertThat(derived.avatar()).isEqualTo("https://example.com/custom.png");
    }

    @Test
    void avatarAutoDerivedFromArchetype() {
        var d = AgentDescriptor.builder()
            .agentId("auto-av").name("Auto").slot("s").tenancyId(TENANCY)
            .archetype("detective")
            .build();
        var derived = ArchetypeDeriver.deriveArchetype(d);
        assertThat(derived.avatar()).isNotNull();
        assertThat(derived.avatar()).startsWith("mythic:P");
        assertThat(AvatarCodec.isExternalUrl(derived.avatar())).isFalse();
    }

    @Test
    void avatarAppearsInA2aCard() {
        var d = AgentDescriptor.builder()
            .agentId("a2a-av").name("A2A").slot("s").tenancyId(TENANCY)
            .avatar("mythic:P1B")
            .build();
        var rendered = renderer.render(d, AgentPromptContext.forFormat(RenderFormat.A2A_CARD));
        assertThat(rendered.content()).contains("\"avatar\"");
        assertThat(rendered.content()).contains("mythic:P1B");
    }

    @Test
    void avatarAbsentFromMarkdown() {
        var d = AgentDescriptor.builder()
            .agentId("md-av").name("MD").slot("s").tenancyId(TENANCY)
            .avatar("mythic:P1B")
            .build();
        var rendered = renderer.render(d, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
        assertThat(rendered.content()).doesNotContain("mythic:P1B");
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=AvatarScenarioTest`

- [ ] **Step 3: Update consumer guide**

Add avatar to the AgentDescriptor field list:
```markdown
- **Avatar:** optional `String` — compact visual identity code (e.g. `"mythic:P1B"`) or external image URL (`"https://..."`). Auto-derived from archetype at registration time when not set explicitly. Collection codes are rendered client-side from SVG part libraries; external URLs are rendered as `<img>`. See blocks-ui#167.
```

- [ ] **Step 4: Update CLAUDE.md**

Add `avatar` to the AgentDescriptor description line.

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

- [ ] **Step 6: Commit**

```bash
git add examples/ docs/ CLAUDE.md
git commit -m "feat: avatar integration tests and documentation

End-to-end scenarios: explicit avatar preserved, auto-derivation from
archetype, A2A_CARD rendering, absence from MARKDOWN. Consumer guide
and CLAUDE.md updated with avatar field documentation.

Closes #181"
```

---

## References

- [specs/issue-181-avatar-field/2026-09-21-avatar-field-design.md] — design spec
- [specs/avatar-generator-contract.md] — avatar generation payload (Section 5)
- [vocab/src/main/java/io/casehub/eidos/vocab/ArchetypeTerm.java] — 48 sub-archetypes
- [runtime/src/main/java/io/casehub/eidos/runtime/registrar/ArchetypeDeriver.java] — derivation location
- [eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java] — render pipeline
- [GitHub #181] — focal issue
