# extensionData Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #177 — Add extensionData field to AgentDescriptor
**Issue group:** #177

**Goal:** Add `Map<String, Object> extensionData` to AgentDescriptor with YAML, JPA, and annotation support.

**Architecture:** New field on the AgentDescriptor record with deep-copy immutability via ExtensionDataCopier utility, construction-time size validation, JPA persistence as JSON TEXT column, YAML deserialization via tree-walking, and flat key-value annotation support via @ExtensionData/@ExtensionEntry.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson, JPA/Hibernate, Jandex

## Global Constraints

- Java 21 source, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn ...`
- api module is pure Java — no Jackson, no CDI, no Quarkus dependencies
- `Map<String, Object>` values restricted to: Map, List, String, Number, Boolean, null
- MAX_EXTENSION_DATA_SIZE = 65536 estimated characters
- extensionData is NEVER rendered in any prompt format
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`

---

## Batch 1: API Foundation

### Task 1: ExtensionDataCopier utility

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/ExtensionDataCopier.java`
- Test: `api/src/test/java/io/casehub/eidos/api/ExtensionDataCopierTest.java`

**Interfaces:**
- Consumes: nothing (standalone utility)
- Produces: `ExtensionDataCopier.deepCopy(Map<String, Object>) → Map<String, Object>`, `ExtensionDataCopier.estimateSize(Map<String, Object>) → long` — used by AgentDescriptor compact constructor (Task 2)

- [ ] **Step 1: Write the failing test for deepCopy**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class ExtensionDataCopierTest {

    @Test
    void deepCopy_nestedMap_producesSeparateCopy() {
        var inner = new LinkedHashMap<String, Object>();
        inner.put("key", "value");
        var original = new LinkedHashMap<String, Object>();
        original.put("nested", inner);

        var copy = ExtensionDataCopier.deepCopy(original);

        assertEquals(original, copy);
        inner.put("mutated", "yes");
        assertNull(((Map<?, ?>) copy.get("nested")).get("mutated"));
    }

    @Test
    void deepCopy_nestedList_producesSeparateCopy() {
        var list = new ArrayList<>(List.of("a", "b"));
        var original = new LinkedHashMap<String, Object>();
        original.put("items", list);

        var copy = ExtensionDataCopier.deepCopy(original);

        assertEquals(original, copy);
        list.add("c");
        assertEquals(2, ((List<?>) copy.get("items")).size());
    }

    @Test
    void deepCopy_primitivesPassedThrough() {
        var original = new LinkedHashMap<String, Object>();
        original.put("str", "hello");
        original.put("num", 42);
        original.put("dbl", 3.14);
        original.put("bool", true);
        original.put("nil", null);

        var copy = ExtensionDataCopier.deepCopy(original);

        assertEquals("hello", copy.get("str"));
        assertEquals(42, copy.get("num"));
        assertEquals(3.14, copy.get("dbl"));
        assertEquals(true, copy.get("bool"));
        assertNull(copy.get("nil"));
    }

    @Test
    void deepCopy_unknownType_throws() {
        var original = new LinkedHashMap<String, Object>();
        original.put("bad", new Object());

        assertThrows(IllegalArgumentException.class,
            () -> ExtensionDataCopier.deepCopy(original));
    }

    @Test
    void deepCopy_emptyMap() {
        var copy = ExtensionDataCopier.deepCopy(Map.of());
        assertTrue(copy.isEmpty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ExtensionDataCopierTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — ExtensionDataCopier doesn't exist

- [ ] **Step 3: Implement ExtensionDataCopier**

```java
package io.casehub.eidos.api;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class ExtensionDataCopier {

    private ExtensionDataCopier() {}

    public static Map<String, Object> deepCopy(Map<String, Object> source) {
        var result = new LinkedHashMap<String, Object>(source.size());
        for (var entry : source.entrySet()) {
            result.put(entry.getKey(), copyValue(entry.getValue()));
        }
        return Map.copyOf(result);
    }

    public static long estimateSize(Map<String, Object> source) {
        return estimateMapSize(source);
    }

    private static Object copyValue(Object value) {
        if (value == null) return null;
        if (value instanceof String || value instanceof Number || value instanceof Boolean) {
            return value;
        }
        if (value instanceof Map<?, ?> map) {
            var result = new LinkedHashMap<String, Object>(map.size());
            for (var entry : map.entrySet()) {
                result.put((String) entry.getKey(), copyValue(entry.getValue()));
            }
            return Map.copyOf(result);
        }
        if (value instanceof List<?> list) {
            var result = new ArrayList<>(list.size());
            for (var item : list) {
                result.add(copyValue(item));
            }
            return List.copyOf(result);
        }
        throw new IllegalArgumentException(
            "extensionData values must be Map, List, String, Number, Boolean, or null; got: "
            + value.getClass().getName());
    }

    private static long estimateMapSize(Map<?, ?> map) {
        long size = 2; // {}
        for (var entry : map.entrySet()) {
            size += ((String) entry.getKey()).length() + 6; // "key":,
            size += estimateValueSize(entry.getValue());
        }
        return size;
    }

    private static long estimateListSize(List<?> list) {
        long size = 2; // []
        for (var item : list) {
            size += estimateValueSize(item) + 2; // ,
        }
        return size;
    }

    private static long estimateValueSize(Object value) {
        if (value == null) return 4; // null
        if (value instanceof String s) return s.length();
        if (value instanceof Number || value instanceof Boolean) {
            return String.valueOf(value).length();
        }
        if (value instanceof Map<?, ?> map) return estimateMapSize(map);
        if (value instanceof List<?> list) return estimateListSize(list);
        return 0;
    }
}
```

- [ ] **Step 4: Add estimateSize tests**

Append to `ExtensionDataCopierTest.java`:

```java
@Test
void estimateSize_emptyMap_returns2() {
    assertEquals(2, ExtensionDataCopier.estimateSize(Map.of()));
}

