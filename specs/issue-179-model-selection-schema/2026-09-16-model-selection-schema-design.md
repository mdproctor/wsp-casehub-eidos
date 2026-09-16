# Design: Integrate model-selection schema from platform for task-level model requirements

**Issue:** casehubio/eidos#179
**Date:** 2026-09-16
**Status:** Draft

## Overview

Replaces the flat `modelTier` (String) and `modelCapabilities` (Set<String>) fields on
`AgentCapability` with a single `ModelQuery model` field from `casehub-platform-api`.
This gives eidos the full model selection surface — tier, capabilities, vendor, family,
locality, cost ceiling, context window, output size, and vendor preference — expressed
as either a string shorthand or an inline constraints object in YAML.

**Boundary principle (unchanged from #172):** eidos is the requirements declaration layer.
Platform (`RoutingAgentProvider`) is the selection authority. Eidos declares `ModelQuery`
on capabilities; platform reads them and resolves to a concrete model at dispatch time.

## 1. AgentCapability Record Change

Remove `modelTier` and `modelCapabilities`. Add `ModelQuery model`:

```java
public record AgentCapability(
    String name,
    String description,
    String capabilityVocabulary,
    Double qualityHint,
    Long latencyHintP50Ms,
    String costHint,
    ModelQuery model,               // NEW — replaces modelTier + modelCapabilities
    List<String> inputTypes,
    List<String> outputTypes,
    List<String> tags,
    Map<String, Double> epistemicDomains,
    Set<String> excludedDomains
) { ... }
```

### Compact Constructor Validation

```java
// ModelQuery is an opaque platform type — eidos validates only that
// it's non-null when provided. Tier vocabulary validation moves to
// CapabilityVocabularyValidator (see §4).
```

No structural validation on `ModelQuery` itself in the compact constructor — `ModelQuery`
owns its own invariants. Eidos validates the tier value against the vocabulary at
registration time (§4), not at construction time.

### Builder

```java
public Builder model(ModelQuery v) { this.model = v; return this; }
// Remove: modelTier(String), modelCapabilities(Set<String>)
```

## 2. eidos-api Dependency

Add `casehub-platform-api` to `api/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-api</artifactId>
    <version>${project.version}</version>
</dependency>
```

`platform-api` is pure Java (no CDI, no Quarkus) — same tier as eidos-api. No framework
pull-in.

## 3. YAML Deserialization — Union Type

The `model` field in descriptor YAML supports the union type from
`model-selection.schema.json`:

```yaml
# String shorthand — alias, tier ref, or model ID
capabilities:
  - name: code-review
    model: reasoning-heavy

# Inline constraints object
capabilities:
  - name: lint-check
    model:
      tier: FAST
      capabilities: [text]
      max-cost: LOW
```

### AgentDescriptorDeserializer Changes

In `AgentDescriptorDeserializer`, when parsing a capability node:

```java
JsonNode modelNode = node.get("model");
if (modelNode != null) {
    if (modelNode.isTextual()) {
        // String shorthand: could be alias, tier ref, or model ID.
        // Store as a ModelQuery with just the string metadata.
        // The platform router resolves aliases and tier refs at dispatch.
        String ref = modelNode.asText();
        if (ModelRef.isTierRef(ref)) {
            b.model(ModelQuery.builder()
                .tier(ModelTier.valueOf(ModelRef.parseTier(ref).name()))
                .build());
        } else {
            // Alias or model ID — store the raw string.
            // RoutingAgentProvider resolves aliases from ManifestResult.
            // Eidos stores this as a vendor field (convention: string
            // refs go through the alias path at dispatch time).
            b.model(ModelQuery.builder().vendor(ref).build());
        }
    } else if (modelNode.isObject()) {
        // Inline constraints — parse each field
        var qb = ModelQuery.builder();
        optionalText(modelNode, "vendor", qb::vendor);
        optionalText(modelNode, "family", qb::family);
        optionalText(modelNode, "tier", v -> qb.tier(ModelTier.valueOf(v)));
        optionalStringSet(modelNode, "capabilities", qb::requiredCapabilities);
        optionalText(modelNode, "locality",
            v -> qb.locality(ModelLocality.valueOf(v)));
        optionalText(modelNode, "max-cost",
            v -> qb.maxCostTier(CostTier.valueOf(v)));
        optionalInt(modelNode, "min-context", qb::minContextWindow);
        optionalInt(modelNode, "min-output", qb::minMaxOutput);
        optionalText(modelNode, "prefer-vendor", qb::preferVendor);
        b.model(qb.build());
    }
}
```

**YAML property naming:** The schema uses kebab-case (`max-cost`, `min-context`,
`prefer-vendor`). The YAML deserializer accepts kebab-case to match the published
schema. This is a deliberate local convention within the `model` block — it matches
what YAML authors will see in IDE autocompletion from the schema.

**String shorthand resolution:** When the model is a plain string that is not a tier
ref, it represents an alias or model ID. The eidos deserializer cannot resolve it at
parse time — alias resolution requires `ManifestResult` from the platform runtime.
Two options for how to carry the string through the descriptor:

**Approach: `preferVendor` carrier field.** Store the raw string in
`ModelQuery.preferVendor()` as a carrier. At dispatch time, the engine checks
`preferVendor()` for alias/model-ID strings before falling back to constraint matching.
This avoids adding a new field to `ModelQuery`.

**Alternative: Separate `modelRef` field on AgentCapability.** Add a `String modelRef`
alongside `ModelQuery model` — the string form goes into `modelRef`, the constraints
form goes into `model`. Cleaner semantics but adds a field and re-introduces some of the
flat-field complexity we're removing. Rejected — keep it simple.

> **Open question:** The `preferVendor` carrier approach is a semantic mismatch —
> `preferVendor` means "tiebreaker preference", not "resolve this alias." A cleaner
> solution is to add a `String alias` field to `ModelQuery` in platform-api. This is
> a platform-side change. For eidos #179, we use the `preferVendor` carrier and
> document the convention. Platform can add `alias` later if the pattern warrants it.

## 4. Registration-Time Validation

`CapabilityVocabularyValidator` changes to validate the `model.tier()` field:

```java
// Before: cap.modelTier() against urn:casehub:vocab:model-tier
// After:  cap.model().tier() against urn:casehub:vocab:model-tier
if (cap.model() != null && cap.model().tier() != null) {
    String tierValue = cap.model().tier().name().toLowerCase();
    validateAgainstVocabulary(tierValue, MODEL_TIER_URI, registry);
}
```

`modelCapabilities` had no vocabulary validation (open strings) — same applies to
`model.requiredCapabilities()`. No change needed.

## 5. Rendering

Follows the existing renderer protocol: routing signals are A2A_CARD only.

### A2A_CARD

The `model` field renders as a JSON object inside each capability, replacing the
separate `modelTier` and `modelCapabilities` fields:

```json
{
  "name": "code-review",
  "description": "Reviews code for quality and correctness",
  "model": {
    "tier": "FLAGSHIP",
    "capabilities": ["text", "tool-use"],
    "max-cost": "HIGH"
  },
  "qualityHint": 0.95,
  "latencyHintP50Ms": 8000,
  "costHint": "high"
}
```

Only non-null fields from `ModelQuery` render. When `model` has only a `tier` set,
it renders as `"model": {"tier": "FLAGSHIP"}`.

**A2A structural assembly hash:** The `model` field replaces `modelTier` and
`modelCapabilities` in the hash payload. All non-null ModelQuery fields contribute
to the hash.

### MARKDOWN / PROSE

`model` is suppressed — routing signals for machine consumption, not LLM prompt
content. Same treatment as the fields it replaces.

## 6. Annotations

### @AgentCapabilityDef

```java
@Repeatable(AgentCapabilityDefs.class)
public @interface AgentCapabilityDef {
    // ... existing fields ...
    String model() default "";              // NEW — string shorthand (alias, tier ref, model ID)
    String modelTier() default "";          // KEPT — structured form, maps to ModelQuery.tier
    String[] modelCapabilities() default {}; // KEPT — structured form, maps to ModelQuery.requiredCapabilities
}
```

**Validation:** Setting both `model` and any of `modelTier`/`modelCapabilities`
simultaneously is a build-time error in `EidosAnnotationsProcessor`.

### EidosAnnotationsProcessor

Extracts annotation values, builds `AnnotatedAgentConfig` with model data:

```java
if (notEmpty(ann, "model")) {
    cap.modelRef = stringValue(ann, "model");  // string shorthand
}
if (notEmpty(ann, "modelTier")) {
    cap.modelTier = stringValue(ann, "modelTier");
}
cap.modelCapabilities = stringArrayValue(ann, "modelCapabilities");

// Validation: mutual exclusion
if (notEmpty(cap.modelRef) && (notEmpty(cap.modelTier) || cap.modelCapabilities.length > 0)) {
    throw buildError("@AgentCapabilityDef: 'model' and 'modelTier'/'modelCapabilities' "
        + "are mutually exclusive on capability '" + cap.name + "'");
}
```

### EidosAnnotationsRecorder

The recorder builds `ModelQuery` from whichever annotation form was used:

```java
if (notEmpty(cap.modelRef)) {
    // String shorthand — same resolution as YAML string form
    String ref = cap.modelRef;
    if (ModelRef.isTierRef(ref)) {
        cb.model(ModelQuery.builder()
            .tier(ModelTier.valueOf(ModelRef.parseTier(ref).name()))
            .build());
    } else {
        cb.model(ModelQuery.builder().preferVendor(ref).build());
    }
} else if (notEmpty(cap.modelTier)) {
    var qb = ModelQuery.builder();
    qb.tier(ModelTier.valueOf(cap.modelTier.toUpperCase()));
    if (cap.modelCapabilities != null && cap.modelCapabilities.length > 0) {
        qb.requiredCapabilities(Set.of(cap.modelCapabilities));
    }
    cb.model(qb.build());
}
```

**Hybrid vocabulary validation** at build time: when `casehub-eidos-vocab` is on the
classpath, validate `modelTier` annotation values against `ModelTierTerm` enum values
(same pattern as before #179 — the check target changes from the record field to the
annotation attribute).

## 7. JPA Schema

Replace `model_tier` and `model_capabilities` columns on `agent_capability` table with
a single `model` JSON column:

```sql
-- V<next>__model_query_on_capability.sql
ALTER TABLE agent_capability DROP COLUMN model_tier;
ALTER TABLE agent_capability DROP COLUMN model_capabilities;
ALTER TABLE agent_capability ADD COLUMN model JSONB;
```

**No existing installations** — per project schema convention, this goes directly into
`V1__initial_schema.sql` as a modification, not a separate migration. The `model_tier`
and `model_capabilities` columns are replaced with:

```sql
model JSONB,
```

### Entity Mapping

```java
@Column(name = "model", columnDefinition = "JSONB")
@Convert(converter = ModelQueryConverter.class)
private ModelQuery model;
```

`ModelQueryConverter` implements `AttributeConverter<ModelQuery, String>` using Jackson
for JSON serialization. `ModelQuery` is already a Jackson-friendly record.

## 8. Module Impact Summary

| Module | Change |
|--------|--------|
| `casehub-eidos-api` | Add `platform-api` dep. Replace `modelTier`/`modelCapabilities` with `ModelQuery model` on `AgentCapability`. Update `Builder`. Update `CapabilityVocabularyValidator`. |
| `casehub-eidos` (runtime) | `AgentDescriptorDeserializer` — union type parsing. `EidosRenderPipeline` — A2A_CARD rendering. `AgentCapabilityEntity` + `AgentDescriptorMapper` — JPA mapping. `V1__initial_schema.sql` — column change. |
| `casehub-eidos-annotations` (deployment) | `@AgentCapabilityDef` — add `model` attribute. `EidosAnnotationsProcessor` — mutual exclusion validation. `EidosAnnotationsRecorder` — build ModelQuery. `AnnotatedAgentConfig` — add `modelRef` field. |
| `casehub-eidos-vocab` | No change — `ModelTierTerm` stays as-is. |
| `casehub-eidos-memory` | No change |
| `casehub-eidos-routing` | No change |
| `casehub-eidos-org-*` | No change (D3) |
| `casehub-eidos-eval` | Update eval YAML profiles to use `model:` field |
| `examples/` | Update YAML descriptors and test code |

## 9. Migration Checklist

Since this is a breaking change to the `AgentCapability` record, all call sites must
update. The migration is mechanical:

1. `.modelTier("flagship")` → `.model(ModelQuery.builder().tier(ModelTier.FLAGSHIP).build())`
2. `.modelCapabilities(Set.of("text"))` → fold into the same `ModelQuery.builder()` call
3. `cap.modelTier()` → `cap.model() != null ? cap.model().tier() : null`
4. `cap.modelCapabilities()` → `cap.model() != null ? cap.model().requiredCapabilities() : Set.of()`

## References

- `api/src/main/java/io/casehub/eidos/api/AgentCapability.java` — existing record with modelTier + modelCapabilities
- `api/src/main/java/io/casehub/eidos/api/CapabilityVocabularyValidator.java` — tier validation
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java` — A2A_CARD rendering (modelTier, modelCapabilities)
- `runtime/src/main/java/io/casehub/eidos/runtime/yaml/AgentDescriptorDeserializer.java` — YAML capability parsing
- `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentCapabilityEntity.java` — JPA entity
- `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java` — entity↔record mapping
- `runtime/src/main/resources/db/eidos/migration/V1__initial_schema.sql` — schema definition
- `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilityDef.java` — annotation
- `annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java` — build extension
- `platform-api/.../ModelQuery.java` — upstream model query record
- `platform-api/.../ModelTier.java` — upstream tier enum
- `agent-config-core/src/main/resources/schema/model-selection.schema.json` — upstream schema
- `agent-router-core/.../RoutingAgentProvider.java` — upstream router (resolves ModelQuery)
- `docs/specs/issue-172-model-selection-vocab/2026-09-14-model-selection-vocab-design.md` — prior design
- `docs/specs/issue-172-model-selection-vocab/decisions.md` — prior decisions (D1-D6)
- `docs/protocols/renderer/capability-metadata-rendering.md` — routing signals render in A2A_CARD only
- `docs/protocols/renderer/a2a-structural-assembly-hash-coverage.md` — hash coverage protocol
