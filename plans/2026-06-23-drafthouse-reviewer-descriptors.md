# DraftHouse Reviewer Descriptors Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce declarative agent persona registration (SPI + YAML loader), fix CDI displacement for `SystemPromptRenderer`, and validate with 4 DraftHouse reviewer descriptors.

**Architecture:** Declarative `AgentDescriptorRegistrar` SPI (returns `List<AgentDescriptor>`) discovered by a separate `AgentDescriptorBootstrap` bean using `@Observes StartupEvent`. `ClasspathYamlDescriptorRegistrar` reads `META-INF/eidos/descriptors.yaml` from all JARs via `ClassLoader.getResources()`. `EidosSystemPromptRenderer` changes from `@DefaultBean` to plain `@ApplicationScoped` for correct CDI displacement.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson YAML (jackson-dataformat-yaml), JUnit 5, AssertJ

**Spec:** `specs/issue-64-drafthouse-reviewer-descriptors/2026-06-23-drafthouse-reviewer-descriptors-design.md`

## Global Constraints

- Java 21 source, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` (use `mvn` not `./mvnw`)
- All commits reference eidos#64: `Refs eidos#64` or `feat(eidos#64): ...`
- No Flyway migrations — purely API/runtime changes
- `AgentDescriptor.Builder` for test/config code; positional constructor only for JPA mapper (PP-20260608-e694ab)

---

### Task 1: MAX_BRIEFING Increase + ThomasKilmannTerm Alias

Two one-line changes with zero dependencies. Both are prerequisite for later tasks (briefings exceed 500 chars; YAML uses "collaborative" as alias).

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java:24`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`
- Modify: `vocab/src/main/java/io/casehub/eidos/vocab/ThomasKilmannTerm.java:18`
- Test: `vocab/src/test/java/io/casehub/eidos/vocab/ThomasKilmannTermTest.java` (create)

**Interfaces:**
- Consumes: nothing
- Produces: `MAX_BRIEFING = 2000` (used by Task 5 briefings); `ThomasKilmannTerm.COLLABORATING` resolves alias `"collaborative"` (used by Task 5 YAML)

- [ ] **Step 1: Write the failing test for MAX_BRIEFING**

In `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java`, add a test that constructs a descriptor with a 1500-character briefing:

```java
@Test
void briefing_at_1500_chars_is_valid() {
    final String briefing = "A".repeat(1500);
    assertThatNoException().isThrownBy(() ->
        AgentDescriptor.builder()
            .agentId(VALID_ID).name(VALID_NAME).slot(VALID_SLOT).tenancyId(VALID_TID)
            .briefing(briefing)
            .build());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest#briefing_at_1500_chars_is_valid`
Expected: FAIL — `AgentValidationException: briefing exceeds maximum length 500 (was 1500)`

- [ ] **Step 3: Change MAX_BRIEFING from 500 to 2000**

In `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java`, line 24:

```java
    static final int MAX_BRIEFING            = 2000;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorValidatorTest#briefing_at_1500_chars_is_valid`
Expected: PASS

- [ ] **Step 5: Write the failing test for ThomasKilmann alias**

Create `vocab/src/test/java/io/casehub/eidos/vocab/ThomasKilmannTermTest.java`:

```java
package io.casehub.eidos.vocab;

import io.casehub.eidos.api.VocabularyRegistry;
import io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ThomasKilmannTermTest {

    static VocabularyRegistry registry;

    @BeforeAll
    static void setUp() {
        registry = new CdiVocabularyRegistry();
        registry.register(ThomasKilmannTerm.class);
    }

    @Test
    void collaborative_alias_resolves_to_collaborating() {
        var resolved = registry.resolve(ThomasKilmannTerm.URI, "collaborative");
        assertThat(resolved).isPresent();
        assertThat(resolved.get()).isEqualTo(ThomasKilmannTerm.COLLABORATING);
    }

    @Test
    void cooperative_alias_still_resolves() {
        var resolved = registry.resolve(ThomasKilmannTerm.URI, "cooperative");
        assertThat(resolved).isPresent();
        assertThat(resolved.get()).isEqualTo(ThomasKilmannTerm.COLLABORATING);
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ThomasKilmannTermTest#collaborative_alias_resolves_to_collaborating`
Expected: FAIL — `Optional.empty` (no match for "collaborative")

