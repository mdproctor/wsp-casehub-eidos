# Annotation Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #146 — close annotation parity gaps
**Issue group:** #146

**Goal:** Close every annotation parity gap so any `AgentDescriptor` achievable via YAML is also achievable via annotations.

**Architecture:** 8 new annotation types in `casehub-eidos-annotations` runtime module, AnnotatedAgentConfig expanded with 6 new inner classes, EidosAnnotationsProcessor updated for merged extraction + validation, EidosAnnotationsRecorder switched from Supplier to BeanCreator pattern for VocabularyRegistry injection, PersonalityTypeDeriver extracted to api module as shared utility.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jandex, Jackson YAML

## Global Constraints

- Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn` not `./mvnw`
- All new annotations in package `io.casehub.eidos.annotations`
- All new annotation types in `annotations/runtime/src/main/java/io/casehub/eidos/annotations/`
- `PersonalityTypeDeriver` + `PersonalityInput` in `api/src/main/java/io/casehub/eidos/api/`
- Sentinel values: `double qualityHint() default -1` (not 0), `long latencyHintP50Ms() default -1`
- NaN guard on all `double` annotation fields: `!Double.isNaN(v)` before range check
- Test with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use `ide_insert_member` for new methods/fields, `ide_replace_member` for modifying existing ones
- Use `ide_find_references` before modifying any public API to check impact

---

## Batch 1: PersonalityTypeDeriver

### Task 1: PersonalityTypeDeriver + DispositionDeserializer refactor

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/PersonalityInput.java`
- Create: `api/src/main/java/io/casehub/eidos/api/PersonalityTypeDeriver.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/yaml/DispositionDeserializer.java`
- Test: `api/src/test/java/io/casehub/eidos/api/PersonalityTypeDeriverTest.java`

**Interfaces:**
- Produces: `PersonalityInput(String mbtiType, String enneagramType, boolean hasExplicitProfile, Map<DispositionAxis, String> explicitAxes)` record
- Produces: `PersonalityTypeDeriver.derive(PersonalityInput input, VocabularyRegistry vocabRegistry, AgentDisposition.Builder builder)` static method

- [ ] **Step 1: Write PersonalityTypeDeriverTest**

Create `api/src/test/java/io/casehub/eidos/api/PersonalityTypeDeriverTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.EnumMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class PersonalityTypeDeriverTest {

    @Test
    void mbtiType_derivesProfile_whenNoExplicitProfile() {
        var builder = AgentDisposition.builder();
        var input = new PersonalityInput("ENTJ", "", false, Map.of());
        PersonalityTypeDeriver.derive(input, testVocabRegistry(), builder);
        var disp = builder.build();
        assertThat(disp.dispositionProfile()).hasSize(8);
        assertThat(disp.dispositionProfile().get(0).term()).isEqualTo("te");
    }

    @Test
    void mbtiType_skipped_whenExplicitProfilePresent() {
        var builder = AgentDisposition.builder()
            .dispositionProfile(new DispositionValue("ti", 0.5));
        var input = new PersonalityInput("ENTJ", "", true, Map.of());
        PersonalityTypeDeriver.derive(input, testVocabRegistry(), builder);
        var disp = builder.build();
        assertThat(disp.dispositionProfile()).hasSize(1);
        assertThat(disp.dispositionProfile().get(0).term()).isEqualTo("ti");
    }

    @Test
    void mbtiType_caseInsensitive() {
        var builder = AgentDisposition.builder();
        var input = new PersonalityInput("entj", "", false, Map.of());
        PersonalityTypeDeriver.derive(input, testVocabRegistry(), builder);
        assertThat(builder.build().dispositionProfile()).hasSize(8);
    }

    @Test
    void enneagramType_derivesAxes_whenNoExplicitValues() {
        var builder = AgentDisposition.builder();
        var input = new PersonalityInput("", "challenger", false, Map.of());
        PersonalityTypeDeriver.derive(input, testVocabRegistry(), builder);
        var disp = builder.build();
        assertThat(disp.primaryTerm(DispositionAxis.RULE_FOLLOWING)).isEqualTo("flexible");
        assertThat(disp.primaryTerm(DispositionAxis.RISK_APPETITE)).isEqualTo("bold");
        assertThat(disp.primaryTerm(DispositionAxis.CONFLICT_MODE)).isEqualTo("competing");
    }

    @Test
    void enneagramType_doesNotOverrideExplicitAxes() {
        var explicitAxes = new EnumMap<DispositionAxis, String>(DispositionAxis.class);
        explicitAxes.put(DispositionAxis.SOCIAL_ORIENTATION, "collaborative");
        var builder = AgentDisposition.builder().socialOrient("collaborative");
        var input = new PersonalityInput("", "challenger", false, explicitAxes);
        PersonalityTypeDeriver.derive(input, testVocabRegistry(), builder);
        var disp = builder.build();
        assertThat(disp.primaryTerm(DispositionAxis.SOCIAL_ORIENTATION)).isEqualTo("collaborative");
    }

    @Test
    void nullRegistry_noOp() {
        var builder = AgentDisposition.builder();
        var input = new PersonalityInput("ENTJ", "challenger", false, Map.of());
        PersonalityTypeDeriver.derive(input, null, builder);
        var disp = builder.build();
        assertThat(disp.dispositionProfile()).isEmpty();
    }

    @Test
    void emptyTypes_noOp() {
        var builder = AgentDisposition.builder();
        var input = new PersonalityInput("", "", false, Map.of());
        PersonalityTypeDeriver.derive(input, testVocabRegistry(), builder);
        var disp = builder.build();
        assertThat(disp.dispositionProfile()).isEmpty();
    }

    private static VocabularyRegistry testVocabRegistry() {
        var registry = new io.casehub.eidos.runtime.vocabulary.CdiVocabularyRegistry();
        registry.register(io.casehub.eidos.vocab.JungianFunctionTerm.class);
        registry.register(io.casehub.eidos.vocab.MbtiTypeTerm.class);
        registry.register(io.casehub.eidos.vocab.EnneagramTerm.class);
        registry.register(io.casehub.eidos.vocab.ConscientiousnessTerm.class);
        registry.register(io.casehub.eidos.vocab.ThomasKilmannTerm.class);
        return registry;
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=PersonalityTypeDeriverTest`
Expected: compilation failure — `PersonalityInput` and `PersonalityTypeDeriver` don't exist yet.

- [ ] **Step 3: Create PersonalityInput record**

Create `api/src/main/java/io/casehub/eidos/api/PersonalityInput.java`:

```java
package io.casehub.eidos.api;

import java.util.Map;

public record PersonalityInput(
        String mbtiType,
        String enneagramType,
        boolean hasExplicitProfile,
        Map<DispositionAxis, String> explicitAxes) {

    public PersonalityInput {
        mbtiType = mbtiType != null ? mbtiType : "";
        enneagramType = enneagramType != null ? enneagramType : "";
        explicitAxes = explicitAxes != null ? Map.copyOf(explicitAxes) : Map.of();
    }
}
```

- [ ] **Step 4: Create PersonalityTypeDeriver**

Create `api/src/main/java/io/casehub/eidos/api/PersonalityTypeDeriver.java`.

Extract the logic from `DispositionDeserializer.java:51-95` into a static `derive()` method. The method:
1. If `mbtiType` is non-empty AND `!hasExplicitProfile` AND registry is non-null: resolve `urn:casehub:vocab:mbti` → `term.defaultProfile()` → set on builder
2. If `enneagramType` is non-empty AND registry is non-null: for each `DispositionAxis`, if `explicitAxes` doesn't contain that axis, resolve equivalent value via registry and set on builder (CONFLICT_MODE uses thomas-kilmann vocab, others use conscientiousness vocab)

