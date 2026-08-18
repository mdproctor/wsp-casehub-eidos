# casehub-eidos-annotations — Design Spec

**Date:** 2026-08-18
**Issue:** casehubio/eidos#139
**Epic:** casehubio/blocks#115 (annotation-driven agent programming model)
**Status:** Draft

## Motivation

CaseHub's eidos module provides the richest agent identity system in the ecosystem (`AgentDescriptor`, `AgentDisposition`, `AgentGoal`, `AgentConstraint`, `AgentCapability`). But defining an agent requires builder chains or YAML — 20+ lines for a simple agent. The annotation model makes identity declarative:

```java
@Identity(slot = "legal-analyst", briefing = "Senior legal analyst")
@Disposition(socialOrient = "collaborative", ruleFollowing = "strict")
public interface LegalAnalyst {
    // agent methods...
}
```

This is part of the platform-wide annotation-driven programming model (blocks#115). Each repo gets an optional `*-annotations` module. Eidos owns identity annotations; blocks owns governance composition.

## Architecture

### Four build extensions, not three

The blocks#115 spec shows three build extensions (LC4j, Engine, Blocks). This design adds a fourth — the eidos build extension — to avoid a layering problem: apps that want `@Identity` should not need `casehub-blocks` on the classpath.

```
LC4j Extension     Engine Ext     Eidos Ext        Blocks Ext
(existing)         (new)          (this spec)      (new)

@*Agent            @Case          @Identity        @DebateAgent
@Tool              @Worker        @Disposition     @VotingAgent
@SystemMessage     @Bind          @AgentGoals      @OversightGate
                   @Goal          @AgentConstraints @TrustRouted
                   @Milestone     @Discoverable    + cross-cutting
```

Each repo processes its own annotations. Blocks adds cross-cutting governance composition separately (e.g., `@OversightGate` on an `@Identity`-annotated class).

### Module structure

New Quarkus extension — opt-in, not forced on all eidos consumers:

```
casehub-eidos-annotations/           ← NEW runtime module
  src/main/java/io/casehub/eidos/annotations/
    Identity.java
    Disposition.java
    AgentGoals.java
    AgentGoalDef.java
    AgentConstraints.java
    AgentConstraintDef.java
    Discoverable.java

casehub-eidos-annotations-deployment/ ← NEW deployment module
  src/main/java/io/casehub/eidos/annotations/deployment/
    EidosAnnotationsProcessor.java
```

**Dependencies:**
- `casehub-eidos-annotations` depends on `casehub-eidos-api` (for `GoalPriority`, `Visibility`, `ConstraintSeverity` enums)
- `casehub-eidos-annotations-deployment` depends on `casehub-eidos-annotations` (runtime artifact) + `casehub-eidos-deployment` (build step ordering) + `quarkus-core-deployment` + `quarkus-arc-deployment`
- No langchain4j-agentic dependency — eidos annotations reference only eidos-api types

**Package:** `io.casehub.eidos.annotations` — separate from `io.casehub.eidos.api` to avoid split packages across JARs.

## Annotations

### @Identity

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface Identity {
    String id() default "";                  // agentId — default: kebab-cased simple class name
    String name() default "";                // display name — default: spaced simple class name
    String slot();                           // required — role in the system
    String provider() default "";
    String modelFamily() default "";
    String jurisdiction() default "";
    String dataHandlingPolicy() default "";
    String briefing() default "";
    String vocabulary() default "";          // domain vocabulary URI
    String slotVocabulary() default "";
    String dispositionVocabulary() default "";
    String styleVocabulary() default "";
    String version() default "";
}
```

**Field mapping to AgentDescriptor:**

| @Identity field | AgentDescriptor field | Derivation |
|---|---|---|
| `id()` | `agentId` | If empty: kebab-cased simple class name (e.g. `LegalAnalystAgent` → `legal-analyst-agent`) |
| `name()` | `name` | If empty: spaced simple class name (e.g. `LegalAnalystAgent` → `Legal Analyst Agent`) |
| `slot()` | `slot` | Direct — required |
| `vocabulary()` | `domainVocabulary` | Direct |
| `slotVocabulary()` | `slotVocabulary` | Direct |
| `dispositionVocabulary()` | `dispositionVocabulary` | Direct |
| `styleVocabulary()` | `styleVocabulary` | Direct |
| All others | Same-named field | Direct mapping |

**tenancyId:** Sourced from MicroProfile config `casehub.eidos.annotations.default-tenancy-id`. Not an annotation field — tenancyId is operational (varies per deployment), not definitional (varies per agent type).

**agentId derivation rules:**
- Simple class name, not fully qualified — `LegalAnalyst` not `com.app.agents.LegalAnalyst`
- Kebab-case conversion: `LegalAnalystAgent` → `legal-analyst-agent`
- Collision detection: if two classes derive the same agentId, the build extension emits a compile-time error: `"Duplicate derived agentId 'reviewer' from classes com.app.agents.Reviewer and com.app.support.Reviewer — add explicit id() to at least one @Identity"`

**Fields NOT on @Identity:** `modelVersion`, `weightsFingerprint`, `axisVocabularies`, `templates` — these are power fields best served by the builder or `AgentDescriptorRegistrar` SPI. The annotation surface covers the 80% case.

### @Disposition

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface Disposition {
    String socialOrient() default "";
    String ruleFollowing() default "";
    String riskAppetite() default "";
    String autonomy() default "";
    String conflictMode() default "";
    boolean delegation() default false;
    String[] dispositionProfile() default {};
    String[] styleProfile() default {};
}
```

**Type mapping:** Each `String` axis value maps to `DispositionValue.of(term)` with weight 1.0. This matches the `AgentDisposition.Builder` simple API (e.g., `builder.socialOrient("collaborative")`).

**`dispositionProfile` / `styleProfile`:** Each string becomes `DispositionValue.of(term)` with weight 1.0. `DescriptorCollector.deriveDispositionAxes()` normalizes weights across all profile terms.

**Weight limitation:** Annotations create equal-weight disposition values. For weighted profiles (e.g., `new DispositionValue("collaborative", 0.8)` vs `new DispositionValue("analytical", 0.3)`), use `AgentDisposition.builder()` directly via the `AgentDescriptorRegistrar` SPI. This is an intentional progressive-disclosure boundary — annotations serve the common case; the builder API serves nuanced personality profiles.

**Field naming:** Annotation fields match the `AgentDisposition` record fields exactly (`socialOrient`, not `socialOrientation`) to maintain consistency with the eidos API.

### @AgentGoals / @AgentGoalDef

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface AgentGoals {
    AgentGoalDef[] value();
}

@Retention(RUNTIME)
@Target({})
public @interface AgentGoalDef {
    String name();
    String description() default "";
    GoalPriority priority() default GoalPriority.PRIMARY;
    Visibility visibility() default Visibility.PUBLIC;
    String[] capabilities() default {};
}
```

**Note:** `GoalPriority` values are `PRIMARY` and `SECONDARY` (the eidos API). The blocks#115 spec uses `HIGH`/`CRITICAL`/`MEDIUM` which do not exist — annotations use the actual API enums.

`capabilities()` references declared `AgentCapability.name()` values. When used with `@Discoverable`, cross-validation ensures referenced capability names exist. When used standalone (no `@Discoverable`), capability references are not validated — they may refer to capabilities registered via other paths (builder, YAML).

### @AgentConstraints / @AgentConstraintDef

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface AgentConstraints {
    AgentConstraintDef[] value();
}

@Retention(RUNTIME)
@Target({})
public @interface AgentConstraintDef {
    String name();
    String description() default "";
    ConstraintSeverity severity() default ConstraintSeverity.HARD;
    Visibility visibility() default Visibility.PUBLIC;
}
```

**Note:** `ConstraintSeverity` values are `HARD` and `SOFT` (the eidos API). The blocks#115 spec uses `MUST` which does not exist.

### @Discoverable

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface Discoverable {
    String[] capabilities();
    String[] tags() default {};
}
```

**Capability mapping:** Each string in `capabilities()` creates an `AgentCapability` with `name` set and all other fields empty/default. This is name-only — `description`, `qualityHint`, `latencyHintP50Ms`, `costHint`, `epistemicDomains`, `excludedDomains`, `capabilityVocabulary`, `inputTypes`, `outputTypes` are not set.

**Capability metadata limitation:** Annotated agents with `@Discoverable` get name-only capabilities. This means:
- `CapabilityHealth` probe steps for epistemic weakness and declared exclusions are no-ops (no data to check)
- A2A_CARD rendering produces capabilities without descriptions
- `BehavioralExpectations.latencyBound()` returns empty (no latency compliance observation)

For agents that need rich capability metadata, implement `AgentDescriptorRegistrar` or use the builder API. This is an intentional progressive-disclosure boundary. A future `@Capability` repeatable annotation could close this gap without changing the current design.

`tags()` provides free-form discovery metadata. Tags do not map to any `AgentDescriptor` field today — they are stored as build-time metadata for future registry query extensions.

## Build Extension

### EidosAnnotationsProcessor

Located in `casehub-eidos-annotations-deployment`. Responsibilities:

1. **Scan** for `@Identity`-annotated classes at build time via Jandex
2. **Extract** annotation values from `@Identity`, `@Disposition`, `@AgentGoals`, `@AgentConstraints`, `@Discoverable` on the same class
3. **Validate** at build time:
   - Required fields present (`@Identity.slot()` is non-empty)
   - Enum values valid (compile-time — Java enforces this)
   - No duplicate derived agentIds across `@Identity`-annotated classes
   - Goal capability references valid when `@Discoverable` is present on the same class
   - Disposition term existence against Jandex-indexed `@VocabularyMetadata` enums (hybrid — only when vocabulary modules are on the classpath)
4. **Generate** synthetic CDI beans implementing `AgentDescriptorRegistrar`
5. **Register** the Quarkus extension feature name `eidos-annotations`

### Bean generation

For each `@Identity`-annotated class, the build extension generates a synthetic CDI bean implementing `AgentDescriptorRegistrar`:

```java
// Generated at build time (conceptual — actual implementation uses Quarkus synthetic bean API)
@ApplicationScoped
public class LegalAnalystAgent_EidosRegistrar implements AgentDescriptorRegistrar {
    @Override
    public List<AgentDescriptor> descriptors() {
        return List.of(
            AgentDescriptor.builder()
                .agentId("legal-analyst-agent")
                .name("Legal Analyst Agent")
                .slot("legal-analyst")
                .briefing("Senior legal analyst")
                .disposition(AgentDisposition.builder()
                    .socialOrient("collaborative")
                    .ruleFollowing("strict")
                    .build())
                .tenancyId(/* injected from config */)
                .build()
        );
    }
}
```

The generated registrar is picked up by the existing `AgentDescriptorBootstrap` → `DescriptorCollector.collectAndValidate()` pipeline. This ensures:
- Disposition axis derivation via `deriveDispositionAxes()` works as normal
- Duplicate (agentId, tenancyId) detection works across all registration paths (annotation, builder, YAML)
- Template validation works if templates are set via other means
- Briefing coherence validation runs

### Hybrid vocabulary validation

When `@VocabularyMetadata`-annotated enum classes are Jandex-indexed on the build classpath (e.g., `casehub-eidos-vocab` is a dependency):

1. Build extension scans for all `@VocabularyMetadata` annotations in the Jandex index
2. For each vocabulary, extracts enum constant names (the valid terms)
3. For each `@Disposition` annotation, checks axis values and `dispositionProfile`/`styleProfile` terms against the vocabulary identified by `@Identity.vocabulary()` / `@Identity.dispositionVocabulary()`
4. Invalid terms produce a compile-time error: `"@Disposition.socialOrient value 'collaboartive' is not a valid term in vocabulary 'urn:casehub:vocab:svo'. Did you mean 'collaborative'?"`

When no vocabulary modules are on the classpath, validation is skipped. The runtime `DescriptorCollector.deriveDispositionAxes()` silently produces no axis derivation for unrecognized terms — this is degradation, not validation. Build-time is the only path that catches term typos.

### Three registration paths

Eidos now supports three registration paths for `AgentDescriptor`:

| Path | Module | Mechanism |
|---|---|---|
| Builder / SPI | `casehub-eidos` (core) | Implement `AgentDescriptorRegistrar`, return built `AgentDescriptor` instances |
| YAML | `casehub-eidos` (core) | `META-INF/eidos/descriptors.yaml` via `ClasspathYamlDescriptorRegistrar` |
| Annotations | `casehub-eidos-annotations` (opt-in) | `@Identity` + companions → generated `AgentDescriptorRegistrar` |

All three flow through `DescriptorCollector.collectAndValidate()`. An agent defined via multiple paths with the same `(agentId, tenancyId)` pair fails at startup with a duplicate detection error.

## Examples

### agent-identity-annotated

Demonstrates `@Identity` + `@Disposition` — the minimal annotated agent:

```java
@Identity(slot = "document-analyst",
          briefing = "Analyses documents and extracts key findings",
          vocabulary = "urn:casehub:vocab:svo")
@Disposition(socialOrient = "collaborative",
             ruleFollowing = "moderate",
             riskAppetite = "cautious")
public interface DocumentAnalyst {
    // Agent methods defined here or via LC4j @Agent
}
```

### goals-constraints-annotated

Demonstrates goals + constraints with capability references:

```java
@Identity(slot = "legal-analyst",
          jurisdiction = "EU",
          dataHandlingPolicy = "gdpr-compliant",
          briefing = "Senior legal analyst specialising in regulatory compliance")
@Disposition(socialOrient = "collaborative",
             ruleFollowing = "strict",
             riskAppetite = "cautious",
             autonomy = "guided",
             conflictMode = "accommodating")
@Discoverable(capabilities = {"document-analysis", "clause-extraction", "risk-assessment"})
@AgentGoals({
    @AgentGoalDef(name = "accurate-analysis",
                  description = "Produce accurate legal analysis",
                  priority = GoalPriority.PRIMARY,
                  capabilities = {"document-analysis"}),
    @AgentGoalDef(name = "regulatory-compliance",
                  description = "Ensure all outputs meet regulatory requirements",
                  priority = GoalPriority.SECONDARY)
})
@AgentConstraints({
    @AgentConstraintDef(name = "no-legal-advice",
                        description = "Must not provide binding legal advice",
                        severity = ConstraintSeverity.HARD),
    @AgentConstraintDef(name = "source-citation",
                        description = "Should cite regulatory sources when possible",
                        severity = ConstraintSeverity.SOFT)
})
public interface LegalAnalystAgent {
    // Agent methods
}
```

## Testing Strategy

### Unit tests (annotations module)

- Annotation retention and target validation (reflection tests)
- Class-name to agentId derivation: standard cases, edge cases (single word, acronyms, inner classes)
- Class-name to display name derivation

### Build extension tests (deployment module)

- `@QuarkusTest` with `@RegisterExtension`: verify generated `AgentDescriptorRegistrar` beans produce correct `AgentDescriptor` instances
- Generated descriptor equivalence: `assertThat(annotatedDescriptor).isEqualTo(builderEquivalent)` for each annotation combination
- Build-time validation: `@Identity` without `slot` → compile error
- Duplicate agentId detection: two classes with same simple name → compile error with helpful message
- Goal capability cross-validation with `@Discoverable`
- Hybrid vocabulary validation: disposition term check when vocab on classpath
- Vocabulary not on classpath: no build error, graceful skip
- Multiple registration paths: annotation + YAML with same agentId → startup duplicate error

### Parity tests

- For each `@Identity` field, assert `AgentDescriptor.Builder` has a matching setter
- For each `@Disposition` field, assert `AgentDisposition.Builder` has a matching setter
- For each `@AgentGoalDef` field, assert `AgentGoal` record has a matching component
- For each `@AgentConstraintDef` field, assert `AgentConstraint` record has a matching component

## Known Limitations

1. **Weight-less disposition profiles:** `String[]` annotations create `DispositionValue` with weight 1.0. Weighted profiles require the builder API.
2. **Name-only capabilities:** `@Discoverable.capabilities()` creates `AgentCapability` with name only. Rich metadata (description, epistemic domains, excluded domains, latency/quality/cost hints) requires the builder API.
3. **No templates via annotations:** `AgentDescriptor.templates` is not settable via annotations. Use `AgentDescriptorRegistrar` SPI.
4. **No per-agent tenancyId:** All annotation-defined agents share the configured default tenancy. Multi-tenant annotations require the builder API.
5. **No axisVocabularies override:** Per-axis vocabulary overrides require the builder API.
6. **Silent runtime degradation for unrecognized terms:** When vocabulary modules are not on the build classpath, disposition term typos are not caught. At runtime, `DescriptorCollector.deriveDispositionAxes()` silently produces no axis derivation for unrecognized terms — it does not error.

## References

- [blocks#115 design spec](/Users/mdproctor/claude/casehub/blocks/docs/specs/annotation-driven-model/2026-08-14-annotation-driven-agent-model-design.md) — parent epic architecture, Design Principles, Build Extension Architecture
- [AgentDescriptor.java](api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java) — record definition, Builder, validation
- [AgentDisposition.java](api/src/main/java/io/casehub/eidos/api/AgentDisposition.java) — 5 axes + dispositionProfile + styleProfile
- [AgentGoal.java](api/src/main/java/io/casehub/eidos/api/AgentGoal.java) — GoalPriority (PRIMARY/SECONDARY), Visibility, capabilities
- [AgentConstraint.java](api/src/main/java/io/casehub/eidos/api/AgentConstraint.java) — ConstraintSeverity (HARD/SOFT), Visibility
- [DescriptorCollector.java](runtime/src/main/java/io/casehub/eidos/runtime/registrar/DescriptorCollector.java) — collectAndValidate(), deriveDispositionAxes()
- [AgentDescriptorBootstrap.java](runtime/src/main/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrap.java) — startup registration pipeline
- [EidosProcessor.java](deployment/src/main/java/io/casehub/eidos/deployment/EidosProcessor.java) — existing build extension (feature + native resources only)
- [Decision review R1](../../reviews/casehub-eidos/issue-139-decision-20260818-040301/responses/reviewer-1.md) — 21 findings, weight-loss and capability-metadata limitations documented