@Test
void estimateSize_simpleEntry() {
    var map = Map.<String, Object>of("key", "value");
    long size = ExtensionDataCopier.estimateSize(map);
    // 2 (braces) + 3 (key len) + 6 (structural) + 5 (value len) = 16
    assertTrue(size > 0);
    assertTrue(size < 100);
}

@Test
void estimateSize_largeMap_exceedsLimit() {
    var map = new LinkedHashMap<String, Object>();
    var bigValue = "x".repeat(60000);
    map.put("huge", bigValue);
    assertTrue(ExtensionDataCopier.estimateSize(map) > 60000);
}
```

- [ ] **Step 5: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ExtensionDataCopierTest`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/ExtensionDataCopier.java api/src/test/java/io/casehub/eidos/api/ExtensionDataCopierTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#177): add ExtensionDataCopier utility for deep copy and size estimation

Refs #177"
```

### Task 2: AgentDescriptor field, Builder, Validator, Comparator

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java` (existing, extend)
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java` (existing, extend)

**Interfaces:**
- Consumes: `ExtensionDataCopier.deepCopy()`, `ExtensionDataCopier.estimateSize()` from Task 1
- Produces: `AgentDescriptor.extensionData() → Map<String, Object>`, `AgentDescriptor.Builder.extensionData(Map<String, Object>)` — used by all subsequent tasks

- [ ] **Step 1: Write failing test for extensionData on AgentDescriptor**

Add to existing `AgentDescriptorTest.java` (or create if absent):

```java
@Test
void extensionData_nullByDefault() {
    var d = AgentDescriptor.builder()
        .agentId("test").name("Test").slot("tester").tenancyId("default")
        .build();
    assertNull(d.extensionData());
}

@Test
void extensionData_deepCopiedAtConstruction() {
    var inner = new LinkedHashMap<String, Object>();
    inner.put("key", "value");
    var ext = new LinkedHashMap<String, Object>();
    ext.put("nested", inner);

    var d = AgentDescriptor.builder()
        .agentId("test").name("Test").slot("tester").tenancyId("default")
        .extensionData(ext)
        .build();

    inner.put("mutated", "yes");
    assertNull(((Map<?, ?>) d.extensionData().get("nested")).get("mutated"));
}