- [ ] **Step 7: Add "collaborative" alias to COLLABORATING**

In `vocab/src/main/java/io/casehub/eidos/vocab/ThomasKilmannTerm.java`, line 18, change:

```java
        List.of("cooperative")),
```

to:

```java
        List.of("cooperative", "collaborative")),
```

- [ ] **Step 8: Run both alias tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl vocab -Dtest=ThomasKilmannTermTest`
Expected: PASS (both tests)

- [ ] **Step 9: Run full api + vocab test suites**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,vocab`
Expected: All tests PASS

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java \
        api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java \
        vocab/src/main/java/io/casehub/eidos/vocab/ThomasKilmannTerm.java \
        vocab/src/test/java/io/casehub/eidos/vocab/ThomasKilmannTermTest.java
git commit -m "feat(eidos#64): increase MAX_BRIEFING to 2000 and add 'collaborative' alias to ThomasKilmannTerm"
```

---

### Task 2: AgentDescriptorRegistrar SPI

New `@FunctionalInterface` in `casehub-eidos-api`. Pure Tier 1 — no CDI, no runtime deps.

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/spi/AgentDescriptorRegistrar.java`
- Test: `api/src/test/java/io/casehub/eidos/api/spi/AgentDescriptorRegistrarTest.java` (create)

**Interfaces:**
- Consumes: `AgentDescriptor`, `AgentRegistry` (both already in api)
- Produces: `AgentDescriptorRegistrar` interface — consumed by Task 3 (bootstrap), Task 4 (YAML registrar)

- [ ] **Step 1: Write the contract test**

Create `api/src/test/java/io/casehub/eidos/api/spi/AgentDescriptorRegistrarTest.java`:

```java
package io.casehub.eidos.api.spi;

import io.casehub.eidos.api.AgentDescriptor;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class AgentDescriptorRegistrarTest {

    @Test
    void registrar_returns_descriptors() {
        AgentDescriptorRegistrar registrar = () -> List.of(
            AgentDescriptor.builder()
                .agentId("test-1").name("Test").slot("tester").tenancyId("t")
                .build()
        );

        var descriptors = registrar.descriptors();
        assertThat(descriptors).hasSize(1);
        assertThat(descriptors.get(0).agentId()).isEqualTo("test-1");
    }

    @Test
    void registrar_is_functional_interface() {
        assertThat(AgentDescriptorRegistrar.class.isAnnotationPresent(FunctionalInterface.class))
            .isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails (class does not exist)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorRegistrarTest`
Expected: Compilation error — `AgentDescriptorRegistrar` not found

- [ ] **Step 3: Create the SPI interface**

Create `api/src/main/java/io/casehub/eidos/api/spi/AgentDescriptorRegistrar.java`:

```java
package io.casehub.eidos.api.spi;

import io.casehub.eidos.api.AgentDescriptor;

import java.util.List;

@FunctionalInterface
public interface AgentDescriptorRegistrar {
    List<AgentDescriptor> descriptors();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorRegistrarTest`
Expected: PASS

- [ ] **Step 5: Run full api test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/spi/AgentDescriptorRegistrar.java \
        api/src/test/java/io/casehub/eidos/api/spi/AgentDescriptorRegistrarTest.java
git commit -m "feat(eidos#64): add AgentDescriptorRegistrar SPI — declarative persona registration"
```

---

### Task 3: CDI Scope Fix + AgentDescriptorBootstrap

Remove `@DefaultBean` from `EidosSystemPromptRenderer` and add the bootstrap bean that auto-discovers registrars.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java:19` (remove `@DefaultBean`)
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrap.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java` (existing — verify still passes)
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrapTest.java` (create)

**Interfaces:**
- Consumes: `AgentDescriptorRegistrar` (Task 2), `AgentRegistry` (api), `StartupEvent` (Quarkus)
- Produces: `AgentDescriptorBootstrap` — wires registrars to registry at startup. `EidosSystemPromptRenderer` now `@ApplicationScoped` (displaces any `@DefaultBean` implementation).

- [ ] **Step 1: Remove @DefaultBean from EidosSystemPromptRenderer**

In `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java`, line 19, remove the `@DefaultBean` annotation. Also remove the import `import io.quarkus.arc.DefaultBean;` at line 9. The class remains `@ApplicationScoped`.

