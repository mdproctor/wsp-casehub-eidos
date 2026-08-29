# Descriptor Codegen — Design Spec

**Issue:** casehubio/eidos#147
**Date:** 2026-08-29
**Status:** Draft

## Summary

Replace `ClasspathYamlDescriptorRegistrar`'s hand-maintained `DescriptorConfig` intermediate
classes with a Jackson Module pattern (custom deserializer + mixins), following engine's
`CaseDefinitionModule` precedent. YAML deserializes directly into `AgentDescriptor` via the
builder. Convenience fields (`mbtiType`, `enneagramType`) handled by the custom deserializer
with `VocabularyRegistry` access via CDI injection into the Jackson Module.

## This Branch's Scope

| Decision | In scope | Notes |
|---|---|---|
| D2: Eliminate DescriptorConfig | Yes | Jackson Module + custom deserializer |
| D5: Convenience fields in Module | Yes | Part of the deserializer |
| D1: Generate builders | Deferred | Needs shared schema generator or standalone tool |
| D3: Shared schema generator | Deferred | Platform work — spec filed on parent |
| D4: Generate AnnotatedAgentConfig | Deferred | Independent of D3, but separate branch |
| D6: Publish JSON Schema | Deferred | Needs D3 |

## Architecture

### Current state

```
descriptors.yaml → Jackson YAML → DescriptorConfig (inner POJO)
                                       ↓
                               toDescriptor() manual mapping
                                       ↓
                               AgentDescriptor (record)
```

`DescriptorConfig` and 6 inner classes (`DispositionConfig`, `CapabilityConfig`, `GoalConfig`,
`ConstraintConfig`, `TemplateRefConfig`, `DispositionValueConfig`) are hand-maintained in
`ClasspathYamlDescriptorRegistrar`. When `AgentDescriptor` gains a field, `DescriptorConfig`
and `toDescriptor()` must be updated manually.

### Target state

```
descriptors.yaml → Jackson YAML + EidosDescriptorModule → AgentDescriptor.Builder → AgentDescriptor
```

No intermediate config classes. Jackson deserialises YAML directly into `AgentDescriptor`
via a custom deserializer that uses the builder. The `EidosDescriptorModule` (Jackson Module)
registers the deserializer and provides `VocabularyRegistry` access for convenience fields.

### Components

```
runtime/src/main/java/io/casehub/eidos/runtime/
  yaml/
    EidosDescriptorModule.java       — Jackson Module, CDI-injected VocabularyRegistry
    AgentDescriptorDeserializer.java — custom JsonDeserializer<AgentDescriptor>
    DispositionDeserializer.java     — String axis values → List<DispositionValue>
    CapabilityDeserializer.java      — handles full capability metadata
  registrar/
    ClasspathYamlDescriptorRegistrar.java — simplified: uses EidosDescriptorModule, no config classes
```

## Detailed Design

### EidosDescriptorModule

```java
@ApplicationScoped
public class EidosDescriptorModule extends SimpleModule {

    @Inject
    Instance<VocabularyRegistry> vocabularyRegistry;

    @PostConstruct
    void init() {
        addDeserializer(AgentDescriptor.class,
            new AgentDescriptorDeserializer(vocabularyRegistry));
        addDeserializer(AgentDisposition.class,
            new DispositionDeserializer(vocabularyRegistry));
    }
}
```

CDI-managed. The `VocabularyRegistry` is injected as `Instance<>` (optional — may not be
available in all contexts). When the registrar creates its `ObjectMapper`, it registers this
module.

### AgentDescriptorDeserializer

Custom `JsonDeserializer<AgentDescriptor>` that:

1. Reads the YAML node tree
2. Constructs `AgentDescriptor` via the builder
3. Handles field-by-field mapping including:
   - **Simple fields** — direct string forwarding (`agentId`, `name`, `slot`, etc.)
   - **tenancyId** — read from YAML if present; fall back to MicroProfile Config
     (`casehub.eidos.annotations.default-tenancy-id`, default `"default"`) when absent
   - **axisVocabularies** — `Map<String, String>` in YAML → `Map<DispositionAxis, String>`
     via `DispositionAxis.valueOf(key)`. This mapping lives here (not in DispositionDeserializer)
     because `axisVocabularies` is on `AgentDescriptor`, not `AgentDisposition`.
   - **Capabilities** — delegates to `CapabilityDeserializer` for full metadata
   - **Goals and constraints** — direct record construction with enum parsing
   - **Templates** — direct `TemplateRef` construction
   - **Disposition** — delegates to `DispositionDeserializer`

**tenancyId handling:** The current `ClasspathYamlDescriptorRegistrar` doesn't set tenancyId
on descriptors (the `DescriptorConfig` has no tenancyId field). The recorder path injects it
from config. For the YAML path, the deserializer reads an optional `tenancyId` field from
YAML (if present), falling back to the MicroProfile Config value.

