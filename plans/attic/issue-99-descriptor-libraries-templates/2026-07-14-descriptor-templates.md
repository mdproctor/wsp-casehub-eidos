# Descriptor Templates Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #99 — Descriptor libraries and templates — reusable prose for agent personalities
**Issue group:** #99

**Goal:** Add reusable prose templates that compose into agent system prompts, eliminating briefing duplication across descriptors that share genre or role conventions.

**Architecture:** Full SPI pattern in api (DescriptorTemplate, TemplateRef, TemplateRegistry, TemplateRegistrar), CdiTemplateRegistry in runtime with @PostConstruct self-population, ClasspathYamlTemplateRegistrar for classpath loading, InMemoryTemplateRegistry in persistence-memory. Templates are identity — declared on AgentDescriptor, resolved at render time by EidosRenderPipeline.

**Tech Stack:** Java 21, Quarkus 3.32, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- Java 21 source on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn clean install` from project root
- Tests: `mvn test -pl <module>`
- IntelliJ MCP mandatory for all .java edits — use `ide_create_file`, `ide_insert_member`, `ide_edit_member`, `ide_replace_member`
- `AgentDescriptorValidator` is package-private — new constants must be `static final` with package access
- Compact constructor validation pattern: records validate in their compact constructor via `AgentDescriptorValidator`
- `FAIL_ON_UNKNOWN_PROPERTIES = true` on YAML mapper — unknown YAML fields are hard errors
- Pre-release: breaking changes to `AgentDescriptor` record (new field) are fine
- All validation failures are `AgentValidationException` (hard errors, fail-fast)
- Flyway next version: V7

---

### Task 1: API types — DescriptorTemplate, TemplateRef, validation constants

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/DescriptorTemplate.java`
- Create: `api/src/main/java/io/casehub/eidos/api/TemplateRef.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` — add constants
- Create: `api/src/test/java/io/casehub/eidos/api/DescriptorTemplateTest.java`
- Create: `api/src/test/java/io/casehub/eidos/api/TemplateRefTest.java`

**Interfaces:**
- Produces: `DescriptorTemplate(String id, String name, List<String> parameters, String content)` record
- Produces: `TemplateRef(String templateId, Map<String, String> args)` record
- Produces: `AgentDescriptorValidator.MAX_TEMPLATE_ID` (100), `MAX_TEMPLATE_NAME` (200), `MAX_TEMPLATE_CONTENT` (4000), `MAX_PARAMETER_NAME` (100)

- [ ] **Step 1: Add validation constants to AgentDescriptorValidator**

Add to `AgentDescriptorValidator.java` after `MAX_BRIEFING`:

```java
static final int MAX_TEMPLATE_ID      = 100;
static final int MAX_TEMPLATE_NAME    = 200;
static final int MAX_TEMPLATE_CONTENT = 4000;
static final int MAX_PARAMETER_NAME   = 100;
```

Also add the `validateRequired` varargs overload (mirrors `validateOptional` varargs):

```java
static void validateRequired(final String fieldName, final String value, final int maxLength,
                              final int... allowedCodePoints) {
    validateField(fieldName, value, maxLength, allowedCodePoints);
}
```