- [ ] **Step 2: Verify existing renderer tests still pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosSystemPromptRendererTest`
Expected: All PASS (tests use the package-private constructor, not CDI)

- [ ] **Step 3: Write the bootstrap duplicate-detection test**

Create `runtime/src/test/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrapTest.java`:

```java
package io.casehub.eidos.runtime.registrar;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.spi.AgentDescriptorRegistrar;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AgentDescriptorBootstrapTest {

    static AgentDescriptor desc(String agentId, String tenancyId) {
        return AgentDescriptor.builder()
            .agentId(agentId).name("N").slot("s").tenancyId(tenancyId).build();
    }

    @Test
    void duplicate_agentId_tenancyId_pair_throws() {
        var registrars = List.<AgentDescriptorRegistrar>of(
            () -> List.of(desc("a1", "t1")),
            () -> List.of(desc("a1", "t1"))
        );

        assertThatThrownBy(() -> AgentDescriptorBootstrap.registerAll(registrars, new ListRegistry()))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Duplicate descriptor")
            .hasMessageContaining("a1")
            .hasMessageContaining("t1");
    }

    @Test
    void same_agentId_different_tenancy_is_allowed() {
        var registry = new ListRegistry();
        var registrars = List.<AgentDescriptorRegistrar>of(
            () -> List.of(desc("a1", "t1")),
            () -> List.of(desc("a1", "t2"))
        );

        AgentDescriptorBootstrap.registerAll(registrars, registry);
        assertThat(registry.registered).hasSize(2);
    }

    @Test
    void empty_registrars_registers_nothing() {
        var registry = new ListRegistry();
        AgentDescriptorBootstrap.registerAll(List.of(), registry);
        assertThat(registry.registered).isEmpty();
    }

    static class ListRegistry implements AgentRegistry {
        final List<AgentDescriptor> registered = new ArrayList<>();

        @Override public void register(AgentDescriptor d) { registered.add(d); }
        @Override public java.util.Optional<AgentDescriptor> findById(String id, String tid) {
            return java.util.Optional.empty();
        }
        @Override public List<AgentDescriptor> find(io.casehub.eidos.api.AgentQuery q) {
            return List.of();
        }
    }
}
```

- [ ] **Step 4: Run test to verify it fails (class does not exist)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentDescriptorBootstrapTest`
Expected: Compilation error — `AgentDescriptorBootstrap` not found

- [ ] **Step 5: Create AgentDescriptorBootstrap**

Create `runtime/src/main/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrap.java`:

```java
package io.casehub.eidos.runtime.registrar;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.spi.AgentDescriptorRegistrar;
import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled",
                 stringValue = "false", enableIfMissing = true)
@ApplicationScoped
public class AgentDescriptorBootstrap {

    @Inject AgentRegistry registry;
    @Inject @Any Instance<AgentDescriptorRegistrar> registrars;

    void onStartup(@Observes StartupEvent ev) {
        registerAll(registrars, registry);
    }

    static void registerAll(Iterable<AgentDescriptorRegistrar> registrars,
                            AgentRegistry registry) {
        final var all = new ArrayList<AgentDescriptor>();
        registrars.forEach(r -> all.addAll(r.descriptors()));

        final var seen = new HashSet<String>();
        for (final var d : all) {
            final var key = d.agentId() + "\0" + d.tenancyId();
            if (!seen.add(key)) {
                throw new IllegalStateException(
                    "Duplicate descriptor: agentId=" + d.agentId()
                    + ", tenancyId=" + d.tenancyId());
            }
        }

        all.forEach(registry::register);
    }
}
```

- [ ] **Step 6: Run bootstrap tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentDescriptorBootstrapTest`
Expected: PASS

- [ ] **Step 7: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java \
        runtime/src/main/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrap.java \
        runtime/src/test/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrapTest.java
git commit -m "feat(eidos#64): remove @DefaultBean from renderer, add AgentDescriptorBootstrap"
```

---

### Task 4: ClasspathYamlDescriptorRegistrar + Maven Dependency

YAML-driven `AgentDescriptorRegistrar` implementation that reads `META-INF/eidos/descriptors.yaml` from all JARs.

