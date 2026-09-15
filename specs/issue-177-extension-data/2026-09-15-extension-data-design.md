# Add extensionData field to AgentDescriptor

**Issue:** casehubio/eidos#177
**Date:** 2026-09-15
**Status:** Design

## Problem

`AgentDescriptor` has no mechanism for application-specific configuration. Applications like wacky-manor need to embed custom config (social profiles, game parameters, deployment metadata) alongside the standard descriptor. The current workaround is a separate YAML file (`META-INF/eidos/social-config.yaml` loaded by `ManorSocialConfigLoader`), which splits character definition across two files and requires a custom loader.

## Solution

Add `Map<String, Object> extensionData` to `AgentDescriptor` — an opaque, application-owned configuration bag. The field is:

- **Optional** — `null` when absent (not empty map), consistent with `axisVocabularies`
- **Deeply immutable** — recursive deep copy at construction time via `ExtensionDataCopier`
- **Size-limited** — estimated size validated at construction time (`MAX_EXTENSION_DATA_SIZE = 65536`)
- **Never rendered** — excluded from all prompt formats; does not participate in cache hashing
- **Namespace-by-convention** — reverse-domain keys recommended (e.g. `io.casehub.manor.socialConfig`)
- **Dual declaration path** — full nested structures via YAML, flat key-value pairs via annotations

## API Changes

### AgentDescriptor (api/)

Add `Map<String, Object> extensionData` as the 23rd record parameter, after `constraints`.

Compact constructor additions:
```java
if (extensionData != null) {
    extensionData = ExtensionDataCopier.deepCopy(extensionData);
    AgentDescriptorValidator.validateExtensionData(extensionData);
}
```

Builder: add `extensionData(Map<String, Object> v)` method.
`toBuilder()`: include `extensionData`.

### ExtensionDataCopier (api/)

New utility class. Two static methods:

**`deepCopy(Map<String, Object>)`** — recursively copies the map tree. Handles only the types YAML/JSON deserialization produces:
- `Map` → new `LinkedHashMap` with recursively copied entries
- `List` → new `ArrayList` with recursively copied elements
- `String`, `Number`, `Boolean`, `null` → passed through (immutable)
- Any other type → `IllegalArgumentException`

**`estimateSize(Map<String, Object>)`** — walks the tree, returns an estimated character count:
- Each string key/value: `String.length()`
- Each map entry: key length + value estimate + 6 (structural: `"":,` + overhead)
- Each list element: value estimate + 2 (structural: `,` + overhead)
- Each map/list container: +2 (`{}` or `[]`)
- Each number/boolean: `String.valueOf().length()`
- Null: 4 (`null`)

The estimate is conservative (slightly over-counts). Maps near the 65536 boundary may be rejected even if their actual JSON serialization is under the limit. This is acceptable — the purpose is abuse prevention, not exact enforcement.

### AgentDescriptorValidator (api/)

Add:
```java
public static final int MAX_EXTENSION_DATA_SIZE = 65_536;

public static void validateExtensionData(Map<String, Object> extensionData) {
    long estimated = ExtensionDataCopier.estimateSize(extensionData);
    if (estimated > MAX_EXTENSION_DATA_SIZE) {
        throw new AgentValidationException("extensionData",
            "estimated size " + estimated + " exceeds maximum " + MAX_EXTENSION_DATA_SIZE);
    }
}
```

### AgentDescriptorComparator (api/)

Add `extensionData` to `compareSimpleFields()`:
```java
compareField(drifts, "extensionData", desired.extensionData(), actual.extensionData());
```

Increment `COMPARED_FIELD_COUNT` from 20 to 21.

`Objects.equals()` → `Map.equals()` works recursively for the types allowed by `ExtensionDataCopier` (all implement `equals()` correctly). After deep copy, both sides are guaranteed to be composed of these types.

## YAML Deserialization (eidos-core/)

### AgentDescriptorDeserializer

Add handling for the `extensionData` key:
```java
if (root.has("extensionData") && root.get("extensionData").isObject()) {
    builder.extensionData(readMapTree(root.get("extensionData")));
}
```

`readMapTree(JsonNode)` — new private method that recursively converts a Jackson tree node to `Map<String, Object>`:
- ObjectNode → `LinkedHashMap<String, Object>` with recursively converted values
- ArrayNode → `ArrayList<Object>` with recursively converted elements
- TextNode → `String`
- NumericNode → `Number` (preserving Integer/Long/Double)
- BooleanNode → `Boolean`
- NullNode → `null`

### YAML syntax

```yaml
agentId: wacky-manor-host
name: Manor Host
slot: host
tenancyId: default
extensionData:
  io.casehub.manor.socialConfig:
    drives: [belonging, approval]
    norms:
      formality: 0.3
      humor: 0.8
    initial-beliefs:
      - "The manor is a place of refinement"
      - "Guests should feel welcomed"
  io.casehub.manor.gameConfig:
    maxTurns: 20
    difficultyLevel: medium
```

