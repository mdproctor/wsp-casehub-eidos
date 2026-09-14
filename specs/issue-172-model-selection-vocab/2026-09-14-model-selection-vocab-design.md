# Design: LLM Model Selection via Eidos Vocabulary — Identity-Scoped Routing

**Issue:** casehubio/eidos#172
**Date:** 2026-09-14
**Status:** Draft

## Overview

Integrates the platform `ModelRegistry` with eidos vocabulary and identity systems
for capability-based model selection. Eidos declares model requirements on agent
descriptors; the platform resolves them against available models at dispatch time.

**Boundary principle:** eidos is the requirements declaration layer. Platform
(`RoutingAgentProvider`) is the selection authority. No `ModelSelector` SPI in eidos,
no dependency on `platform-api`'s `ModelRegistry`. Dependency flows one way: platform
reads eidos descriptors.

## 1. New Fields on AgentCapability

Two new optional fields on `AgentCapability`:

```java
public record AgentCapability(
    String name,
    String description,
    String capabilityVocabulary,
    Double qualityHint,
    Long latencyHintP50Ms,
    String costHint,
    String modelTier,              // NEW — vocabulary-grounded tier requirement
    Set<String> modelCapabilities,  // NEW — required model capabilities (open strings)
    List<String> inputTypes,
    List<String> outputTypes,
    List<String> tags,
    Map<String, Double> epistemicDomains,
    Set<String> excludedDomains
) { ... }
```

- **`modelTier`** — optional, vocabulary-grounded string (e.g., `"flagship"`,
  `"standard"`, `"fast"`, `"embedding"`). Validated against `ModelTierTerm` vocabulary
  at registration time via the same pattern as `capabilityVocabulary`. When null, the
  agent doesn't constrain model tier — the platform router falls back to the tenant's
  default.

- **`modelCapabilities`** — optional `Set<String>`, open strings matching platform's
  `ModelDescriptor.capabilities()` (e.g., `"text"`, `"vision"`, `"tool-use"`). No
  vocabulary grounding — model capabilities are vendor feature flags that evolve
  quarterly. Pass-through filter criteria for the platform router.