**Files:**
- Modify: `runtime/pom.xml` (add jackson-dataformat-yaml dependency)
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java` (create)

**Interfaces:**
- Consumes: `AgentDescriptorRegistrar` (Task 2), `AgentDescriptor.Builder` (api)
- Produces: `ClasspathYamlDescriptorRegistrar` — CDI-discovered; returns descriptors parsed from YAML. Package-private `loadFrom(InputStream)` for unit tests.

- [ ] **Step 1: Add jackson-dataformat-yaml to runtime/pom.xml**

In `runtime/pom.xml`, after the `quarkus-jackson` dependency (around line 69), add:

```xml
    <!-- YAML descriptor loading -->
    <dependency>
      <groupId>com.fasterxml.jackson.dataformat</groupId>
      <artifactId>jackson-dataformat-yaml</artifactId>
    </dependency>
```

(Version managed by Quarkus BOM.)

- [ ] **Step 2: Write the unit tests**

Create `runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java`:

```java
package io.casehub.eidos.runtime.registrar;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.DispositionAxis;
import org.junit.jupiter.api.Test;

import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ClasspathYamlDescriptorRegistrarTest {

    static final ClasspathYamlDescriptorRegistrar registrar = new ClasspathYamlDescriptorRegistrar();

    List<AgentDescriptor> parse(String yaml) {
        return registrar.loadFrom(new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));
    }

    @Test
    void empty_descriptors_list_returns_empty() {
        var result = parse("descriptors: []");
        assertThat(result).isEmpty();
    }

    @Test
    void null_input_returns_empty() {
        var result = registrar.loadFrom(null);
        assertThat(result).isEmpty();
    }

    @Test
    void valid_single_descriptor_maps_all_fields() {
        var yaml = """
            descriptors:
              - agentId: test-1
                name: Test Agent
                slot: reviewer
                tenancyId: default
                version: "1.0"
                disposition:
                  conflictMode: collaborating
                  ruleFollowing: strict
                  delegation: false
                capabilities:
                  - name: code-review
                    tags: [quality]
                    inputTypes: [code]
                    outputTypes: [review]
                briefing: |
                  You are a test agent.
            """;

        var result = parse(yaml);
        assertThat(result).hasSize(1);
        var d = result.get(0);
        assertThat(d.agentId()).isEqualTo("test-1");
        assertThat(d.name()).isEqualTo("Test Agent");
        assertThat(d.slot()).isEqualTo("reviewer");
        assertThat(d.tenancyId()).isEqualTo("default");
        assertThat(d.version()).isEqualTo("1.0");
        assertThat(d.disposition().conflictMode()).isEqualTo("collaborating");
        assertThat(d.disposition().ruleFollowing()).isEqualTo("strict");
        assertThat(d.disposition().delegation()).isFalse();
        assertThat(d.capabilities()).hasSize(1);
        assertThat(d.capabilities().get(0).name()).isEqualTo("code-review");
        assertThat(d.capabilities().get(0).tags()).containsExactly("quality");
        assertThat(d.briefing()).startsWith("You are a test agent.");
    }

    @Test
    void optional_fields_default_to_null() {
        var yaml = """
            descriptors:
              - agentId: minimal
                name: Minimal
                slot: s
                tenancyId: t
            """;

        var result = parse(yaml);
        assertThat(result).hasSize(1);
        var d = result.get(0);
        assertThat(d.version()).isNull();
        assertThat(d.provider()).isNull();
        assertThat(d.briefing()).isNull();
        assertThat(d.disposition()).isNull();
        assertThat(d.capabilities()).isEmpty();
    }

    @Test
    void axis_vocabularies_deserialised_to_enum_keys() {
        var yaml = """
            descriptors:
              - agentId: vocab-test
                name: N
                slot: s
                tenancyId: t
                axisVocabularies:
                  CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
                  RULE_FOLLOWING: urn:casehub:vocab:conscientiousness
            """;

        var result = parse(yaml);
        var axisVocabs = result.get(0).axisVocabularies();
        assertThat(axisVocabs).containsEntry(DispositionAxis.CONFLICT_MODE,
            "urn:casehub:vocab:thomas-kilmann");
        assertThat(axisVocabs).containsEntry(DispositionAxis.RULE_FOLLOWING,
            "urn:casehub:vocab:conscientiousness");
    }

    @Test
    void invalid_axis_vocabulary_key_throws() {
        var yaml = """
            descriptors:
              - agentId: bad
                name: N
                slot: s
                tenancyId: t
                axisVocabularies:
                  INVALID_AXIS: urn:foo
            """;

        assertThatThrownBy(() -> parse(yaml))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void missing_required_field_throws_validation_exception() {
        var yaml = """
            descriptors:
              - name: No ID
                slot: s
                tenancyId: t
            """;

        assertThatThrownBy(() -> parse(yaml))
            .isInstanceOf(io.casehub.eidos.api.AgentValidationException.class)
            .hasMessageContaining("agentId");
    }

    @Test
    void capability_with_epistemic_domains_and_excluded_domains() {
        var yaml = """
            descriptors:
              - agentId: epistemic
                name: N
                slot: s
                tenancyId: t
                capabilities:
                  - name: review
                    epistemicDomains:
                      java: 0.95
                      rust: 0.3
                    excludedDomains: [cobol]
            """;

        var result = parse(yaml);
        var cap = result.get(0).capabilities().get(0);
        assertThat(cap.epistemicDomains()).containsEntry("java", 0.95);
        assertThat(cap.excludedDomains()).containsExactly("cobol");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail (class does not exist)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlDescriptorRegistrarTest`
Expected: Compilation error

- [ ] **Step 4: Create ClasspathYamlDescriptorRegistrar**

Create `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`:

```java
package io.casehub.eidos.runtime.registrar;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.eidos.api.AgentCapability;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.spi.AgentDescriptorRegistrar;
import jakarta.enterprise.context.ApplicationScoped;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Enumeration;
import java.util.List;
import java.util.Map;
import java.util.Set;

@ApplicationScoped
public class ClasspathYamlDescriptorRegistrar implements AgentDescriptorRegistrar {

    private static final String RESOURCE_PATH = "META-INF/eidos/descriptors.yaml";
    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory())
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, true);

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

    List<AgentDescriptor> loadFrom(final InputStream yaml) {
        if (yaml == null) return List.of();
        final DescriptorFile file;
        try {
            file = YAML_MAPPER.readValue(yaml, DescriptorFile.class);
        } catch (final IOException e) {
            throw new IllegalStateException("Failed to parse YAML: " + e.getMessage(), e);
        }
        if (file.descriptors == null || file.descriptors.isEmpty()) return List.of();

        final var result = new ArrayList<AgentDescriptor>(file.descriptors.size());
        for (final var cfg : file.descriptors) {
            result.add(toDescriptor(cfg));
        }
        return result;
    }

    private static AgentDescriptor toDescriptor(final DescriptorConfig cfg) {
        final var builder = AgentDescriptor.builder()
            .agentId(cfg.agentId).name(cfg.name).slot(cfg.slot).tenancyId(cfg.tenancyId)
            .version(cfg.version).provider(cfg.provider)
            .modelFamily(cfg.modelFamily).modelVersion(cfg.modelVersion)
            .weightsFingerprint(cfg.weightsFingerprint)
            .domainVocabulary(cfg.domainVocabulary)
            .slotVocabulary(cfg.slotVocabulary)
            .dispositionVocabulary(cfg.dispositionVocabulary)
            .jurisdiction(cfg.jurisdiction)
            .dataHandlingPolicy(cfg.dataHandlingPolicy)
            .briefing(cfg.briefing);

        if (cfg.axisVocabularies != null && !cfg.axisVocabularies.isEmpty()) {
            final var axisMap = new java.util.LinkedHashMap<DispositionAxis, String>();
            cfg.axisVocabularies.forEach((key, uri) -> axisMap.put(DispositionAxis.valueOf(key), uri));
            builder.axisVocabularies(axisMap);
        }

        if (cfg.disposition != null) {
            builder.disposition(new AgentDisposition(
                cfg.disposition.socialOrient, cfg.disposition.ruleFollowing,
                cfg.disposition.riskAppetite, cfg.disposition.autonomy,
                cfg.disposition.conflictMode, cfg.disposition.delegation));
        }

        if (cfg.capabilities != null) {
            builder.capabilities(cfg.capabilities.stream().map(c ->
                new AgentCapability(c.name, c.qualityHint, c.latencyHintP50Ms, c.costHint,
                    c.inputTypes, c.outputTypes, c.tags, c.epistemicDomains, c.excludedDomains)
            ).toList());
        }

        return builder.build();
    }

    static class DescriptorFile {
        public List<DescriptorConfig> descriptors;
    }

    static class DescriptorConfig {
        public String agentId, name, slot, tenancyId, version, provider,
                      modelFamily, modelVersion, weightsFingerprint,
                      domainVocabulary, slotVocabulary, dispositionVocabulary,
                      jurisdiction, dataHandlingPolicy, briefing;
        public Map<String, String> axisVocabularies;
        public DispositionConfig disposition;
        public List<CapabilityConfig> capabilities;
    }

    static class DispositionConfig {
        public String socialOrient, ruleFollowing, riskAppetite, autonomy, conflictMode;
        public boolean delegation;
    }

    static class CapabilityConfig {
        public String name;
        public Double qualityHint;
        public Long latencyHintP50Ms;
        public String costHint;
        public List<String> inputTypes, outputTypes, tags;
        public Map<String, Double> epistemicDomains;
        public Set<String> excludedDomains;
    }
}
```

- [ ] **Step 5: Run YAML registrar tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlDescriptorRegistrarTest`
Expected: All PASS