**Error handling:** The `AgentDescriptor` compact constructor validates on construction.
Invalid YAML will throw `AgentValidationException` from the constructor, wrapped in
Jackson's `JsonMappingException`. This is fail-fast — the error message will reference
the validation constraint that failed (e.g., "duplicate capability name: X").

### DispositionDeserializer

Handles the type adaptation: YAML `socialOrient: "collaborative"` → record
`List<DispositionValue>`.

```java
public class DispositionDeserializer extends JsonDeserializer<AgentDisposition> {

    private final Instance<VocabularyRegistry> vocabRegistry;

    @Override
    public AgentDisposition deserialize(JsonParser p, DeserializationContext ctxt)
            throws IOException {
        ObjectNode root = p.readValueAsTree();
        var builder = AgentDisposition.builder();

        // String axis values → List<DispositionValue>
        ifString(root, "socialOrient", builder::socialOrient);
        ifString(root, "ruleFollowing", builder::ruleFollowing);
        ifString(root, "riskAppetite", builder::riskAppetite);
        ifString(root, "autonomy", builder::autonomy);
        ifString(root, "conflictMode", builder::conflictMode);

        // Boolean
        if (root.has("delegation")) builder.delegation(root.get("delegation").asBoolean());

        // Weighted profiles
        ifArray(root, "dispositionProfile", builder::dispositionProfile);
        ifArray(root, "styleProfile", builder::styleProfile);

        // Convenience: mbtiType → dispositionProfile via VocabularyRegistry
        if (root.has("mbtiType") && vocabRegistry.isResolvable()) {
            var registry = vocabRegistry.get();
            String mbtiType = root.get("mbtiType").asText().toLowerCase(Locale.ROOT);
            registry.resolve("urn:casehub:vocab:mbti", mbtiType)
                .ifPresent(term -> builder.dispositionProfile(term.defaultProfile()));
        }

        // Convenience: enneagramType → axis values via VocabularyRegistry
        if (root.has("enneagramType") && vocabRegistry.isResolvable()) {
            // Port existing logic from ClasspathYamlDescriptorRegistrar lines 75-99
            resolveEnneagramAxes(root, builder);
        }

        return builder.build();
    }
}
```

**Key point:** The `String → List<DispositionValue>` adaptation uses the builder's existing
`socialOrient(String)` convenience method (which creates `List.of(DispositionValue.of(s))`).
No new conversion logic needed — the builder already handles this.

### CapabilityDeserializer

Not strictly required — `AgentCapability` is a record with all-JSON-friendly types. Jackson
can deserialize it directly. But `epistemicDomains` (Map<String, Double>) and `excludedDomains`
(Set<String>) benefit from explicit handling to produce the correct collection types.

Evaluate during implementation: if Jackson handles `AgentCapability` correctly without a
custom deserializer, skip this class.

### ClasspathYamlDescriptorRegistrar changes

The registrar simplifies dramatically:

```java
@ApplicationScoped
public class ClasspathYamlDescriptorRegistrar implements AgentDescriptorRegistrar {

    private static final String RESOURCE_PATH = "META-INF/eidos/descriptors.yaml";

    @Inject
    EidosDescriptorModule eidosModule;

    private ObjectMapper createMapper() {
        var mapper = new ObjectMapper(new YAMLFactory())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, true);
        mapper.registerModule(eidosModule);
        return mapper;
    }

    @Override
    public List<AgentDescriptor> descriptors() {
        // Same classpath scanning as today
        // But deserialization is now:
        //   mapper.readValue(yaml, DescriptorFile.class)
        // where DescriptorFile is just:
        //   record DescriptorFile(List<AgentDescriptor> descriptors) {}
    }
}
```

**Deleted:** `DescriptorConfig`, `DispositionConfig`, `CapabilityConfig`, `GoalConfig`,
`ConstraintConfig`, `TemplateRefConfig`, `DispositionValueConfig` (all 7 inner classes),
and the `toDescriptor()` method.

**Retained:** `DescriptorFile` — kept as a static inner class with public field
(`public List<AgentDescriptor> descriptors`), not converted to a record. This avoids
Jackson record-deserialization issues (records need `@JsonCreator` or parameter-names module).
The class has a single field and no drift risk.

**Non-CDI path:** A static factory method creates an `ObjectMapper` with the module
configured without CDI — passing `null` for `VocabularyRegistry`. This preserves the
existing `loadFrom(InputStream)` test API and matches the current `loadFrom(yaml, null)`
behaviour where convenience fields (`mbtiType`, `enneagramType`) silently skip when
no vocabulary registry is available.

## Test Strategy

### Unit tests for the deserializer

1. **Round-trip from YAML string** — parse a complete descriptor YAML, verify all fields
   populated on the `AgentDescriptor`
2. **String axis adaptation** — `socialOrient: "collaborative"` produces
   `List.of(DispositionValue.of("collaborative"))`