**Why per-capability, not per-descriptor:** `AgentCapability` is already the unit of
operational metadata (`qualityHint`, `latencyHintP50Ms`, `costHint`, `epistemicDomains`).
Different capabilities have different computational demands — code review needs FLAGSHIP,
lint checking is fine with FAST. CBR feedback (blocks#270) scores by
`(agentId, modelId, capabilityName)` — per-capability requirements give it a natural
grouping key.

**Why no cost ceiling or locality fields:** cost budgets and model locality (CLOUD/LOCAL)
are tenant/deployment policy, not agent identity. The platform router applies
`maxCostTier` from tenant preferences and `locality` from deployment policy at
resolution time. `costHint` on `AgentCapability` already signals expense for discovery.
Single authority per dimension — no competing constraints.

### Compact Constructor Validation

```java
AgentDescriptorValidator.validateOptional("modelTier", modelTier,
    AgentDescriptorValidator.MAX_CAPABILITY_STRING);
if (modelCapabilities != null) {
    AgentDescriptorValidator.validateItems("modelCapabilities",
        modelCapabilities, AgentDescriptorValidator.MAX_CAPABILITY_STRING);
    modelCapabilities = Set.copyOf(modelCapabilities);
}
```

### Builder Additions

```java
public Builder modelTier(String v)              { this.modelTier = v; return this; }
public Builder modelCapabilities(Set<String> v) { this.modelCapabilities = v; return this; }
```

## 2. ModelTierTerm Vocabulary

New enum in `casehub-eidos-vocab`:

```java
@VocabularyMetadata(
    uri = "urn:casehub:vocab:model-tier",
    name = "Model Tier",
    version = "1.0"
)
public enum ModelTierTerm implements VocabularyTerm {
    FLAGSHIP,
    STANDARD,
    FAST,
    EMBEDDING;

    @Override
    public List<String> exactMatch() { return List.of(); }

    @Override
    public List<String> axisExactMatch() { return List.of(); }

    @Override
    public VocabularyTerm specializes() {
        return switch (this) {
            case FLAGSHIP  -> STANDARD;
            case STANDARD  -> FAST;
            case FAST      -> null;
            case EMBEDDING -> null;
        };
    }
}
```

**Hierarchy:** `FLAGSHIP → STANDARD → FAST` (linear subsumption chain). `EMBEDDING`
is an island — different modality, not a tier level.

**Matching semantics:**
- `match("flagship", "standard")` → `MatchDegree.Plugin(1)` — FLAGSHIP satisfies
  STANDARD (superset capability)
- `match("fast", "standard")` → `MatchDegree.Specialization(1)` — FAST is less
  capable than STANDARD (downgrade signal)
- `match("embedding", "standard")` → `MatchDegree.None` — different modality

The platform router uses these match degrees to implement fallback: when the preferred
tier is unavailable or over budget, it can accept a Plugin match (upgrade) freely, and
decide whether to accept a Specialization match (downgrade) based on cost constraints.

**VocabularyRegistrar:** standard `@ApplicationScoped` CDI bean in `casehub-eidos-vocab`,
same pattern as `BelbinRegistrar`, `DiscRegistrar`, etc. No `axisExactMatch` or
cross-vocabulary mappings — model tiers are a standalone concept.

## 3. Registration-Time Validation

Follows the existing `capabilityVocabulary` validation pattern:

- **`DescriptorCollector`** validates `modelTier` when present: if a
  `VocabularyRegistry` is available, resolve the term against
  `urn:casehub:vocab:model-tier`. If the vocabulary is registered but the term doesn't
  exist → `AgentValidationException`. If the vocabulary isn't registered
  (`casehub-eidos-vocab` not on classpath) → skip validation, the field is just a string.

- **`modelCapabilities`** — structural validation only: non-blank strings, max length
  per item (reuses `MAX_CAPABILITY_STRING`). No vocabulary lookup.

- **Annotation build-time validation** in `EidosAnnotationsProcessor`: when
  `casehub-eidos-vocab` is on the build classpath, validate `modelTier` values at
  compile time (same hybrid validation pattern as `capabilityVocabulary`).

## 4. Rendering

Follows the renderer protocol (`capability-metadata-rendering.md`): routing signals
are A2A_CARD only.

**A2A_CARD:** `modelTier` and `modelCapabilities` render in the capability's JSON
object alongside existing routing signals:

```json
{
  "name": "code-review",
  "description": "Reviews code for quality and correctness",
  "modelTier": "flagship",
  "modelCapabilities": ["text", "tool-use"],
  "qualityHint": 0.95,
  "latencyHintP50Ms": 8000,
  "costHint": "high"
}
```

**MARKDOWN / PROSE:** `modelTier` and `modelCapabilities` are suppressed — routing
signals for machine consumption, not LLM prompt content. Same treatment as
`qualityHint` and `latencyHintP50Ms`.

**A2A structural assembly hash:** `modelTier` and `modelCapabilities` are included in
the A2A_CARD hash payload per the `a2a-structural-assembly-hash-coverage` protocol.

## 5. Annotations and YAML

### Annotations

Extend `@AgentCapabilityDef`:

```java
@Repeatable(AgentCapabilityDefs.class)
public @interface AgentCapabilityDef {
    // ... existing fields ...
    String modelTier() default "";
    String[] modelCapabilities() default {};
}
```

`EidosAnnotationsProcessor` extracts the values and passes them through
`AnnotatedAgentConfig` to the recorder. Hybrid vocabulary validation at build time:
when `casehub-eidos-vocab` is on the classpath, validate `modelTier` against
`ModelTierTerm` values.

### YAML

The existing `AgentDescriptorDeserializer` picks up new fields naturally — it builds
capabilities via `AgentCapability.Builder`, which gets the two new builder methods:

```yaml
agents:
  - agentId: code-reviewer
    name: Code Reviewer
    tenancyId: acme
    slot: reviewer
    provider: anthropic
    modelFamily: claude
    capabilities:
      - name: code-review
        modelTier: flagship
        modelCapabilities: [text, tool-use]
        qualityHint: 0.95
      - name: lint-check
        modelTier: fast
        modelCapabilities: [text]
```

No new deserializer needed.

## 6. Module Impact

| Module | Change |
|--------|--------|
| `casehub-eidos-api` | `modelTier`, `modelCapabilities` on `AgentCapability` record + Builder + validation |
| `casehub-eidos-vocab` | `ModelTierTerm` enum + `ModelTierRegistrar` CDI bean |
| `casehub-eidos` (runtime) | `DescriptorCollector` vocab validation. `EidosSystemPromptRenderer` A2A_CARD rendering + hash. |
| `casehub-eidos-annotations` (deployment) | `@AgentCapabilityDef` extension + `EidosAnnotationsProcessor` extraction + hybrid validation |
| `casehub-eidos-memory` | No change |
| `casehub-eidos-routing` | No change |
| `casehub-eidos-eval` | Update eval profiles with `modelTier` |
| JPA (runtime) | `model_tier` and `model_capabilities` columns on capability table |

### What Eidos Does NOT Do

- No `ModelSelector` SPI — selection lives in platform's `RoutingAgentProvider`
- No dependency on `platform-api`'s `ModelRegistry` — no dependency cycle
- No cost ceiling or locality fields — tenant policy, not agent identity
- No vocabulary for model capabilities — vendor feature flags, not stable enough

### Integration Flow (Platform Responsibility)

```
AgentDescriptor declares:
  capability[code-review].modelTier = "flagship"
  capability[code-review].modelCapabilities = ["text", "tool-use"]

AgentSelector picks the agent (capability + health + trust)

Engine invokes via RoutingAgentProvider:
  → reads descriptor's model requirements
  → queries ModelRegistry: ModelQuery.builder()
      .tier(ModelTier.FLAGSHIP)
      .requiredCapabilities(Set.of("text", "tool-use"))
      .maxCostTier(tenantBudget)
      .locality(tenantPolicy)
      .build()
  → applies CBR signals from RoutingSignalAssembler
  → resolves to concrete model ID
  → dispatches
```

## References

- `api/src/main/java/io/casehub/eidos/api/AgentCapability.java` — existing per-capability operational metadata
- `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — existing modelFamily/modelVersion identity fields
- `api/src/main/java/io/casehub/eidos/api/AgentSelector.java` — selection SPI
- `runtime/src/main/java/io/casehub/eidos/runtime/selector/SimpleAgentSelector.java` — default selector implementation
- `platform-api/.../ModelRegistry.java` — upstream SPI (platform-api)
- `platform-api/.../ModelDescriptor.java` — upstream model record (tier, capabilities, costTier, locality)
- `platform-api/.../ModelTier.java` — platform enum: FLAGSHIP, STANDARD, FAST, EMBEDDING
- `platform-api/.../CostTier.java` — platform enum: FREE → LOW → MEDIUM → HIGH → PREMIUM
- `docs/protocols/renderer/capability-metadata-rendering.md` — routing signals render in A2A_CARD only
- `docs/protocols/renderer/a2a-structural-assembly-hash-coverage.md` — hash coverage protocol
- casehubio/platform#285 — LLM model registry epic (upstream)
- casehubio/blocks#270 — CBR model routing integration