- [ ] **Step 6: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/pom.xml \
        runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java \
        runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java
git commit -m "feat(eidos#64): add ClasspathYamlDescriptorRegistrar — YAML-driven persona loading"
```

---

### Task 5: DraftHouse Reviewer YAML + Scenario Test

The end-to-end integration: 4 DraftHouse reviewer descriptors defined in YAML, auto-registered at startup, rendered to MARKDOWN structural prompts.

**Files:**
- Create: `examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml`
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DraftHouseReviewerScenarioTest.java`

**Interfaces:**
- Consumes: `AgentDescriptorBootstrap` (Task 3), `ClasspathYamlDescriptorRegistrar` (Task 4), `InMemoryAgentRegistry` (persistence-memory), `EidosSystemPromptRenderer` (runtime), `CdiVocabularyRegistry` + TK/Conscientiousness vocabs (vocab)
- Produces: End-to-end validation — no downstream tasks

- [ ] **Step 1: Create the YAML persona file**

Create `examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml`:

```yaml
descriptors:
  - agentId: drafthouse-structural-reviewer
    name: Structural Reviewer
    slot: document-reviewer
    tenancyId: drafthouse
    axisVocabularies:
      CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
      RULE_FOLLOWING: urn:casehub:vocab:conscientiousness
    disposition:
      conflictMode: collaborating
      ruleFollowing: strict
      delegation: false
    capabilities:
      - name: document-review
        tags: [structural]
    briefing: |
      You are the Structural Reviewer for DraftHouse. Your focus is document
      organisation, logical flow, and section coherence. You verify that headings
      form a navigable hierarchy, that each section has a clear purpose, and that
      the document can be read top-to-bottom without backtracking.

      Approach reviews collaboratively — flag structural issues with specific
      suggestions for reordering, splitting, or merging sections. Follow strict
      structural conventions when they exist; propose new ones when they do not.

  - agentId: drafthouse-content-reviewer
    name: Content Reviewer
    slot: document-reviewer
    tenancyId: drafthouse
    axisVocabularies:
      CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
      RISK_APPETITE: urn:casehub:vocab:conscientiousness
    disposition:
      conflictMode: competing
      riskAppetite: conservative
      delegation: false
    capabilities:
      - name: document-review
        tags: [content]
    briefing: |
      You are the Content Reviewer for DraftHouse. Your focus is factual accuracy,
      technical correctness, and completeness of claims. You challenge assertions
      that lack evidence, flag unsupported conclusions, and verify that code
      examples match the described behaviour.

      Take a competing stance — accuracy is non-negotiable. Be conservative with
      what you accept as correct; demand evidence for technical claims. Missing
      citations or unverified assertions are findings, not suggestions.

  - agentId: drafthouse-readability-reviewer
    name: Readability Reviewer
    slot: document-reviewer
    tenancyId: drafthouse
    axisVocabularies:
      CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
      AUTONOMY: urn:casehub:vocab:conscientiousness
    disposition:
      conflictMode: accommodating
      autonomy: directed
      delegation: false
    capabilities:
      - name: document-review
        tags: [readability]
    briefing: |
      You are the Readability Reviewer for DraftHouse. Your focus is clarity,
      sentence structure, jargon density, and audience appropriateness. You ensure
      the document communicates its ideas to the intended reader without
      unnecessary complexity.

      Accommodate the author's voice and style — suggest improvements that
      preserve their intent while improving clarity. Follow the project's style
      guide when one exists; defer to the author when it does not.

  - agentId: drafthouse-completeness-reviewer
    name: Completeness Reviewer
    slot: document-reviewer
    tenancyId: drafthouse
    axisVocabularies:
      CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
      RULE_FOLLOWING: urn:casehub:vocab:conscientiousness
    disposition:
      conflictMode: collaborating
      ruleFollowing: strict
      delegation: false
    capabilities:
      - name: document-review
        tags: [completeness]
    briefing: |
      You are the Completeness Reviewer for DraftHouse. Your focus is coverage —
      whether the document addresses all stated goals, whether edge cases are
      discussed, whether prerequisites and dependencies are documented, and whether
      the reader has enough information to act without external reference.

      Work collaboratively — missing coverage is often unintentional. Frame
      findings as gaps to fill rather than failures. Follow strict checklist
      conventions when they exist for the document type.
```