```java
package io.casehub.eidos.api;

import java.util.Locale;

public final class PersonalityTypeDeriver {

    private PersonalityTypeDeriver() {}

    public static void derive(PersonalityInput input, VocabularyRegistry vocabRegistry,
                              AgentDisposition.Builder builder) {
        if (vocabRegistry == null) return;

        if (!input.mbtiType().isEmpty() && !input.hasExplicitProfile()) {
            String mbtiType = input.mbtiType().toLowerCase(Locale.ROOT);
            vocabRegistry.resolve("urn:casehub:vocab:mbti", mbtiType)
                .ifPresent(term -> builder.dispositionProfile(term.defaultProfile()));
        }

        if (!input.enneagramType().isEmpty()) {
            String enneaValue = input.enneagramType().toLowerCase(Locale.ROOT);
            if (vocabRegistry.resolve("urn:casehub:vocab:enneagram", enneaValue).isPresent()) {
                for (var axis : DispositionAxis.values()) {
                    if (input.explicitAxes().containsKey(axis)) continue;
                    if (axis == DispositionAxis.CONFLICT_MODE) {
                        vocabRegistry.equivalentValues(
                            "urn:casehub:vocab:enneagram", enneaValue,
                            "urn:casehub:vocab:thomas-kilmann", axis)
                            .ifPresent(builder::conflictMode);
                    } else {
                        vocabRegistry.equivalentValues(
                            "urn:casehub:vocab:enneagram", enneaValue,
                            "urn:casehub:vocab:conscientiousness", axis)
                            .ifPresent(val -> {
                                switch (axis) {
                                    case SOCIAL_ORIENTATION -> builder.socialOrient(val);
                                    case RULE_FOLLOWING -> builder.ruleFollowing(val);
                                    case RISK_APPETITE -> builder.riskAppetite(val);
                                    case AUTONOMY -> builder.autonomy(val);
                                    default -> {}
                                }
                            });
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 5: Run PersonalityTypeDeriverTest — verify green**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=PersonalityTypeDeriverTest`
Expected: all 7 tests pass.

- [ ] **Step 6: Refactor DispositionDeserializer to use PersonalityTypeDeriver**

In `runtime/src/main/java/io/casehub/eidos/runtime/yaml/DispositionDeserializer.java`, replace the inline mbtiType/enneagramType logic (lines 51-95) with a call to `PersonalityTypeDeriver.derive()`. Build the `PersonalityInput` from the parsed YAML fields:

```java
var explicitAxes = new EnumMap<DispositionAxis, String>(DispositionAxis.class);
if (explicitSocialOrient != null) explicitAxes.put(DispositionAxis.SOCIAL_ORIENTATION, explicitSocialOrient);
if (explicitRuleFollowing != null) explicitAxes.put(DispositionAxis.RULE_FOLLOWING, explicitRuleFollowing);
if (explicitRiskAppetite != null) explicitAxes.put(DispositionAxis.RISK_APPETITE, explicitRiskAppetite);
if (explicitAutonomy != null) explicitAxes.put(DispositionAxis.AUTONOMY, explicitAutonomy);
if (explicitConflictMode != null) explicitAxes.put(DispositionAxis.CONFLICT_MODE, explicitConflictMode);

String mbtiType = root.has("mbtiType") ? root.get("mbtiType").asText() : "";
String enneagramType = root.has("enneagramType") ? root.get("enneagramType").asText() : "";

PersonalityTypeDeriver.derive(
    new PersonalityInput(mbtiType, enneagramType, explicitProfile != null, explicitAxes),
    vocabRegistry, builder);
```

- [ ] **Step 7: Run DispositionDeserializerTest — verify regression green**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DispositionDeserializerTest`
Expected: all existing tests pass (mbtiType, enneagramType, explicit override tests).

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/PersonalityTypeDeriver.java api/src/main/java/io/casehub/eidos/api/PersonalityInput.java api/src/test/java/io/casehub/eidos/api/PersonalityTypeDeriverTest.java runtime/src/main/java/io/casehub/eidos/runtime/yaml/DispositionDeserializer.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#146): PersonalityTypeDeriver utility — extract shared mbti/enneagram derivation  Refs #146"
```

---

## Batch 2: Annotation Definitions + @Identity/@Disposition Pipeline

### Task 2: Annotation definitions + AnnotatedAgentConfig + update test agents

**Files:**
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilityDef.java`
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilities.java`
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/EpistemicDomain.java`
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/DispositionWeight.java`
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AxisVocabulary.java`
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentTemplateRef.java`
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentTemplates.java`
- Create: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/TemplateArg.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/Identity.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/Disposition.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedAgentConfig.java`
- Modify: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/SimpleAnnotatedAgent.java`
- Modify: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/FullAnnotatedAgent.java`

**Interfaces:**
- Produces: All 8 new annotation types available for use
- Produces: `AnnotatedAgentConfig` with `CapabilityConfig`, `DispositionWeightConfig`, `AxisVocabConfig`, `TemplateRefConfig`, `TemplateArgConfig`, `EpistemicDomainConfig` inner classes

- [ ] **Step 1: Create @DispositionWeight annotation**

Create `annotations/runtime/src/main/java/io/casehub/eidos/annotations/DispositionWeight.java`:

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
public @interface DispositionWeight {
    String value();
    double weight() default 1.0;
}
```

- [ ] **Step 2: Create @AxisVocabulary annotation**

Create `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AxisVocabulary.java`:

```java
package io.casehub.eidos.annotations;

import io.casehub.eidos.api.DispositionAxis;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
public @interface AxisVocabulary {
    DispositionAxis axis();
    String uri();
}
```

- [ ] **Step 3: Create @EpistemicDomain annotation**

Create `annotations/runtime/src/main/java/io/casehub/eidos/annotations/EpistemicDomain.java`:

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
public @interface EpistemicDomain {
    String value();
    double score();
}
```

- [ ] **Step 4: Create @AgentCapabilityDef + @AgentCapabilities**

Create `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilityDef.java`:

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Repeatable(AgentCapabilities.class)
public @interface AgentCapabilityDef {
    String name();
    String description() default "";
    String capabilityVocabulary() default "";
    double qualityHint() default -1;
    long latencyHintP50Ms() default -1;
    String costHint() default "";
    String[] inputTypes() default {};
    String[] outputTypes() default {};
    String[] tags() default {};
    EpistemicDomain[] epistemicDomains() default {};
    String[] excludedDomains() default {};
}
```

Create `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentCapabilities.java`:

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AgentCapabilities {
    AgentCapabilityDef[] value();
}
```

- [ ] **Step 5: Create @AgentTemplateRef + @AgentTemplates + @TemplateArg**

Create `annotations/runtime/src/main/java/io/casehub/eidos/annotations/TemplateArg.java`:

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Retention(RetentionPolicy.RUNTIME)
public @interface TemplateArg {
    String key();
    String value();
}
```

Create `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentTemplateRef.java`:

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Repeatable(AgentTemplates.class)
public @interface AgentTemplateRef {
    String id();
    TemplateArg[] args() default {};
}
```