3. **mbtiType convenience** — `mbtiType: "INTJ"` populates `dispositionProfile` via vocabulary
   lookup (requires `VocabularyRegistry` with MBTI vocab registered)
4. **enneagramType convenience** — `enneagramType: "type_1"` populates axes via cross-vocabulary
   equivalentValues
5. **tenancyId injection** — YAML without `tenancyId` uses config default; YAML with explicit
   `tenancyId` uses the declared value
6. **Validation error** — YAML with duplicate capability names throws
   `AgentValidationException` (from compact constructor, wrapped in `JsonMappingException`)
7. **Unknown properties** — YAML with unknown fields fails (`FAIL_ON_UNKNOWN_PROPERTIES = true`)
8. **Full capability metadata** — `qualityHint`, `latencyHintP50Ms`, `costHint`,
   `epistemicDomains`, `excludedDomains`, `inputTypes`, `outputTypes`, `tags` all deserialize
9. **Templates** — `templates` with `ref` and `args` deserialize to `TemplateRef`
10. **Goals and constraints** — priority, visibility, severity, capability references all
    deserialize correctly
11. **axisVocabularies** — `Map<String, String>` in YAML deserializes to
    `Map<DispositionAxis, String>` on the descriptor
12. **loadFrom without CDI** — `loadFrom(InputStream)` works with null VocabularyRegistry,
    convenience fields silently skip

### Existing test migration

`ClasspathYamlDescriptorRegistrarTest` (if it exists) and the YAML schema tests from #143
must still pass. The YAML format doesn't change — only the internal deserialization path.

### Parity test

A new test verifies that every field on `AgentDescriptor` is handled by the deserializer.
Uses reflection to list record components and asserts each has a corresponding YAML test case.
This prevents new fields from being silently ignored.

## Migration — What Changes for YAML Users

**Nothing.** The YAML format is unchanged. All existing `META-INF/eidos/descriptors.yaml`
files continue to work. The change is internal — how the YAML is deserialized.

## Files Changed

| File | Change |
|---|---|
| `runtime/.../yaml/EidosDescriptorModule.java` | New — Jackson Module |
| `runtime/.../yaml/AgentDescriptorDeserializer.java` | New — custom deserializer |
| `runtime/.../yaml/DispositionDeserializer.java` | New — type adaptation |
| `runtime/.../registrar/ClasspathYamlDescriptorRegistrar.java` | Simplified — delete config classes, use module |
| Tests | New deserializer tests + parity test |

## Risks

1. **Compact constructor fail-fast** — validation errors surface as `JsonMappingException`
   wrapping `AgentValidationException`. Error messages will differ from the current path
   (which validates at `toDescriptor()` time). The content is the same; the wrapping changes.

2. **VocabularyRegistry availability** — in test contexts where no CDI container is running,
   the `Instance<VocabularyRegistry>` won't be resolvable. Convenience fields (`mbtiType`,
   `enneagramType`) silently skip when registry is unavailable. This matches current behaviour
   (the `loadFrom(InputStream)` overload passes `null` for registry).

3. **Ordering** — `AgentDescriptor` compact constructor validates goal-capability cross-references.
   The deserializer must set capabilities before goals on the builder. Field order in YAML
   shouldn't matter (Jackson reads the full tree first), but the builder call order matters.

4. **ObjectMapper lifecycle** — current code uses a `static final ObjectMapper`. The new
   design creates the mapper at CDI init time (non-static, module-injected). Any code path
   that caches or shares the ObjectMapper outside CDI scope won't have the module registered.
   Mitigated by keeping the mapper internal to the registrar.

5. **enneagramType resolution complexity** — the enneagram → axis resolution involves
   cross-vocabulary `equivalentValues` lookups across 5 axes plus a special case for
   `CONFLICT_MODE` (Thomas-Kilmann). ~25 lines of axis-specific logic to port from
   `ClasspathYamlDescriptorRegistrar` lines 75-99.

## References

- D2, D5 in `specs/issue-147-descriptor-codegen/decisions.md`
- Decision review in `specs/issue-147-descriptor-codegen/decision-review.md`
- `io.casehub.api.model.converter.CaseDefinitionModule` in engine — pattern precedent
- `io.casehub.api.model.converter.deser.CaseDefinitionDeserializer` in engine — custom deserializer precedent
- `io.casehub.model.marshaller.WorkerMarshaller` in engine — custom (de)serializer precedent
- `ClasspathYamlDescriptorRegistrar.toDescriptor()` — current mapping to port
- `ClasspathYamlDescriptorRegistrar` lines 70-99 — mbtiType/enneagramType logic to port
- Platform specs: `2026-08-29-shared-yaml-core-design.md`, `2026-08-29-shared-schema-generator-design.md`
- casehubio/eidos#146 — annotation parity gaps (partially addressed by D4, deferred)
- casehubio/quarkmind#283 — personality generator wizard (downstream consumer of schema)