- [ ] **Step 2: Write the scenario test**

Create `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DraftHouseReviewerScenarioTest.java`:

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DraftHouseReviewerScenarioTest {

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;

    // ── Registration (YAML-driven, no @BeforeEach) ────────────────────────────

    @Test
    void all_four_reviewers_found_by_slot() {
        var reviewers = registry.find(AgentQuery.bySlot("document-reviewer", "drafthouse"));
        assertThat(reviewers).hasSize(4);
        assertThat(reviewers.stream().map(AgentDescriptor::agentId).toList())
            .containsExactlyInAnyOrder(
                "drafthouse-structural-reviewer",
                "drafthouse-content-reviewer",
                "drafthouse-readability-reviewer",
                "drafthouse-completeness-reviewer");
    }

    @Test
    void all_four_reviewers_found_by_capability() {
        var reviewers = registry.find(AgentQuery.byCapability("document-review", "drafthouse"));
        assertThat(reviewers).hasSize(4);
    }

    @ParameterizedTest
    @ValueSource(strings = {
        "drafthouse-structural-reviewer",
        "drafthouse-content-reviewer",
        "drafthouse-readability-reviewer",
        "drafthouse-completeness-reviewer"
    })
    void each_reviewer_found_by_id(String agentId) {
        var desc = registry.findById(agentId, "drafthouse");
        assertThat(desc).isPresent();
        assertThat(desc.get().slot()).isEqualTo("document-reviewer");
        assertThat(desc.get().tenancyId()).isEqualTo("drafthouse");
        assertThat(desc.get().briefing()).isNotNull().isNotBlank();
        assertThat(desc.get().capabilities()).hasSize(1);
        assertThat(desc.get().capabilities().get(0).name()).isEqualTo("document-review");
    }

    @Test
    void tenancy_isolation_from_default() {
        var defaultReviewers = registry.find(
            AgentQuery.bySlot("document-reviewer", "default"));
        assertThat(defaultReviewers).isEmpty();
    }

    // ── Renderer (structural MARKDOWN — no LLM in examples module) ──────────

    @ParameterizedTest
    @ValueSource(strings = {
        "drafthouse-structural-reviewer",
        "drafthouse-content-reviewer",
        "drafthouse-readability-reviewer",
        "drafthouse-completeness-reviewer"
    })
    void rendered_output_contains_identity_and_capability(String agentId) {
        var desc = registry.findById(agentId, "drafthouse").orElseThrow();
        var ctx = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
        var result = renderer.render(desc, ctx);

        assertThat(result.content()).contains(desc.name());
        assertThat(result.content()).contains("document-review");
        assertThat(result.format()).isEqualTo(RenderFormat.MARKDOWN);
    }

    @Test
    void structural_reviewer_shows_thomas_kilmann_label() {
        var desc = registry.findById("drafthouse-structural-reviewer", "drafthouse").orElseThrow();
        var result = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));

        assertThat(result.content()).contains("Collaborating (Thomas-Kilmann Conflict Modes)");
    }

    @Test
    void content_reviewer_shows_competing_label() {
        var desc = registry.findById("drafthouse-content-reviewer", "drafthouse").orElseThrow();
        var result = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));

        assertThat(result.content()).contains("Competing (Thomas-Kilmann Conflict Modes)");
    }

    @Test
    void readability_reviewer_shows_accommodating_label() {
        var desc = registry.findById("drafthouse-readability-reviewer", "drafthouse").orElseThrow();
        var result = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));

        assertThat(result.content()).contains("Accommodating (Thomas-Kilmann Conflict Modes)");
    }

    @ParameterizedTest
    @ValueSource(strings = {
        "drafthouse-structural-reviewer",
        "drafthouse-content-reviewer",
        "drafthouse-readability-reviewer",
        "drafthouse-completeness-reviewer"
    })
    void rendered_output_has_structural_markdown_sections(String agentId) {
        var desc = registry.findById(agentId, "drafthouse").orElseThrow();
        var result = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));

        assertThat(result.content()).contains("## Role");
        assertThat(result.content()).contains("## How You Operate");
        assertThat(result.content()).contains("## Operating Principles");
    }

    @Test
    void structural_reviewer_briefing_in_rendered_output() {
        var desc = registry.findById("drafthouse-structural-reviewer", "drafthouse").orElseThrow();
        var result = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));

        assertThat(result.content()).contains("logical flow");
    }

    @Test
    void content_reviewer_briefing_in_rendered_output() {
        var desc = registry.findById("drafthouse-content-reviewer", "drafthouse").orElseThrow();
        var result = renderer.render(desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));

        assertThat(result.content()).contains("factual accuracy");
    }
}
```

- [ ] **Step 3: Run scenario tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=DraftHouseReviewerScenarioTest`
Expected: All PASS