Create `annotations/runtime/src/main/java/io/casehub/eidos/annotations/AgentTemplates.java`:

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AgentTemplates {
    AgentTemplateRef[] value();
}
```

- [ ] **Step 6: Modify @Identity — add weightsFingerprint + modelVersion**

Add to `annotations/runtime/src/main/java/io/casehub/eidos/annotations/Identity.java`:

```java
String weightsFingerprint() default "";
String modelVersion() default "";
```

- [ ] **Step 7: Modify @Disposition — type changes + new fields**

Replace `annotations/runtime/src/main/java/io/casehub/eidos/annotations/Disposition.java` with:

```java
package io.casehub.eidos.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Disposition {
    String socialOrient() default "";
    String ruleFollowing() default "";
    String riskAppetite() default "";
    String autonomy() default "";
    String conflictMode() default "";
    boolean delegation() default false;
    DispositionWeight[] dispositionProfile() default {};
    DispositionWeight[] styleProfile() default {};
    String mbtiType() default "";
    String enneagramType() default "";
    AxisVocabulary[] axisVocabularies() default {};
}
```

- [ ] **Step 8: Update AnnotatedAgentConfig — add all new inner classes**

Add to `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedAgentConfig.java`:

New fields on the main class:
```java
public String weightsFingerprint;
public String modelVersion;
public String mbtiType;
public String enneagramType;
public DispositionWeightConfig[] dispositionProfile;
public DispositionWeightConfig[] styleProfile;
public AxisVocabConfig[] axisVocabularies;
public TemplateRefConfig[] templateRefs;
public CapabilityConfig[] richCapabilities;
```

Remove the old `String[] dispositionProfile` and `String[] styleProfile` fields (replaced by `DispositionWeightConfig[]`).

New inner classes:
```java
public static class DispositionWeightConfig {
    public String value;
    public double weight;
    public DispositionWeightConfig() {}
}

public static class AxisVocabConfig {
    public String axis;
    public String uri;
    public AxisVocabConfig() {}
}

public static class TemplateRefConfig {
    public String id;
    public TemplateArgConfig[] args;
    public TemplateRefConfig() {}
}

public static class TemplateArgConfig {
    public String key;
    public String value;
    public TemplateArgConfig() {}
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
    public CapabilityConfig() {}
}

public static class EpistemicDomainConfig {
    public String value;
    public double score;
    public EpistemicDomainConfig() {}
}
```

- [ ] **Step 9: Update existing test agents for @Disposition breaking change**

`SimpleAnnotatedAgent.java` — no `dispositionProfile` or `styleProfile`, no change needed.

`FullAnnotatedAgent.java` — no `dispositionProfile` or `styleProfile`, no change needed.

Both test agents use only axis string fields (`socialOrient`, `ruleFollowing`, etc.), which are unchanged. No migration needed for these test agents.

- [ ] **Step 10: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl annotations/runtime,annotations/deployment`
Expected: compiles successfully.

Note: existing `EidosAnnotationsProcessorTest` will fail because the processor's `extractDisposition()` still references the old `String[]` fields on config. This is expected — Task 3 fixes it.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations/runtime/src/main/java/io/casehub/eidos/annotations/ annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedAgentConfig.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#146): annotation definitions + config classes for parity  Refs #146"
```

---

### Task 3: Processor + Recorder — @Identity fields, weighted profiles, axis vocabularies, personality types

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java`
- Create: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/WeightedDispositionAgent.java`
- Modify: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessorTest.java`

**Interfaces:**
- Consumes: All annotation types from Task 2, `PersonalityTypeDeriver.derive()` from Task 1
- Consumes: `AnnotatedAgentConfig` with new inner classes from Task 2
- Produces: Full pipeline from annotations → processor → config → recorder → AgentDescriptor for @Identity fields, weighted profiles, axis vocabularies, and personality types

- [ ] **Step 1: Write test agent — WeightedDispositionAgent**

Create `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/WeightedDispositionAgent.java`:

```java
package io.casehub.eidos.annotations.deployment.test;

import io.casehub.eidos.annotations.*;
import io.casehub.eidos.api.DispositionAxis;

@Identity(slot = "weighted-agent", briefing = "Agent with weighted profiles",
          weightsFingerprint = "sha256:abc123", modelVersion = "2024-Q3")
@Disposition(
    socialOrient = "collaborative",
    dispositionProfile = {
        @DispositionWeight(value = "collaborative", weight = 0.8),
        @DispositionWeight(value = "analytical", weight = 0.4)
    },
    styleProfile = {
        @DispositionWeight(value = "concise", weight = 0.7)
    },
    axisVocabularies = {
        @AxisVocabulary(axis = DispositionAxis.CONFLICT_MODE,
                       uri = "urn:casehub:vocab:thomas-kilmann")
    })
public interface WeightedDispositionAgent {}
```

- [ ] **Step 2: Write tests for @Identity fields, weighted profiles, axis vocabularies**

Add to `EidosAnnotationsProcessorTest.java`:

Add `WeightedDispositionAgent.class` to the `QuarkusUnitTest` `.withApplicationRoot()`.

```java
@Test
void identityFieldsWeightsFingerprintAndModelVersion() {
    var d = registry.findById("weighted-disposition-agent", "test-tenant").orElseThrow();
    assertThat(d.weightsFingerprint()).isEqualTo("sha256:abc123");
    assertThat(d.modelVersion()).isEqualTo("2024-Q3");
}

@Test
void weightedDispositionProfile() {
    var d = registry.findById("weighted-disposition-agent", "test-tenant").orElseThrow();
    assertThat(d.disposition().dispositionProfile()).hasSize(2);
    assertThat(d.disposition().dispositionProfile().get(0).term()).isEqualTo("collaborative");
    assertThat(d.disposition().dispositionProfile().get(0).weight()).isEqualTo(0.8);
    assertThat(d.disposition().dispositionProfile().get(1).term()).isEqualTo("analytical");
    assertThat(d.disposition().dispositionProfile().get(1).weight()).isEqualTo(0.4);
}

@Test
void weightedStyleProfile() {
    var d = registry.findById("weighted-disposition-agent", "test-tenant").orElseThrow();
    assertThat(d.disposition().styleProfile()).hasSize(1);
    assertThat(d.disposition().styleProfile().get(0).term()).isEqualTo("concise");
    assertThat(d.disposition().styleProfile().get(0).weight()).isEqualTo(0.7);
}

@Test
void axisVocabularies() {
    var d = registry.findById("weighted-disposition-agent", "test-tenant").orElseThrow();
    assertThat(d.axisVocabularies()).containsEntry(
        io.casehub.eidos.api.DispositionAxis.CONFLICT_MODE,
        "urn:casehub:vocab:thomas-kilmann");
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations/deployment -Dtest=EidosAnnotationsProcessorTest`
Expected: FAIL — processor doesn't extract new fields yet.

- [ ] **Step 4: Update EidosAnnotationsProcessor — extractConfig for @Identity fields**

In `extractConfig()`, add after the existing `config.version` line:

```java
config.weightsFingerprint = stringValue(identity, "weightsFingerprint");
config.modelVersion = stringValue(identity, "modelVersion");
```

- [ ] **Step 5: Update EidosAnnotationsProcessor — extractDisposition for weighted profiles + axis vocabs + personality types**

Replace the body of `extractDisposition()` to handle `@DispositionWeight[]` instead of `String[]` for disposition/style profiles, extract `@AxisVocabulary[]`, and extract `mbtiType`/`enneagramType`:

