# Archetype Identity Auto-Derivation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #192 — consolidate archetype/personality data into shared YAML resource
**Issue group:** #192, #193

**Goal:** Wire ArchetypeCompatibility to load from YAML, add `domain` field, auto-derive disposition from archetype, and auto-compose briefing at render time — so a 6–7 field YAML descriptor produces a fully rendered system prompt.

**Architecture:** ArchetypeCompatibility replaces hardcoded static initializers with YAML parsing from `META-INF/eidos/archetypes.yaml`. `domain` is added as an optional String field across the full descriptor stack. Auto-derivation chains archetype → first MBTI affinity → PersonalityTypeDeriver → 5-axis disposition in DescriptorCollector. Auto-briefing composes structurally in EidosRenderPipeline at render time.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, Jackson + YAMLFactory

## Global Constraints

- Java enums (`ArchetypeTerm`, `ArchetypeFamily`) stay as compile-time model — YAML provides data, not structure
- `domain` is open String — no vocabulary validation (same pattern as `slot`)
- No existing database installations — schema changes go directly into V1
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- Use `mvn` not `./mvnw`

---

## Batch 1: YAML-backed ArchetypeCompatibility (#192)

### Task 1: Wire ArchetypeCompatibility to load from YAML

**Files:**
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/ArchetypeCompatibility.java`
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/ArchetypeCompatibilityTest.java` (existing tests, verify they still pass)

**Interfaces:**
- Consumes: `META-INF/eidos/archetypes.yaml` (classpath resource, already landed)
- Produces: Same static API — `compatibleFamilies(String, String)`, `subArchetypeAffinity(ArchetypeTerm, String, String)` — unchanged signatures. New package-private `mbtiAffinities(ArchetypeTerm)` → `List<String>` for Task 4.

- [ ] **Step 1: Verify existing tests pass before changes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ArchetypeCompatibilityTest`
Expected: All tests PASS (baseline)

- [ ] **Step 2: Add jackson-dataformat-yaml compile dependency to vocab/pom.xml**

The test-scoped dependency already exists. Change scope from `test` to compile (or add a new compile-scope entry) since the runtime static initializer now needs it:

```xml
<dependency>
  <groupId>com.fasterxml.jackson.dataformat</groupId>
  <artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