- [ ] **Step 4: Run full examples test suite to check no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios`
Expected: All PASS (including existing MultiAgentTeamTest, SystemPromptRendererTest, etc.)

- [ ] **Step 5: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and pass tests

- [ ] **Step 6: Commit**

```bash
git add examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml \
        examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DraftHouseReviewerScenarioTest.java
git commit -m "feat(eidos#64): add DraftHouse reviewer YAML personas and scenario test"
```

---

### Task 6: File Follow-Up Issues

File the deferred issues identified in the spec before closing out.

**Files:** None (GitHub issue management only)

- [ ] **Step 1: File CdiVocabularyRegistry @DefaultBean fix issue**

```bash
gh issue create --repo casehubio/eidos \
  --title "fix: CdiVocabularyRegistry @DefaultBean → @ApplicationScoped" \
  --body "Same CDI displacement risk as EidosSystemPromptRenderer (fixed in eidos#64). CdiVocabularyRegistry is @DefaultBean @ApplicationScoped — should be plain @ApplicationScoped for correct displacement when a consumer ships a @DefaultBean VocabularyRegistry.

Refs eidos#64

**Scale:** XS
**Complexity:** Low"
```

- [ ] **Step 2: File ARC42STORIES stale VocabularyRegistrar descriptions issue**

```bash
gh issue create --repo casehubio/eidos \
  --title "docs: fix stale VocabularyRegistrar descriptions in ARC42STORIES.MD" \
  --body "Lines 709, 714, and 724 describe the old imperative VocabularyRegistrar pattern. Actual source (VocabularyRegistrar.java:12) is declarative: \`Class<? extends Enum<? extends VocabularyTerm>> vocabulary()\`.

- Line 709: says \`void register(VocabularyRegistry registry)\`
- Line 714: says \`registrar.register(this)\` — actual code is \`registerRaw(r.vocabulary())\`
- Line 724: says \`domain apps write registry.register()\` — domain apps implement VocabularyRegistrar and return the class

Identified during eidos#64 spec review.

**Scale:** XS
**Complexity:** Low"
```

- [ ] **Step 3: File reactive-mode bootstrap issue**

```bash
gh issue create --repo casehubio/eidos \
  --title "feat: reactive-mode AgentDescriptorBootstrap" \
  --body "AgentDescriptorBootstrap (eidos#64) is gated to blocking mode via @IfBuildProperty. Reactive-only deployments without casehub-eidos-memory have no AgentRegistry bean, so the blocking bootstrap cannot run.

A reactive bootstrap using ReactiveAgentRegistry.register() (returns Uni<Void>) is needed for reactive-only deployments that want YAML-driven persona registration.

Refs eidos#64

**Scale:** S
**Complexity:** Med"
```

- [ ] **Step 4: Commit (no files — issue numbers recorded)**

No git commit needed — this step creates GitHub issues only.