```java
private void extractDisposition(ClassInfo classInfo, AnnotatedAgentConfig config) {
    var ann = classInfo.annotation(DISPOSITION);
    if (ann == null) {
        config.hasDisposition = false;
        return;
    }
    config.hasDisposition = true;
    config.socialOrient = stringValue(ann, "socialOrient");
    config.ruleFollowing = stringValue(ann, "ruleFollowing");
    config.riskAppetite = stringValue(ann, "riskAppetite");
    config.autonomy = stringValue(ann, "autonomy");
    config.conflictMode = stringValue(ann, "conflictMode");
    var del = ann.value("delegation");
    config.delegation = del != null && del.asBoolean();

    // Weighted disposition profiles (D4)
    var dp = ann.value("dispositionProfile");
    if (dp != null) {
        var nested = dp.asNestedArray();
        config.dispositionProfile = new AnnotatedAgentConfig.DispositionWeightConfig[nested.length];
        for (int i = 0; i < nested.length; i++) {
            var dwc = new AnnotatedAgentConfig.DispositionWeightConfig();
            dwc.value = nested[i].value("value").asString();
            var w = nested[i].value("weight");
            dwc.weight = w != null ? w.asDouble() : 1.0;
            config.dispositionProfile[i] = dwc;
        }
    }
    var sp = ann.value("styleProfile");
    if (sp != null) {
        var nested = sp.asNestedArray();
        config.styleProfile = new AnnotatedAgentConfig.DispositionWeightConfig[nested.length];
        for (int i = 0; i < nested.length; i++) {
            var dwc = new AnnotatedAgentConfig.DispositionWeightConfig();
            dwc.value = nested[i].value("value").asString();
            var w = nested[i].value("weight");
            dwc.weight = w != null ? w.asDouble() : 1.0;
            config.styleProfile[i] = dwc;
        }
    }

    // Axis vocabularies (D7)
    var av = ann.value("axisVocabularies");
    if (av != null) {
        var nested = av.asNestedArray();
        config.axisVocabularies = new AnnotatedAgentConfig.AxisVocabConfig[nested.length];
        for (int i = 0; i < nested.length; i++) {
            var avc = new AnnotatedAgentConfig.AxisVocabConfig();
            avc.axis = nested[i].value("axis").asEnum();
            avc.uri = nested[i].value("uri").asString();
            config.axisVocabularies[i] = avc;
        }
    }

    // Personality types (D10)
    config.mbtiType = stringValue(ann, "mbtiType");
    config.enneagramType = stringValue(ann, "enneagramType");
}
```

- [ ] **Step 6: Update validateArrayTerms for @DispositionWeight type change**

The existing `validateArrayTerms()` calls `v.asStringArray()` which won't work for `@DispositionWeight[]`. Update it to handle both old-style `String[]` (for `dispositionProfile` which is now `DispositionWeight[]`) and extract the `value()` field:

```java
private void validateArrayTerms(AnnotationInstance ann, String field,
                                java.util.Set<String> validTerms, String vocabUri, ClassInfo classInfo) {
    var v = ann.value(field);
    if (v == null) return;
    if (field.equals("dispositionProfile") || field.equals("styleProfile")) {
        for (var nested : v.asNestedArray()) {
            var term = nested.value("value").asString();
            if (term.isEmpty()) continue;
            if (validTerms.stream().noneMatch(t -> t.equalsIgnoreCase(term))) {
                LOG.warnf("@Disposition.%s value '%s' on %s may not be a valid term in vocabulary '%s'"
                          + " (build-time check uses enum constant names, not VocabularyTerm.value())",
                          field, term, classInfo.name(), vocabUri);
            }
        }
    } else {
        for (var term : v.asStringArray()) {
            if (term.isEmpty()) continue;
            if (validTerms.stream().noneMatch(t -> t.equalsIgnoreCase(term))) {
                LOG.warnf("@Disposition.%s value '%s' on %s may not be a valid term in vocabulary '%s'"
                          + " (build-time check uses enum constant names, not VocabularyTerm.value())",
                          field, term, classInfo.name(), vocabUri);
            }
        }
    }
}
```

- [ ] **Step 7: Update EidosAnnotationsRecorder — BeanCreator + @Identity fields + weighted profiles + axis vocabs + personality types**

Replace the entire `EidosAnnotationsRecorder` class. Key changes:
1. `createRegistrar()` returns `Function<SyntheticCreationalContext<AgentDescriptorRegistrar>, AgentDescriptorRegistrar>` instead of `Supplier<AgentDescriptorRegistrar>`
2. Inject `VocabularyRegistry` from `SyntheticCreationalContext`
3. Add `weightsFingerprint`, `modelVersion` to builder
4. Build `DispositionValue` from `DispositionWeightConfig` (with explicit weight, not `DispositionValue.of()`)
5. Build `axisVocabularies` map from `AxisVocabConfig`
6. Call `PersonalityTypeDeriver.derive()` for mbtiType/enneagramType