```

- [ ] **Step 3: Replace static initializer in ArchetypeCompatibility**

Replace the entire static block and all 7 private init methods (`initMbti`, `initEnneagram`, `initDisc`, `initBelbin`, `initBigFive`, `initSdi`, `initAffinities`) plus the `aff` helper with a single static initializer that reads from YAML.

The YAML `compatibility:` section uses short names (`mbti`, `enneagram`, `disc`, etc.) but the Java API uses URIs (`urn:casehub:vocab:mbti`). Add a URI mapping constant:

```java
private static final Map<String, String> FRAMEWORK_URIS = Map.of(
    "mbti", MbtiTypeTerm.URI,
    "enneagram", EnneagramTerm.URI,
    "disc", DiscTerm.URI,
    "belbin", BelbinTerm.URI,
    "bigFive", BigFiveTerm.URI,
    "sdi", SdiTerm.URI
);
```

Static initializer reads YAML:

```java
static {
    try {
        var yaml = new ObjectMapper(new YAMLFactory());
        var stream = Thread.currentThread().getContextClassLoader()
                .getResourceAsStream("META-INF/eidos/archetypes.yaml");
        if (stream == null) throw new IllegalStateException("archetypes.yaml not found on classpath");
        var root = yaml.readTree(stream);

        // Parse compatibility section → FAMILY_MAP
        var compat = root.get("compatibility");
        compat.fields().forEachRemaining(framework -> {
            var uri = FRAMEWORK_URIS.get(framework.getKey());
            if (uri == null) return;
            var termMap = new HashMap<String, Set<ArchetypeFamily>>();
            framework.getValue().fields().forEachRemaining(entry -> {
                var families = new HashSet<ArchetypeFamily>();
                entry.getValue().forEach(f ->
                    families.add(ArchetypeFamily.valueOf(f.asText().toUpperCase(Locale.ROOT))));
                termMap.put(entry.getKey().toLowerCase(Locale.ROOT), Set.copyOf(families));
            });
            FAMILY_MAP.put(uri, Map.copyOf(termMap));
        });

        // Parse families.*.subArchetypes.*.affinities → AFFINITY_MAP
        var families = root.get("families");
        families.fields().forEachRemaining(family ->
            family.getValue().get("subArchetypes").fields().forEachRemaining(sub -> {
                var term = ArchetypeTerm.valueOf(sub.getKey().toUpperCase(Locale.ROOT));
                var affinities = sub.getValue().get("affinities");
                if (affinities == null) return;
                var uriMap = new HashMap<String, Set<String>>();
                affinities.fields().forEachRemaining(fw -> {
                    var fwUri = FRAMEWORK_URIS.get(fw.getKey());
                    if (fwUri == null) return;
                    var values = new ArrayList<String>();
                    fw.getValue().forEach(v -> values.add(v.asText().toLowerCase(Locale.ROOT)));
                    uriMap.put(fwUri, List.copyOf(values));
                });
                AFFINITY_MAP.put(term, Map.copyOf(uriMap));
            })
        );
    } catch (Exception e) {
        throw new ExceptionInInitializerError(e);
    }
}
```

Remove: `initMbti()`, `initEnneagram()`, `initDisc()`, `initBelbin()`, `initBigFive()`, `initSdi()`, `initAffinities()`, `aff()` — all 8 methods (lines 44–251).

Add package-private method for Task 4's auto-derivation:

```java
static List<String> mbtiAffinities(ArchetypeTerm term) {
    var affinities = AFFINITY_MAP.get(term);
    if (affinities == null) return List.of();
    var mbti = affinities.get(MbtiTypeTerm.URI);
    return mbti != null ? mbti : List.of();
}
```

Update the `AFFINITY_MAP` field type from `Map<ArchetypeTerm, Map<String, Set<String>>>` to `Map<ArchetypeTerm, Map<String, List<String>>>`. Update `subArchetypeAffinity()` to use `List.contains()` (same API, works identically).

Add imports: `com.fasterxml.jackson.databind.ObjectMapper`, `com.fasterxml.jackson.dataformat.yaml.YAMLFactory`, `java.util.ArrayList`, `java.util.HashSet`.

- [ ] **Step 4: Run existing tests to verify API compatibility**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab`
Expected: All tests PASS — same API, different data source

- [ ] **Step 5: Commit**

```bash
git add vocab/pom.xml vocab/src/main/java/io/casehub/eidos/vocab/ArchetypeCompatibility.java
git commit -m "feat(#192): wire ArchetypeCompatibility to load from archetypes.yaml

Replace 7 hardcoded init methods with YAML-backed static initializer.
Public API unchanged — compatibleFamilies() and subArchetypeAffinity()
retain same signatures and behavior.

Refs #192

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Add bidirectional validation and schema-YAML sync tests

**Files:**
- Modify: `vocab/src/test/java/io/casehub/eidos/vocab/ArchetypeDataValidationTest.java`

**Interfaces:**
- Consumes: `archetypes.yaml` (classpath), `descriptor.schema.json` (classpath), Java enums
- Produces: Build-time validation — fails if YAML↔Java or YAML↔schema drift

- [ ] **Step 1: Add reverse-direction validation assertions**

The existing tests check YAML→Java (e.g., `yaml_contains_all_12_families` checks YAML families match Java enum values). The `containsExactlyInAnyOrderElementsOf` assertion is already bidirectional for families and sub-archetypes. Verify the existing assertions cover both directions — they use `containsExactlyInAnyOrderElementsOf` which is bidirectional.

Check if description and adjective tests are also bidirectional. If any test only checks one direction, add the reverse assertion.

- [ ] **Step 2: Add compatibility section validation test**

```java
@Test
void yaml_compatibility_framework_keys_are_known() {
    var knownFrameworks = Set.of("mbti", "enneagram", "disc", "belbin", "bigFive", "sdi");
    var yamlFrameworks = fieldNames(root.get("compatibility"));
    assertThat(yamlFrameworks).isSubsetOf(knownFrameworks);
}

