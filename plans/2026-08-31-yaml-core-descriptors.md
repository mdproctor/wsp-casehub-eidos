# yaml-core Parameterized Descriptor YAML — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #149 — feat: adopt yaml-core for parameterized descriptor YAML — variables, forEach, when, CSV
**Issue group:** #149

**Goal:** Insert yaml-core preprocessing between YAML parsing and Jackson deserialization in `ClasspathYamlDescriptorRegistrar`, enabling variables, forEach expansion, conditional inclusion, and CSV data sources in `descriptors.yaml`.

**Architecture:** Two-pass pipeline — parse YAML to generic `Map<String, Object>` (lenient), preprocess with yaml-core (resolve variables, expand forEach, evaluate `when`, load CSV), then deserialize each resolved map through the existing `EidosDescriptorModule` (strict). New classes: `DescriptorPreprocessor` (orchestrates pipeline) and `DescriptorForEachAdapter` (adapts descriptor maps to yaml-core's `ForEachAdapter` interface).

**Tech Stack:** Java 21, casehub-platform-yaml-core (VariableResolver, ForEachExpander, CsvParser), Jackson YAML, Quarkus CDI

## Global Constraints

- Java 21 source level, Java 26 JVM
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
- yaml-core Maven coordinate: `io.casehub:casehub-platform-yaml-core` (version managed by parent BOM)
- All new classes in package `io.casehub.eidos.runtime.yaml`
- ForEach expansion limit: 100 per template (hardcoded constant `MAX_EXPANSION = 100`)
- `FAIL_ON_UNKNOWN_PROPERTIES=true` on per-descriptor deserialization (existing behavior preserved)
- Backward compatibility: existing descriptors.yaml files without preprocessing keys must work unchanged

---

## Batch 1: Adapter + Preprocessor Core

### Task 1: Maven dependency + DescriptorForEachAdapter

**Files:**
- Modify: `runtime/pom.xml` — add yaml-core dependency
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/DescriptorForEachAdapter.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/DescriptorForEachAdapterTest.java`

**Interfaces:**
- Consumes: `io.casehub.yaml.core.foreach.ForEachAdapter<Map<String, Object>>` (from yaml-core)
- Produces: `DescriptorForEachAdapter` — used by `DescriptorPreprocessor` in Task 2

- [ ] **Step 1: Add yaml-core dependency to runtime/pom.xml**

Add after the `jackson-dataformat-yaml` dependency:

```xml
<!-- yaml-core — variable resolution, forEach expansion, conditional inclusion, CSV -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-core</artifactId>
</dependency>
```

- [ ] **Step 2: Verify the dependency resolves**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:resolve -pl runtime -q`
Expected: SUCCESS (no missing artifact errors)

- [ ] **Step 3: Write failing tests for DescriptorForEachAdapter**

```java
package io.casehub.eidos.runtime.yaml;

import io.casehub.yaml.core.resolver.VariableResolver;
import io.casehub.yaml.core.resolver.VariableSource;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class DescriptorForEachAdapterTest {

    private final DescriptorForEachAdapter adapter = new DescriptorForEachAdapter();

    @Test
    void getId_returns_agentId() {
        var map = Map.<String, Object>of("agentId", "test-agent", "name", "Test");
        assertThat(adapter.getId(map)).isEqualTo("test-agent");
    }

    @Test
    void getForEach_returns_forEach_value() {
        var map = Map.<String, Object>of("agentId", "a", "forEach", "teams");
        assertThat(adapter.getForEach(map)).isEqualTo("teams");
    }

    @Test
    void getForEach_returns_null_when_absent() {
        var map = Map.<String, Object>of("agentId", "a");
        assertThat(adapter.getForEach(map)).isNull();
    }

    @Test
    void getWhen_returns_when_value() {
        var map = Map.<String, Object>of("agentId", "a", "when", "${var.enabled}");
        assertThat(adapter.getWhen(map)).isEqualTo("${var.enabled}");
    }

    @Test
    void getWhen_returns_null_when_absent() {
        var map = Map.<String, Object>of("agentId", "a");
        assertThat(adapter.getWhen(map)).isNull();
    }

    @Test
    void stamp_resolves_variables_in_map() {
        var template = new LinkedHashMap<String, Object>();
        template.put("agentId", "${each.team}-reviewer");
        template.put("name", "${each.team} Reviewer");
        template.put("slot", "reviewer");

        var resolver = new VariableResolver(Map.of(), Set.of())
                .withEachContext(Map.of("team", "frontend"));

        var result = adapter.stamp(template, "tpl.frontend", resolver);
        assertThat(result.get("agentId")).isEqualTo("frontend-reviewer");
        assertThat(result.get("name")).isEqualTo("frontend Reviewer");
        assertThat(result.get("slot")).isEqualTo("reviewer");
    }

    @Test
    void stamp_strips_forEach_and_when() {
        var template = new LinkedHashMap<String, Object>();
        template.put("agentId", "a");
        template.put("forEach", "teams");
        template.put("when", "${var.enabled}");

        var resolver = new VariableResolver(Map.of(), Set.of());
        var result = adapter.stamp(template, "a", resolver);

        assertThat(result).doesNotContainKey("forEach");
        assertThat(result).doesNotContainKey("when");
        assertThat(result).containsKey("agentId");
    }

    @Test
    void stamp_resolves_nested_map_values() {
        var caps = new LinkedHashMap<String, Object>();
        caps.put("name", "${each.team}-review");
        var template = new LinkedHashMap<String, Object>();
        template.put("agentId", "a");
        template.put("capability", caps);

        var resolver = new VariableResolver(Map.of(), Set.of())
                .withEachContext(Map.of("team", "backend"));

        var result = adapter.stamp(template, "a.backend", resolver);
        @SuppressWarnings("unchecked")
        var resolvedCap = (Map<String, Object>) result.get("capability");
        assertThat(resolvedCap.get("name")).isEqualTo("backend-review");
    }

    @Test
    void stamp_does_not_mutate_template() {
        var template = new LinkedHashMap<String, Object>();
        template.put("agentId", "${each.x}");
        template.put("forEach", "group");

        var resolver = new VariableResolver(Map.of(), Set.of())
                .withEachContext(Map.of("x", "val"));

        adapter.stamp(template, "tpl.val", resolver);

        assertThat(template.get("agentId")).isEqualTo("${each.x}");
        assertThat(template).containsKey("forEach");
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DescriptorForEachAdapterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 5: Implement DescriptorForEachAdapter**

```java
package io.casehub.eidos.runtime.yaml;

import io.casehub.yaml.core.foreach.ForEachAdapter;
import io.casehub.yaml.core.resolver.VariableResolver;

import java.util.LinkedHashMap;
import java.util.Map;

public final class DescriptorForEachAdapter implements ForEachAdapter<Map<String, Object>> {

    @Override
    public Map<String, Object> stamp(Map<String, Object> template, String stampedId,
                                      VariableResolver scopedResolver) {
        var resolved = new LinkedHashMap<>(scopedResolver.resolveMap(template, stampedId));
        resolved.remove("forEach");
        resolved.remove("when");
        return resolved;
    }

    @Override
    public Object getForEach(Map<String, Object> element) {
        return element.get("forEach");
    }

    @Override
    public String getId(Map<String, Object> element) {
        Object id = element.get("agentId");
        return id != null ? id.toString() : null;
    }

    @Override
    public String getWhen(Map<String, Object> element) {
        Object when = element.get("when");
        return when != null ? when.toString() : null;
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DescriptorForEachAdapterTest`
Expected: all 7 tests PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/pom.xml runtime/src/main/java/io/casehub/eidos/runtime/yaml/DescriptorForEachAdapter.java runtime/src/test/java/io/casehub/eidos/runtime/yaml/DescriptorForEachAdapterTest.java
git commit -m "feat(#149): add DescriptorForEachAdapter — yaml-core ForEachAdapter for descriptor maps

Refs #149"
```

---

### Task 2: DescriptorPreprocessor — variables + forEach expansion

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/DescriptorPreprocessor.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/DescriptorPreprocessorTest.java`

**Interfaces:**
- Consumes: `DescriptorForEachAdapter` (from Task 1), `VariableResolver`, `ForEachExpander`, `IterationGroup` (from yaml-core)
- Produces: `DescriptorPreprocessor.preprocess(Map<String, Object> rawYaml, Map<String, VariableSource> externalSources, ClassLoader classLoader) → List<Map<String, Object>>`

- [ ] **Step 1: Write failing tests for variable resolution and forEach**

```java
package io.casehub.eidos.runtime.yaml;

import io.casehub.yaml.core.resolver.VariableSource;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DescriptorPreprocessorTest {

    static Map<String, Object> descriptor(String agentId) {
        return descriptor(agentId, Map.of());
    }

    static Map<String, Object> descriptor(String agentId, Map<String, Object> extras) {
        var map = new LinkedHashMap<String, Object>();
        map.put("agentId", agentId);
        map.put("name", agentId);
        map.put("slot", "worker");
        map.put("tenancyId", "default");
        map.putAll(extras);
        return map;
    }

    @Test
    void passthrough_no_preprocessing_keys() {
        var raw = Map.<String, Object>of(
                "descriptors", List.of(descriptor("agent-1"), descriptor("agent-2")));

        var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

        assertThat(result).hasSize(2);
        assertThat(result.get(0).get("agentId")).isEqualTo("agent-1");
        assertThat(result.get(1).get("agentId")).isEqualTo("agent-2");
    }

    @Test
    void variable_resolution_in_descriptor_fields() {
        var raw = Map.<String, Object>of(
                "variables", Map.of("tenant", "acme"),
                "descriptors", List.of(
                        descriptor("${var.tenant}-reviewer", Map.of(
                                "tenancyId", "${var.tenant}"))));

        var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).get("agentId")).isEqualTo("acme-reviewer");
        assertThat(result.get(0).get("tenancyId")).isEqualTo("acme");
    }

    @Test
    void forEach_named_group_expands() {
        var raw = new LinkedHashMap<String, Object>();
        raw.put("iterations", Map.of("teams",
                Map.of("as", "team", "in", List.of("frontend", "backend"))));
        raw.put("descriptors", List.of(
                descriptor("${each.team}-reviewer", Map.of("forEach", "teams"))));

        var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

        assertThat(result).hasSize(2);
        assertThat(result.get(0).get("agentId")).isEqualTo("frontend-reviewer");
        assertThat(result.get(1).get("agentId")).isEqualTo("backend-reviewer");
    }

    @Test
    void forEach_inline_expands() {
        var forEach = Map.<String, Object>of(
                "as", "env", "in", List.of("dev", "prod"));
        var raw = Map.<String, Object>of(
                "descriptors", List.of(
                        descriptor("${each.env}-agent", Map.of("forEach", forEach))));

        var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

        assertThat(result).hasSize(2);
        assertThat(result.get(0).get("agentId")).isEqualTo("dev-agent");
        assertThat(result.get(1).get("agentId")).isEqualTo("prod-agent");
    }

    @Test
    void forEach_strips_forEach_key_from_result() {
        var raw = new LinkedHashMap<String, Object>();
        raw.put("iterations", Map.of("teams",
                Map.of("as", "team", "in", List.of("a"))));
        raw.put("descriptors", List.of(
                descriptor("${each.team}", Map.of("forEach", "teams"))));

        var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

        assertThat(result.get(0)).doesNotContainKey("forEach");
    }

    @Test
    void mixed_static_and_forEach_preserves_order() {
        var raw = new LinkedHashMap<String, Object>();
        raw.put("iterations", Map.of("env",
                Map.of("as", "e", "in", List.of("a", "b"))));
        var descs = new java.util.ArrayList<Map<String, Object>>();
        descs.add(descriptor("first"));
        descs.add(descriptor("${each.e}-expand", Map.of("forEach", "env")));
        descs.add(descriptor("last"));
        raw.put("descriptors", descs);

        var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

        assertThat(result).hasSize(4);
        assertThat(result.stream().map(m -> m.get("agentId")).toList())
                .containsExactly("first", "a-expand", "b-expand", "last");
    }

    @Test
    void expansion_limit_exceeded_throws() {
        var values = new java.util.ArrayList<String>();
        for (int i = 0; i < 101; i++) values.add("v" + i);
        var forEach = Map.<String, Object>of("as", "x", "in", values);
        var raw = Map.<String, Object>of(
                "descriptors", List.of(
                        descriptor("${each.x}", Map.of("forEach", forEach))));

        assertThatThrownBy(() ->
                DescriptorPreprocessor.preprocess(raw, Map.of(), null))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("101")
                .hasMessageContaining("100");
    }

    @Test
    void unresolved_variable_throws() {
        var raw = Map.<String, Object>of(
                "descriptors", List.of(descriptor("${var.missing}")));

        assertThatThrownBy(() ->
                DescriptorPreprocessor.preprocess(raw, Map.of(), null))
                .isInstanceOf(RuntimeException.class);
    }

    @Test
    void external_variable_source_resolved() {
        VariableSource configSource = name ->
                "db.host".equals(name) ? "localhost" : null;
        var raw = Map.<String, Object>of(
                "descriptors", List.of(
                        descriptor("${config.db.host}-agent")));

        var result = DescriptorPreprocessor.preprocess(
                raw, Map.of("config", configSource), null);

        assertThat(result.get(0).get("agentId")).isEqualTo("localhost-agent");
    }

    @Test
    void variables_combined_with_forEach() {
        var raw = new LinkedHashMap<String, Object>();
        raw.put("variables", Map.of("org", "acme"));
        raw.put("iterations", Map.of("teams",
                Map.of("as", "team", "in", List.of("fe", "be"))));
        raw.put("descriptors", List.of(
                descriptor("${var.org}-${each.team}",
                        Map.of("forEach", "teams",
                                "tenancyId", "${var.org}"))));

        var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

        assertThat(result).hasSize(2);
        assertThat(result.get(0).get("agentId")).isEqualTo("acme-fe");
        assertThat(result.get(0).get("tenancyId")).isEqualTo("acme");
        assertThat(result.get(1).get("agentId")).isEqualTo("acme-be");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DescriptorPreprocessorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement DescriptorPreprocessor (variables + forEach, no CSV yet)**

```java
package io.casehub.eidos.runtime.yaml;

import io.casehub.yaml.core.foreach.ForEachExpander;
import io.casehub.yaml.core.foreach.IterationGroup;
import io.casehub.yaml.core.resolver.VariableResolver;
import io.casehub.yaml.core.resolver.VariableSource;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

public final class DescriptorPreprocessor {

    static final int MAX_EXPANSION = 100;

    private DescriptorPreprocessor() {}

    @SuppressWarnings("unchecked")
    public static List<Map<String, Object>> preprocess(
            Map<String, Object> rawYaml,
            Map<String, VariableSource> externalSources,
            ClassLoader classLoader) {

        var prefixSources = new LinkedHashMap<String, VariableSource>();

        // 1. Extract variables → var source
        var variables = (Map<String, Object>) rawYaml.get("variables");
        if (variables != null && !variables.isEmpty()) {
            Map<String, String> vars = new LinkedHashMap<>();
            variables.forEach((k, v) -> vars.put(k, v.toString()));
            prefixSources.put("var", name -> vars.get(name));
        }

        // 2. Merge external sources (e.g., config)
        prefixSources.putAll(externalSources);

        var resolver = new VariableResolver(prefixSources, Set.of());

        // 3. Extract iterations → IterationGroups
        var iterationGroups = new LinkedHashMap<String, IterationGroup>();
        var iterations = (Map<String, Object>) rawYaml.get("iterations");
        if (iterations != null) {
            for (var entry : iterations.entrySet()) {
                var groupMap = (Map<String, Object>) entry.getValue();
                String as = (String) groupMap.get("as");
                Object in = groupMap.get("in");
                iterationGroups.put(entry.getKey(), new IterationGroup(as, in));
            }
        }

        // 4. Extract dataSources → load CSV (Task 3)
        // CSV expansion handled in expandCsvDescriptors()

        // 5. Extract descriptors list → keyed map
        var descriptorsList = (List<Map<String, Object>>) rawYaml.get("descriptors");
        if (descriptorsList == null || descriptorsList.isEmpty()) {
            return List.of();
        }

        // 6. Partition: CSV-backed vs normal
        var normalDescriptors = new LinkedHashMap<String, Map<String, Object>>();
        var descriptorOrder = new ArrayList<String>();

        for (var desc : descriptorsList) {
            String agentId = desc.get("agentId").toString();
            normalDescriptors.put(agentId, desc);
            descriptorOrder.add(agentId);
        }

        // 7. Expand normal descriptors via ForEachExpander
        var adapter = new DescriptorForEachAdapter();
        var expanded = ForEachExpander.expand(
                normalDescriptors, iterationGroups, resolver, adapter, MAX_EXPANSION);

        return List.copyOf(expanded.elements());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DescriptorPreprocessorTest`
Expected: all 10 tests PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/yaml/DescriptorPreprocessor.java runtime/src/test/java/io/casehub/eidos/runtime/yaml/DescriptorPreprocessorTest.java
git commit -m "feat(#149): add DescriptorPreprocessor — variable resolution + forEach expansion

Refs #149"
```

---

## Batch 2: CSV Data Sources + Conditional Inclusion

### Task 3: DescriptorPreprocessor — CSV expansion + when conditions

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/DescriptorPreprocessor.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/DescriptorPreprocessorTest.java`
- Create: `runtime/src/test/resources/io/casehub/eidos/runtime/yaml/test-agents.csv` (test fixture)

**Interfaces:**
- Consumes: `CsvParser`, `CsvDataSource`, `Truthiness` (from yaml-core), `VariableResolver.withEachRowContext()` (from yaml-core)
- Produces: CSV expansion and `when` conditional logic in `DescriptorPreprocessor.preprocess()`

- [ ] **Step 1: Write failing tests for when conditions**

Add to `DescriptorPreprocessorTest`:

```java
@Test
void when_true_includes_descriptor() {
    var raw = new LinkedHashMap<String, Object>();
    raw.put("variables", Map.of("enabled", "true"));
    raw.put("descriptors", List.of(
            descriptor("gated", Map.of("when", "${var.enabled}"))));

    var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

    assertThat(result).hasSize(1);
    assertThat(result.get(0).get("agentId")).isEqualTo("gated");
    assertThat(result.get(0)).doesNotContainKey("when");
}

@Test
void when_false_excludes_descriptor() {
    var raw = new LinkedHashMap<String, Object>();
    raw.put("variables", Map.of("enabled", "false"));
    raw.put("descriptors", List.of(
            descriptor("gated", Map.of("when", "${var.enabled}"))));

    var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

    assertThat(result).isEmpty();
}

@Test
void when_with_forEach_per_copy_exclusion() {
    var forEach = Map.<String, Object>of(
            "as", "flag", "in", List.of("true", "false"));
    var raw = Map.<String, Object>of(
            "descriptors", List.of(
                    descriptor("${each.flag}-agent",
                            Map.of("forEach", forEach, "when", "${each.flag}"))));

    var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

    assertThat(result).hasSize(1);
    assertThat(result.get(0).get("agentId")).isEqualTo("true-agent");
}
```

- [ ] **Step 2: Run tests — when tests should already pass (ForEachExpander handles when)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DescriptorPreprocessorTest`
Expected: PASS — ForEachExpander already evaluates `when` via the adapter's `getWhen()`

- [ ] **Step 3: Write failing tests for CSV data sources**

Add to `DescriptorPreprocessorTest`:

```java
@Test
void csv_inline_expansion() {
    var csvContent = "name:STRING, role:STRING\nalice, reviewer\nbob, planner";
    var raw = new LinkedHashMap<String, Object>();
    raw.put("dataSources", Map.of("roster",
            Map.of("csv", csvContent)));
    var forEach = Map.<String, Object>of("as", "agent", "in", "roster");
    raw.put("descriptors", List.of(
            descriptor("${each.agent.name}-agent",
                    Map.of("forEach", forEach,
                            "slot", "${each.agent.role}"))));

    var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

    assertThat(result).hasSize(2);
    assertThat(result.get(0).get("agentId")).isEqualTo("alice-agent");
    assertThat(result.get(0).get("slot")).isEqualTo("reviewer");
    assertThat(result.get(1).get("agentId")).isEqualTo("bob-agent");
    assertThat(result.get(1).get("slot")).isEqualTo("planner");
}

@Test
void csv_classpath_file_expansion() {
    var raw = new LinkedHashMap<String, Object>();
    raw.put("dataSources", Map.of("roster",
            Map.of("file", "io/casehub/eidos/runtime/yaml/test-agents.csv")));
    var forEach = Map.<String, Object>of("as", "agent", "in", "roster");
    raw.put("descriptors", List.of(
            descriptor("${each.agent.name}-agent",
                    Map.of("forEach", forEach,
                            "slot", "${each.agent.role}"))));

    var result = DescriptorPreprocessor.preprocess(
            raw, Map.of(), Thread.currentThread().getContextClassLoader());

    assertThat(result).hasSize(2);
    assertThat(result.get(0).get("agentId")).isEqualTo("carol-agent");
    assertThat(result.get(0).get("slot")).isEqualTo("analyst");
}

@Test
void csv_with_when_excludes_rows() {
    var csvContent = "name:STRING, active:BOOLEAN\nalice, true\nbob, false";
    var raw = new LinkedHashMap<String, Object>();
    raw.put("dataSources", Map.of("roster",
            Map.of("csv", csvContent)));
    var forEach = Map.<String, Object>of("as", "agent", "in", "roster");
    raw.put("descriptors", List.of(
            descriptor("${each.agent.name}-agent",
                    Map.of("forEach", forEach,
                            "when", "${each.agent.active}"))));

    var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

    assertThat(result).hasSize(1);
    assertThat(result.get(0).get("agentId")).isEqualTo("alice-agent");
}

@Test
void csv_expansion_limit_exceeded_throws() {
    var sb = new StringBuilder("name:STRING\n");
    for (int i = 0; i < 101; i++) sb.append("v").append(i).append("\n");
    var raw = new LinkedHashMap<String, Object>();
    raw.put("dataSources", Map.of("big",
            Map.of("csv", sb.toString())));
    var forEach = Map.<String, Object>of("as", "row", "in", "big");
    raw.put("descriptors", List.of(
            descriptor("${each.row.name}", Map.of("forEach", forEach))));

    assertThatThrownBy(() ->
            DescriptorPreprocessor.preprocess(raw, Map.of(), null))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("101")
            .hasMessageContaining("100");
}

@Test
void datasource_and_iteration_namespace_collision_throws() {
    var raw = new LinkedHashMap<String, Object>();
    raw.put("iterations", Map.of("shared",
            Map.of("as", "x", "in", List.of("a"))));
    raw.put("dataSources", Map.of("shared",
            Map.of("csv", "name:STRING\nalice")));
    raw.put("descriptors", List.of(descriptor("a")));

    assertThatThrownBy(() ->
            DescriptorPreprocessor.preprocess(raw, Map.of(), null))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("shared");
}

@Test
void mixed_csv_and_normal_forEach_preserves_order() {
    var csvContent = "name:STRING\nalice\nbob";
    var raw = new LinkedHashMap<String, Object>();
    raw.put("dataSources", Map.of("people",
            Map.of("csv", csvContent)));
    raw.put("iterations", Map.of("env",
            Map.of("as", "e", "in", List.of("dev", "prod"))));
    var descs = new java.util.ArrayList<Map<String, Object>>();
    descs.add(descriptor("first"));
    descs.add(descriptor("${each.person.name}-agent",
            Map.of("forEach", Map.of("as", "person", "in", "people"))));
    descs.add(descriptor("${each.e}-env",
            Map.of("forEach", "env")));
    descs.add(descriptor("last"));
    raw.put("descriptors", descs);

    var result = DescriptorPreprocessor.preprocess(raw, Map.of(), null);

    assertThat(result.stream().map(m -> m.get("agentId")).toList())
            .containsExactly("first", "alice-agent", "bob-agent",
                    "dev-env", "prod-env", "last");
}
```

- [ ] **Step 4: Create CSV test fixture**

Create `runtime/src/test/resources/io/casehub/eidos/runtime/yaml/test-agents.csv`:

```
name:STRING, role:STRING
carol, analyst
dave, planner
```

- [ ] **Step 5: Run tests to verify CSV tests fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DescriptorPreprocessorTest`
Expected: CSV-related tests FAIL (CSV expansion not implemented)

- [ ] **Step 6: Implement CSV expansion and ordering in DescriptorPreprocessor**

Update `DescriptorPreprocessor.preprocess()` to:
1. Parse `dataSources` section — load CSV from `file` (classpath) or `csv` (inline)
2. Validate no namespace collision between dataSources and iterations
3. Partition descriptors: CSV-backed forEach (inline forEach where `in` is a String matching a data source name) vs. normal
4. Expand normal descriptors via ForEachExpander (already done)
5. Expand CSV descriptors directly using `VariableResolver.withEachContext()` + `withEachRowContext()`, with `Truthiness` for `when` evaluation
6. Merge results maintaining original descriptor position order

The key addition — a private `expandCsvDescriptors()` method:

```java
@SuppressWarnings("unchecked")
private static List<Map<String, Object>> expandCsvDescriptors(
        Map<String, Object> descriptor,
        CsvDataSource dataSource,
        VariableResolver resolver) {

    var forEach = (Map<String, Object>) descriptor.get("forEach");
    String as = (String) forEach.get("as");
    var rows = dataSource.rows();

    if (rows.size() > MAX_EXPANSION) {
        throw new IllegalStateException(
                "forEach CSV template '" + descriptor.get("agentId")
                + "' would expand to " + rows.size()
                + " elements (limit: " + MAX_EXPANSION + ").");
    }

    var results = new ArrayList<Map<String, Object>>();
    for (int i = 0; i < rows.size(); i++) {
        var row = rows.get(i);
        String rowKey = String.valueOf(i);
        var rowResolver = resolver
                .withEachContext(Map.of(as, rowKey))
                .withEachRowContext(Map.of(as, row));

        String when = (String) descriptor.get("when");
        if (when != null) {
            String resolved = rowResolver.resolveString(when,
                    descriptor.get("agentId") + "." + rowKey);
            if (!Truthiness.isTruthy(resolved)) continue;
        }

        var resolved = new LinkedHashMap<>(
                rowResolver.resolveMap(descriptor, descriptor.get("agentId") + "." + rowKey));
        resolved.remove("forEach");
        resolved.remove("when");
        results.add(resolved);
    }
    return results;
}
```

The main `preprocess()` method is updated to:
- Parse dataSources
- Validate namespace collisions
- Partition descriptors into ordered slots: each slot is either a single static descriptor, a normalForEach key, or a csvForEach key
- Expand each slot in order and collect results

- [ ] **Step 7: Run all preprocessor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DescriptorPreprocessorTest`
Expected: all tests PASS (including when + CSV tests)

- [ ] **Step 8: Run full module tests to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all existing tests PASS

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/yaml/DescriptorPreprocessor.java runtime/src/test/java/io/casehub/eidos/runtime/yaml/DescriptorPreprocessorTest.java runtime/src/test/resources/io/casehub/eidos/runtime/yaml/test-agents.csv
git commit -m "feat(#149): add CSV data source expansion and when conditions to DescriptorPreprocessor

Refs #149"
```

---

## Batch 3: Integration — Wire into ClasspathYamlDescriptorRegistrar

### Task 4: Modify ClasspathYamlDescriptorRegistrar to use preprocessing pipeline

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java`

**Interfaces:**
- Consumes: `DescriptorPreprocessor.preprocess()` (from Task 2-3), `EidosDescriptorModule.createMapper()` (existing)
- Produces: Modified `loadFrom()` that supports parameterized YAML → `List<AgentDescriptor>`

- [ ] **Step 1: Write failing integration tests**

Add to `ClasspathYamlDescriptorRegistrarTest`:

```java
@Test
void variable_resolution_produces_correct_descriptors() {
    var yaml = """
            variables:
              tenant: wacky-races
            descriptors:
              - agentId: racer
                name: Racer
                slot: driver
                tenancyId: ${var.tenant}
            """;
    var result = parse(yaml);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).tenancyId()).isEqualTo("wacky-races");
}

@Test
void forEach_expansion_produces_multiple_descriptors() {
    var yaml = """
            iterations:
              teams:
                as: team
                in: [frontend, backend]
            descriptors:
              - agentId: ${each.team}-reviewer
                name: ${each.team} Reviewer
                slot: reviewer
                tenancyId: default
                forEach: teams
                capabilities:
                  - name: code-review
                    tags: [${each.team}]
            """;
    var result = parse(yaml);
    assertThat(result).hasSize(2);
    assertThat(result.get(0).agentId()).isEqualTo("frontend-reviewer");
    assertThat(result.get(0).capabilities().get(0).tags())
            .containsExactly("frontend");
    assertThat(result.get(1).agentId()).isEqualTo("backend-reviewer");
}

@Test
void when_false_excludes_from_result() {
    var yaml = """
            variables:
              audit_enabled: "false"
            descriptors:
              - agentId: always
                name: Always
                slot: s
                tenancyId: t
              - agentId: gated
                name: Gated
                slot: s
                tenancyId: t
                when: "${var.audit_enabled}"
            """;
    var result = parse(yaml);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).agentId()).isEqualTo("always");
}

