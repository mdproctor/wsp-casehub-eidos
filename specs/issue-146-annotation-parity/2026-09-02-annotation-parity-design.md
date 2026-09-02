# casehub-eidos-annotations — Parity Design Spec

**Date:** 2026-09-02
**Issue:** casehubio/eidos#146
**Predecessor:** casehubio/eidos#139 (initial annotations design)
**Status:** Draft

## Motivation

Issue #139 delivered the annotation surface for eidos with intentional scope boundaries: name-only capabilities (`@Discoverable`), equal-weight disposition profiles (`String[]`), no templates, no axis vocabularies, no convenience personality types. These were documented as Known Limitations §1-6.

Issue #146 closes every gap from issue #139's Known Limitations §1-6. The goal: any `AgentDescriptor` achievable through the YAML surface is also achievable via annotations. This matters for personality generation workflows — a generator targeting the annotation surface should produce the same descriptor as one targeting YAML. Residual Builder-only capabilities (multi-valued disposition axes) are documented in §Residual Limitations.

## Scope

### In scope

| Gap | Solution | Decision |
|-----|----------|----------|
| Rich capability metadata (11 fields) | `@AgentCapabilityDef` with `@Repeatable` | D2 |
| `epistemicDomains` (Map<String, Double>) | Nested `@EpistemicDomain(value, score)` | D3 |
| Weighted `dispositionProfile` / `styleProfile` | `@DispositionWeight` with `weight() default 1.0` | D4 |
| `mbtiType` / `enneagramType` convenience | Fields on `@Disposition` + shared `PersonalityTypeDeriver` | D5, D10 |
| Templates (`List<TemplateRef>`) | `@AgentTemplateRef` with `@Repeatable` + nested `@TemplateArg` | D6 |
| `axisVocabularies` (Map<DispositionAxis, String>) | Nested `@AxisVocabulary(axis, uri)[]` on `@Disposition` | D7 |
| `weightsFingerprint`, `modelVersion` | String fields on `@Identity` | D9 |
| Build-time validation for new fields | Extended `EidosAnnotationsProcessor` | D8 |

### Out of scope