The `extensionData` key is optional. When absent, the field is `null`.

## JPA Persistence (runtime/)

### AgentDescriptorEntity

Add:
```java
@Column(name = "extension_data", columnDefinition = "TEXT")
String extensionData;
```

Stored as serialized JSON, consistent with `disposition`, `templates`, and `axisVocabularies`.

### AgentDescriptorMapper

`toRecord()` — add to the constructor call:
```java
readJson(e.extensionData, new TypeReference<Map<String, Object>>() {})
```

`toEntity()` — add:
```java
e.extensionData = writeJson(d.extensionData());
```

### Schema

Rewrite the base migration file to add `extension_data TEXT` to the `agent_descriptor` table. No Flyway versioned migration needed (no deployed instances — per schema convention in CLAUDE.md).

## Annotations (casehub-eidos-annotations/)

### New annotations (runtime/)

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ExtensionData {
    ExtensionEntry[] value();
}
```

```java
@Target({})
@Retention(RetentionPolicy.RUNTIME)
public @interface ExtensionEntry {
    String key();
    String value();
}
```

`@ExtensionData` is a separate container annotation, not nested in `@Identity`. This positions extension data as secondary metadata, not core identity. `@ExtensionEntry` uses `@Target({})` (nested-only, consistent with `@EpistemicDomain` and `@TemplateArg`).

### Usage

```java
@Identity(id = "manor-host", name = "Manor Host", slot = "host")
@ExtensionData({
    @ExtensionEntry(key = "io.casehub.manor.maxTurns", value = "20"),
    @ExtensionEntry(key = "io.casehub.manor.difficulty", value = "medium")
})
public class ManorHostAgent { }
```

Values are always `String` in the annotation path. The resulting `Map<String, Object>` contains `String` values — type-compatible with the field type. Applications needing nested structures use YAML.

### EidosAnnotationsProcessor (deployment/)

Extract `@ExtensionData` entries from Jandex scan. Build `Map<String, String>` from `@ExtensionEntry` annotations. Pass to `AnnotatedAgentConfig` for runtime reconstruction.

### AnnotatedAgentConfig (runtime/)

Add `Map<String, String> extensionData` field. The recorder converts this to `Map<String, Object>` when constructing the `AgentDescriptor` via Builder.

## Rendering

**extensionData is NOT rendered in any format.** The `EidosRenderPipeline` and `EidosSystemPromptRenderer` do not read or reference this field. It does not appear in `buildDescriptorPayload()` for any format.

**Protocol guard:** Per protocol PP-20260613-608684, any future change that renders extensionData in any format MUST also add it to `buildDescriptorPayload()` to maintain cache coherence. This is documented here as a standing constraint.

**Caching implication:** Descriptors differing only in `extensionData` produce identical rendered prompts and share cache entries. This is correct and intentional — extension data is not prompt-relevant.

## In-Memory (persistence-memory/)

No changes needed. `InMemoryAgentRegistry` stores `AgentDescriptor` records directly.

## Namespacing Convention

Top-level keys in `extensionData` SHOULD use reverse-domain notation to avoid collisions:
- `io.casehub.manor.socialConfig`
- `io.casehub.manor.gameConfig`
- `com.example.myapp.settings`

This is a convention, not enforced structurally. This decision assumes each descriptor is owned by a single application. If descriptor composition across application boundaries is introduced, structural namespacing enforcement must be revisited.

## Test Plan

1. **ExtensionDataCopier tests** — deep copy correctness (nested maps, lists, mixed types), unknown type rejection, null handling, empty map handling
2. **ExtensionDataCopier.estimateSize tests** — size estimation accuracy, boundary cases near MAX_EXTENSION_DATA_SIZE
3. **AgentDescriptor compact constructor** — null extensionData, valid extensionData, oversized extensionData rejection, deep immutability verification (mutating source map after construction doesn't affect descriptor)
4. **AgentDescriptorComparator** — extensionData participation in comparison, equal/unequal cases
5. **AgentDescriptorDeserializer** — YAML with extensionData (nested maps, lists, primitives), YAML without extensionData (null)
6. **JPA round-trip** — register descriptor with extensionData, retrieve, verify equality
7. **Annotations** — @ExtensionData with @ExtensionEntry, verify Map<String, Object> construction
8. **Builder/toBuilder** — extensionData preservation through builder round-trip

## References

- casehubio/eidos#177 — Add extensionData field to AgentDescriptor
- casehubio/examples#56 — Move SocialConfig to Eidos extensionData (motivating use case)
- AgentDescriptor.java:7-300 — current record definition
- AgentDescriptorDeserializer.java:27-164 — YAML deserialization
- AgentDescriptorEntity.java:27-81 — JPA entity
- AgentDescriptorMapper.java:23-177 — entity ↔ record mapping
- AgentDescriptorComparator.java:13-201 — drift detection
- EidosAnnotationsProcessor.java — build extension for annotation processing
- Protocol PP-20260613-608684 — A2A structural assembly hash coverage
