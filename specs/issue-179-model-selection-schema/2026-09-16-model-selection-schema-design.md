# Design: Integrate model-selection schema from platform for task-level model requirements

**Issue:** casehubio/eidos#179
**Date:** 2026-09-16
**Status:** Draft

## Overview

Replaces the flat `modelTier` (String) and `modelCapabilities` (Set<String>) fields on
`AgentCapability` with two mutually exclusive model selection fields from
`casehub-platform-api`: `String modelRef` for string shorthands (aliases, tier refs,
model IDs) and `ModelQuery model` for inline constraint queries. This mirrors the
platform's own `AgentSessionConfig` which has both `String model` and `ModelQuery
modelQuery`.

YAML supports the union type from `model-selection.schema.json`: `model: "reasoning-heavy"`
(string shorthand → `modelRef`) or `model: {tier: FAST, capabilities: [reasoning]}`
(inline constraints → `model`).

**Boundary principle (unchanged from #172):** eidos is the requirements declaration layer.
Platform (`RoutingAgentProvider`) is the selection authority. Eidos declares model
requirements on capabilities; platform resolves to a concrete model at dispatch time.

## 1. AgentCapability Record Change

Remove `modelTier` and `modelCapabilities`. Add `String modelRef` and `ModelQuery model`:

```java
public record AgentCapability(
    String name,
    String description,
    String capabilityVocabulary,
    Double qualityHint,
    Long latencyHintP50Ms,
    String costHint,
    String modelRef,                // NEW — string shorthand (alias, tier ref, model ID)
    ModelQuery model,               // NEW — inline constraint query
    List<String> inputTypes,
    List<String> outputTypes,
    List<String> tags,
    Map<String, Double> epistemicDomains,
    Set<String> excludedDomains
) { ... }
```

`modelRef` and `model` are mutually exclusive — setting both is a validation error.
This mirrors `AgentSessionConfig` on the platform side, which has both `String model`
and `ModelQuery modelQuery`.

The engine reads the descriptor and dispatches accordingly:
- `modelRef` set → `config.withModel(modelRef)` → string resolution (aliases, tier refs, registry IDs)
- `model` set → `config.withModel(model)` → direct query resolution
- Neither set → no model constraint, platform uses default

### Compact Constructor Validation

```java
if (modelRef != null && model != null) {
    throw new AgentValidationException("model",
        "modelRef and model are mutually exclusive");
}
AgentDescriptorValidator.validateOptional("modelRef", modelRef,
    AgentDescriptorValidator.MAX_CAPABILITY_STRING);
// ModelQuery owns its own invariants. Tier vocabulary validation
// happens at registration time via CapabilityVocabularyValidator (§4).
```

### Builder

```java
public Builder modelRef(String v) { this.modelRef = v; return this; }
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
        // String shorthand → modelRef (alias, tier ref, or model ID)
        b.modelRef(modelNode.asText());
    } else if (modelNode.isObject()) {
        // Inline constraints → ModelQuery
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

String shorthands go to `modelRef`, constraint objects go to `model`. No conversion
needed at parse time — the engine dispatches each form through the appropriate
`RoutingAgentProvider` path at invocation time.

**YAML property naming:** The schema uses kebab-case (`max-cost`, `min-context`,
`prefer-vendor`). The YAML deserializer accepts kebab-case to match the published
schema. This is a deliberate local convention within the `model` block — it matches
what YAML authors will see in IDE autocompletion from the schema.

## 4. Registration-Time Validation

`CapabilityVocabularyValidator` changes to validate model tier from both paths:

```java
// From ModelQuery (inline constraints)
if (cap.model() != null && cap.model().tier() != null) {
    String tierValue = cap.model().tier().name().toLowerCase();
    validateAgainstVocabulary(tierValue, MODEL_TIER_URI, registry);
}

// From modelRef (string shorthand) — validate tier refs only
if (cap.modelRef() != null && ModelRef.isTierRef(cap.modelRef())) {
    String tierValue = ModelRef.parseTier(cap.modelRef()).name().toLowerCase();
    validateAgainstVocabulary(tierValue, MODEL_TIER_URI, registry);
}
// Non-tier-ref strings (aliases, model IDs) pass through unvalidated —
// they resolve at dispatch time via RoutingAgentProvider.
```

`model.requiredCapabilities()` — no vocabulary validation (open strings). No change
from previous behaviour.

## 5. Rendering

Follows the existing renderer protocol: routing signals are A2A_CARD only.

### A2A_CARD

Replaces the separate `modelTier` and `modelCapabilities` fields with a unified `model`
field. The value depends on which form is set:

When `modelRef` is set (string shorthand):
```json
{
  "name": "code-review",
  "model": "reasoning-heavy",
  "qualityHint": 0.95
}
```

When `model` is set (inline constraints):
```json
{
  "name": "code-review",
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

Only non-null fields from `ModelQuery` render in the object form.

**A2A structural assembly hash:** The `model` field replaces `modelTier` and
`modelCapabilities` in the hash payload. For `modelRef`, the string value contributes.
For `model`, all non-null `ModelQuery` fields contribute.

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
    cap.modelRef = stringValue(ann, "model");
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

The recorder maps annotation values to the dual-field model:

```java
if (notEmpty(cap.modelRef)) {
    // String shorthand → modelRef (same path as YAML string form)
    cb.modelRef(cap.modelRef);
} else if (notEmpty(cap.modelTier)) {
    // Structured form → ModelQuery
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
`model_ref` and `model` columns:

**No existing installations** — per project schema convention, this goes directly into
`V1__initial_schema.sql` as a modification. The `model_tier` and `model_capabilities`
columns are replaced with:

```sql
model_ref VARCHAR(200),
model JSONB,
CONSTRAINT chk_model_exclusion CHECK (
    model_ref IS NULL OR model IS NULL
),
```

### Entity Mapping

```java
@Column(name = "model_ref", length = 200)
private String modelRef;

@Column(name = "model", columnDefinition = "JSONB")
@Convert(converter = ModelQueryConverter.class)
private ModelQuery model;
```

`ModelQueryConverter` implements `AttributeConverter<ModelQuery, String>` using Jackson
for JSON serialization. `ModelQuery` is already a Jackson-friendly record. The DB-level
check constraint enforces mutual exclusion.

## 8. Module Impact Summary

| Module | Change |
|--------|--------|
| `casehub-eidos-api` | Add `platform-api` dep. Replace `modelTier`/`modelCapabilities` with `String modelRef` + `ModelQuery model` on `AgentCapability`. Update `Builder`. Update `CapabilityVocabularyValidator`. |
| `casehub-eidos` (runtime) | `AgentDescriptorDeserializer` — union type parsing. `EidosRenderPipeline` — A2A_CARD rendering. `AgentCapabilityEntity` + `AgentDescriptorMapper` — JPA mapping. `V1__initial_schema.sql` — column change. `ModelQueryConverter` — new JPA converter. |
| `casehub-eidos-annotations` (deployment) | `@AgentCapabilityDef` — add `model` attribute. `EidosAnnotationsProcessor` — mutual exclusion validation. `EidosAnnotationsRecorder` — map to modelRef/model. `AnnotatedAgentConfig` — add `modelRef` field. |
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