```java
package io.casehub.eidos.annotations.runtime;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.spi.AgentDescriptorRegistrar;
import io.quarkus.arc.SyntheticCreationalContext;
import io.quarkus.runtime.annotations.Recorder;
import org.eclipse.microprofile.config.ConfigProvider;

import java.util.*;
import java.util.function.Function;

@Recorder
public class EidosAnnotationsRecorder {

    private static final String TENANCY_CONFIG_KEY = "casehub.eidos.annotations.default-tenancy-id";

    public Function<SyntheticCreationalContext<AgentDescriptorRegistrar>, AgentDescriptorRegistrar>
            createRegistrar(AnnotatedAgentConfig config) {
        return ctx -> {
            var vocabRegistry = ctx.getInjectedReference(VocabularyRegistry.class);
            var tenancyId = ConfigProvider.getConfig()
                    .getOptionalValue(TENANCY_CONFIG_KEY, String.class)
                    .orElse("default");

            var builder = AgentDescriptor.builder()
                    .agentId(config.agentId).name(config.name).slot(config.slot).tenancyId(tenancyId);

            if (notEmpty(config.provider)) builder.provider(config.provider);
            if (notEmpty(config.modelFamily)) builder.modelFamily(config.modelFamily);
            if (notEmpty(config.jurisdiction)) builder.jurisdiction(config.jurisdiction);
            if (notEmpty(config.dataHandlingPolicy)) builder.dataHandlingPolicy(config.dataHandlingPolicy);
            if (notEmpty(config.briefing)) builder.briefing(config.briefing);
            if (notEmpty(config.domainVocabulary)) builder.domainVocabulary(config.domainVocabulary);
            if (notEmpty(config.slotVocabulary)) builder.slotVocabulary(config.slotVocabulary);
            if (notEmpty(config.dispositionVocabulary)) builder.dispositionVocabulary(config.dispositionVocabulary);
            if (notEmpty(config.styleVocabulary)) builder.styleVocabulary(config.styleVocabulary);
            if (notEmpty(config.version)) builder.version(config.version);
            if (notEmpty(config.weightsFingerprint)) builder.weightsFingerprint(config.weightsFingerprint);
            if (notEmpty(config.modelVersion)) builder.modelVersion(config.modelVersion);

            if (config.hasDisposition) {
                var db = AgentDisposition.builder().delegation(config.delegation);
                if (notEmpty(config.socialOrient)) db.socialOrient(config.socialOrient);
                if (notEmpty(config.ruleFollowing)) db.ruleFollowing(config.ruleFollowing);
                if (notEmpty(config.riskAppetite)) db.riskAppetite(config.riskAppetite);
                if (notEmpty(config.autonomy)) db.autonomy(config.autonomy);
                if (notEmpty(config.conflictMode)) db.conflictMode(config.conflictMode);

                if (config.dispositionProfile != null && config.dispositionProfile.length > 0) {
                    var values = new ArrayList<DispositionValue>();
                    for (var dp : config.dispositionProfile) {
                        if (notEmpty(dp.value)) values.add(new DispositionValue(dp.value, dp.weight));
                    }
                    if (!values.isEmpty()) db.dispositionProfile(values);
                }
                if (config.styleProfile != null && config.styleProfile.length > 0) {
                    var values = new ArrayList<DispositionValue>();
                    for (var sp : config.styleProfile) {
                        if (notEmpty(sp.value)) values.add(new DispositionValue(sp.value, sp.weight));
                    }
                    if (!values.isEmpty()) db.styleProfile(values);
                }

                // Personality type derivation (D5, D10)
                var explicitAxes = new EnumMap<DispositionAxis, String>(DispositionAxis.class);
                if (notEmpty(config.socialOrient)) explicitAxes.put(DispositionAxis.SOCIAL_ORIENTATION, config.socialOrient);
                if (notEmpty(config.ruleFollowing)) explicitAxes.put(DispositionAxis.RULE_FOLLOWING, config.ruleFollowing);
                if (notEmpty(config.riskAppetite)) explicitAxes.put(DispositionAxis.RISK_APPETITE, config.riskAppetite);
                if (notEmpty(config.autonomy)) explicitAxes.put(DispositionAxis.AUTONOMY, config.autonomy);
                if (notEmpty(config.conflictMode)) explicitAxes.put(DispositionAxis.CONFLICT_MODE, config.conflictMode);

                PersonalityTypeDeriver.derive(
                    new PersonalityInput(
                        config.mbtiType != null ? config.mbtiType : "",
                        config.enneagramType != null ? config.enneagramType : "",
                        config.dispositionProfile != null && config.dispositionProfile.length > 0,
                        explicitAxes),
                    vocabRegistry, db);

                builder.disposition(db.build());
            }

            // Axis vocabularies (D7)
            if (config.axisVocabularies != null && config.axisVocabularies.length > 0) {
                var map = new EnumMap<DispositionAxis, String>(DispositionAxis.class);
                for (var av : config.axisVocabularies) {
                    map.put(DispositionAxis.valueOf(av.axis), av.uri);
                }
                builder.axisVocabularies(map);
            }

            if (config.goals != null) {
                var goals = new ArrayList<AgentGoal>();
                for (var g : config.goals) {
                    goals.add(new AgentGoal(g.name, g.description,
                            GoalPriority.valueOf(g.priority), Visibility.valueOf(g.visibility),
                            g.capabilities != null ? List.of(g.capabilities) : List.of(), null));
                }
                builder.goals(goals);
            }

            if (config.constraints != null) {
                var constraints = new ArrayList<AgentConstraint>();
                for (var c : config.constraints) {
                    constraints.add(new AgentConstraint(c.name, c.description,
                            Visibility.valueOf(c.visibility), ConstraintSeverity.valueOf(c.severity)));
                }
                builder.constraints(constraints);
            }

            // Capabilities — merged from @Discoverable + @AgentCapabilityDef (D2)
            buildCapabilities(config, builder);

            // Templates (D6) — handled in Task 6
            buildTemplates(config, builder);

            return (AgentDescriptorRegistrar) () -> List.of(builder.build());
        };
    }

    private static void buildCapabilities(AnnotatedAgentConfig config, AgentDescriptor.Builder builder) {
        var caps = new ArrayList<AgentCapability>();
        if (config.capabilities != null && config.capabilities.length > 0) {
            for (var name : config.capabilities) {
                caps.add(new AgentCapability.Builder().name(name).build());
            }
        }
        if (config.richCapabilities != null) {
            for (var cap : config.richCapabilities) {
                var cb = AgentCapability.builder().name(cap.name);
                if (notEmpty(cap.description)) cb.description(cap.description);
                if (notEmpty(cap.capabilityVocabulary)) cb.capabilityVocabulary(cap.capabilityVocabulary);
                if (cap.qualityHint >= 0) cb.qualityHint(cap.qualityHint);
                if (cap.latencyHintP50Ms >= 0) cb.latencyHintP50Ms(cap.latencyHintP50Ms);
                if (notEmpty(cap.costHint)) cb.costHint(cap.costHint);
                if (cap.inputTypes != null && cap.inputTypes.length > 0) cb.inputTypes(List.of(cap.inputTypes));
                if (cap.outputTypes != null && cap.outputTypes.length > 0) cb.outputTypes(List.of(cap.outputTypes));
                if (cap.tags != null && cap.tags.length > 0) cb.tags(List.of(cap.tags));
                if (cap.epistemicDomains != null && cap.epistemicDomains.length > 0) {
                    var map = new HashMap<String, Double>();
                    for (var ed : cap.epistemicDomains) map.put(ed.value, ed.score);
                    cb.epistemicDomains(map);
                }
                if (cap.excludedDomains != null && cap.excludedDomains.length > 0) {
                    cb.excludedDomains(Set.of(cap.excludedDomains));
                }
                caps.add(cb.build());
            }
        }
        if (!caps.isEmpty()) builder.capabilities(caps);
    }

    private static void buildTemplates(AnnotatedAgentConfig config, AgentDescriptor.Builder builder) {
        if (config.templateRefs == null || config.templateRefs.length == 0) return;
        var refs = new ArrayList<TemplateRef>();
        for (var ref : config.templateRefs) {
            var args = new HashMap<String, String>();
            if (ref.args != null) {
                for (var arg : ref.args) args.put(arg.key, arg.value);
            }
            refs.add(new TemplateRef(ref.id, args));
        }
        builder.templates(refs);
    }

    private static boolean notEmpty(String s) {
        return s != null && !s.isEmpty();
    }
}
```

- [ ] **Step 8: Update EidosAnnotationsProcessor — change SyntheticBeanBuildItem from .supplier() to .createWith()**

In `processAnnotations()`, change the synthetic bean creation:

```java
syntheticBeans.produce(SyntheticBeanBuildItem
        .configure(AgentDescriptorRegistrar.class)
        .scope(ApplicationScoped.class)
        .identifier("eidos-ann-" + className)
        .setRuntimeInit()
        .createWith(recorder.createRegistrar(config))
        .addInjectionPoint(io.casehub.eidos.api.VocabularyRegistry.class)
        .done());
```

- [ ] **Step 9: Run tests — verify green**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations/deployment -Dtest=EidosAnnotationsProcessorTest`
Expected: all existing + new tests pass.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#146): processor + recorder — identity fields, weighted profiles, axis vocabs, personality types  Refs #146"
```

---

## Batch 3: Rich Capabilities + Templates

### Task 4: Processor + Recorder — Rich capabilities + templates

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`
- Create: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/RichCapabilityAgent.java`
- Create: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/CapabilityMergeAgent.java`
- Create: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/TemplatedAgent.java`
- Modify: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessorTest.java`

**Interfaces:**
- Consumes: `@AgentCapabilityDef`, `@EpistemicDomain`, `@AgentTemplateRef`, `@TemplateArg` from Task 2
- Consumes: `AnnotatedAgentConfig.CapabilityConfig`, `TemplateRefConfig` from Task 2
- Consumes: `buildCapabilities()`, `buildTemplates()` already in recorder from Task 3
- Produces: Full pipeline for rich capabilities (merge semantics) and templates

- [ ] **Step 1: Write test agents**

Create `RichCapabilityAgent.java`:

```java
package io.casehub.eidos.annotations.deployment.test;

import io.casehub.eidos.annotations.*;

@Identity(slot = "rich-cap-agent", briefing = "Agent with rich capabilities")
@AgentCapabilityDef(name = "analysis",
    description = "Deep analysis capability",
    capabilityVocabulary = "urn:casehub:vocab:capability",
    qualityHint = 0.95,
    latencyHintP50Ms = 3000,
    costHint = "medium",
    inputTypes = {"application/pdf", "text/plain"},
    outputTypes = {"application/json"},
    tags = {"nlp", "extraction"},
    epistemicDomains = {
        @EpistemicDomain(value = "legal", score = 0.95),
        @EpistemicDomain(value = "financial", score = 0.6)
    },
    excludedDomains = {"criminal-law"})
@AgentCapabilityDef(name = "summarization",
    description = "Summarizes documents",
    qualityHint = 0.8)
public interface RichCapabilityAgent {}
```

Create `CapabilityMergeAgent.java`:

```java
package io.casehub.eidos.annotations.deployment.test;