- `@Discoverable` changes — stays name-only in `casehub-eidos-api`
- `@AgentGoals`/`@AgentConstraints` — already at full parity
- Retrofit `@Repeatable` onto `@AgentGoalDef`/`@AgentConstraintDef` — separate concern (casehubio/eidos#152)
- Runtime `@Inject VocabularyRegistry` in recorder — infrastructure only, not a feature

## Architecture

### New annotations (in `casehub-eidos-annotations` runtime module)

#### @AgentCapabilityDef (Repeatable)

```java
@Retention(RUNTIME)
@Target(TYPE)
@Repeatable(AgentCapabilities.class)
public @interface AgentCapabilityDef {
    String name();
    String description() default "";
    String capabilityVocabulary() default "";
    double qualityHint() default -1;      // -1 = not set
    long latencyHintP50Ms() default -1;    // -1 = not set
    String costHint() default "";
    String[] inputTypes() default {};
    String[] outputTypes() default {};
    String[] tags() default {};
    EpistemicDomain[] epistemicDomains() default {};
    String[] excludedDomains() default {};
}
```

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface AgentCapabilities {
    AgentCapabilityDef[] value();
}
```

```java
@Retention(RUNTIME)
@Target({})
public @interface EpistemicDomain {
    String value();      // domain name
    double score();      // 0.0-1.0
}
```

**Merge semantics with `@Discoverable`:** When both `@Discoverable` and `@AgentCapabilityDef` are present, capabilities from both form a union. Name-only capabilities from `@Discoverable` and rich capabilities from `@AgentCapabilityDef` coexist. If the same capability name appears in both → build-time error.

#### @DispositionWeight

```java
@Retention(RUNTIME)
@Target({})
public @interface DispositionWeight {
    String value();               // term name
    double weight() default 1.0;  // 0.0-1.0
}
```

**Breaking change to `@Disposition`:** `dispositionProfile` and `styleProfile` change from `String[]` to `DispositionWeight[]`:

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
    DispositionWeight[] dispositionProfile() default {};
    DispositionWeight[] styleProfile() default {};
    String mbtiType() default "";           // NEW (D10)
    String enneagramType() default "";      // NEW (D10)
    AxisVocabulary[] axisVocabularies() default {};  // NEW (D7)
}
```

Simple usage unchanged in spirit: `@DispositionWeight("collaborative")` (weight defaults to 1.0). Weighted: `@DispositionWeight(value = "analytical", weight = 0.3)`.

#### @AxisVocabulary

```java
@Retention(RUNTIME)
@Target({})
public @interface AxisVocabulary {
    DispositionAxis axis();
    String uri();
}
```

#### @AgentTemplateRef (Repeatable)

```java
@Retention(RUNTIME)
@Target(TYPE)
@Repeatable(AgentTemplates.class)
public @interface AgentTemplateRef {
    String id();
    TemplateArg[] args() default {};
}
```

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface AgentTemplates {
    AgentTemplateRef[] value();
}
```

```java
@Retention(RUNTIME)
@Target({})
public @interface TemplateArg {
    String key();
    String value();
}
```

### @Identity changes (D9)

```java
@Retention(RUNTIME)
@Target(TYPE)
public @interface Identity {
    // ... existing fields unchanged ...
    String weightsFingerprint() default "";  // NEW
    String modelVersion() default "";        // NEW
}
```

### AnnotatedAgentConfig changes

New fields to carry build-time-extracted values to the recorder:

```java
public class AnnotatedAgentConfig {
    // ... existing fields ...

    // D4: weighted disposition profiles (replaces String[] dispositionProfile/styleProfile)
    public DispositionWeightConfig[] dispositionProfile;
    public DispositionWeightConfig[] styleProfile;

    // D9: new Identity fields
    public String weightsFingerprint;
    public String modelVersion;

    // D10: personality convenience
    public String mbtiType;
    public String enneagramType;

    // D7: axis vocabularies
    public AxisVocabConfig[] axisVocabularies;

    // D6: templates
    public TemplateRefConfig[] templateRefs;

    // D2: rich capabilities (replaces String[] capabilities)
    public CapabilityConfig[] richCapabilities;

    public static class AxisVocabConfig {
        public String axis;  // DispositionAxis enum name
        public String uri;
    }

    public static class TemplateRefConfig {
        public String id;
        public TemplateArgConfig[] args;
    }

    public static class TemplateArgConfig {
        public String key;
        public String value;
    }

    public static class CapabilityConfig {
        public String name;
        public String description;
        public String capabilityVocabulary;
        public double qualityHint = -1;
        public long latencyHintP50Ms = -1;
        public String costHint;
        public String[] inputTypes;
        public String[] outputTypes;
        public String[] tags;
        public EpistemicDomainConfig[] epistemicDomains;
        public String[] excludedDomains;
    }

    public static class EpistemicDomainConfig {
        public String value;
        public double score;
    }

    public static class DispositionWeightConfig {
        public String value;
        public double weight;
    }
}
```

### EidosAnnotationsProcessor changes

#### Capability extraction (D2)

Replace `extractCapabilities()` with merged extraction:

1. Collect name-only capabilities from `@Discoverable.capabilities()` (if present)
2. Collect rich capabilities from `@AgentCapabilityDef` annotations — use `classInfo.annotationsWithRepeatable(AGENT_CAPABILITY_DEF, AGENT_CAPABILITIES)` to handle both single and container forms (Jandex requires the container `DotName` for `@Repeatable` annotations; `classInfo.annotation()` returns only the container when multiple are present)
3. Check for name collisions between the two sets → build-time error
4. Check for duplicate names within `@AgentCapabilityDef` annotations → build-time error
5. Store merged result in `config.richCapabilities` (name-only entries get a `CapabilityConfig` with only `name` set)

#### Disposition extraction (D4, D7, D10)

Update `extractDisposition()`:
- `dispositionProfile` / `styleProfile`: iterate `@DispositionWeight[]` nested annotation instances via `ann.value("dispositionProfile").asNestedArray()`, extract `value()` and `weight()` from each. Store as `DispositionWeightConfig[]` on config.
- `axisVocabularies`: iterate `@AxisVocabulary[]`, extract `axis()` enum name and `uri()`
- `mbtiType` / `enneagramType`: extract string values

Update `validateArrayTerms()` for the `@DispositionWeight[]` type change: after the type change, `v.asStringArray()` throws because values are now nested `AnnotationInstance` objects, not strings. Replace with iteration over `v.asNestedArray()`, extracting the `value()` field from each `@DispositionWeight` instance for vocabulary validation.

#### Template extraction (D6)

New `extractTemplates()` method:
- Collect `@AgentTemplateRef` annotations from the class (via Jandex, handles `@Repeatable`)
- For each ref: extract `id()` and nested `@TemplateArg[]` (key/value pairs)
- Store in `config.templateRefs`

#### Build-time validation (D8)

Extend validation in `processAnnotations()`:

1. `qualityHint` range: if set (not -1), guard `Double.isNaN()` first, then check 0.0-1.0. `NaN` is a valid annotation compile-time constant that defeats naïve range checks — `NaN < 0.0` and `NaN > 1.0` both return `false` per IEEE 754 (ARC42STORIES §8 Anti-pattern 4).
2. `epistemicDomains` scores: each `@EpistemicDomain.score()` must pass `!Double.isNaN(score)` and be 0.0-1.0
3. `excludedDomains ∩ epistemicDomains = ∅`: no domain in both sets
4. Goal-capability cross-validation: `@AgentGoalDef.capabilities()` validated against union of `@Discoverable` and `@AgentCapabilityDef` capability names. `validateGoalCapabilities()` must iterate `config.richCapabilities` (a `CapabilityConfig[]`) rather than the removed `config.capabilities` (a `String[]`), extracting `cap.name` for the name set.
5. `@DispositionWeight.weight()` range: `!Double.isNaN(weight)` and 0.0-1.0. Without this, invalid weights pass to `DispositionValue` compact constructor at runtime.
6. Duplicate `@AxisVocabulary.axis()` detection: two `@AxisVocabulary` entries with the same `DispositionAxis` → build-time error.
7. Duplicate `@AgentCapabilityDef.name()` detection: two `@AgentCapabilityDef` annotations on the same class with the same `name()` → build-time error. (Cross-surface collisions between `@Discoverable` and `@AgentCapabilityDef` are covered by D2 step 3.)

#### Orphan annotation warnings (D8b)

Extend `warnDiscoverableWithoutIdentity()` to also warn for `@AgentCapabilityDef` and `@AgentTemplateRef` on classes without `@Identity`. These are `@Target(TYPE)` annotations that are silently ignored when no `@Identity` is present — the processor only iterates `@Identity`-annotated classes for extraction. Without the warning, a user could annotate a class with rich capabilities or templates, forget `@Identity`, and get no feedback.

Scan the Jandex index for `AGENT_CAPABILITY_DEF`, `AGENT_CAPABILITIES`, `AGENT_TEMPLATE_REF`, and `AGENT_TEMPLATES` — any class carrying these annotations but not in `processedClasses` gets a warning:

```java
LOG.warnf("Class %s has @AgentCapabilityDef but no @Identity — capabilities will not be registered", className);
LOG.warnf("Class %s has @AgentTemplateRef but no @Identity — templates will not be registered", className);
```

### EidosAnnotationsRecorder changes (D5)

Switch from `Supplier<AgentDescriptorRegistrar>` to `BeanCreator` pattern:

```java
public Function<SyntheticCreationalContext<AgentDescriptorRegistrar>, AgentDescriptorRegistrar>
    createRegistrar(AnnotatedAgentConfig config) {
    return ctx -> {
        var vocabRegistry = ctx.getInjectedReference(VocabularyRegistry.class);
        // ... build descriptor with full capability, template, disposition support ...
        // ... PersonalityTypeDeriver.derive(config, vocabRegistry, builder) ...
        return () -> List.of(builder.build());
    };
}
```

The build step changes from `.supplier(recorder.createRegistrar(config))` to `.createWith(recorder.createRegistrar(config))` with `.addInjectionPoint(VocabularyRegistry.class)`.

**Dependency:** `casehub-eidos` runtime is a mandatory transitive dependency of `casehub-eidos-annotations`. The annotations module produces `AgentDescriptorRegistrar` beans consumed by `AgentDescriptorBootstrap` (runtime module) — without the runtime module, registered beans are inert. The `BeanCreator` pattern makes this coupling explicit via the `VocabularyRegistry` injection point: if `CdiVocabularyRegistry` (runtime module) is absent, Arc reports an unsatisfied dependency at startup. No fallback is needed — the annotations pipeline is non-functional without the runtime module.

#### Rich capability construction

For each `CapabilityConfig` in `config.richCapabilities`:

```java
var capBuilder = AgentCapability.builder().name(cap.name);
if (notEmpty(cap.description)) capBuilder.description(cap.description);
if (notEmpty(cap.capabilityVocabulary)) capBuilder.capabilityVocabulary(cap.capabilityVocabulary);
if (cap.qualityHint >= 0) capBuilder.qualityHint(cap.qualityHint);
if (cap.latencyHintP50Ms >= 0) capBuilder.latencyHintP50Ms(cap.latencyHintP50Ms);
if (notEmpty(cap.costHint)) capBuilder.costHint(cap.costHint);
if (cap.inputTypes != null && cap.inputTypes.length > 0)
    capBuilder.inputTypes(List.of(cap.inputTypes));
if (cap.outputTypes != null && cap.outputTypes.length > 0)
    capBuilder.outputTypes(List.of(cap.outputTypes));
if (cap.tags != null && cap.tags.length > 0)
    capBuilder.tags(List.of(cap.tags));
if (cap.epistemicDomains != null && cap.epistemicDomains.length > 0) {
    var map = new HashMap<String, Double>();
    for (var ed : cap.epistemicDomains) map.put(ed.value, ed.score);
    capBuilder.epistemicDomains(map);
}
if (cap.excludedDomains != null && cap.excludedDomains.length > 0)
    capBuilder.excludedDomains(Set.of(cap.excludedDomains));
```

#### Template construction

For each `TemplateRefConfig`:

```java
var args = new HashMap<String, String>();
if (ref.args != null) {
    for (var arg : ref.args) args.put(arg.key, arg.value);
}
templateRefs.add(new TemplateRef(ref.id, args));
```

**Template validation:** Templates from the annotation path flow through the standard `DescriptorCollector.collectAndValidate()` pipeline, which validates template IDs against `TemplateRegistry.resolve()` and checks for missing/extra args against `DescriptorTemplate.parameters()`. This is the same validation applied to Builder and YAML paths. If a template ID doesn't exist: `"Descriptor '<agentId>' references unknown template: <templateId>"`. If args don't match: `"Descriptor '<agentId>', template '<id>': missing args [...]"` or `"... unexpected args [...]"`.

#### Axis vocabularies construction

For `config.axisVocabularies` → `Map<DispositionAxis, String>` on `builder.axisVocabularies(map)`:

```java
if (config.axisVocabularies != null && config.axisVocabularies.length > 0) {
    var map = new EnumMap<DispositionAxis, String>(DispositionAxis.class);
    for (var av : config.axisVocabularies) {
        map.put(DispositionAxis.valueOf(av.axis), av.uri);
    }
    builder.axisVocabularies(map);
}
```

`AxisVocabConfig.axis` stores the `DispositionAxis` enum name as a string (extracted by the processor via `ann.value("axis").asEnum()`). `DispositionAxis.valueOf()` converts it back. The `valueOf()` call is safe — the processor already validates against the `DispositionAxis` enum at build time (the annotation field is typed as `DispositionAxis axis()`, so only valid enum constants compile).

#### Weighted disposition profile construction

The updated `dispositionProfile` and `styleProfile` extraction (replaces current `DispositionValue.of(t)` from plain strings):

```java
if (config.dispositionProfile != null && config.dispositionProfile.length > 0) {
    var values = new ArrayList<DispositionValue>();
    for (var dp : config.dispositionProfile) {
        if (notEmpty(dp.value)) values.add(new DispositionValue(dp.value, dp.weight));
    }
    if (!values.isEmpty()) db.dispositionProfile(values);
}
// Same pattern for config.styleProfile
```

#### PersonalityTypeDeriver integration

After building the disposition from explicit fields, call the deriver:

```java
var explicitAxes = new EnumMap<DispositionAxis, String>(DispositionAxis.class);
if (notEmpty(config.socialOrient)) explicitAxes.put(SOCIAL_ORIENTATION, config.socialOrient);
if (notEmpty(config.ruleFollowing)) explicitAxes.put(RULE_FOLLOWING, config.ruleFollowing);
if (notEmpty(config.riskAppetite)) explicitAxes.put(RISK_APPETITE, config.riskAppetite);
if (notEmpty(config.autonomy)) explicitAxes.put(AUTONOMY, config.autonomy);
if (notEmpty(config.conflictMode)) explicitAxes.put(CONFLICT_MODE, config.conflictMode);

PersonalityTypeDeriver.derive(
    new PersonalityInput(config.mbtiType, config.enneagramType,
        config.dispositionProfile != null && config.dispositionProfile.length > 0,
        explicitAxes),
    vocabRegistry, db);
```

### PersonalityTypeDeriver utility (D5)

New class in `io.casehub.eidos.api` (shared by `DispositionDeserializer` and recorder). Lives in the api module because all its dependencies are api types (`VocabularyRegistry`, `AgentDisposition.Builder`, `DispositionValue`) — follows the `CapabilityResolver` / `BehavioralExpectations` precedent for static utilities in api:

```java
public record PersonalityInput(
        String mbtiType,
        String enneagramType,
        boolean hasExplicitProfile,
        Map<DispositionAxis, String> explicitAxes) {}

public final class PersonalityTypeDeriver {
    public static void derive(
            PersonalityInput input,
            VocabularyRegistry vocabRegistry,
            AgentDisposition.Builder builder) {
        // mbtiType → dispositionProfile (only when profile not explicitly set)
        // enneagramType → per-axis values (only when explicit axis not in input.explicitAxes())
        // Same precedence rules as DispositionDeserializer:51-95
        // Uses input.explicitAxes().containsKey(axis) to determine precedence
    }
}
```

`PersonalityInput` lives alongside `PersonalityTypeDeriver` in `io.casehub.eidos.api`. The `Map<DispositionAxis, String>` captures only axes with non-empty explicit values — an axis absent from the map means "derive from personality type if available".

`DispositionDeserializer` refactored to call `PersonalityTypeDeriver.derive()` instead of inline logic.

## Migration

### Breaking change: @Disposition.dispositionProfile / styleProfile

**Before (issue #139):**
```java
@Disposition(dispositionProfile = {"collaborative", "analytical"})
```

**After (issue #146):**
```java
@Disposition(dispositionProfile = {
    @DispositionWeight("collaborative"),
    @DispositionWeight("analytical")
})
```

Mechanical migration. No semantic change when all weights are 1.0 (the default).

### Additive changes (no migration needed)

- New fields on `@Identity`: `weightsFingerprint`, `modelVersion`
- New fields on `@Disposition`: `mbtiType`, `enneagramType`, `axisVocabularies`
- New annotations: `@AgentCapabilityDef`, `@AgentTemplateRef`, `@TemplateArg`, `@EpistemicDomain`, `@DispositionWeight`, `@AxisVocabulary`

## Residual Limitations

1. **Multi-valued disposition axes:** The Builder API supports `socialOrient(DispositionValue... values)` for multi-valued weighted axis values (e.g., `socialOrient(new DispositionValue("collaborative", 0.7), new DispositionValue("directive", 0.3))`). The annotation surface supports only a single unweighted string per axis (`String socialOrient() default ""`). This matches the YAML surface, which also uses single axis strings. Multi-valued axes are typically derived from `dispositionProfile` via `DescriptorCollector.deriveDispositionAxes()`, which runs on all paths including annotations.

2. **Per-agent tenancyId:** Inherited from issue #139 Known Limitations §4 — all annotation-defined agents share the configured default tenancy.

## Testing Strategy

### Parity tests

For each `AgentDescriptor` field, assert the annotation surface can produce the same value as the Builder:
- Build a descriptor via Builder with all fields populated
- Build the equivalent via annotations on a test class
- `assertThat(annotatedDescriptor).isEqualTo(builderDescriptor)`

### Capability merge tests

- `@Discoverable` only → name-only capabilities (backward compat)
- `@AgentCapabilityDef` only → rich capabilities
- Both present, no collision → union
- Both present, name collision → build-time error

### Weighted profile tests

- `@DispositionWeight("term")` → weight 1.0
- `@DispositionWeight(value = "term", weight = 0.5)` → explicit weight
- Mixed weights in same profile
- `dispositionProfile` + `mbtiType` present → `dispositionProfile` wins (profile takes precedence)

### Personality type convenience tests

- `mbtiType = "INTJ"` alone → derives dispositionProfile via `MbtiTypeTerm.defaultProfile()`
- `enneagramType = "type-1"` alone → derives axis values via `equivalentValues()`
- `mbtiType` + explicit `dispositionProfile` → profile wins
- `enneagramType` + explicit axis value → explicit wins
- No vocab module on classpath → warning, no derivation (graceful fallback)

### Template tests

- Single `@AgentTemplateRef(id = "safety-primer")` → no args
- Multiple `@AgentTemplateRef` (repeatable) → ordered list
- `@TemplateArg(key, value)` → args map
- Template ref validation against `TemplateRegistry` at startup

### Axis vocabulary tests

- `@AxisVocabulary(axis = SOCIAL_ORIENTATION, uri = "urn:...")` → `axisVocabularies` map
- Multiple axes → map with multiple entries
- Duplicate axis → build-time error

### Build-time validation tests

- `qualityHint = 1.5` → build-time error
- `qualityHint = Double.NaN` → build-time error (ARC42STORIES §8 Anti-pattern 4)
- `@EpistemicDomain(value = "java", score = 2.0)` → build-time error
- `@EpistemicDomain(value = "java", score = Double.NaN)` → build-time error
- `excludedDomains = {"java"}` + `@EpistemicDomain(value = "java", ...)` → build-time error
- `@AgentGoalDef(capabilities = {"missing-cap"})` with no matching `@AgentCapabilityDef` or `@Discoverable` → build-time error
- `@DispositionWeight(value = "term", weight = 1.5)` → build-time error
- `@DispositionWeight(value = "term", weight = Double.NaN)` → build-time error
- Two `@AgentCapabilityDef` with same `name()` on one class → build-time error
- `@AgentCapabilityDef` without `@Identity` on class → build-time warning
- `@AgentTemplateRef` without `@Identity` on class → build-time warning

## Annotation Inventory

### New annotations (8)

| Annotation | Target | Repeatable | Module |
|-----------|--------|-----------|--------|
| `@AgentCapabilityDef` | TYPE | Yes (`@AgentCapabilities`) | eidos-annotations |
| `@AgentCapabilities` | TYPE | No (container) | eidos-annotations |
| `@EpistemicDomain` | — (nested) | No | eidos-annotations |
| `@DispositionWeight` | — (nested) | No | eidos-annotations |
| `@AxisVocabulary` | — (nested) | No | eidos-annotations |
| `@AgentTemplateRef` | TYPE | Yes (`@AgentTemplates`) | eidos-annotations |
| `@AgentTemplates` | TYPE | No (container) | eidos-annotations |
| `@TemplateArg` | — (nested) | No | eidos-annotations |

### Modified annotations (2)

| Annotation | Changes |
|-----------|---------|
| `@Identity` | +`weightsFingerprint`, +`modelVersion` |
| `@Disposition` | `dispositionProfile`/`styleProfile` type change (`String[]` → `DispositionWeight[]`), +`mbtiType`, +`enneagramType`, +`axisVocabularies` |

### New utility classes (2)

| Class | Package | Purpose |
|-------|---------|---------|
| `PersonalityTypeDeriver` | `io.casehub.eidos.api` | Shared mbtiType/enneagramType derivation |
| `PersonalityInput` | `io.casehub.eidos.api` | Structured input for `PersonalityTypeDeriver.derive()` |

### Modified classes (3)

| Class | Changes |
|-------|---------|
| `AnnotatedAgentConfig` | +`CapabilityConfig`, +`TemplateRefConfig`, +`TemplateArgConfig`, +`AxisVocabConfig`, +`EpistemicDomainConfig`, +`DispositionWeightConfig`, `dispositionProfile`/`styleProfile` type change (`String[]` → `DispositionWeightConfig[]`), +identity fields, +personality fields |
| `EidosAnnotationsProcessor` | Capability merge extraction, disposition weight/axis extraction, template extraction, extended validation |
| `EidosAnnotationsRecorder` | `Supplier` → `BeanCreator` pattern, rich capability construction, template construction, `PersonalityTypeDeriver` call |

## Example — Full Parity Agent

```java
@Identity(slot = "legal-analyst",
          jurisdiction = "EU",
          dataHandlingPolicy = "gdpr-compliant",
          briefing = "Senior legal analyst specialising in regulatory compliance",
          vocabulary = "urn:casehub:vocab:svo",
          dispositionVocabulary = "urn:casehub:vocab:conscientiousness",
          weightsFingerprint = "sha256:abc123",
          modelVersion = "2024-Q3")
@Disposition(socialOrient = "collaborative",
             ruleFollowing = "strict",
             riskAppetite = "cautious",
             autonomy = "guided",
             conflictMode = "accommodating",
             delegation = true,
             dispositionProfile = {
                 @DispositionWeight(value = "collaborative", weight = 0.8),
                 @DispositionWeight(value = "analytical", weight = 0.6)
             },
             axisVocabularies = {
                 @AxisVocabulary(axis = DispositionAxis.CONFLICT_MODE,
                                uri = "urn:casehub:vocab:thomas-kilmann")
             })
@AgentCapabilityDef(name = "document-analysis",
                    description = "Analyses legal documents and extracts key findings",
                    capabilityVocabulary = "urn:casehub:vocab:capability",
                    qualityHint = 0.95,
                    latencyHintP50Ms = 3000,
                    costHint = "medium",
                    inputTypes = {"application/pdf", "text/plain"},
                    outputTypes = {"application/json"},
                    epistemicDomains = {
                        @EpistemicDomain(value = "eu-regulatory", score = 0.95),
                        @EpistemicDomain(value = "contract-law", score = 0.8)
                    },
                    excludedDomains = {"criminal-law"})
@AgentCapabilityDef(name = "clause-extraction",
                    description = "Extracts specific clauses from legal documents",
                    qualityHint = 0.9)
@Discoverable(capabilities = {"risk-assessment"})
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
                        severity = ConstraintSeverity.HARD)
})
@AgentTemplateRef(id = "safety-primer",
                  args = {@TemplateArg(key = "domain", value = "legal")})