@Test
void yaml_compatibility_family_values_are_valid() {
    root.get("compatibility").fields().forEachRemaining(framework ->
        framework.getValue().fields().forEachRemaining(entry ->
            entry.getValue().forEach(f ->
                assertThat(f.asText().toUpperCase(Locale.ROOT))
                    .as("compatibility.%s.%s value '%s'", framework.getKey(), entry.getKey(), f.asText())
                    .isIn(Arrays.stream(ArchetypeFamily.values()).map(Enum::name).toArray())
            )
        )
    );
}
```

- [ ] **Step 3: Add schema-YAML sync validation test**

Read `descriptor.schema.json` from classpath and verify its archetype enum array matches YAML sub-archetype values:

```java
@Test
void descriptor_schema_archetype_enum_matches_yaml() throws IOException {
    var schemaStream = Thread.currentThread().getContextClassLoader()
            .getResourceAsStream("META-INF/eidos/descriptor.schema.json");
    assertThat(schemaStream).as("descriptor.schema.json must be on classpath").isNotNull();
    var json = new ObjectMapper();
    var schema = json.readTree(schemaStream);

    var schemaArchetypes = new HashSet<String>();
    schema.at("/properties/agents/items/properties/archetype/enum")
          .forEach(n -> schemaArchetypes.add(n.asText()));

    var yamlArchetypes = new HashSet<String>();
    root.get("families").fields().forEachRemaining(family ->
            family.getValue().get("subArchetypes").fields().forEachRemaining(sub ->
                    yamlArchetypes.add(sub.getKey())));

    assertThat(schemaArchetypes)
            .as("descriptor.schema.json archetype enum must match archetypes.yaml")
            .containsExactlyInAnyOrderElementsOf(yamlArchetypes);
}
```

Note: The exact JSON path for the archetype enum in `descriptor.schema.json` may differ — verify by reading the schema structure. Adjust the `at()` path accordingly.

Also add `jackson-databind` as a test dependency if not already present (for reading JSON schema).

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ArchetypeDataValidationTest`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add vocab/src/test/java/io/casehub/eidos/vocab/ArchetypeDataValidationTest.java
git commit -m "test(#192): add bidirectional validation and schema-YAML sync tests

Verify compatibility section framework keys and family values.
Validate descriptor.schema.json archetype enum matches YAML values.
Any YAML↔Java or YAML↔schema drift now fails the build.