- [ ] **Step 2: Write failing tests for DescriptorTemplate**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class DescriptorTemplateTest {

    @Test void valid_static_template() {
        var t = new DescriptorTemplate("style-guide", "Style Guide", List.of(), "You follow these conventions.");
        assertThat(t.id()).isEqualTo("style-guide");
        assertThat(t.name()).isEqualTo("Style Guide");
        assertThat(t.parameters()).isEmpty();
        assertThat(t.content()).isEqualTo("You follow these conventions.");
    }

    @Test void valid_parameterized_template() {
        var t = new DescriptorTemplate("villain", "Villain", List.of("catchphrase", "nemesis"),
            "Your catchphrase is \"${catchphrase}\". Your nemesis is ${nemesis}.");
        assertThat(t.parameters()).containsExactly("catchphrase", "nemesis");
    }

    @Test void null_parameters_defaulted_to_empty() {
        var t = new DescriptorTemplate("x", "X", null, "content");
        assertThat(t.parameters()).isEmpty();
    }

    @Test void parameters_are_immutable() {
        var params = new java.util.ArrayList<>(List.of("a", "b"));
        var t = new DescriptorTemplate("x", "X", params, "content");
        assertThatThrownBy(() -> t.parameters().add("c")).isInstanceOf(UnsupportedOperationException.class);
    }

    @Test void null_id_throws() {
        assertThatThrownBy(() -> new DescriptorTemplate(null, "X", List.of(), "content"))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName()).isEqualTo("template.id"));
    }

    @Test void blank_id_throws() {
        assertThatThrownBy(() -> new DescriptorTemplate("  ", "X", List.of(), "content"))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void null_content_throws() {
        assertThatThrownBy(() -> new DescriptorTemplate("x", "X", List.of(), null))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName()).isEqualTo("template.content"));
    }

    @Test void content_allows_newlines() {
        assertThatNoException().isThrownBy(() ->
            new DescriptorTemplate("x", "X", List.of(), "line one\nline two\nline three"));
    }

    @Test void content_rejects_over_4000_chars() {
        assertThatThrownBy(() -> new DescriptorTemplate("x", "X", List.of(), "x".repeat(4001)))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void id_rejects_over_100_chars() {
        assertThatThrownBy(() -> new DescriptorTemplate("x".repeat(101), "X", List.of(), "content"))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void content_rejects_bidi_override() {
        assertThatThrownBy(() -> new DescriptorTemplate("x", "X", List.of(), "text‪hidden"))
            .isInstanceOf(AgentValidationException.class);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=DescriptorTemplateTest`
Expected: Compilation failure — `DescriptorTemplate` does not exist

- [ ] **Step 4: Implement DescriptorTemplate**

```java
package io.casehub.eidos.api;

import java.util.List;

public record DescriptorTemplate(
        String id,
        String name,
        List<String> parameters,
        String content
) {
    public DescriptorTemplate {
        AgentDescriptorValidator.validateRequired("template.id", id, AgentDescriptorValidator.MAX_TEMPLATE_ID);
        AgentDescriptorValidator.validateRequired("template.name", name, AgentDescriptorValidator.MAX_TEMPLATE_NAME);
        AgentDescriptorValidator.validateRequired("template.content", content,
                AgentDescriptorValidator.MAX_TEMPLATE_CONTENT, 0x000A);
        parameters = parameters != null ? List.copyOf(parameters) : List.of();
        AgentDescriptorValidator.validateItems("template.parameters", parameters,
                AgentDescriptorValidator.MAX_PARAMETER_NAME);
    }
}
```

- [ ] **Step 5: Run DescriptorTemplate tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=DescriptorTemplateTest`
Expected: All PASS

- [ ] **Step 6: Write failing tests for TemplateRef**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class TemplateRefTest {

    @Test void valid_ref_no_args() {
        var ref = new TemplateRef("style-guide", Map.of());
        assertThat(ref.templateId()).isEqualTo("style-guide");
        assertThat(ref.args()).isEmpty();
    }

    @Test void valid_ref_with_args() {
        var ref = new TemplateRef("villain", Map.of("catchphrase", "Nyah-ha-ha!"));
        assertThat(ref.args()).containsEntry("catchphrase", "Nyah-ha-ha!");
    }

    @Test void null_args_defaulted_to_empty() {
        var ref = new TemplateRef("x", null);
        assertThat(ref.args()).isEmpty();
    }

    @Test void args_are_immutable() {
        var args = new java.util.HashMap<>(Map.of("k", "v"));
        var ref = new TemplateRef("x", args);
        assertThatThrownBy(() -> ref.args().put("k2", "v2")).isInstanceOf(UnsupportedOperationException.class);
    }

    @Test void null_templateId_throws() {
        assertThatThrownBy(() -> new TemplateRef(null, Map.of()))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName()).isEqualTo("templateRef.templateId"));
    }

    @Test void arg_value_rejects_over_1000_chars() {
        assertThatThrownBy(() -> new TemplateRef("x", Map.of("k", "v".repeat(1001))))
            .isInstanceOf(AgentValidationException.class);
    }
}
```

- [ ] **Step 7: Implement TemplateRef**

```java
package io.casehub.eidos.api;

import java.util.Map;

public record TemplateRef(
        String templateId,
        Map<String, String> args
) {
    static final int MAX_TEMPLATE_ARG_VALUE = 1000;

    public TemplateRef {
        AgentDescriptorValidator.validateRequired("templateRef.templateId", templateId,
                AgentDescriptorValidator.MAX_TEMPLATE_ID);
        args = args != null ? Map.copyOf(args) : Map.of();
        AgentDescriptorValidator.validateMapKeys("templateRef.args", args.keySet(),
                AgentDescriptorValidator.MAX_PARAMETER_NAME);
        AgentDescriptorValidator.validateItems("templateRef.args.values", args.values(),
                MAX_TEMPLATE_ARG_VALUE);
    }
}
```

- [ ] **Step 8: Run all api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: All PASS (including existing tests — no regressions)

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/DescriptorTemplate.java \
        api/src/main/java/io/casehub/eidos/api/TemplateRef.java \
        api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java \
        api/src/test/java/io/casehub/eidos/api/DescriptorTemplateTest.java \
        api/src/test/java/io/casehub/eidos/api/TemplateRefTest.java
git commit -m "feat(#99): add DescriptorTemplate and TemplateRef records with validation"
```

---

### Task 2: SPI interfaces — TemplateRegistry, TemplateRegistrar

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/TemplateRegistry.java`
- Create: `api/src/main/java/io/casehub/eidos/api/spi/TemplateRegistrar.java`

**Interfaces:**
- Consumes: `DescriptorTemplate` from Task 1
- Produces: `TemplateRegistry { void register(DescriptorTemplate); Optional<DescriptorTemplate> resolve(String id); List<DescriptorTemplate> all(); }`
- Produces: `TemplateRegistrar { List<DescriptorTemplate> templates(); }`

- [ ] **Step 1: Create TemplateRegistry SPI**

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Optional;

public interface TemplateRegistry {
    void register(DescriptorTemplate template);
    Optional<DescriptorTemplate> resolve(String id);
    List<DescriptorTemplate> all();
}
```

- [ ] **Step 2: Create TemplateRegistrar CDI SPI**

```java
package io.casehub.eidos.api.spi;

import io.casehub.eidos.api.DescriptorTemplate;
import java.util.List;

@FunctionalInterface
public interface TemplateRegistrar {
    List<DescriptorTemplate> templates();
}
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: All PASS

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/TemplateRegistry.java \
        api/src/main/java/io/casehub/eidos/api/spi/TemplateRegistrar.java
git commit -m "feat(#99): add TemplateRegistry SPI and TemplateRegistrar CDI SPI"
```

---

### Task 3: AgentDescriptor — add templates field

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — new field, builder method, compact constructor validation
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java` — new tests
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java` — include templates in drift detection
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java` — new test

**Interfaces:**
- Consumes: `TemplateRef` from Task 1
- Produces: `AgentDescriptor.templates()` — `List<TemplateRef>`, nullable
- Produces: `AgentDescriptor.Builder.templates(List<TemplateRef>)` — builder method

**BREAKING CHANGE:** `AgentDescriptor` record gains a 19th component. All direct constructor calls (in mapper, tests, YAML registrar) must be updated. The builder is unaffected. This is fine — pre-release.

- [ ] **Step 1: Write failing tests**

Add to `AgentDescriptorTest.java`:

```java
@Test void templates_null_by_default() {
    var d = minimal("a1", "t1");
    assertThat(d.templates()).isNull();
}

@Test void templates_round_trips_through_builder() {
    var ref = new TemplateRef("style-guide", Map.of());
    var d = AgentDescriptor.builder()
        .agentId("a1").name("n").slot("s").tenancyId("t")
        .templates(List.of(ref))
        .build();
    assertThat(d.templates()).containsExactly(ref);
}

@Test void templates_are_immutable_when_set() {
    var refs = new java.util.ArrayList<>(List.of(new TemplateRef("x", Map.of())));
    var d = AgentDescriptor.builder()
        .agentId("a1").name("n").slot("s").tenancyId("t")
        .templates(refs)
        .build();
    assertThatThrownBy(() -> d.templates().add(new TemplateRef("y", Map.of())))
        .isInstanceOf(UnsupportedOperationException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorTest`
Expected: Compilation failure — `templates()` not on record

- [ ] **Step 3: Add templates field to AgentDescriptor**

Add `List<TemplateRef> templates` as the 19th field on the record (after `briefing`). Update the compact constructor to handle null → `List.copyOf()`. Add `templates(List<TemplateRef>)` to the Builder. Update `build()` to include templates.

Every direct constructor call site must be updated — use `ide_find_references` on the `AgentDescriptor` constructor to find all callers, then fix each one by adding `null` (or the correct value) as the 19th argument.

- [ ] **Step 4: Run api tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: All PASS

- [ ] **Step 5: Update AgentDescriptorComparator**

Add templates comparison to `compareSimpleFields()`. The comparison logic: convert template refs to a comparable string representation (JSON or toString) and diff. Increment `COMPARED_FIELD_COUNT` from 16 to 17.

Add test to `AgentDescriptorComparatorTest`:

```java
@Test void templatesDrifted() {
    var ref = new TemplateRef("style-guide", Map.of());
    var result = AgentDescriptorComparator.compare(
        base(),
        withField(b -> b.templates(List.of(ref))));
    assertThat(result.matches()).isFalse();
    assertThat(result.drifts()).extracting("field").contains("templates");
}
```

- [ ] **Step 6: Run full api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: All PASS (structural sync test `comparatorCoversAllDescriptorComponents` verifies completeness)

- [ ] **Step 7: Fix compilation in downstream modules**

The record change breaks `runtime` and `persistence-memory` modules. Run `mvn compile` to find all sites. Fix each:
- `AgentDescriptorMapper.toRecord()` — add `null` for templates (or read from entity)
- `ClasspathYamlDescriptorRegistrar.toDescriptor()` — add `null` for templates (YAML parsing comes in Task 5)
- Any test helpers or builder calls in runtime/examples

- [ ] **Step 8: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All PASS across all modules

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#99): add templates field to AgentDescriptor record"
```

---

### Task 4: CdiTemplateRegistry and InMemoryTemplateRegistry

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/template/CdiTemplateRegistry.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/template/CdiTemplateRegistryTest.java`
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryTemplateRegistry.java`
- Create: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryTemplateRegistryTest.java`

**Interfaces:**
- Consumes: `TemplateRegistry`, `TemplateRegistrar`, `DescriptorTemplate` from Tasks 1-2
- Produces: `CdiTemplateRegistry` — `@ApplicationScoped`, `@PostConstruct` self-populating, `Instance<TemplateRegistrar>` discovery
- Produces: `InMemoryTemplateRegistry` — `@Alternative @Priority(1)`, `ConcurrentHashMap` backed

- [ ] **Step 1: Write failing tests for InMemoryTemplateRegistry**

```java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.DescriptorTemplate;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class InMemoryTemplateRegistryTest {

    @Test void register_and_resolve() {
        var registry = new InMemoryTemplateRegistry();
        var t = new DescriptorTemplate("style", "Style", List.of(), "content");
        registry.register(t);
        assertThat(registry.resolve("style")).contains(t);
    }

    @Test void resolve_unknown_returns_empty() {
        var registry = new InMemoryTemplateRegistry();
        assertThat(registry.resolve("nope")).isEmpty();
    }

    @Test void all_returns_registered_templates() {
        var registry = new InMemoryTemplateRegistry();
        var t1 = new DescriptorTemplate("a", "A", List.of(), "content a");
        var t2 = new DescriptorTemplate("b", "B", List.of(), "content b");
        registry.register(t1);
        registry.register(t2);
        assertThat(registry.all()).containsExactlyInAnyOrder(t1, t2);
    }

    @Test void duplicate_id_throws() {
        var registry = new InMemoryTemplateRegistry();
        registry.register(new DescriptorTemplate("dup", "A", List.of(), "a"));
        assertThatThrownBy(() -> registry.register(new DescriptorTemplate("dup", "B", List.of(), "b")))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("dup");
    }

    @Test void placeholder_validation_rejects_undeclared_parameter() {
        var registry = new InMemoryTemplateRegistry();
        var t = new DescriptorTemplate("bad", "Bad", List.of("name"),
            "Hello ${name}, your nemesis is ${nemesis}.");
        assertThatThrownBy(() -> registry.register(t))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("nemesis");
    }

    @Test void placeholder_validation_accepts_matching_params() {
        var registry = new InMemoryTemplateRegistry();
        var t = new DescriptorTemplate("ok", "OK", List.of("name", "nemesis"),
            "Hello ${name}, your nemesis is ${nemesis}.");
        assertThatNoException().isThrownBy(() -> registry.register(t));
    }

    @Test void static_template_with_no_placeholders_passes() {
        var registry = new InMemoryTemplateRegistry();
        var t = new DescriptorTemplate("plain", "Plain", List.of(), "No variables here.");
        assertThatNoException().isThrownBy(() -> registry.register(t));
    }
}
```

- [ ] **Step 2: Implement InMemoryTemplateRegistry**

```java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.DescriptorTemplate;
import io.casehub.eidos.api.TemplateRegistry;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.regex.Pattern;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryTemplateRegistry implements TemplateRegistry {

    private static final Pattern PLACEHOLDER = Pattern.compile("\\$\\{([^}]+)}");
    private final ConcurrentHashMap<String, DescriptorTemplate> store = new ConcurrentHashMap<>();

    @Override
    public void register(DescriptorTemplate template) {
        validatePlaceholders(template);
        if (store.putIfAbsent(template.id(), template) != null) {
            throw new IllegalStateException("Duplicate template ID: " + template.id());
        }
    }

    @Override
    public Optional<DescriptorTemplate> resolve(String id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<DescriptorTemplate> all() {
        return List.copyOf(store.values());
    }

    public void clear() { store.clear(); }

    static void validatePlaceholders(DescriptorTemplate template) {
        var matcher = PLACEHOLDER.matcher(template.content());
        var declared = Set.copyOf(template.parameters());
        var undeclared = new ArrayList<String>();
        while (matcher.find()) {
            var param = matcher.group(1);
            if (!declared.contains(param)) undeclared.add(param);
        }
        if (!undeclared.isEmpty()) {
            throw new IllegalStateException("Template '" + template.id()
                + "' has undeclared placeholder(s): " + undeclared);
        }
    }
}
```

- [ ] **Step 3: Run InMemoryTemplateRegistry tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryTemplateRegistryTest`
Expected: All PASS

- [ ] **Step 4: Implement CdiTemplateRegistry**

```java
package io.casehub.eidos.runtime.template;

import io.casehub.eidos.api.DescriptorTemplate;
import io.casehub.eidos.api.TemplateRegistry;
import io.casehub.eidos.api.spi.TemplateRegistrar;
import io.casehub.eidos.memory.InMemoryTemplateRegistry;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Optional;

@ApplicationScoped
public class CdiTemplateRegistry implements TemplateRegistry {

    @Inject @Any Instance<TemplateRegistrar> registrars;

    private final InMemoryTemplateRegistry delegate = new InMemoryTemplateRegistry();

    @PostConstruct
    void init() {
        for (TemplateRegistrar r : registrars) {
            r.templates().forEach(delegate::register);
        }
    }

    @Override public void register(DescriptorTemplate template) { delegate.register(template); }
    @Override public Optional<DescriptorTemplate> resolve(String id) { return delegate.resolve(id); }
    @Override public List<DescriptorTemplate> all() { return delegate.all(); }
}
```

Note: `CdiTemplateRegistry` delegates to `InMemoryTemplateRegistry` for storage and validation logic. This avoids duplicating the placeholder validation and duplicate-ID detection. The `@PostConstruct` pattern follows `CdiVocabularyRegistry.init()`.

- [ ] **Step 5: Write CdiTemplateRegistry test**

```java
package io.casehub.eidos.runtime.template;

import io.casehub.eidos.api.DescriptorTemplate;
import io.casehub.eidos.api.TemplateRegistry;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class CdiTemplateRegistryTest {

    @Inject TemplateRegistry registry;

    @Test void registry_is_injectable() {
        assertThat(registry).isNotNull();
    }

    @Test void programmatic_register_and_resolve() {
        var t = new DescriptorTemplate("test-cdi", "Test CDI", List.of(), "test content");
        registry.register(t);
        assertThat(registry.resolve("test-cdi")).contains(t);
    }
}
```

- [ ] **Step 6: Run runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CdiTemplateRegistryTest`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#99): implement CdiTemplateRegistry and InMemoryTemplateRegistry"
```

---

### Task 5: YAML loading — ClasspathYamlTemplateRegistrar + descriptor template refs

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/template/ClasspathYamlTemplateRegistrar.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/template/ClasspathYamlTemplateRegistrarTest.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java` — parse template refs
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java` — template ref tests
- Create: `runtime/src/test/resources/META-INF/eidos/templates.yaml` — test fixture

**Interfaces:**
- Consumes: `DescriptorTemplate`, `TemplateRef`, `TemplateRegistrar` from Tasks 1-2
- Produces: `ClasspathYamlTemplateRegistrar` — `@ApplicationScoped`, loads from `META-INF/eidos/templates.yaml`
- Produces: `ClasspathYamlDescriptorRegistrar.TemplateRefConfig` — YAML config class mapping `ref` → `templateId`

- [ ] **Step 1: Create test YAML fixture**

Create `runtime/src/test/resources/META-INF/eidos/templates.yaml`:

```yaml
templates:
  - id: test-style-guide
    name: Test Style Guide
    content: |
      You follow the test style conventions.
      Always be explicit about your intentions.

  - id: test-role-template
    name: Test Role Template
    parameters: [role_name, communication_style]
    content: |
      As a ${role_name}, you communicate in a ${communication_style} manner.
```

- [ ] **Step 2: Write failing tests for ClasspathYamlTemplateRegistrar**

```java
package io.casehub.eidos.runtime.template;

import org.junit.jupiter.api.Test;
import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;
import static org.assertj.core.api.Assertions.*;

class ClasspathYamlTemplateRegistrarTest {

    @Test void loads_static_template_from_yaml() {
        var yaml = """
            templates:
              - id: style
                name: Style Guide
                content: "Follow these conventions."
            """;
        var registrar = new ClasspathYamlTemplateRegistrar();
        var templates = registrar.loadFrom(new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));
        assertThat(templates).hasSize(1);
        assertThat(templates.get(0).id()).isEqualTo("style");
        assertThat(templates.get(0).parameters()).isEmpty();
    }

    @Test void loads_parameterized_template_from_yaml() {
        var yaml = """
            templates:
              - id: villain
                name: Villain
                parameters: [catchphrase, nemesis]
                content: "Your catchphrase is \\"${catchphrase}\\". Nemesis: ${nemesis}."
            """;
        var registrar = new ClasspathYamlTemplateRegistrar();
        var templates = registrar.loadFrom(new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));
        assertThat(templates).hasSize(1);
        assertThat(templates.get(0).parameters()).containsExactly("catchphrase", "nemesis");
    }

    @Test void empty_file_returns_empty_list() {
        var registrar = new ClasspathYamlTemplateRegistrar();
        var templates = registrar.loadFrom(new ByteArrayInputStream("templates:\n".getBytes(StandardCharsets.UTF_8)));
        assertThat(templates).isEmpty();
    }
}
```

- [ ] **Step 3: Implement ClasspathYamlTemplateRegistrar**

Follow the exact pattern of `ClasspathYamlDescriptorRegistrar`: classpath scanning with `Thread.currentThread().getContextClassLoader().getResources()`, `FAIL_ON_UNKNOWN_PROPERTIES = true`, package-visible `loadFrom(InputStream)` for testability.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlTemplateRegistrarTest`
Expected: All PASS

- [ ] **Step 5: Add template ref parsing to ClasspathYamlDescriptorRegistrar**

Add `TemplateRefConfig` inner class to `ClasspathYamlDescriptorRegistrar`:

```java
static class TemplateRefConfig {
    public String ref;
    public Map<String, String> args;
}
```

Add `public List<TemplateRefConfig> templates;` field to `DescriptorConfig`.

Update `toDescriptor()` to map `TemplateRefConfig` → `TemplateRef`:

```java
.templates(cfg.templates != null
    ? cfg.templates.stream()
        .map(tr -> new TemplateRef(tr.ref, tr.args))
        .toList()
    : null)
```

- [ ] **Step 6: Test descriptor YAML with template refs**

Add test to `ClasspathYamlDescriptorRegistrarTest`:

```java
@Test void parses_template_refs_from_descriptor_yaml() {
    var yaml = """
        descriptors:
          - agentId: test-agent
            name: Test
            slot: tester
            tenancyId: default
            templates:
              - ref: style-guide
              - ref: villain
                args:
                  catchphrase: "Nyah!"
                  nemesis: Penelope
        """;
    var registrar = new ClasspathYamlDescriptorRegistrar();
    var descriptors = registrar.loadFrom(new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));
    assertThat(descriptors).hasSize(1);
    assertThat(descriptors.get(0).templates()).hasSize(2);
    assertThat(descriptors.get(0).templates().get(0).templateId()).isEqualTo("style-guide");
    assertThat(descriptors.get(0).templates().get(1).args()).containsEntry("catchphrase", "Nyah!");
}
```

- [ ] **Step 7: Run all runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#99): YAML loading for templates and descriptor template refs"
```

---

### Task 6: Bootstrap ordering and DescriptorCollector template ref validation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/DescriptorCollector.java` — add `TemplateRegistry` parameter, validate refs
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrap.java` — inject `TemplateRegistry`, pass to collector
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrapTest.java` — update tests
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/DescriptorCollectorTest.java` — template ref validation tests

**Interfaces:**
- Consumes: `TemplateRegistry` from Task 2, `DescriptorCollector.collectAndValidate` (existing)
- Produces: `DescriptorCollector.collectAndValidate(Iterable<AgentDescriptorRegistrar>, TemplateRegistry)` — new signature

- [ ] **Step 1: Write failing tests for template ref validation**

```java
package io.casehub.eidos.runtime.registrar;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.spi.AgentDescriptorRegistrar;
import io.casehub.eidos.memory.InMemoryTemplateRegistry;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class DescriptorCollectorTest {

    private InMemoryTemplateRegistry templateRegistry() {
        var reg = new InMemoryTemplateRegistry();
        reg.register(new DescriptorTemplate("style", "Style", List.of(), "Follow conventions."));
        reg.register(new DescriptorTemplate("role", "Role", List.of("role_name"),
            "You are a ${role_name}."));
        return reg;
    }

    private AgentDescriptor descriptorWithTemplates(List<TemplateRef> refs) {
        return AgentDescriptor.builder()
            .agentId("a").name("n").slot("s").tenancyId("t")
            .templates(refs).build();
    }

    @Test void valid_refs_pass() {
        var refs = List.of(
            new TemplateRef("style", Map.of()),
            new TemplateRef("role", Map.of("role_name", "reviewer")));
        AgentDescriptorRegistrar registrar = () -> List.of(descriptorWithTemplates(refs));
        assertThatNoException().isThrownBy(() ->
            DescriptorCollector.collectAndValidate(List.of(registrar), templateRegistry()));
    }

    @Test void unknown_template_id_throws() {
        var refs = List.of(new TemplateRef("nonexistent", Map.of()));
        AgentDescriptorRegistrar registrar = () -> List.of(descriptorWithTemplates(refs));
        assertThatThrownBy(() ->
            DescriptorCollector.collectAndValidate(List.of(registrar), templateRegistry()))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("nonexistent");
    }

    @Test void missing_required_arg_throws() {
        var refs = List.of(new TemplateRef("role", Map.of()));
        AgentDescriptorRegistrar registrar = () -> List.of(descriptorWithTemplates(refs));
        assertThatThrownBy(() ->
            DescriptorCollector.collectAndValidate(List.of(registrar), templateRegistry()))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("role_name");
    }

    @Test void extra_arg_throws() {
        var refs = List.of(new TemplateRef("style", Map.of("bogus", "value")));
        AgentDescriptorRegistrar registrar = () -> List.of(descriptorWithTemplates(refs));
        assertThatThrownBy(() ->
            DescriptorCollector.collectAndValidate(List.of(registrar), templateRegistry()))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("bogus");
    }

    @Test void null_templates_passes() {
        var d = AgentDescriptor.builder().agentId("a").name("n").slot("s").tenancyId("t").build();
        AgentDescriptorRegistrar registrar = () -> List.of(d);
        assertThatNoException().isThrownBy(() ->
            DescriptorCollector.collectAndValidate(List.of(registrar), templateRegistry()));
    }
}
```

- [ ] **Step 2: Implement template ref validation in DescriptorCollector**

Change signature to `collectAndValidate(Iterable<AgentDescriptorRegistrar> registrars, TemplateRegistry templateRegistry)`. After duplicate-ID check, add template ref validation loop:

```java
for (var d : all) {
    if (d.templates() != null) {
        for (var ref : d.templates()) {
            var template = templateRegistry.resolve(ref.templateId())
                .orElseThrow(() -> new IllegalStateException(
                    "Descriptor '" + d.agentId() + "' references unknown template: " + ref.templateId()));
            var declared = Set.copyOf(template.parameters());
            var provided = ref.args().keySet();
            var missing = new TreeSet<>(declared);
            missing.removeAll(provided);
            if (!missing.isEmpty()) {
                throw new IllegalStateException("Descriptor '" + d.agentId()
                    + "', template '" + ref.templateId() + "': missing args " + missing);
            }
            var extra = new TreeSet<>(provided);
            extra.removeAll(declared);
            if (!extra.isEmpty()) {
                throw new IllegalStateException("Descriptor '" + d.agentId()
                    + "', template '" + ref.templateId() + "': unexpected args " + extra);
            }
        }
    }
}
```

- [ ] **Step 3: Update AgentDescriptorBootstrap to inject TemplateRegistry**

Add `@Inject TemplateRegistry templateRegistry;` and update the call:

```java
static void registerAll(Iterable<AgentDescriptorRegistrar> registrars,
                        AgentRegistry registry, TemplateRegistry templateRegistry) {
    DescriptorCollector.collectAndValidate(registrars, templateRegistry).forEach(registry::register);
}
```

Update `onStartup` to pass `templateRegistry`.

- [ ] **Step 4: Fix any compilation issues and run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#99): template ref validation in DescriptorCollector with bootstrap ordering"
```

---

### Task 7: Render pipeline integration — template resolution and assembly

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java` — inject `TemplateRegistry`, resolve templates, update assembly, update enrichment payload, update `PROMPT_TEMPLATE`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java` — template rendering tests
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java` — integration tests

**Interfaces:**
- Consumes: `TemplateRegistry`, `DescriptorTemplate`, `TemplateRef` from earlier tasks
- Produces: Resolved template content in MARKDOWN, PROSE assembly; template content in enrichment payload

- [ ] **Step 1: Write failing tests for template resolution**

Add to `EidosRenderPipelineTest`:

```java
@Test void markdown_includes_resolved_templates_before_disposition() {
    // Register a static template
    templateRegistry.register(new DescriptorTemplate("cartoon-style", "Cartoon Style",
        List.of(), "You are a cartoon character. Be expressive and theatrical."));

    var descriptor = AgentDescriptor.builder()
        .agentId("agent-1").name("Test Agent").slot("tester").tenancyId("default")
        .templates(List.of(new TemplateRef("cartoon-style", Map.of())))
        .disposition(AgentDisposition.builder().socialOrient("collaborative").build())
        .build();

    var context = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
    // render structurally (no LLM)
    var result = pipeline.buildStage1(descriptor, context);
    var assembled = pipeline.assemble(result, Optional.empty(), Optional.empty(), descriptor, context);

    assertThat(assembled.text()).contains("You are a cartoon character");
    // templates appear before disposition
    int templatePos = assembled.text().indexOf("cartoon character");
    int dispositionPos = assembled.text().indexOf("How You Operate");
    if (dispositionPos > 0) {
        assertThat(templatePos).isLessThan(dispositionPos);
    }
}

@Test void parameterized_template_substitution() {
    templateRegistry.register(new DescriptorTemplate("villain", "Villain",
        List.of("catchphrase", "nemesis"),
        "Your catchphrase is \"${catchphrase}\". Your nemesis is ${nemesis}."));

    var descriptor = AgentDescriptor.builder()
        .agentId("agent-1").name("Test").slot("villain").tenancyId("default")
        .templates(List.of(new TemplateRef("villain",
            Map.of("catchphrase", "Nyah-ha-ha!", "nemesis", "Penelope"))))
        .build();

    var context = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
    var result = pipeline.buildStage1(descriptor, context);
    var assembled = pipeline.assemble(result, Optional.empty(), Optional.empty(), descriptor, context);

    assertThat(assembled.text()).contains("Your catchphrase is \"Nyah-ha-ha!\"");
    assertThat(assembled.text()).contains("Your nemesis is Penelope");
}

@Test void substitution_is_single_pass_no_cross_injection() {
    templateRegistry.register(new DescriptorTemplate("inject-test", "Test",
        List.of("greeting", "name"),
        "Hello ${name}. ${greeting}"));

    var descriptor = AgentDescriptor.builder()
        .agentId("agent-1").name("Test").slot("tester").tenancyId("default")
        .templates(List.of(new TemplateRef("inject-test",
            Map.of("greeting", "Hi ${name}", "name", "Alice"))))
        .build();

    var context = AgentPromptContext.forFormat(RenderFormat.MARKDOWN);
    var result = pipeline.buildStage1(descriptor, context);
    var assembled = pipeline.assemble(result, Optional.empty(), Optional.empty(), descriptor, context);

    assertThat(assembled.text()).contains("Hello Alice");
    assertThat(assembled.text()).contains("Hi ${name}");
}

@Test void a2a_card_does_not_include_templates() {
    templateRegistry.register(new DescriptorTemplate("cartoon-style", "Cartoon Style",
        List.of(), "You are a cartoon character."));

    var descriptor = AgentDescriptor.builder()
        .agentId("agent-1").name("Test").slot("tester").tenancyId("default")
        .templates(List.of(new TemplateRef("cartoon-style", Map.of())))
        .build();

    var context = AgentPromptContext.forFormat(RenderFormat.A2A_CARD);
    var result = pipeline.buildStage1(descriptor, context);
    var assembled = pipeline.assemble(result, Optional.empty(), Optional.empty(), descriptor, context);

    assertThat(assembled.text()).doesNotContain("cartoon character");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosRenderPipelineTest`
Expected: Compilation failure or assertion failure — pipeline doesn't handle templates yet

- [ ] **Step 3: Implement template resolution in EidosRenderPipeline**

Add `TemplateRegistry` as a constructor parameter. Add template resolution method:

```java
String resolveTemplates(AgentDescriptor descriptor) {
    if (descriptor.templates() == null || descriptor.templates().isEmpty()) return null;
    var sb = new StringBuilder();
    for (var ref : descriptor.templates()) {
        var template = templateRegistry.resolve(ref.templateId())
            .orElseThrow(() -> new IllegalStateException("Unknown template: " + ref.templateId()));
        sb.append(substitute(template.content(), ref.args())).append("\n\n");
    }
    return sb.toString().trim();
}

private static final Pattern PLACEHOLDER = Pattern.compile("\\$\\{([^}]+)}");

static String substitute(String content, Map<String, String> args) {
    if (args == null || args.isEmpty()) return content;
    return PLACEHOLDER.matcher(content).replaceAll(match -> {
        var param = match.group(1);
        var value = args.get(param);
        return value != null ? java.util.regex.Matcher.quoteReplacement(value) : match.group();
    });
}
```

- [ ] **Step 4: Update assembly methods**

In `assembleMarkdown()`, add templates section after capabilities, before disposition:

```java
// Templates — resolved prose, before disposition
String templates = resolveTemplates(descriptor);
if (templates != null) {
    sb.append("\n## Behavioral Conventions\n").append(templates).append("\n");
}
```

Same pattern for `assembleProse()` — templates as paragraphs before disposition.

`assembleA2aCard()` — no change (templates don't appear in A2A_CARD).

- [ ] **Step 5: Update enrichment payload**

In `buildDescriptorPayload()`, add resolved templates to the JSON payload:

```java
String resolvedTemplates = resolveTemplates(descriptor);
if (resolvedTemplates != null) {
    node.put("templates", resolvedTemplates);
}
```

In `buildEnrichmentPayload()`, add:

```java
copyIfPresent(payload, descriptorNode, "templates");
```

Update `PROMPT_TEMPLATE` to mention templates in the "payload may contain" list and in the disposition narrative instruction.

- [ ] **Step 6: Run all render tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All PASS

- [ ] **Step 7: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All PASS across all modules

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#99): render pipeline integration — template resolution, assembly, and enrichment"
```

---

### Task 8: JPA persistence — entity, mapper, Flyway migration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java` — add `templates` TEXT column
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` — map templates field
- Create: `runtime/src/main/resources/db/eidos/migration/V7__descriptor_templates.sql`

**Interfaces:**
- Consumes: `TemplateRef` from Task 1
- Produces: round-trip persistence of `AgentDescriptor.templates()` via JSON TEXT column

- [ ] **Step 1: Create Flyway migration**

Create `runtime/src/main/resources/db/eidos/migration/V7__descriptor_templates.sql`:

```sql
ALTER TABLE agent_descriptor ADD COLUMN templates TEXT NULL;
```

- [ ] **Step 2: Add templates field to AgentDescriptorEntity**

Add after `briefing`:

```java
@Column(columnDefinition = "TEXT")
String templates;
```

- [ ] **Step 3: Update AgentDescriptorMapper**

In `toRecord()`, read templates from entity:

```java
readJson(e.templates, new TypeReference<List<TemplateRef>>() {})
```

In `toEntity()`, write templates:

```java
e.templates = writeJson(d.templates());
```

- [ ] **Step 4: Run full project build to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#99): JPA persistence for descriptor template refs — entity, mapper, V7 migration"
```

---

### Task 9: Example scenario test

**Files:**
- Modify: `examples/agent-scenarios/src/test/resources/META-INF/eidos/templates.yaml` — example templates
- Modify: `examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml` — add template refs to an existing descriptor
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DescriptorTemplateScenarioTest.java`

**Interfaces:**
- Consumes: All infrastructure from Tasks 1-8
- Produces: Integration test proving end-to-end: YAML load → registry → descriptor validation → render with templates

- [ ] **Step 1: Create example templates YAML**

Create `examples/agent-scenarios/src/test/resources/META-INF/eidos/templates.yaml`:

```yaml
templates:
  - id: document-review-conventions
    name: Document Review Conventions
    content: |
      When reviewing documents, always provide specific line references.
      Categorise findings as structural, content, or stylistic.
      Never suggest removing content without explaining what is lost.

  - id: communication-style
    name: Communication Style Template
    parameters: [formality, feedback_approach]
    content: |
      Communicate in a ${formality} register. When providing feedback,
      use a ${feedback_approach} approach — frame suggestions constructively.
```

- [ ] **Step 2: Add template refs to an existing descriptor**

Add `templates:` to the `drafthouse-structural-reviewer` in `descriptors.yaml`:

```yaml
    templates:
      - ref: document-review-conventions
      - ref: communication-style
        args:
          formality: professional
          feedback_approach: collaborative
```

- [ ] **Step 3: Write scenario test**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class DescriptorTemplateScenarioTest {

    @Inject AgentRegistry registry;
    @Inject TemplateRegistry templateRegistry;
    @Inject SystemPromptRenderer renderer;

    @Test void templates_loaded_from_classpath() {
        assertThat(templateRegistry.resolve("document-review-conventions")).isPresent();
        assertThat(templateRegistry.resolve("communication-style")).isPresent();
    }

    @Test void descriptor_with_templates_registered() {
        var desc = registry.findById("drafthouse-structural-reviewer", "drafthouse");
        assertThat(desc).isPresent();
        assertThat(desc.get().templates()).isNotNull().hasSize(2);
    }

    @Test void rendered_prompt_includes_template_content() {
        var desc = registry.findById("drafthouse-structural-reviewer", "drafthouse").orElseThrow();
        var ctx = AgentPromptContext.forFormat(SystemPromptRenderer.RenderFormat.MARKDOWN);
        var rendered = renderer.render(desc, ctx);
        assertThat(rendered.text()).contains("specific line references");
        assertThat(rendered.text()).contains("professional register");
        assertThat(rendered.text()).contains("collaborative approach");
    }

    @Test void parameterized_template_substituted() {
        var desc = registry.findById("drafthouse-structural-reviewer", "drafthouse").orElseThrow();
        var ctx = AgentPromptContext.forFormat(SystemPromptRenderer.RenderFormat.MARKDOWN);
        var rendered = renderer.render(desc, ctx);
        assertThat(rendered.text()).doesNotContain("${formality}");
        assertThat(rendered.text()).doesNotContain("${feedback_approach}");
    }
}
```

- [ ] **Step 4: Run example tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios`
Expected: All PASS

- [ ] **Step 5: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All PASS across all modules

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#99): example scenario test — end-to-end template loading, registration, and rendering"
```