import io.casehub.eidos.annotations.*;
import io.casehub.eidos.api.Discoverable;

@Identity(slot = "merge-agent", briefing = "Agent with merged capabilities")
@Discoverable(capabilities = {"simple-cap"})
@AgentCapabilityDef(name = "rich-cap", description = "A rich capability", qualityHint = 0.9)
public interface CapabilityMergeAgent {}
```

Create `TemplatedAgent.java`:

```java
package io.casehub.eidos.annotations.deployment.test;

import io.casehub.eidos.annotations.*;

@Identity(slot = "templated-agent", briefing = "Agent with templates")
@AgentTemplateRef(id = "safety-primer",
    args = {@TemplateArg(key = "domain", value = "legal")})
@AgentTemplateRef(id = "jurisdiction-notice",
    args = {@TemplateArg(key = "region", value = "EU")})
public interface TemplatedAgent {}
```

- [ ] **Step 2: Write tests**

Add test agents to `QuarkusUnitTest.withApplicationRoot()`, then add test methods:

```java
@Test
void richCapability_fullMetadata() {
    var d = registry.findById("rich-capability-agent", "test-tenant").orElseThrow();
    assertThat(d.capabilities()).hasSize(2);
    var analysis = d.capabilities().stream().filter(c -> c.name().equals("analysis")).findFirst().orElseThrow();
    assertThat(analysis.description()).isEqualTo("Deep analysis capability");
    assertThat(analysis.capabilityVocabulary()).isEqualTo("urn:casehub:vocab:capability");
    assertThat(analysis.qualityHint()).isEqualTo(0.95);
    assertThat(analysis.latencyHintP50Ms()).isEqualTo(3000L);
    assertThat(analysis.costHint()).isEqualTo("medium");
    assertThat(analysis.inputTypes()).containsExactly("application/pdf", "text/plain");
    assertThat(analysis.outputTypes()).containsExactly("application/json");
    assertThat(analysis.tags()).containsExactly("nlp", "extraction");
    assertThat(analysis.epistemicDomains()).containsEntry("legal", 0.95);
    assertThat(analysis.epistemicDomains()).containsEntry("financial", 0.6);
    assertThat(analysis.excludedDomains()).containsExactly("criminal-law");
}

@Test
void capabilityMerge_unionOfDiscoverableAndRichCaps() {
    var d = registry.findById("merge-agent", "test-tenant").orElseThrow();
    assertThat(d.capabilities()).hasSize(2);
    assertThat(d.capabilities().stream().map(c -> c.name()).toList())
        .containsExactlyInAnyOrder("simple-cap", "rich-cap");
    var richCap = d.capabilities().stream().filter(c -> c.name().equals("rich-cap")).findFirst().orElseThrow();
    assertThat(richCap.qualityHint()).isEqualTo(0.9);
}