Refs #192

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: `domain` field across the descriptor stack (#193)

### Task 3: Add `domain` field to AgentDescriptor and supporting types

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` (record component, builder, toBuilder)
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/yaml/AgentDescriptorDeserializer.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Modify: `api/src/main/resources/META-INF/eidos/descriptor.schema.json`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/Identity.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedAgentConfig.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java` (existing)

**Interfaces:**
- Produces: `AgentDescriptor.domain()` → `String` (nullable), `Builder.domain(String)` → `Builder`

- [ ] **Step 1: Write test for domain field**

Add test in existing `AgentDescriptorTest` (or create if needed):

```java
@Test
void domain_field_round_trips_through_builder() {
    var d = AgentDescriptor.builder()
            .agentId("test")
            .tenancyId("t1")
            .domain("software-engineering")
            .build();
    assertThat(d.domain()).isEqualTo("software-engineering");
}

@Test
void domain_defaults_to_null() {
    var d = AgentDescriptor.builder()
            .agentId("test")
            .tenancyId("t1")
            .build();
    assertThat(d.domain()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorTest`
Expected: FAIL — `domain()` method does not exist

- [ ] **Step 3: Add `domain` to AgentDescriptor record**

Add `domain` as a record component between `slot` (line 21) and `archetype` (line 22):

```java
String slot,
String domain,
String archetype,
```

Add `domain` to the compact constructor's validation (no special validation needed — nullable open String).

Add `domain` to the `Builder` class:
- Field: `private String domain;` (after `slot` field at line 219)
- Method: `public Builder domain(String v) { this.domain = v; return this; }`
- Update `build()` method to include `domain` in constructor call
- Update `toBuilder()` method to include `.domain(this.domain())`

- [ ] **Step 4: Update AgentDescriptorDeserializer**

In `deserialize()` method (line 32–127), add after the `slot` field parsing:

```java
ifString(obj, "domain", b::domain);
```

- [ ] **Step 5: Update JPA entity + mapper + schema**

`AgentDescriptorEntity`: Add field after `slot` (line 56):
```java
String domain;
```

`AgentDescriptorMapper.toRecord()` (line 28–50): Add `domain` to builder:
```java
.domain(e.domain)
```

`AgentDescriptorMapper.toEntity()` (line 52–88): Add:
```java
e.domain = d.domain();
```

`V1__initial_schema.sql`: Add column after `slot` (line 17):
```sql
domain              VARCHAR(255),
```

- [ ] **Step 6: Update annotations**

`@Identity` annotation: Add after `slot()`:
```java
String domain() default "";
```

`AnnotatedAgentConfig`: Add `domain` field.

`EidosAnnotationsRecorder`: Pass `domain` through to builder.

`EidosAnnotationsProcessor`: Extract `domain` from `@Identity` annotation.

- [ ] **Step 7: Update descriptor.schema.json**

Add `domain` field definition in the properties section:
```json
"domain": {
  "type": "string",
  "description": "Human-readable domain label (e.g. software-engineering, healthcare, legal)"
}
```

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,eidos-core,runtime,annotations/runtime,annotations/deployment`
Expected: All tests PASS (some test descriptors may need `domain` added to constructor calls — fix compilation errors)

- [ ] **Step 9: Commit**

```bash
git add api/ eidos-core/ runtime/ annotations/
git commit -m "feat(#193): add domain field to AgentDescriptor

Optional String field for human-readable domain label (e.g.
software-engineering, healthcare). Distinct from domainVocabulary
(vocabulary URI). Added to YAML, JPA, annotations, and JSON schema.

Refs #193

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Auto-derivation and auto-briefing (#193)

### Task 4: Auto-derive disposition from archetype in DescriptorCollector

**Files:**
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/registrar/DescriptorCollector.java`
- Test: `eidos-core/src/test/java/io/casehub/eidos/core/registrar/DescriptorCollectorTest.java` (existing)

**Interfaces:**
- Consumes: `ArchetypeCompatibility.subArchetypeAffinity()`, `PersonalityTypeDeriver.derive()`, `VocabularyRegistry`
- Produces: Modified `AgentDescriptor` with auto-derived disposition when archetype is set and disposition is absent

- [ ] **Step 1: Write failing test for auto-derivation**

In `DescriptorCollectorTest`, add a test that creates a descriptor with just `archetype: "sage"` and no disposition, then verifies that `deriveDispositionAxes()` produces a non-null disposition with axis values:

```java
@Test
void archetype_auto_derives_disposition_when_absent() {
    var descriptor = AgentDescriptor.builder()
            .agentId("test-sage")
            .tenancyId("t1")
            .archetype("sage")
            .build();

    var derived = DescriptorCollector.deriveDispositionAxes(descriptor, vocabRegistry);

    assertThat(derived.disposition()).isNotNull();
    assertThat(derived.disposition().get(DispositionAxis.SOCIAL_ORIENTATION)).isNotEmpty();
}
```

Note: `vocabRegistry` needs to have MBTI and Jungian vocabularies registered. Check existing test setup for how this is done.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core -Dtest=DescriptorCollectorTest#archetype_auto_derives_disposition_when_absent`
Expected: FAIL — disposition is null (no derivation logic yet)

- [ ] **Step 3: Implement auto-derivation in DescriptorCollector**

Add a new method `deriveDispositionFromArchetype`:

```java
private static AgentDescriptor deriveDispositionFromArchetype(AgentDescriptor descriptor,
                                                               VocabularyRegistry vocabRegistry) {
    if (descriptor.archetype() == null) return descriptor;
    if (descriptor.disposition() != null
        && (descriptor.disposition().dispositionProfile() != null
            || !descriptor.disposition().allAxesEmpty())) {
        return descriptor;
    }

    // Resolve archetype to enum
    ArchetypeTerm term;
    try {
        term = ArchetypeTerm.valueOf(descriptor.archetype().toUpperCase(Locale.ROOT));
    } catch (IllegalArgumentException e) {
        return descriptor; // Unknown archetype — skip silently
    }

    // Get first MBTI affinity
    var affinities = /* extract from ArchetypeCompatibility AFFINITY_MAP */;
    // Since AFFINITY_MAP is private, we need to use the public API differently.
    // Check all 16 MBTI types and find which ones have affinity:
    var mbtiTypes = List.of("INTJ","INTP","ENTJ","ENTP","INFJ","INFP","ENFJ","ENFP",
                            "ISTJ","ISFJ","ESTJ","ESFJ","ISTP","ISFP","ESTP","ESFP");
    String firstMbti = null;
    for (var mbti : mbtiTypes) {
        if (ArchetypeCompatibility.subArchetypeAffinity(term, MbtiTypeTerm.URI, mbti.toLowerCase(Locale.ROOT))) {
            firstMbti = mbti;
            break;
        }
    }
    if (firstMbti == null) return descriptor;

    // Derive via PersonalityTypeDeriver
    var input = new PersonalityInput(firstMbti, null, false, Set.of());
    var derived = PersonalityTypeDeriver.derive(input, vocabRegistry);
    if (derived.dispositionProfile() == null) return descriptor;

    return descriptor.toBuilder()
            .disposition(derived)
            .build();
}
```

**Important refinement:** The approach above iterates all 16 MBTI types, which is suboptimal. A better approach: add a package-private static method to `ArchetypeCompatibility` that returns the MBTI affinity set for a given `ArchetypeTerm`:

```java
static Set<String> mbtiAffinities(ArchetypeTerm term) {
    var affinities = AFFINITY_MAP.get(term);
    if (affinities == null) return Set.of();
    var mbti = affinities.get(MbtiTypeTerm.URI);
    return mbti != null ? mbti : Set.of();
}
```

Then in the derivation:
```java
var mbtiSet = ArchetypeCompatibility.mbtiAffinities(term);
if (mbtiSet.isEmpty()) return descriptor;
String firstMbti = mbtiSet.iterator().next();
```

Note: `Set` iteration order is not guaranteed. Since `AFFINITY_MAP` uses `Set.copyOf()` from the YAML parse order, and the YAML lists MBTI types in relevance order, we need to preserve ordering. Change the affinity parsing to use `LinkedHashSet` or `List` to preserve YAML order. Alternatively, use the YAML node directly to get ordered affinities.

**Better approach:** Change `AFFINITY_MAP` value type from `Set<String>` to `List<String>` to preserve YAML ordering. This requires updating:
- `AFFINITY_MAP` value type changed to `List<String>` in Task 1 to preserve YAML ordering
- `subArchetypeAffinity()` uses `.contains()` (works for both Set and List)
- New package-private method added in Task 1: `static List<String> mbtiAffinities(ArchetypeTerm term)`

Call `deriveDispositionFromArchetype()` in `collectAndValidate()` before the existing `deriveDispositionAxes()` call (before line 78):

```java
var withArchetypeDerived = deriveDispositionFromArchetype(d, vocabRegistry);
var derived = deriveDispositionAxes(withArchetypeDerived, vocabRegistry);
```

- [ ] **Step 4: Write test for explicit disposition takes precedence**

```java
@Test
void explicit_disposition_wins_over_archetype() {
    var explicitDisposition = AgentDisposition.builder()
            .socialOrientation("collaborative")
            .build();
    var descriptor = AgentDescriptor.builder()
            .agentId("test")
            .tenancyId("t1")
            .archetype("sage")
            .disposition(explicitDisposition)
            .build();

    var derived = DescriptorCollector.deriveDispositionAxes(descriptor, vocabRegistry);
    assertThat(derived.disposition().get(DispositionAxis.SOCIAL_ORIENTATION))
            .extracting(dv -> dv.term())
            .contains("collaborative");
}
```

- [ ] **Step 5: Write test for unknown archetype skips silently**

```java
@Test
void unknown_archetype_skips_derivation() {
    var descriptor = AgentDescriptor.builder()
            .agentId("test")
            .tenancyId("t1")
            .archetype("custom-domain-role")
            .build();

    var derived = DescriptorCollector.deriveDispositionAxes(descriptor, vocabRegistry);
    assertThat(derived.disposition()).isNull();
}
```

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add vocab/src/main/java/io/casehub/eidos/vocab/ArchetypeCompatibility.java eidos-core/
git commit -m "feat(#193): auto-derive disposition from archetype

When archetype is set and disposition is absent, derive via:
archetype → first MBTI affinity → PersonalityTypeDeriver → 5-axis.
Explicit disposition or mbtiType/enneagramType always wins.
Unknown archetypes skip derivation silently.

Refs #193

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 5: Auto-compose briefing at render time

**Files:**
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java`
- Test: `eidos-core/src/test/java/io/casehub/eidos/core/renderer/EidosRenderPipelineTest.java` (existing)

**Interfaces:**
- Consumes: `AgentDescriptor.archetype()`, `.slot()`, `.domain()`, `.archetypeAdjectives()`, `VocabularyRegistry`
- Produces: Auto-composed `## Operating Principles` section in MARKDOWN/PROSE when briefing is null

- [ ] **Step 1: Write failing test for auto-briefing**

```java
@Test
void auto_briefing_composes_from_archetype_and_slot() {
    var descriptor = AgentDescriptor.builder()
            .agentId("test")
            .tenancyId("t1")
            .name("Code Reviewer")
            .slot("reviewer")
            .domain("software-engineering")
            .archetype("sage")
            .archetypeAdjectives(List.of("cautious", "methodical"))
            .build();
    var context = AgentPromptContext.builder().format(RenderFormat.MARKDOWN).build();

    var result = pipeline.assemble(
            pipeline.buildStage1(descriptor, context),
            Optional.empty(), Optional.empty(),
            descriptor, context);

    assertThat(result.systemPrompt())
            .contains("Operating Principles")
            .contains("reviewer")
            .contains("software-engineering")
            .contains("sage")
            .contains("cautious and methodical");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core -Dtest=EidosRenderPipelineTest#auto_briefing_composes_from_archetype_and_slot`
Expected: FAIL — no auto-briefing logic exists

- [ ] **Step 3: Implement auto-briefing in assembleMarkdown**

In `assembleMarkdown()` (line 754–758), replace the briefing block:

```java
// Briefing — explicit or auto-composed
if (!(enrichment.isPresent() && enrichment.get().dispositionNarrative().isPresent())) {
    if (descriptor.briefing() != null) {
        sb.append("\n## Operating Principles\n").append(descriptor.briefing()).append("\n");
    } else if (descriptor.archetype() != null && descriptor.slot() != null) {
        sb.append("\n## Operating Principles\n");
        sb.append("You are a ").append(descriptor.slot());
        if (descriptor.domain() != null) {
            sb.append(" in the ").append(descriptor.domain()).append(" domain");
        }
        sb.append(", embodying the ").append(descriptor.archetype()).append(" archetype.\n");
        if (!descriptor.archetypeAdjectives().isEmpty()) {
            sb.append("Your approach is ").append(String.join(" and ", descriptor.archetypeAdjectives())).append(".\n");
        }
        vocab.resolve("urn:casehub:vocab:archetype", descriptor.archetype())
             .ifPresent(term -> sb.append(term.description()).append("\n"));
    }
}
```

Also add the same logic to `assembleProse()` method — find the equivalent briefing block and apply the same pattern.

Do NOT add auto-briefing to `assembleA2aCard()` — A2A_CARD carries structured data only.

- [ ] **Step 4: Write test for explicit briefing wins**

```java
@Test
void explicit_briefing_wins_over_auto() {
    var descriptor = AgentDescriptor.builder()
            .agentId("test")
            .tenancyId("t1")
            .name("Test")
            .slot("reviewer")
            .archetype("sage")
            .briefing("Custom operating instructions.")
            .build();
    var context = AgentPromptContext.builder().format(RenderFormat.MARKDOWN).build();

    var result = pipeline.assemble(
            pipeline.buildStage1(descriptor, context),
            Optional.empty(), Optional.empty(),
            descriptor, context);

    assertThat(result.systemPrompt())
            .contains("Custom operating instructions.")
            .doesNotContain("embodying the");
}
```

- [ ] **Step 5: Write test for minimal auto-briefing (no domain, no adjectives)**

```java
@Test
void auto_briefing_works_with_minimal_fields() {
    var descriptor = AgentDescriptor.builder()
            .agentId("test")
            .tenancyId("t1")
            .name("Test")
            .slot("reviewer")
            .archetype("sage")
            .build();
    var context = AgentPromptContext.builder().format(RenderFormat.MARKDOWN).build();

    var result = pipeline.assemble(
            pipeline.buildStage1(descriptor, context),
            Optional.empty(), Optional.empty(),
            descriptor, context);

    assertThat(result.systemPrompt())
            .contains("You are a reviewer, embodying the sage archetype.")
            .doesNotContain("domain");
}
```

- [ ] **Step 6: Write test for A2A_CARD has no auto-briefing**

```java
@Test
void a2a_card_has_no_auto_briefing() {
    var descriptor = AgentDescriptor.builder()
            .agentId("test")
            .tenancyId("t1")
            .name("Test")
            .slot("reviewer")
            .archetype("sage")
            .build();
    var context = AgentPromptContext.builder().format(RenderFormat.A2A_CARD).build();

    var result = pipeline.assemble(
            pipeline.buildStage1(descriptor, context),
            Optional.empty(), Optional.empty(),
            descriptor, context);

    assertThat(result.systemPrompt()).doesNotContain("Operating Principles");
}
```

- [ ] **Step 7: Render domain in role section**

In `assembleMarkdownRole()` (line 535–549), add domain after slot rendering:

```java
if (descriptor.domain() != null) {
    sb.append("**Domain:** ").append(descriptor.domain()).append("\n");
}
```

Also add `domain` to the A2A_CARD `buildDescriptorPayload()` method — add to the identity section:
```java
addIfNonBlank(identity, "domain", descriptor.domain());
```

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add eidos-core/
git commit -m "feat(#193): auto-compose briefing from archetype + slot + domain

When briefing is null and archetype + slot are present, compose
structural Operating Principles at render time. Format-discriminated:
MARKDOWN/PROSE compose, A2A_CARD does not. Explicit briefing always
wins. Domain rendered in role section and A2A_CARD identity.

Refs #193

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 6: End-to-end minimal identity test

**Files:**
- Create: `eidos-core/src/test/java/io/casehub/eidos/core/MinimalIdentityTest.java`
- Modify: `eidos-core/src/test/resources/META-INF/eidos/descriptors.yaml` (add minimal test descriptor)

**Interfaces:**
- Consumes: Full pipeline — YAML parsing → DescriptorCollector → EidosRenderPipeline
- Produces: Verification that 6-field YAML produces complete rendered prompt

- [ ] **Step 1: Add minimal descriptor to test YAML**

Add to `eidos-core/src/test/resources/META-INF/eidos/descriptors.yaml`:

```yaml
  - id: minimal-demo
    name: Code Reviewer
    slot: reviewer
    domain: software-engineering
    archetype: sage
    archetypeAdjectives: [cautious, methodical]
    tenancyId: test
```

- [ ] **Step 2: Write end-to-end test**

```java
@QuarkusTest
class MinimalIdentityTest {

    @Inject
    AgentRegistry registry;

    @Inject
    EidosRenderPipeline pipeline;

    @Test
    void minimal_descriptor_produces_full_prompt() {
        var match = registry.findById("minimal-demo", "test");
        assertThat(match).isPresent();
        var descriptor = match.get();

        // Verify auto-derivation happened
        assertThat(descriptor.disposition()).isNotNull();
        assertThat(descriptor.disposition().get(DispositionAxis.SOCIAL_ORIENTATION)).isNotEmpty();

        // Verify domain field
        assertThat(descriptor.domain()).isEqualTo("software-engineering");

        // Render and verify
        var context = AgentPromptContext.builder().format(RenderFormat.MARKDOWN).build();
        var result = pipeline.assemble(
                pipeline.buildStage1(descriptor, context),
                Optional.empty(), Optional.empty(),
                descriptor, context);

        assertThat(result.systemPrompt())
                .contains("Code Reviewer")
                .contains("reviewer")
                .contains("software-engineering")
                .contains("sage")
                .contains("cautious")
                .contains("Operating Principles");
    }
}
```

- [ ] **Step 3: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core -Dtest=MinimalIdentityTest`
Expected: PASS

- [ ] **Step 4: Run full project test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,vocab,eidos-core,persistence-memory,deployment`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add eidos-core/
git commit -m "test(#193): end-to-end minimal identity test

Verify 6-field YAML descriptor (id, name, slot, domain, archetype,
archetypeAdjectives) produces complete rendered prompt with auto-derived
disposition and auto-composed briefing.

Closes #193
Refs #192

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-10-30-archetype-identity-auto-design.md] — design spec this plan implements
- `vocab/src/main/java/io/casehub/eidos/vocab/ArchetypeCompatibility.java:11-252` — hardcoded data to replace
- `vocab/src/main/resources/META-INF/eidos/archetypes.yaml` — YAML data source
- `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java:7-393` — record to extend with domain
- `eidos-core/src/main/java/io/casehub/eidos/core/registrar/DescriptorCollector.java:26-91` — auto-derivation hook
- `eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java:711-786` — auto-briefing hook
- `api/src/main/java/io/casehub/eidos/api/PersonalityTypeDeriver.java` — MBTI → disposition
- `vocab/src/test/java/io/casehub/eidos/vocab/ArchetypeDataValidationTest.java` — validation tests
- GitHub #192 — consolidate archetype data
- GitHub #193 — minimal demo identity