@Test
void extensionData_oversized_throws() {
    var ext = new LinkedHashMap<String, Object>();
    ext.put("huge", "x".repeat(70000));

    assertThrows(AgentValidationException.class, () ->
        AgentDescriptor.builder()
            .agentId("test").name("Test").slot("tester").tenancyId("default")
            .extensionData(ext)
            .build());
}

@Test
void extensionData_preservedInToBuilder() {
    var ext = Map.<String, Object>of("key", "value");
    var d = AgentDescriptor.builder()
        .agentId("test").name("Test").slot("tester").tenancyId("default")
        .extensionData(ext)
        .build();

    var rebuilt = d.toBuilder().build();
    assertEquals(d.extensionData(), rebuilt.extensionData());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AgentDescriptorTest`
Expected: compilation failure — no `extensionData` method on AgentDescriptor

- [ ] **Step 3: Add extensionData to AgentDescriptor record**

Modify `AgentDescriptor.java`:

1. Add `Map<String, Object> extensionData` as the 23rd parameter after `constraints` in the record declaration
2. In the compact constructor, add after the constraints validation block:
```java
if (extensionData != null) {
    extensionData = ExtensionDataCopier.deepCopy(extensionData);
    long estimatedSize = ExtensionDataCopier.estimateSize(extensionData);
    if (estimatedSize > AgentDescriptorValidator.MAX_EXTENSION_DATA_SIZE) {
        throw new AgentValidationException("extensionData",
            "estimated size " + estimatedSize + " exceeds maximum " + AgentDescriptorValidator.MAX_EXTENSION_DATA_SIZE);
    }
}
```
3. In `toBuilder()`, add `.extensionData(this.extensionData)`
4. In the Builder class, add:
```java
private Map<String, Object> extensionData;
```
and:
```java
public Builder extensionData(Map<String, Object> v) {
    this.extensionData = v;
    return this;
}
```
5. In `Builder.build()`, add `extensionData` as the last argument to the constructor call

- [ ] **Step 4: Add MAX_EXTENSION_DATA_SIZE to AgentDescriptorValidator**

Add to `AgentDescriptorValidator.java`:
```java
static final int MAX_EXTENSION_DATA_SIZE = 65_536;
```

- [ ] **Step 5: Add extensionData to AgentDescriptorComparator**

In `AgentDescriptorComparator.java`:
1. Change `COMPARED_FIELD_COUNT = 20` to `COMPARED_FIELD_COUNT = 21`
2. Add to `compareSimpleFields()`:
```java
compareField(drifts, "extensionData", desired.extensionData(), actual.extensionData());
```

- [ ] **Step 6: Write failing comparator test**

Add to existing `AgentDescriptorComparatorTest.java`:

```java
@Test
void extensionData_drift_detected() {
    var d1 = base().extensionData(Map.of("k", "v1")).build();
    var d2 = base().extensionData(Map.of("k", "v2")).build();
    var result = AgentDescriptorComparator.compare(d1, d2);
    assertFalse(result.matches());
    assertTrue(result.drifts().stream().anyMatch(d -> d.field().equals("extensionData")));
}

@Test
void extensionData_null_vs_present_drift() {
    var d1 = base().build();
    var d2 = base().extensionData(Map.of("k", "v")).build();
    var result = AgentDescriptorComparator.compare(d1, d2);
    assertFalse(result.matches());
}

@Test
void extensionData_equal_no_drift() {
    var d1 = base().extensionData(Map.of("k", "v")).build();
    var d2 = base().extensionData(Map.of("k", "v")).build();
    var result = AgentDescriptorComparator.compare(d1, d2);
    assertTrue(result.drifts().stream().noneMatch(d -> d.field().equals("extensionData")));
}
```

(Where `base()` is a helper method returning a pre-configured builder — check existing test file for the pattern.)

- [ ] **Step 7: Fix any compilation errors in existing tests**

Adding a 23rd record parameter breaks all existing constructor call sites that use the positional constructor directly. The `AgentDescriptorMapper.toRecord()` call is the main one — that's in Batch 2. For api tests, check `AgentDescriptorComparatorTest` and `AgentDescriptorTest` for direct constructor calls and add `null` for extensionData where needed.

- [ ] **Step 8: Run all api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#177): add extensionData field to AgentDescriptor

Map<String, Object> with deep-copy immutability, size validation,
and comparator participation.

Refs #177"
```

---

## Batch 2: YAML + JPA Persistence

### Task 3: YAML deserialization

**Files:**
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/yaml/AgentDescriptorDeserializer.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializerTest.java` (existing, extend)

**Interfaces:**
- Consumes: `AgentDescriptor.Builder.extensionData(Map<String, Object>)` from Task 2
- Produces: YAML descriptors with `extensionData` key deserialize into `AgentDescriptor` with populated extensionData field

- [ ] **Step 1: Write failing deserializer test**

Add to existing `AgentDescriptorDeserializerTest.java`:

```java
@Test
void extensionData_deserializesNestedStructure() throws Exception {
    String yaml = """
        agentId: test-agent
        name: Test Agent
        slot: tester
        tenancyId: default
        extensionData:
          io.casehub.manor.social:
            drives:
              - belonging
              - approval
            norms:
              formality: 0.3
        """;
    var descriptor = deserialize(yaml);
    assertNotNull(descriptor.extensionData());
    var social = (Map<?, ?>) descriptor.extensionData().get("io.casehub.manor.social");
    assertNotNull(social);
    var drives = (List<?>) social.get("drives");
    assertEquals(List.of("belonging", "approval"), drives);
    var norms = (Map<?, ?>) social.get("norms");
    assertEquals(0.3, norms.get("formality"));
}

@Test
void extensionData_absent_isNull() throws Exception {
    String yaml = """
        agentId: test-agent
        name: Test Agent
        slot: tester
        tenancyId: default
        """;
    var descriptor = deserialize(yaml);
    assertNull(descriptor.extensionData());
}
```

(Check existing test file for the `deserialize()` helper method pattern.)

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentDescriptorDeserializerTest`
Expected: FAIL — extensionData is always null

- [ ] **Step 3: Add extensionData handling to AgentDescriptorDeserializer**

In `AgentDescriptorDeserializer.java`, add before `return builder.build();`:

```java
if (root.has("extensionData") && root.get("extensionData").isObject()) {
    builder.extensionData(readMapTree(root.get("extensionData")));
}
```

Add the `readMapTree` private method:

```java
private static Map<String, Object> readMapTree(JsonNode node) {
    var map = new LinkedHashMap<String, Object>();
    node.fields().forEachRemaining(e -> map.put(e.getKey(), readValueTree(e.getValue())));
    return map;
}

private static Object readValueTree(JsonNode node) {
    if (node.isNull()) return null;
    if (node.isTextual()) return node.asText();
    if (node.isBoolean()) return node.asBoolean();
    if (node.isInt()) return node.asInt();
    if (node.isLong()) return node.asLong();
    if (node.isDouble() || node.isFloat()) return node.asDouble();
    if (node.isObject()) return readMapTree(node);
    if (node.isArray()) {
        var list = new ArrayList<>();
        for (JsonNode item : node) list.add(readValueTree(item));
        return list;
    }
    return node.asText();
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentDescriptorDeserializerTest`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eidos-core/ runtime/src/test/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#177): add extensionData YAML deserialization

Supports nested maps, lists, and primitives via tree-walking.

Refs #177"
```

### Task 4: JPA persistence

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`
- Modify: `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java` (existing, extend)

**Interfaces:**
- Consumes: `AgentDescriptor.extensionData()` from Task 2
- Produces: JPA round-trip preserves extensionData (register → findById → extensionData matches)

- [ ] **Step 1: Write failing JPA round-trip test**

Add to existing `JpaAgentRegistryTest.java`:

```java
@Test
void extensionData_roundTrip() {
    var ext = new LinkedHashMap<String, Object>();
    ext.put("io.casehub.test.config", Map.of("level", 5, "name", "test"));
    ext.put("io.casehub.test.flags", List.of("alpha", "beta"));

    var descriptor = baseDescriptor().extensionData(ext).build();
    registry.register(descriptor);

    var found = registry.findById(descriptor.agentId(), descriptor.tenancyId());
    assertTrue(found.isPresent());
    assertEquals(ext, found.get().extensionData());
}

@Test
void extensionData_null_roundTrip() {
    var descriptor = baseDescriptor().build();
    registry.register(descriptor);

    var found = registry.findById(descriptor.agentId(), descriptor.tenancyId());
    assertTrue(found.isPresent());
    assertNull(found.get().extensionData());
}
```

(Check existing test file for `baseDescriptor()` helper and `registry` injection pattern.)

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest`
Expected: FAIL — compilation errors or null extensionData on retrieval

- [ ] **Step 3: Add extension_data column to V1 schema**

In `V1__initial_schema.sql`, add `extension_data TEXT,` to the `agent_descriptor` CREATE TABLE statement, before the CONSTRAINT line:

```sql
    templates              TEXT,
    extension_data         TEXT,
    CONSTRAINT uq_agent UNIQUE (agent_id, tenancy_id)
```

- [ ] **Step 4: Add field to AgentDescriptorEntity**

In `AgentDescriptorEntity.java`, add after the `templates` field:

```java
@Column(name = "extension_data", columnDefinition = "TEXT")
String extensionData;
```

- [ ] **Step 5: Update AgentDescriptorMapper**

In `AgentDescriptorMapper.java`:

`toRecord()` — add `readJson(e.extensionData, new TypeReference<Map<String, Object>>() {})` as the last argument to the `new AgentDescriptor(...)` constructor call (after the constraints mapping).

`toEntity()` — add:
```java
e.extensionData = writeJson(d.extensionData());
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentRegistryTest`
Expected: all PASS

- [ ] **Step 7: Run full module tests to catch breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all PASS. Fix any compilation errors from the new record parameter in test files.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#177): add extensionData JPA persistence

JSON TEXT column, same pattern as disposition/templates.

Refs #177"
```

---

## Batch 3: Annotations

### Task 5: @ExtensionData and @ExtensionEntry annotations + processor + recorder

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/ExtensionData.java`
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/ExtensionEntry.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedAgentConfig.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java`
- Modify: `annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`
- Test: annotation integration test (existing test in annotations module or examples)

**Interfaces:**
- Consumes: `AgentDescriptor.Builder.extensionData(Map<String, Object>)` from Task 2
- Produces: `@ExtensionData({@ExtensionEntry(key="...", value="...")})` on a class produces a descriptor with populated extensionData

- [ ] **Step 1: Create @ExtensionEntry annotation**

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({})
@Retention(RetentionPolicy.RUNTIME)
public @interface ExtensionEntry {
    String key();
    String value();
}
```

- [ ] **Step 2: Create @ExtensionData annotation**

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ExtensionData {
    ExtensionEntry[] value();
}
```

- [ ] **Step 3: Add ExtensionEntryConfig to AnnotatedAgentConfig**

In `AnnotatedAgentConfig.java`, add a new inner class:

```java
public static class ExtensionEntryConfig {
    public String key;
    public String value;
    public ExtensionEntryConfig() {}
}
```

Add the field after `templateRefs`:
```java
public ExtensionEntryConfig[] extensionEntries;
```

- [ ] **Step 4: Update EidosAnnotationsProcessor to extract @ExtensionData**

In `EidosAnnotationsProcessor.java`, find the method that populates `AnnotatedAgentConfig` from Jandex annotations. Add extraction of `@ExtensionData`:

```java
// Look for @ExtensionData on the class
var extDataAnnotation = classInfo.annotation(DotName.createSimple("io.casehub.eidos.annotations.ExtensionData"));
if (extDataAnnotation != null) {
    var entries = extDataAnnotation.value().asNestedArray();
    var configs = new AnnotatedAgentConfig.ExtensionEntryConfig[entries.length];
    for (int i = 0; i < entries.length; i++) {
        var ec = new AnnotatedAgentConfig.ExtensionEntryConfig();
        ec.key = entries[i].value("key").asString();
        ec.value = entries[i].value("value").asString();
        configs[i] = ec;
    }
    config.extensionEntries = configs;
}
```

(Check the existing processor code for the exact Jandex API pattern — `classInfo.annotation()` vs `classInfo.declaredAnnotation()`, and how `value()` is accessed.)

- [ ] **Step 5: Update EidosAnnotationsRecorder to build extensionData**

In `EidosAnnotationsRecorder.java`, add before `buildCapabilities(config, builder);`:

```java
if (config.extensionEntries != null && config.extensionEntries.length > 0) {
    var extMap = new LinkedHashMap<String, Object>();
    for (var entry : config.extensionEntries) {
        extMap.put(entry.key, entry.value);
    }
    builder.extensionData(extMap);
}
```

Add `import java.util.LinkedHashMap;` if not already present.

- [ ] **Step 6: Run full build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Write an integration test or example**

If an annotation-based integration test exists in `examples/agent-scenarios/`, add a test class:

```java
@Identity(id = "ext-test", name = "Extension Test", slot = "tester")
@ExtensionData({
    @ExtensionEntry(key = "io.casehub.test.maxRetries", value = "3"),
    @ExtensionEntry(key = "io.casehub.test.mode", value = "strict")
})
public class ExtensionTestAgent {}
```

Then verify the descriptor in the test:
```java
@Test
void annotationBasedExtensionData() {
    var descriptor = registry.findById("ext-test", tenancyId).orElseThrow();
    assertNotNull(descriptor.extensionData());
    assertEquals("3", descriptor.extensionData().get("io.casehub.test.maxRetries"));
    assertEquals("strict", descriptor.extensionData().get("io.casehub.test.mode"));
}
```

(Check existing examples under `examples/agent-scenarios/` for the test pattern — `@QuarkusTest`, registry injection, tenancy ID resolution.)

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations/ examples/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#177): add @ExtensionData annotation support

Flat key-value pairs via @ExtensionEntry, extracted at build time
by EidosAnnotationsProcessor.

Refs #177"
```

### Task 6: Full build verification and consumer guide update

**Files:**
- Modify: `docs/guides/consumer-guide.md` (add extensionData section)

**Interfaces:**
- Consumes: all prior tasks
- Produces: passing full build, updated consumer documentation

- [ ] **Step 1: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS with all tests passing

- [ ] **Step 2: Update consumer guide**

Add an `### Extension Data` section to `docs/guides/consumer-guide.md` under the appropriate heading (after templates or where descriptor fields are documented). Content:

```markdown
### Extension Data

Applications can embed custom configuration alongside the standard descriptor
using the `extensionData` field — an opaque `Map<String, Object>`.

**YAML:**
```yaml
extensionData:
  io.myapp.config:
    maxRetries: 3
    mode: strict
```

**Annotations:**
```java
@ExtensionData({
    @ExtensionEntry(key = "io.myapp.maxRetries", value = "3"),
    @ExtensionEntry(key = "io.myapp.mode", value = "strict")
})
```

**Accessing:**
```java
Map<String, Object> ext = descriptor.extensionData();
var config = (Map<?, ?>) ext.get("io.myapp.config");
```

Use reverse-domain key notation to avoid collisions between extensions.
The annotation path supports flat String key-value pairs; use YAML for
nested structures. Extension data is never rendered in system prompts.
Maximum estimated size: 64KB.
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add docs/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs(#177): add extensionData section to consumer guide

Closes #177"
```

---

## References

- [2026-09-15-extension-data-design.md] — design spec this plan implements
- AgentDescriptor.java:7-300 — current record definition (22 params → 23)
- AgentDescriptorValidator.java:5-133 — validation constants and methods
- AgentDescriptorComparator.java:13-201 — drift detection (COMPARED_FIELD_COUNT 20 → 21)
- AgentDescriptorDeserializer.java:27-164 — YAML deserialization
- AgentDescriptorEntity.java:27-81 — JPA entity
- AgentDescriptorMapper.java:23-177 — entity ↔ record mapping
- V1__initial_schema.sql:1-127 — base schema (add extension_data TEXT)
- AnnotatedAgentConfig.java:3-113 — build-time config transfer
- EidosAnnotationsRecorder.java:13-181 — runtime descriptor construction
- EidosAnnotationsProcessor.java — build extension (Jandex scan)
- Protocol PP-20260613-608684 — A2A structural assembly hash coverage
- GitHub #177 — focal issue