@Test
void templates_repeatable() {
    var d = registry.findById("templated-agent", "test-tenant").orElseThrow();
    assertThat(d.templates()).hasSize(2);
    assertThat(d.templates().get(0).templateId()).isEqualTo("safety-primer");
    assertThat(d.templates().get(0).args()).containsEntry("domain", "legal");
    assertThat(d.templates().get(1).templateId()).isEqualTo("jurisdiction-notice");
    assertThat(d.templates().get(1).args()).containsEntry("region", "EU");
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations/deployment -Dtest=EidosAnnotationsProcessorTest`
Expected: FAIL — processor doesn't extract capabilities or templates yet.

- [ ] **Step 4: Update EidosAnnotationsProcessor — add DotNames for new annotations**

Add class-level constants:

```java
private static final DotName AGENT_CAPABILITY_DEF = DotName.createSimple(AgentCapabilityDef.class);
private static final DotName AGENT_CAPABILITIES = DotName.createSimple(AgentCapabilities.class);
private static final DotName AGENT_TEMPLATE_REF = DotName.createSimple(AgentTemplateRef.class);
private static final DotName AGENT_TEMPLATES = DotName.createSimple(AgentTemplates.class);
```

- [ ] **Step 5: Update extractCapabilities — merged extraction from @Discoverable + @AgentCapabilityDef**

Replace `extractCapabilities()`:

```java
private void extractCapabilities(ClassInfo classInfo, AnnotatedAgentConfig config) {
    // Name-only from @Discoverable
    var discAnn = classInfo.annotation(DISCOVERABLE);
    if (discAnn != null) {
        config.capabilities = discAnn.value("capabilities").asStringArray();
    }

    // Rich from @AgentCapabilityDef (repeatable)
    var capDefs = classInfo.annotationsWithRepeatable(AGENT_CAPABILITY_DEF, AGENT_CAPABILITIES);
    if (!capDefs.isEmpty()) {
        config.richCapabilities = new AnnotatedAgentConfig.CapabilityConfig[capDefs.size()];
        var richNames = new HashSet<String>();
        for (int i = 0; i < capDefs.size(); i++) {
            var ann = capDefs.get(i);
            var cap = new AnnotatedAgentConfig.CapabilityConfig();
            cap.name = ann.value("name").asString();
            if (!richNames.add(cap.name)) {
                throw new IllegalStateException(
                    "Duplicate @AgentCapabilityDef name '" + cap.name + "' on " + classInfo.name());
            }
            cap.description = stringValue(ann, "description");
            cap.capabilityVocabulary = stringValue(ann, "capabilityVocabulary");
            var qh = ann.value("qualityHint");
            cap.qualityHint = qh != null ? qh.asDouble() : -1;
            var lh = ann.value("latencyHintP50Ms");
            cap.latencyHintP50Ms = lh != null ? lh.asLong() : -1;
            cap.costHint = stringValue(ann, "costHint");
            var it = ann.value("inputTypes");
            cap.inputTypes = it != null ? it.asStringArray() : new String[0];
            var ot = ann.value("outputTypes");
            cap.outputTypes = ot != null ? ot.asStringArray() : new String[0];
            var tg = ann.value("tags");
            cap.tags = tg != null ? tg.asStringArray() : new String[0];
            var ed = ann.value("epistemicDomains");
            if (ed != null) {
                var nested = ed.asNestedArray();
                cap.epistemicDomains = new AnnotatedAgentConfig.EpistemicDomainConfig[nested.length];
                for (int j = 0; j < nested.length; j++) {
                    var edc = new AnnotatedAgentConfig.EpistemicDomainConfig();
                    edc.value = nested[j].value("value").asString();
                    edc.score = nested[j].value("score").asDouble();
                    cap.epistemicDomains[j] = edc;
                }
            }
            var exd = ann.value("excludedDomains");
            cap.excludedDomains = exd != null ? exd.asStringArray() : new String[0];
            config.richCapabilities[i] = cap;
        }
    }

    // Collision check: @Discoverable names vs @AgentCapabilityDef names
    if (config.capabilities != null && config.richCapabilities != null) {
        var discNames = new HashSet<>(java.util.Arrays.asList(config.capabilities));
        for (var rc : config.richCapabilities) {
            if (discNames.contains(rc.name)) {
                throw new IllegalStateException(
                    "Capability '" + rc.name + "' on " + classInfo.name()
                    + " appears in both @Discoverable and @AgentCapabilityDef");
            }
        }
    }
}
```

- [ ] **Step 6: Add extractTemplates method**

```java
private void extractTemplates(ClassInfo classInfo, AnnotatedAgentConfig config) {
    var templateDefs = classInfo.annotationsWithRepeatable(AGENT_TEMPLATE_REF, AGENT_TEMPLATES);
    if (templateDefs.isEmpty()) return;
    config.templateRefs = new AnnotatedAgentConfig.TemplateRefConfig[templateDefs.size()];
    for (int i = 0; i < templateDefs.size(); i++) {
        var ann = templateDefs.get(i);
        var ref = new AnnotatedAgentConfig.TemplateRefConfig();
        ref.id = ann.value("id").asString();
        var argsVal = ann.value("args");
        if (argsVal != null) {
            var nested = argsVal.asNestedArray();
            ref.args = new AnnotatedAgentConfig.TemplateArgConfig[nested.length];
            for (int j = 0; j < nested.length; j++) {
                var tac = new AnnotatedAgentConfig.TemplateArgConfig();
                tac.key = nested[j].value("key").asString();
                tac.value = nested[j].value("value").asString();
                ref.args[j] = tac;
            }
        }
        config.templateRefs[i] = ref;
    }
}
```

- [ ] **Step 7: Call extractTemplates from extractConfig**

In `extractConfig()`, add after `extractCapabilities`:

```java
extractTemplates(classInfo, config);
```

- [ ] **Step 8: Update validateGoalCapabilities for merged capability names**

Update to check both `config.capabilities` (name-only) and `config.richCapabilities`:

```java
private void validateGoalCapabilities(ClassInfo classInfo, AnnotatedAgentConfig config) {
    if (config.goals == null) return;
    var capNames = new HashSet<String>();
    if (config.capabilities != null) {
        for (var cap : config.capabilities) capNames.add(cap);
    }
    if (config.richCapabilities != null) {
        for (var cap : config.richCapabilities) capNames.add(cap.name);
    }
    if (capNames.isEmpty()) return;
    for (var goal : config.goals) {
        if (goal.capabilities == null) continue;
        for (var capRef : goal.capabilities) {
            if (!capNames.contains(capRef)) {
                throw new IllegalStateException(
                    "@AgentGoalDef '" + goal.name + "' on " + classInfo.name()
                    + " references capability '" + capRef + "' not declared in @Discoverable or @AgentCapabilityDef");
            }
        }
    }
}
```

- [ ] **Step 9: Run tests — verify green**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations/deployment -Dtest=EidosAnnotationsProcessorTest`
Expected: all tests pass.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#146): rich capabilities with merge semantics + repeatable templates  Refs #146"
```

---

## Batch 4: Validation + Parity

### Task 5: Build-time validation extensions + orphan warnings + full parity test

**Files:**
- Modify: `annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java`
- Create: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/test/FullParityAgent.java`
- Modify: `annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessorTest.java`

**Interfaces:**
- Consumes: All extraction and construction from Tasks 1-4
- Produces: Build-time validation for NaN, ranges, duplicates, orphan warnings. Full parity test proving annotation == builder descriptor.

- [ ] **Step 1: Write FullParityAgent test agent**

Create `FullParityAgent.java` — exercises every field simultaneously:

```java
package io.casehub.eidos.annotations.deployment.test;

import io.casehub.eidos.annotations.*;
import io.casehub.eidos.api.*;

@Identity(id = "parity-agent", name = "Parity Agent", slot = "analyst",
          vocabulary = "urn:casehub:vocab:svo",
          dispositionVocabulary = "urn:casehub:vocab:conscientiousness",
          provider = "test-provider", modelFamily = "test-model",
          modelVersion = "v1", weightsFingerprint = "sha256:parity",
          jurisdiction = "EU", dataHandlingPolicy = "gdpr",
          briefing = "Full parity test agent", version = "1.0")
@Disposition(socialOrient = "collaborative", ruleFollowing = "strict",
             riskAppetite = "cautious", autonomy = "guided",
             conflictMode = "accommodating", delegation = true,
             dispositionProfile = {
                 @DispositionWeight(value = "collaborative", weight = 0.8),
                 @DispositionWeight(value = "analytical", weight = 0.4)
             },
             axisVocabularies = {
                 @AxisVocabulary(axis = DispositionAxis.CONFLICT_MODE,
                                uri = "urn:casehub:vocab:thomas-kilmann")
             })
@AgentCapabilityDef(name = "cap-a", description = "Capability A",
    qualityHint = 0.9, latencyHintP50Ms = 2000, costHint = "low",
    inputTypes = {"text/plain"}, outputTypes = {"application/json"},
    tags = {"tag1"},
    epistemicDomains = {@EpistemicDomain(value = "domain-a", score = 0.95)},
    excludedDomains = {"domain-x"})
@Discoverable(capabilities = {"cap-b"})
@AgentGoals({
    @AgentGoalDef(name = "goal-1", description = "Primary goal",
                  priority = GoalPriority.PRIMARY, capabilities = {"cap-a"}),
    @AgentGoalDef(name = "goal-2", description = "Secondary goal",
                  priority = GoalPriority.SECONDARY)
})
@AgentConstraints({
    @AgentConstraintDef(name = "constraint-1", description = "Hard constraint",
                        severity = ConstraintSeverity.HARD),
    @AgentConstraintDef(name = "constraint-2", description = "Private soft",
                        severity = ConstraintSeverity.SOFT, visibility = Visibility.PRIVATE)
})
public interface FullParityAgent {}
```

- [ ] **Step 2: Write parity test — annotation descriptor equals builder descriptor**

```java
@Test
void fullParity_annotationEqualsBuilder() {
    var annotated = registry.findById("parity-agent", "test-tenant").orElseThrow();
    var built = AgentDescriptor.builder()
        .agentId("parity-agent").name("Parity Agent").slot("analyst")
        .tenancyId("test-tenant")
        .domainVocabulary("urn:casehub:vocab:svo")
        .dispositionVocabulary("urn:casehub:vocab:conscientiousness")
        .provider("test-provider").modelFamily("test-model")
        .modelVersion("v1").weightsFingerprint("sha256:parity")
        .jurisdiction("EU").dataHandlingPolicy("gdpr")
        .briefing("Full parity test agent").version("1.0")
        .disposition(AgentDisposition.builder()
            .socialOrient("collaborative").ruleFollowing("strict")
            .riskAppetite("cautious").autonomy("guided")
            .conflictMode("accommodating").delegation(true)
            .dispositionProfile(
                new DispositionValue("collaborative", 0.8),
                new DispositionValue("analytical", 0.4))
            .build())
        .axisVocabularies(java.util.Map.of(
            DispositionAxis.CONFLICT_MODE, "urn:casehub:vocab:thomas-kilmann"))
        .capabilities(java.util.List.of(
            AgentCapability.builder().name("cap-a").description("Capability A")
                .qualityHint(0.9).latencyHintP50Ms(2000L).costHint("low")
                .inputTypes(java.util.List.of("text/plain"))
                .outputTypes(java.util.List.of("application/json"))
                .tags(java.util.List.of("tag1"))
                .epistemicDomains(java.util.Map.of("domain-a", 0.95))
                .excludedDomains(java.util.Set.of("domain-x")).build(),
            AgentCapability.builder().name("cap-b").build()))
        .goals(java.util.List.of(
            new AgentGoal("goal-1", "Primary goal", GoalPriority.PRIMARY,
                Visibility.PUBLIC, java.util.List.of("cap-a"), null),
            new AgentGoal("goal-2", "Secondary goal", GoalPriority.SECONDARY,
                Visibility.PUBLIC, java.util.List.of(), null)))
        .constraints(java.util.List.of(
            new AgentConstraint("constraint-1", "Hard constraint",
                Visibility.PUBLIC, ConstraintSeverity.HARD),
            new AgentConstraint("constraint-2", "Private soft",
                Visibility.PRIVATE, ConstraintSeverity.SOFT)))
        .build();

    assertThat(annotated.agentId()).isEqualTo(built.agentId());
    assertThat(annotated.name()).isEqualTo(built.name());
    assertThat(annotated.slot()).isEqualTo(built.slot());
    assertThat(annotated.provider()).isEqualTo(built.provider());
    assertThat(annotated.modelFamily()).isEqualTo(built.modelFamily());
    assertThat(annotated.modelVersion()).isEqualTo(built.modelVersion());
    assertThat(annotated.weightsFingerprint()).isEqualTo(built.weightsFingerprint());
    assertThat(annotated.domainVocabulary()).isEqualTo(built.domainVocabulary());
    assertThat(annotated.dispositionVocabulary()).isEqualTo(built.dispositionVocabulary());
    assertThat(annotated.jurisdiction()).isEqualTo(built.jurisdiction());
    assertThat(annotated.dataHandlingPolicy()).isEqualTo(built.dataHandlingPolicy());
    assertThat(annotated.briefing()).isEqualTo(built.briefing());
    assertThat(annotated.version()).isEqualTo(built.version());
    assertThat(annotated.disposition().delegation()).isEqualTo(built.disposition().delegation());
    assertThat(annotated.disposition().dispositionProfile()).isEqualTo(built.disposition().dispositionProfile());
    assertThat(annotated.axisVocabularies()).isEqualTo(built.axisVocabularies());
    assertThat(annotated.capabilities()).hasSize(built.capabilities().size());
    assertThat(annotated.goals()).isEqualTo(built.goals());
    assertThat(annotated.constraints()).isEqualTo(built.constraints());
}
```

- [ ] **Step 3: Add build-time validation to EidosAnnotationsProcessor**

Add a `validateCapabilityMetadata()` method called from `extractCapabilities()` after extraction:

```java
private void validateCapabilityMetadata(ClassInfo classInfo, AnnotatedAgentConfig.CapabilityConfig[] caps) {
    if (caps == null) return;
    for (var cap : caps) {
        if (cap.qualityHint != -1) {
            if (Double.isNaN(cap.qualityHint) || cap.qualityHint < 0.0 || cap.qualityHint > 1.0) {
                throw new IllegalStateException(
                    "@AgentCapabilityDef '" + cap.name + "' on " + classInfo.name()
                    + ": qualityHint must be 0.0-1.0, got " + cap.qualityHint);
            }
        }
        if (cap.epistemicDomains != null) {
            var domainNames = new HashSet<String>();
            for (var ed : cap.epistemicDomains) {
                if (Double.isNaN(ed.score) || ed.score < 0.0 || ed.score > 1.0) {
                    throw new IllegalStateException(
                        "@EpistemicDomain score for '" + ed.value + "' on " + classInfo.name()
                        + " must be 0.0-1.0, got " + ed.score);
                }
                domainNames.add(ed.value);
            }
            if (cap.excludedDomains != null) {
                for (var exd : cap.excludedDomains) {
                    if (domainNames.contains(exd)) {
                        throw new IllegalStateException(
                            "Domain '" + exd + "' on " + classInfo.name()
                            + " appears in both epistemicDomains and excludedDomains");
                    }
                }
            }
        }
    }
}
```

Add duplicate `@AxisVocabulary.axis()` detection in `extractDisposition()`:

```java
if (config.axisVocabularies != null) {
    var seenAxes = new HashSet<String>();
    for (var avc : config.axisVocabularies) {
        if (!seenAxes.add(avc.axis)) {
            throw new IllegalStateException(
                "Duplicate @AxisVocabulary axis " + avc.axis + " on " + classInfo.name());
        }
    }
}
```

Add `@DispositionWeight.weight()` range validation in `extractDisposition()`:

```java
// After extracting each DispositionWeightConfig:
if (Double.isNaN(dwc.weight) || dwc.weight < 0.0 || dwc.weight > 1.0) {
    throw new IllegalStateException(
        "@DispositionWeight weight for '" + dwc.value + "' on " + classInfo.name()
        + " must be 0.0-1.0, got " + dwc.weight);
}
```

- [ ] **Step 4: Add orphan annotation warnings**

Extend `warnDiscoverableWithoutIdentity()` to also check `@AgentCapabilityDef` and `@AgentTemplateRef`:

```java
private void warnDiscoverableWithoutIdentity(CombinedIndexBuildItem index, Set<String> processedClasses) {
    for (var ann : index.getIndex().getAnnotations(DISCOVERABLE)) {
        if (ann.target().kind() != AnnotationTarget.Kind.CLASS) continue;
        var className = ann.target().asClass().name().toString();
        if (!processedClasses.contains(className)) {
            LOG.warnf("Class %s has @Discoverable but no @Identity — capabilities will not be registered", className);
        }
    }
    for (var dotName : List.of(AGENT_CAPABILITY_DEF, AGENT_CAPABILITIES)) {
        for (var ann : index.getIndex().getAnnotations(dotName)) {
            if (ann.target().kind() != AnnotationTarget.Kind.CLASS) continue;
            var className = ann.target().asClass().name().toString();
            if (!processedClasses.contains(className)) {
                LOG.warnf("Class %s has @AgentCapabilityDef but no @Identity — capabilities will not be registered", className);
            }
        }
    }
    for (var dotName : List.of(AGENT_TEMPLATE_REF, AGENT_TEMPLATES)) {
        for (var ann : index.getIndex().getAnnotations(dotName)) {
            if (ann.target().kind() != AnnotationTarget.Kind.CLASS) continue;
            var className = ann.target().asClass().name().toString();
            if (!processedClasses.contains(className)) {
                LOG.warnf("Class %s has @AgentTemplateRef but no @Identity — templates will not be registered", className);
            }
        }
    }
}
```

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl annotations/deployment -Dtest=EidosAnnotationsProcessorTest`
Expected: all tests pass including parity test.

- [ ] **Step 6: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all modules compile and test green.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add annotations/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#146): build-time validation, orphan warnings, full parity test  Refs #146"
```

---

## References

- [2026-09-02-annotation-parity-design.md](/Users/mdproctor/claude/public/casehub/eidos/specs/issue-146-annotation-parity/2026-09-02-annotation-parity-design.md) — design spec this plan implements
- [decisions.md](/Users/mdproctor/claude/public/casehub/eidos/specs/issue-146-annotation-parity/decisions.md) — D1-D10 design decisions
- [EidosAnnotationsProcessor.java](/Users/mdproctor/claude/casehub/eidos/annotations/deployment/src/main/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessor.java) — current build extension
- [EidosAnnotationsRecorder.java](/Users/mdproctor/claude/casehub/eidos/annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java) — current recorder
- [AnnotatedAgentConfig.java](/Users/mdproctor/claude/casehub/eidos/annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedAgentConfig.java) — current config
- [DispositionDeserializer.java](/Users/mdproctor/claude/casehub/eidos/runtime/src/main/java/io/casehub/eidos/runtime/yaml/DispositionDeserializer.java) — personality type derivation logic to extract
- [AgentCapability.java](/Users/mdproctor/claude/casehub/eidos/api/src/main/java/io/casehub/eidos/api/AgentCapability.java) — 11-field capability record
- [AgentDescriptor.java](/Users/mdproctor/claude/casehub/eidos/api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java) — full descriptor record
- [issue-139 annotations spec](/Users/mdproctor/claude/public/casehub/eidos/specs/issue-139-eidos-annotations/2026-08-18-eidos-annotations-design.md) — predecessor design
- [EidosAnnotationsProcessorTest.java](/Users/mdproctor/claude/casehub/eidos/annotations/deployment/src/test/java/io/casehub/eidos/annotations/deployment/EidosAnnotationsProcessorTest.java) — existing test patterns
- [Supervises.java](/Users/mdproctor/claude/casehub/eidos/org-annotations/runtime/src/main/java/io/casehub/eidos/org/annotations/Supervises.java) — @Repeatable precedent
- [GitHub #146](https://github.com/casehubio/eidos/issues/146)