@AgentTemplateRef(id = "jurisdiction-notice",
                  args = {@TemplateArg(key = "region", value = "EU")})
public interface LegalAnalystAgent {
}
```

This produces an `AgentDescriptor` identical to one built via the Builder with the same field values.

## References

- [casehubio/eidos#146](https://github.com/casehubio/eidos/issues/146) — issue body with full gaps table
- [issue-139 design spec](../issue-139-eidos-annotations/2026-08-18-eidos-annotations-design.md) — original annotations architecture, Known Limitations §1-6
- [issue-139 decisions](../issue-139-eidos-annotations/decisions.md) — D1-D6 from initial annotations work
- [AgentCapability.java](../../api/src/main/java/io/casehub/eidos/api/AgentCapability.java) — 11-field capability record
- [AgentDescriptor.java](../../api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java) — full descriptor record with all fields
- [DispositionDeserializer.java](../../runtime/src/main/java/io/casehub/eidos/runtime/yaml/DispositionDeserializer.java) — mbtiType/enneagramType derivation logic (lines 51-95)
- [EidosAnnotationsProcessor.java](../../annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java) — current build extension
- [EidosAnnotationsRecorder.java](../../annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java) — current recorder
- [Supervises.java](../../org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/Supervises.java) — `@Repeatable` precedent
- [TemplateRef.java](../../api/src/main/java/io/casehub/eidos/api/TemplateRef.java) — template ref record
- [DispositionValue.java](../../api/src/main/java/io/casehub/eidos/api/DispositionValue.java) — weighted term record