@Test
void csv_inline_expansion_produces_correct_descriptors() {
    var yaml = """
            dataSources:
              roster:
                csv: |
                  name:STRING, role:STRING
                  alice, reviewer
                  bob, planner
            descriptors:
              - agentId: ${each.agent.name}-agent
                name: ${each.agent.name}
                slot: ${each.agent.role}
                tenancyId: default
                forEach:
                  as: agent
                  in: roster
            """;
    var result = parse(yaml);
    assertThat(result).hasSize(2);
    assertThat(result.get(0).agentId()).isEqualTo("alice-agent");
    assertThat(result.get(0).slot()).isEqualTo("reviewer");
    assertThat(result.get(1).agentId()).isEqualTo("bob-agent");
    assertThat(result.get(1).slot()).isEqualTo("planner");
}

@Test
void existing_yaml_without_preprocessing_works_unchanged() {
    var yaml = """
            descriptors:
              - agentId: plain
                name: Plain Agent
                slot: worker
                tenancyId: default
                disposition:
                  conflictMode: collaborating
                  delegation: false
                capabilities:
                  - name: code-review
                    tags: [quality]
                briefing: You are a plain agent.
            """;
    var result = parse(yaml);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).agentId()).isEqualTo("plain");
    assertThat(result.get(0).disposition().primaryTerm(
            io.casehub.eidos.api.DispositionAxis.CONFLICT_MODE))
            .isEqualTo("collaborating");
    assertThat(result.get(0).capabilities().get(0).name())
            .isEqualTo("code-review");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlDescriptorRegistrarTest`
Expected: new tests FAIL (registrar doesn't do preprocessing yet)

- [ ] **Step 3: Modify ClasspathYamlDescriptorRegistrar**

Update `loadFrom(InputStream yaml, VocabularyRegistry vocabRegistry)`:

```java
public List<AgentDescriptor> loadFrom(final InputStream yaml,
                                       final VocabularyRegistry vocabRegistry) {
    if (yaml == null) return List.of();
    try {
        // Step 1: Parse YAML to generic Map (lenient — no custom module)
        var plainMapper = new ObjectMapper(new YAMLFactory());
        @SuppressWarnings("unchecked")
        var rawMap = plainMapper.readValue(yaml, Map.class);
        if (rawMap == null) return List.of();

        // Step 2: Preprocess — resolve variables, expand forEach, evaluate when, load CSV
        var externalSources = new LinkedHashMap<String, VariableSource>();
        try {
            var config = ConfigProvider.getConfig();
            externalSources.put("config", name ->
                    config.getOptionalValue(name, String.class).orElse(null));
        } catch (Exception ignored) {
            // No MicroProfile Config available (e.g., plain unit test)
        }

        @SuppressWarnings("unchecked")
        var rawTyped = (Map<String, Object>) rawMap;
        var descriptorMaps = DescriptorPreprocessor.preprocess(
                rawTyped, externalSources,
                Thread.currentThread().getContextClassLoader());

        if (descriptorMaps.isEmpty()) return List.of();

        // Step 3: Deserialize each resolved map through EidosDescriptorModule
        var eidosMapper = EidosDescriptorModule.createMapper(vocabRegistry);
        var result = new ArrayList<AgentDescriptor>();
        for (var map : descriptorMaps) {
            var node = eidosMapper.valueToTree(map);
            result.add(eidosMapper.treeToValue(node, AgentDescriptor.class));
        }
        return List.copyOf(result);
    } catch (final IOException e) {
        throw new IllegalStateException("Failed to parse YAML: " + e.getMessage(), e);
    }
}
```

Also update the no-vocabRegistry `loadFrom(InputStream)` overload to delegate to the full version:

```java
List<AgentDescriptor> loadFrom(final InputStream yaml) {
    return loadFrom(yaml, null);
}
```

Add import for `org.eclipse.microprofile.config.ConfigProvider` and `io.casehub.yaml.core.resolver.VariableSource`.

Remove the `DescriptorFile` inner class (no longer used).

- [ ] **Step 4: Run all registrar tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ClasspathYamlDescriptorRegistrarTest`
Expected: all tests PASS (new + existing)

- [ ] **Step 5: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all tests PASS

- [ ] **Step 6: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java
git commit -m "feat(#149): wire yaml-core preprocessing into ClasspathYamlDescriptorRegistrar

Inserts DescriptorPreprocessor between YAML parsing and Jackson
deserialization. Supports variables, forEach, when, CSV data sources.
Existing descriptor YAML works unchanged.

Closes #149"
```

## References

- [2026-08-31-yaml-core-descriptors-design.md] — design spec this plan implements
- [ClasspathYamlDescriptorRegistrar.java:17] — current YAML loading, modification target
- [EidosDescriptorModule.java:11] — Jackson module, reused unchanged
- [AgentDescriptorDeserializer.java:14] — descriptor deserialization, reused unchanged
- [ForEachExpander.java] — yaml-core expansion engine
- [ForEachAdapter.java] — yaml-core adapter interface
- [VariableResolver.java] — yaml-core variable resolution
- [CsvParser.java] — yaml-core CSV support
- [ForEachExpanderTest.java] — yaml-core test patterns (TestAdapter reference)
- [GitHub #149] — focal issue
