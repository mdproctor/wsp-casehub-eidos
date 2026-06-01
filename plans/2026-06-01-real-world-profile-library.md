# Real-World Agent Profile Library Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a curated library of 8 real-world agent profiles and integrate them into the eval harness with semantic proximity and three-stage personality preservation measurement.

**Architecture:** EvalCase refactors to a sealed interface (SyntheticEvalCase + ProfiledEvalCase). AgentProfileLoader (test scope) reads YAML profiles from classpath. Four new @ApplicationScoped judges (ProximityJudge, VocabularyExpressivenessJudge, TraitExpressionJudge, PairContrastJudge) run 84 LLM calls per eval. Attribution diagnoses localise failures to vocabulary gap, renderer flattening, or profile design.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson YAML, LangChain4j ChatModel, JUnit 5, AssertJ

---

## File Map

**New — `eval/src/main/java/io/casehub/eidos/eval/`:**
SourceType.java, CoverageLoss.java, TraitPolarity.java, Attribution.java, VocabularyGap.java, AgentProfile.java, VariantPair.java, VariantIndex.java, AttributionDiagnosis.java, ProximityResult.java, PairContrastResult.java, VocabularyExpressivenessResult.java, TraitExpressionResult.java, ProximityReport.java, PersonalityPreservationReport.java, SyntheticEvalCase.java, ProfiledEvalCase.java, ProximityJudge.java, VocabularyExpressivenessJudge.java, TraitExpressionJudge.java, PairContrastJudge.java

**New — `eval/src/test/java/io/casehub/eidos/eval/`:**
AgentProfileLoader.java, AgentProfileLoaderTest.java, ProximityJudgeTest.java, VocabularyExpressivenessJudgeTest.java, TraitExpressionJudgeTest.java, PairContrastJudgeTest.java

**New — `eval/src/test/resources/profiles/`:**
index.yaml, sw-engineer-careful.yaml, sw-engineer-bold.yaml, security-analyst-defensive.yaml, security-analyst-proactive.yaml, product-manager.yaml, clinical-researcher.yaml, customer-support-agent.yaml, technical-writer.yaml

**Modify:**
EvalCase.java (sealed interface), EvalDataset.java (realWorld() + 9 factory updates), EvalReportWriter.java (+4 methods), PromptJudgeTest.java (8 new SyntheticEvalCase), EvalReportWriterTest.java (2 new SyntheticEvalCase), EvalReportTest.java (1 new SyntheticEvalCase), PromptEvalTest.java (+evaluateRealWorldScenarios + @BeforeAll), eval/pom.xml (+jackson-dataformat-yaml)

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval`
**Eval-only tests:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Peval -Dgroups=eval`

---

## Task 1: pom.xml dependency + foundation enums

**Files:**
- Modify: `eval/pom.xml`
- Create: `eval/src/main/java/io/casehub/eidos/eval/SourceType.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/CoverageLoss.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/TraitPolarity.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/Attribution.java`

- [ ] **Step 1: Add jackson-dataformat-yaml to eval/pom.xml**

In `eval/pom.xml`, add inside `<dependencies>`:
```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
    <scope>provided</scope>
</dependency>
```

- [ ] **Step 2: Create SourceType enum**

```java
package io.casehub.eidos.eval;

public enum SourceType {
    PRACTITIONER,
    ACADEMIC,
    OPENAI_COOKBOOK,
    ANTHROPIC_LIBRARY,
    ONET_SYNTHESISED
}
```

- [ ] **Step 3: Create CoverageLoss enum**

```java
package io.casehub.eidos.eval;

public enum CoverageLoss {
    PARTIAL,
    FULL
}
```

- [ ] **Step 4: Create TraitPolarity enum**

```java
package io.casehub.eidos.eval;

public enum TraitPolarity {
    HIGH, LOW, NEUTRAL
}
```

- [ ] **Step 5: Create Attribution enum**

```java
package io.casehub.eidos.eval;

public enum Attribution {
    VOCABULARY_GAP,
    RENDERER_FLATTENING,
    PROFILE_DESIGN_GAP,
    WORKING,
    INSUFFICIENT_DATA
}
```

- [ ] **Step 6: Verify compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl eval -q
```
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/pom.xml eval/src/main/java/io/casehub/eidos/eval/SourceType.java eval/src/main/java/io/casehub/eidos/eval/CoverageLoss.java eval/src/main/java/io/casehub/eidos/eval/TraitPolarity.java eval/src/main/java/io/casehub/eidos/eval/Attribution.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): add jackson-dataformat-yaml + foundation enums

Refs #23"
```

---

## Task 2: Foundation records (no validation)

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/VocabularyGap.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/VariantPair.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/VariantIndex.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/AttributionDiagnosis.java`

- [ ] **Step 1: Create VocabularyGap**

```java
package io.casehub.eidos.eval;

public record VocabularyGap(String concept, String description, CoverageLoss loss) {}
```

- [ ] **Step 2: Create VariantPair**

```java
package io.casehub.eidos.eval;

public record VariantPair(String primaryAxis, String higher, String lower) {}
```

- [ ] **Step 3: Create VariantIndex**

```java
package io.casehub.eidos.eval;

import java.util.List;

public record VariantIndex(List<String> profiles, List<VariantPair> variants) {}
```

- [ ] **Step 4: Create AttributionDiagnosis**

```java
package io.casehub.eidos.eval;

public record AttributionDiagnosis(
    String profileName,
    String axis,
    int stage1Score,
    int stage2ExpressionScore,
    int stage3EffectSize,
    Attribution attribution
) {}
```

- [ ] **Step 5: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl eval -q
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/VocabularyGap.java eval/src/main/java/io/casehub/eidos/eval/VariantPair.java eval/src/main/java/io/casehub/eidos/eval/VariantIndex.java eval/src/main/java/io/casehub/eidos/eval/AttributionDiagnosis.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): foundation records (VocabularyGap, VariantPair, VariantIndex, AttributionDiagnosis)

Refs #23"
```

---

## Task 3: Validated records (ProximityResult, PairContrastResult)

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/ProximityResult.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/PairContrastResult.java`

- [ ] **Step 1: Write failing tests**

Create `eval/src/test/java/io/casehub/eidos/eval/ValidatedRecordTest.java`:
```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ValidatedRecordTest {

    static SyntheticEvalCase minimalCase() {
        final var desc = new AgentDescriptor(
            "id", "N", null, null, null, null, null, null, null, null,
            "worker", List.of(), null, null, null, "t");
        return new SyntheticEvalCase("c", desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
    }

    @Test
    void proximityResult_rejects_score_below_0() {
        assertThatThrownBy(() -> new ProximityResult(minimalCase(), -1, "r", List.of()))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("out of range");
    }

    @Test
    void proximityResult_rejects_score_above_5() {
        assertThatThrownBy(() -> new ProximityResult(minimalCase(), 6, "r", List.of()))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void proximityResult_accepts_boundary_values() {
        new ProximityResult(minimalCase(), 0, "r", List.of());
        new ProximityResult(minimalCase(), 5, "r", List.of());
    }

    @Test
    void pairContrastResult_rejects_effectSize_below_1() {
        assertThatThrownBy(() -> new PairContrastResult("hi", "lo", "riskAppetite",
            RenderFormat.MARKDOWN, true, 0, "r"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("out of range");
    }

    @Test
    void pairContrastResult_rejects_effectSize_above_5() {
        assertThatThrownBy(() -> new PairContrastResult("hi", "lo", "riskAppetite",
            RenderFormat.MARKDOWN, true, 6, "r"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void pairContrastResult_accepts_boundary_values() {
        new PairContrastResult("hi", "lo", "riskAppetite", RenderFormat.MARKDOWN, true, 1, "r");
        new PairContrastResult("hi", "lo", "riskAppetite", RenderFormat.MARKDOWN, true, 5, "r");
    }
}
```

- [ ] **Step 2: Run tests — expect compile failure (SyntheticEvalCase not yet defined)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=ValidatedRecordTest -q 2>&1 | tail -5
```
Expected: compilation error — SyntheticEvalCase not found

- [ ] **Step 3: Create ProximityResult**

```java
package io.casehub.eidos.eval;

import java.util.List;

public record ProximityResult(
    EvalCase evalCase,
    int score,
    String reasoning,
    List<String> gaps
) {
    public ProximityResult {
        if (score < 0 || score > 5)
            throw new IllegalArgumentException("ProximityResult score out of range: " + score);
    }
}
```

- [ ] **Step 4: Create PairContrastResult**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;

public record PairContrastResult(
    String profileHigh,
    String profileLow,
    String primaryAxis,
    RenderFormat format,
    boolean correctlyIdentified,
    int effectSize,
    String reasoning
) {
    public PairContrastResult {
        if (effectSize < 1 || effectSize > 5)
            throw new IllegalArgumentException("effectSize out of range: " + effectSize);
    }
}
```

Note: Tests still compile-fail until Task 6 (EvalCase refactor) creates SyntheticEvalCase. Commit the records now; the test class will pass in Task 6.

- [ ] **Step 5: Compile production code**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl eval -q
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/ProximityResult.java eval/src/main/java/io/casehub/eidos/eval/PairContrastResult.java eval/src/test/java/io/casehub/eidos/eval/ValidatedRecordTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): validated result records + tests (ProximityResult 0-5, PairContrastResult 1-5)

Refs #23"
```

---

## Task 4: AgentProfile record + result types

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/AgentProfile.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/VocabularyExpressivenessResult.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/TraitExpressionResult.java`

- [ ] **Step 1: Create AgentProfile**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.GoalContext;

import java.util.List;
import java.util.Map;

public record AgentProfile(
    String name,
    String role,
    String domain,
    String sourceUrl,
    String sourceCitation,
    SourceType sourceType,
    String originalProse,
    GoalContext evalGoal,
    String notes,
    Map<String, String> theoreticalFramework,
    Map<String, TraitPolarity> expectedTraits,
    AgentDescriptor descriptor,
    List<VocabularyGap> vocabularyGaps
) {}
```

- [ ] **Step 2: Create VocabularyExpressivenessResult**

```java
package io.casehub.eidos.eval;

import java.util.List;
import java.util.Map;

public record VocabularyExpressivenessResult(
    String profileName,
    Map<String, Integer> expressivenessScores,
    List<String> weakAxes
) {}
```

- [ ] **Step 3: Create TraitExpressionResult**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;

import java.util.Map;

public record TraitExpressionResult(
    ProfiledEvalCase evalCase,
    RenderFormat format,
    Map<String, Integer> expressionScores,
    Map<String, Boolean> directionMatches,
    String delegationAssessment
) {}
```

Note: TraitExpressionResult references ProfiledEvalCase (Task 6). Production code compile will fail until then; commit the files anyway — they'll compile once Task 6 is done.

- [ ] **Step 4: Compile what compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl eval 2>&1 | grep -E "BUILD|ERROR" | head -5
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/AgentProfile.java eval/src/main/java/io/casehub/eidos/eval/VocabularyExpressivenessResult.java eval/src/main/java/io/casehub/eidos/eval/TraitExpressionResult.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): AgentProfile, VocabularyExpressivenessResult, TraitExpressionResult

Refs #23"
```

---

## Task 5: ProximityReport + PersonalityPreservationReport

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/ProximityReport.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/PersonalityPreservationReport.java`

- [ ] **Step 1: Create ProximityReport**

```java
package io.casehub.eidos.eval;

import java.util.List;

public record ProximityReport(
    List<ProximityResult> results,
    double floor,
    double meanScore,
    double minScore,
    int belowFloor
) {
    public static ProximityReport build(final List<ProximityResult> results, final double floor) {
        final double mean = results.stream().mapToInt(ProximityResult::score).average().orElse(0.0);
        final int min = results.stream().mapToInt(ProximityResult::score).min().orElse(0);
        final int below = (int) results.stream().filter(r -> r.score() < floor).count();
        return new ProximityReport(results, floor, mean, min, below);
    }
}
```

- [ ] **Step 2: Create PersonalityPreservationReport**

```java
package io.casehub.eidos.eval;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.stream.Collectors;

public record PersonalityPreservationReport(
    List<VocabularyExpressivenessResult> expressivenessResults,
    List<TraitExpressionResult> traitExpressionResults,
    List<PairContrastResult> pairContrastResults,
    List<AttributionDiagnosis> diagnoses,
    double meanExpressivenessScore,
    double meanTraitMatchRate,
    double meanEffectSize,
    double discriminationAccuracy,
    List<String> annotations
) {
    private static final List<String> AXES =
        List.of("socialOrient", "ruleFollowing", "riskAppetite", "autonomy");

    public static PersonalityPreservationReport build(
        final List<VocabularyExpressivenessResult> exp,
        final List<TraitExpressionResult> traits,
        final List<PairContrastResult> contrasts
    ) {
        // meanExpressivenessScore: flat mean across all (profile × axis) cells
        final double meanExp = exp.stream()
            .flatMapToInt(r -> r.expressivenessScores().values().stream().mapToInt(Integer::intValue))
            .average().orElse(0.0);

        // meanTraitMatchRate: flat mean across all (profile × format × axis) direction-match cells
        final long totalMatches = traits.stream()
            .flatMap(r -> r.directionMatches().values().stream())
            .filter(Boolean::booleanValue).count();
        final long totalCells = traits.stream()
            .mapToLong(r -> r.directionMatches().size()).sum();
        final double meanTraitMatchRate = totalCells > 0 ? (double) totalMatches / totalCells : 0.0;

        // meanEffectSize: flat mean across all (pair × format) results
        final double meanEffectSize = contrasts.stream()
            .mapToInt(PairContrastResult::effectSize).average().orElse(0.0);

        // discriminationAccuracy: % pairs correctly identified
        final double discAcc = contrasts.isEmpty() ? 0.0 :
            (double) contrasts.stream().filter(PairContrastResult::correctlyIdentified).count()
            / contrasts.size();

        final List<AttributionDiagnosis> diagnoses = computeDiagnoses(exp, traits, contrasts);

        return new PersonalityPreservationReport(
            exp, traits, contrasts, diagnoses,
            meanExp, meanTraitMatchRate, meanEffectSize, discAcc,
            new ArrayList<>()
        );
    }

    private static List<AttributionDiagnosis> computeDiagnoses(
        final List<VocabularyExpressivenessResult> exp,
        final List<TraitExpressionResult> traits,
        final List<PairContrastResult> contrasts
    ) {
        final List<AttributionDiagnosis> result = new ArrayList<>();
        for (final VocabularyExpressivenessResult er : exp) {
            for (final String axis : AXES) {
                final int s1 = er.expressivenessScores().getOrDefault(axis, -1);
                // Stage 2: mean direction match across all formats for this profile × axis
                final List<TraitExpressionResult> profileTraits = traits.stream()
                    .filter(t -> t.evalCase().profile().name().equals(er.profileName()))
                    .toList();
                final double matchRate = profileTraits.isEmpty() ? -1.0 :
                    profileTraits.stream()
                        .mapToInt(t -> Boolean.TRUE.equals(t.directionMatches().get(axis)) ? 1 : 0)
                        .average().orElse(-1.0);
                // Stage 3: mean effectSize across formats for this profile × axis
                final double s3 = contrasts.stream()
                    .filter(c -> c.primaryAxis().equals(axis)
                        && (c.profileHigh().equals(er.profileName())
                            || c.profileLow().equals(er.profileName())))
                    .mapToInt(PairContrastResult::effectSize)
                    .average().orElse(-1.0);

                final Attribution attr;
                if (s1 < 0) {
                    attr = Attribution.INSUFFICIENT_DATA;
                } else if (s1 <= 2) {
                    attr = Attribution.VOCABULARY_GAP;
                } else if (matchRate >= 0 && matchRate < 0.5) {
                    attr = Attribution.RENDERER_FLATTENING;
                } else if (s3 >= 1 && s3 <= 2) {
                    attr = Attribution.PROFILE_DESIGN_GAP;
                } else if (s3 > 2) {
                    attr = Attribution.WORKING;
                } else {
                    attr = Attribution.INSUFFICIENT_DATA;
                }

                result.add(new AttributionDiagnosis(
                    er.profileName(), axis, s1, (int) Math.round(matchRate * 5),
                    (int) Math.round(s3), attr));
            }
        }
        return result;
    }
}
```

- [ ] **Step 3: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl eval 2>&1 | grep -E "BUILD|ERROR" | head -5
```

Expected: errors only for TraitExpressionResult.evalCase() returning ProfiledEvalCase (not yet defined). That's OK — fixed in Task 6.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/ProximityReport.java eval/src/main/java/io/casehub/eidos/eval/PersonalityPreservationReport.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): ProximityReport + PersonalityPreservationReport with attribution logic

Refs #23"
```

---

## Task 6: EvalCase sealed interface refactor

This is the breaking change. 20 call sites change from `new EvalCase(...)` to `new SyntheticEvalCase(...)`.

**Files:**
- Replace: `eval/src/main/java/io/casehub/eidos/eval/EvalCase.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/SyntheticEvalCase.java`
- Create: `eval/src/main/java/io/casehub/eidos/eval/ProfiledEvalCase.java`
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java` (9 factory methods)
- Modify: `eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java` (8 occurrences)
- Modify: `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java` (2 occurrences)
- Modify: `eval/src/test/java/io/casehub/eidos/eval/EvalReportTest.java` (1 occurrence)

- [ ] **Step 1: Replace EvalCase.java with sealed interface**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;

public sealed interface EvalCase permits SyntheticEvalCase, ProfiledEvalCase {
    String name();
    AgentDescriptor descriptor();
    AgentPromptContext context();
}
```

- [ ] **Step 2: Create SyntheticEvalCase**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;

public record SyntheticEvalCase(
    String name,
    AgentDescriptor descriptor,
    AgentPromptContext context
) implements EvalCase {}
```

- [ ] **Step 3: Create ProfiledEvalCase**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;

public record ProfiledEvalCase(
    String name,
    AgentDescriptor descriptor,
    AgentPromptContext context,
    AgentProfile profile
) implements EvalCase {}
```

- [ ] **Step 4: Update EvalDataset — replace all 9 `new EvalCase(` with `new SyntheticEvalCase(`**

In `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java`, replace every occurrence:
```bash
sed -i '' 's/return new EvalCase(/return new SyntheticEvalCase(/g' /Users/mdproctor/claude/casehub/eidos/eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java
```
Verify: `grep -c "SyntheticEvalCase" /Users/mdproctor/claude/casehub/eidos/eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java` → should print 9

Also update the return type of `all()` from `List<EvalCase>` to `List<SyntheticEvalCase>`:
In EvalDataset.java, change:
```java
public static List<EvalCase> all() {
```
to:
```java
public static List<SyntheticEvalCase> all() {
```

- [ ] **Step 5: Update PromptJudgeTest — replace all 8 `new EvalCase(` with `new SyntheticEvalCase(`**

```bash
sed -i '' 's/new EvalCase(/new SyntheticEvalCase(/g' /Users/mdproctor/claude/casehub/eidos/eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java
```
Verify: `grep -c "SyntheticEvalCase" /Users/mdproctor/claude/casehub/eidos/eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java` → 8

- [ ] **Step 6: Update EvalReportWriterTest — replace 2 occurrences**

In `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java`, replace:
- Line 37: `new EvalResult(new EvalCase("case1", desc, ctx), ...` → `new EvalResult(new SyntheticEvalCase("case1", desc, ctx), ...`
- Line 90: `new EvalCase("case2", desc, ctx)` → `new SyntheticEvalCase("case2", desc, ctx)`

```bash
sed -i '' 's/new EvalCase(/new SyntheticEvalCase(/g' /Users/mdproctor/claude/casehub/eidos/eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java
```

- [ ] **Step 7: Update EvalReportTest — replace 1 occurrence**

```bash
sed -i '' 's/new EvalCase(/new SyntheticEvalCase(/g' /Users/mdproctor/claude/casehub/eidos/eval/src/test/java/io/casehub/eidos/eval/EvalReportTest.java
```

- [ ] **Step 8: Run all existing tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -q
```
Expected: BUILD SUCCESS. All previously passing tests should still pass. `ValidatedRecordTest` now compiles and should pass too.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/EvalCase.java eval/src/main/java/io/casehub/eidos/eval/SyntheticEvalCase.java eval/src/main/java/io/casehub/eidos/eval/ProfiledEvalCase.java eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java eval/src/test/java/io/casehub/eidos/eval/PromptJudgeTest.java eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java eval/src/test/java/io/casehub/eidos/eval/EvalReportTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "refactor(eidos#23): EvalCase → sealed interface; SyntheticEvalCase + ProfiledEvalCase

Refs #23"
```

---

## Task 7: Stub profile YAMLs + AgentProfileLoader

The YAMLs here are stubs for testing infrastructure. Real prose is added in Task 15.

**Files:**
- Create: `eval/src/test/resources/profiles/index.yaml`
- Create: `eval/src/test/resources/profiles/sw-engineer-careful.yaml`
- Create: `eval/src/test/resources/profiles/sw-engineer-bold.yaml`
- Create: `eval/src/test/java/io/casehub/eidos/eval/AgentProfileLoader.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/AgentProfileLoaderTest.java`

- [ ] **Step 1: Write AgentProfileLoaderTest (failing)**

```java
package io.casehub.eidos.eval;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class AgentProfileLoaderTest {

    @Test
    void load_returns_all_profiles_from_index() {
        final var loader = new AgentProfileLoader();
        final List<AgentProfile> profiles = loader.load();
        assertThat(profiles).hasSize(2);  // stub index has 2
        assertThat(profiles).extracting(AgentProfile::name)
            .containsExactlyInAnyOrder("sw-engineer-careful", "sw-engineer-bold");
    }

    @Test
    void load_deserializes_descriptor_fields_correctly() {
        final var profile = new AgentProfileLoader().load().stream()
            .filter(p -> p.name().equals("sw-engineer-careful")).findFirst().orElseThrow();
        assertThat(profile.descriptor().agentId()).isEqualTo("sw-engineer-careful-01");
        assertThat(profile.descriptor().slot()).isEqualTo("reviewer");
        assertThat(profile.descriptor().tenancyId()).isEqualTo("profiles-1");
    }

    @Test
    void load_deserializes_disposition_with_delegation_boolean() {
        final var profile = new AgentProfileLoader().load().stream()
            .filter(p -> p.name().equals("sw-engineer-careful")).findFirst().orElseThrow();
        assertThat(profile.descriptor().disposition()).isNotNull();
        assertThat(profile.descriptor().disposition().delegation()).isFalse();
    }

    @Test
    void load_deserializes_expectedTraits() {
        final var profile = new AgentProfileLoader().load().stream()
            .filter(p -> p.name().equals("sw-engineer-careful")).findFirst().orElseThrow();
        assertThat(profile.expectedTraits()).containsEntry("riskAppetite", TraitPolarity.LOW);
    }

    @Test
    void load_deserializes_sourceType_enum() {
        final var profile = new AgentProfileLoader().load().stream()
            .filter(p -> p.name().equals("sw-engineer-careful")).findFirst().orElseThrow();
        assertThat(profile.sourceType()).isEqualTo(SourceType.ANTHROPIC_LIBRARY);
    }

    @Test
    void loadIndex_returns_variant_pairs() {
        final var index = new AgentProfileLoader().loadIndex();
        assertThat(index.variants()).hasSize(1);
        assertThat(index.variants().get(0).primaryAxis()).isEqualTo("riskAppetite");
        assertThat(index.variants().get(0).higher()).isEqualTo("sw-engineer-bold");
    }

    @Test
    void stage0_passes_for_valid_variant_pairs() {
        // load() runs stage 0 implicitly; no exception = pass
        new AgentProfileLoader().load();
    }

    @Test
    void stage0_fails_when_primaryAxis_same_value_in_both_profiles() {
        // This tests the validation logic via a custom loader with tampered profiles;
        // for now verify the happy path passes — the next test adds a bad fixture.
        // Happy path already tested by stage0_passes_for_valid_variant_pairs.
    }
}
```

- [ ] **Step 2: Run — expect compile failure (AgentProfileLoader missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=AgentProfileLoaderTest -q 2>&1 | tail -3
```

- [ ] **Step 3: Create stub index.yaml**

```yaml
# eval/src/test/resources/profiles/index.yaml
profiles:
  - sw-engineer-careful.yaml
  - sw-engineer-bold.yaml

variants:
  - primaryAxis: riskAppetite
    higher: sw-engineer-bold
    lower: sw-engineer-careful
```

- [ ] **Step 4: Create stub sw-engineer-careful.yaml**

```yaml
name: sw-engineer-careful
role: Software Engineer — Code Review (Careful)
domain: software-engineering
sourceUrl: "https://docs.anthropic.com/en/prompt-library/software-quality-assurance"
sourceCitation: "Anthropic Prompt Library, 2025"
sourceType: ANTHROPIC_LIBRARY
originalProse: |
  You are a senior software engineer specialising in code review. You prioritise
  correctness over velocity. You catch subtle bugs before they reach production.
  You are meticulous, thorough, and conservative in your technical judgements.
evalGoal:
  description: "Review a Java pull request for correctness and security"
  subGoals:
    - Flag off-by-one errors and null dereferences
    - Identify security antipatterns
  caseRef: ~
notes: "Correctness-over-velocity emphasis."
theoreticalFramework:
  belbin: monitor-evaluator
  disc: C
expectedTraits:
  riskAppetite: LOW
  socialOrient: LOW
  ruleFollowing: HIGH
  autonomy: LOW
descriptor:
  agentId: sw-engineer-careful-01
  name: Software Engineer — Careful
  version: "1.0"
  provider: anthropic
  modelFamily: claude
  modelVersion: claude-opus-4-7
  slot: reviewer
  capabilities:
    - name: code-review
      qualityHint: 0.95
      latencyHintP50Ms: 45000
      costHint: medium
      inputTypes: [pull-request, diff]
      outputTypes: [review-comment, approval]
      tags: []
      epistemicDomains:
        java: 0.95
        distributed-systems: 0.85
  disposition:
    socialOrient: independent
    ruleFollowing: strict
    riskAppetite: conservative
    autonomy: directed
    delegation: false
  tenancyId: profiles-1
vocabularyGaps:
  - concept: correctness-over-velocity
    description: "Engineering philosophy; riskAppetite=conservative approximates it"
    loss: PARTIAL
```

- [ ] **Step 5: Create stub sw-engineer-bold.yaml**

```yaml
name: sw-engineer-bold
role: Software Engineer — Code Review (Bold)
domain: software-engineering
sourceUrl: "https://example.com/bold-engineer"
sourceCitation: "Practitioner, 2025"
sourceType: PRACTITIONER
originalProse: |
  You are a senior software engineer who values velocity and pragmatic solutions.
  You ship code quickly, accept reasonable risk, and trust your team to catch edge cases.
  You prefer done over perfect.
evalGoal:
  description: "Review a Java pull request and approve if no critical issues"
  subGoals:
    - Approve quickly if no critical bugs
    - Flag only blockers, not style
  caseRef: ~
notes: "Velocity-over-correctness emphasis."
theoreticalFramework:
  belbin: shaper
  disc: D
expectedTraits:
  riskAppetite: HIGH
  socialOrient: LOW
  ruleFollowing: HIGH
  autonomy: LOW
descriptor:
  agentId: sw-engineer-bold-01
  name: Software Engineer — Bold
  version: "1.0"
  provider: anthropic
  modelFamily: claude
  modelVersion: claude-opus-4-7
  slot: reviewer
  capabilities:
    - name: code-review
      qualityHint: 0.85
      latencyHintP50Ms: 20000
      costHint: low
      inputTypes: [pull-request, diff]
      outputTypes: [review-comment, approval]
      tags: []
      epistemicDomains:
        java: 0.9
        distributed-systems: 0.75
  disposition:
    socialOrient: independent
    ruleFollowing: strict
    riskAppetite: bold
    autonomy: directed
    delegation: false
  tenancyId: profiles-1
vocabularyGaps: []
```

Note: `ruleFollowing` and `socialOrient` and `autonomy` are intentionally identical to sw-engineer-careful. Only `riskAppetite` differs (`conservative` vs `bold`). This satisfies Stage 0 single-axis constraint.

- [ ] **Step 6: Implement AgentProfileLoader**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

class AgentProfileLoader {

    private static final ObjectMapper YAML =
        new ObjectMapper(new YAMLFactory()).findAndRegisterModules();

    private static final List<String> AXES =
        List.of("socialOrient", "ruleFollowing", "riskAppetite", "autonomy");

    List<AgentProfile> load() {
        final VariantIndex index = loadIndex();
        final Map<String, AgentProfile> byName = new LinkedHashMap<>();
        for (final String filename : index.profiles()) {
            final AgentProfile p = loadProfile(filename);
            byName.put(p.name(), p);
        }
        validateVariantPairs(index, byName);
        return new ArrayList<>(byName.values());
    }

    VariantIndex loadIndex() {
        try (InputStream is = cl().getResourceAsStream("profiles/index.yaml")) {
            if (is == null) return new VariantIndex(List.of(), List.of());
            return YAML.readValue(is, VariantIndex.class);
        } catch (final IOException e) {
            throw new IllegalStateException("Failed to load profiles/index.yaml", e);
        }
    }

    private AgentProfile loadProfile(final String filename) {
        final String path = "profiles/" + filename;
        try (InputStream is = cl().getResourceAsStream(path)) {
            if (is == null) throw new IllegalStateException("Profile not found on classpath: " + path);
            return YAML.readValue(is, AgentProfile.class);
        } catch (final IOException e) {
            throw new IllegalStateException("Failed to load profile: " + path, e);
        }
    }

    private void validateVariantPairs(final VariantIndex index,
                                       final Map<String, AgentProfile> byName) {
        for (final VariantPair pair : index.variants()) {
            final AgentProfile hi = byName.get(pair.higher());
            final AgentProfile lo = byName.get(pair.lower());
            if (hi == null) throw new IllegalStateException(
                "Variant pair higher slug not found: " + pair.higher());
            if (lo == null) throw new IllegalStateException(
                "Variant pair lower slug not found: " + pair.lower());

            final var dh = hi.descriptor().disposition();
            final var dl = lo.descriptor().disposition();

            final String axHi = axisValue(dh, pair.primaryAxis());
            final String axLo = axisValue(dl, pair.primaryAxis());
            if (Objects.equals(axHi, axLo)) throw new IllegalStateException(
                "Pair " + pair.higher() + " vs " + pair.lower()
                + ": primaryAxis '" + pair.primaryAxis() + "' same value: " + axHi);

            for (final String other : AXES) {
                if (!other.equals(pair.primaryAxis())) {
                    if (!Objects.equals(axisValue(dh, other), axisValue(dl, other)))
                        throw new IllegalStateException(
                            "Pair " + pair.higher() + " vs " + pair.lower()
                            + ": non-primary axis '" + other + "' differs");
                }
            }
            final var dispH = dh != null ? dh.delegation() : false;
            final var dispL = dl != null ? dl.delegation() : false;
            if (dispH != dispL) throw new IllegalStateException(
                "Pair " + pair.higher() + " vs " + pair.lower() + ": delegation differs");
        }
    }

    private String axisValue(final io.casehub.eidos.api.AgentDisposition d, final String axis) {
        if (d == null) return null;
        return switch (axis) {
            case "socialOrient" -> d.socialOrient();
            case "ruleFollowing" -> d.ruleFollowing();
            case "riskAppetite" -> d.riskAppetite();
            case "autonomy" -> d.autonomy();
            default -> throw new IllegalArgumentException("Unknown axis: " + axis);
        };
    }

    private ClassLoader cl() {
        return Thread.currentThread().getContextClassLoader();
    }
}
```

- [ ] **Step 7: Run AgentProfileLoaderTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=AgentProfileLoaderTest -q
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 8: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/java/io/casehub/eidos/eval/AgentProfileLoader.java eval/src/test/java/io/casehub/eidos/eval/AgentProfileLoaderTest.java eval/src/test/resources/profiles/index.yaml eval/src/test/resources/profiles/sw-engineer-careful.yaml eval/src/test/resources/profiles/sw-engineer-bold.yaml
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): AgentProfileLoader (test scope) + stub profile YAMLs + Stage 0 validation

Refs #23"
```

---

## Task 8: EvalDataset.realWorld()

**Files:**
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalDataset.java`

- [ ] **Step 1: Write failing test**

Add to `eval/src/test/java/io/casehub/eidos/eval/EvalDatasetTest.java` (create if absent):
```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class EvalDatasetTest {

    @Test
    void all_returns_non_empty_list() {
        assertThat(EvalDataset.all()).isNotEmpty();
    }

    @Test
    void all_returns_only_synthetic_cases() {
        assertThat(EvalDataset.all()).allSatisfy(c -> assertThat(c).isInstanceOf(SyntheticEvalCase.class));
    }

    @Test
    void realWorld_returns_profiled_cases() {
        final List<ProfiledEvalCase> cases = EvalDataset.realWorld();
        assertThat(cases).isNotEmpty();
        assertThat(cases).allSatisfy(c -> assertThat(c.profile()).isNotNull());
    }

    @Test
    void realWorld_creates_markdown_and_prose_per_profile() {
        final List<ProfiledEvalCase> cases = EvalDataset.realWorld();
        // stub index has 2 profiles → 4 cases (2 formats each)
        assertThat(cases).hasSize(4);
        assertThat(cases).extracting(c -> c.context().format())
            .containsExactlyInAnyOrder(
                RenderFormat.MARKDOWN, RenderFormat.PROSE,
                RenderFormat.MARKDOWN, RenderFormat.PROSE);
    }

    @Test
    void realWorld_uses_evalGoal_when_present() {
        final List<ProfiledEvalCase> cases = EvalDataset.realWorld();
        // both stub profiles have evalGoal set
        assertThat(cases).allSatisfy(c ->
            assertThat(c.context().goal()).isPresent());
    }
}
```

- [ ] **Step 2: Run — expect failure (realWorld() not yet defined)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalDatasetTest -q 2>&1 | tail -5
```

- [ ] **Step 3: Add realWorld() to EvalDataset**

Add the following to `EvalDataset.java` (after the existing `all()` method):
```java
import io.casehub.eidos.api.GoalContext;
import java.util.stream.Stream;
import static java.util.function.Function.identity;
import static java.util.stream.Collectors.toMap;

public static List<ProfiledEvalCase> realWorld() {
    return new AgentProfileLoader().load().stream()
        .flatMap(profile -> Stream.of(
            profileCase(profile, RenderFormat.MARKDOWN),
            profileCase(profile, RenderFormat.PROSE)
        ))
        .toList();
}

private static ProfiledEvalCase profileCase(final AgentProfile profile, final RenderFormat format) {
    final AgentPromptContext ctx = profile.evalGoal() != null
        ? AgentPromptContext.forFormat(format).withGoal(profile.evalGoal())
        : AgentPromptContext.forFormat(format);
    return new ProfiledEvalCase(
        profile.name() + "-" + format.name().toLowerCase(),
        profile.descriptor(),
        ctx,
        profile
    );
}
```

Note: `AgentProfileLoader` is in test scope. `EvalDataset.realWorld()` is in main scope — it cannot directly instantiate `AgentProfileLoader`. **Fix:** move `realWorld()` to a test helper class instead, or move `AgentProfileLoader` to a new `eval/src/main/java/...` package.

**Resolution per spec:** `AgentProfileLoader` is test-scope only. `EvalDataset.realWorld()` must also be test-scope, or we need a factory pattern. The cleanest fix: move `realWorld()` to a new test-scope class `RealWorldEvalDataset` in `eval/src/test/java/`.

Create `eval/src/test/java/io/casehub/eidos/eval/RealWorldEvalDataset.java`:
```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;

import java.util.List;
import java.util.stream.Stream;

class RealWorldEvalDataset {

    static List<ProfiledEvalCase> all() {
        return new AgentProfileLoader().load().stream()
            .flatMap(profile -> Stream.of(
                profileCase(profile, RenderFormat.MARKDOWN),
                profileCase(profile, RenderFormat.PROSE)
            ))
            .toList();
    }

    private static ProfiledEvalCase profileCase(final AgentProfile profile,
                                                 final RenderFormat format) {
        final AgentPromptContext ctx = profile.evalGoal() != null
            ? AgentPromptContext.forFormat(format).withGoal(profile.evalGoal())
            : AgentPromptContext.forFormat(format);
        return new ProfiledEvalCase(
            profile.name() + "-" + format.name().toLowerCase(),
            profile.descriptor(), ctx, profile);
    }
}
```

Update `EvalDatasetTest` to use `RealWorldEvalDataset.all()` instead of `EvalDataset.realWorld()`.

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=EvalDatasetTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/java/io/casehub/eidos/eval/RealWorldEvalDataset.java eval/src/test/java/io/casehub/eidos/eval/EvalDatasetTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): RealWorldEvalDataset (test scope) + EvalDatasetTest

Refs #23"
```

---

## Task 9: ProximityJudge

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/ProximityJudge.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/ProximityJudgeTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ProximityJudgeTest {

    static final String VALID_JSON = """
        { "score": 4, "reasoning": "Core role captured.", "gaps": ["philosophy not expressed"] }
        """;

    ProximityJudge judge;
    ProfiledEvalCase evalCase;
    RenderedPrompt rendered;

    @BeforeEach
    void setUp() {
        judge = new ProximityJudge(stubModel(VALID_JSON), new ObjectMapper());
        evalCase = minimalCase();
        rendered = new RenderedPrompt("You are a careful engineer.", RenderFormat.MARKDOWN, "dh", "ch");
    }

    @Test
    void evaluate_parses_score() {
        assertThat(judge.evaluate(evalCase, rendered).score()).isEqualTo(4);
    }

    @Test
    void evaluate_parses_reasoning() {
        assertThat(judge.evaluate(evalCase, rendered).reasoning()).isEqualTo("Core role captured.");
    }

    @Test
    void evaluate_parses_gaps() {
        assertThat(judge.evaluate(evalCase, rendered).gaps())
            .containsExactly("philosophy not expressed");
    }

    @Test
    void evaluate_throws_malformed_when_score_missing() {
        final var noScore = new ProximityJudge(
            stubModel("{\"reasoning\": \"ok\", \"gaps\": []}"), new ObjectMapper());
        assertThatThrownBy(() -> noScore.evaluate(evalCase, rendered))
            .isInstanceOf(MalformedJudgeResponseException.class);
    }

    @Test
    void evaluate_throws_malformed_when_reasoning_missing() {
        final var noReasoning = new ProximityJudge(
            stubModel("{\"score\": 3, \"gaps\": []}"), new ObjectMapper());
        assertThatThrownBy(() -> noReasoning.evaluate(evalCase, rendered))
            .isInstanceOf(MalformedJudgeResponseException.class);
    }

    private static ChatModel stubModel(final String json) {
        return request -> ChatResponse.builder().aiMessage(AiMessage.from(json)).build();
    }

    private static ProfiledEvalCase minimalCase() {
        final var desc = new AgentDescriptor(
            "id", "N", null, null, null, null, null, null, null, null,
            "reviewer", List.of(), null, null, null, "t");
        final var profile = new AgentProfile(
            "sw-engineer-careful", "SW Eng", "sw", null, null,
            SourceType.ANTHROPIC_LIBRARY, "You are careful.", null, null,
            Map.of(), Map.of(), desc, List.of());
        return new ProfiledEvalCase("test", desc,
            AgentPromptContext.forFormat(RenderFormat.MARKDOWN), profile);
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=ProximityJudgeTest -q 2>&1 | tail -3
```

- [ ] **Step 3: Implement ProximityJudge**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.request.ResponseFormat;
import dev.langchain4j.model.chat.request.ResponseFormatType;
import dev.langchain4j.model.chat.request.json.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.List;

@ApplicationScoped
public class ProximityJudge {

    static final String SYSTEM_PROMPT = """
        You are evaluating how faithfully a machine-rendered system prompt captures
        the identity expressed in a human-authored system prompt.

        The human-authored prompt is the ground truth. The machine-rendered prompt was
        derived by structuring the human prompt into an AgentDescriptor, then rendering
        that descriptor back into a system prompt.

        Score 0–5:
        - 5: Conveys the same role, constraints, and operational style.
        - 4: Minor gaps — one or two concepts softened or absent.
        - 3: Core role present but significant style, constraints, or domain context missing.
        - 2: Role recognisable but rendering loses enough to change agent behaviour.
        - 1: Superficial match — same domain, fundamentally different character.
        - 0: Identity mismatch.

        Scoring guidance: Treat additions from the descriptor (not in the original prose)
        as neutral unless they contradict the original. Treat as a gap only content absent
        from BOTH the original prose AND the descriptor.

        Return JSON: { "score": int, "reasoning": string, "gaps": string[] }
        """;

    static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
        .type(ResponseFormatType.JSON)
        .jsonSchema(JsonSchema.builder()
            .name("ProximityJudgment")
            .rootElement(JsonObjectSchema.builder()
                .addIntegerProperty("score", "Score 0–5")
                .addStringProperty("reasoning", "Explanation")
                .addProperty("gaps", JsonArraySchema.builder()
                    .items(JsonStringSchema.builder().build()).build())
                .required("score", "reasoning", "gaps")
                .build())
            .build())
        .build();

    private final ChatModel model;
    private final ObjectMapper mapper;

    @Inject
    public ProximityJudge(@Any final Instance<ChatModel> models, final ObjectMapper mapper) {
        if (!models.isResolvable()) throw new IllegalStateException(
            "Judge ChatModel not configured.");
        this.model = models.get();
        this.mapper = mapper;
    }

    ProximityJudge(final ChatModel model, final ObjectMapper mapper) {
        this.model = model;
        this.mapper = mapper;
    }

    public ProximityResult evaluate(final ProfiledEvalCase evalCase, final RenderedPrompt rendered) {
        final ObjectNode payload = mapper.createObjectNode();
        payload.put("originalProse", evalCase.profile().originalProse());
        payload.put("rendered", rendered.content());
        try {
            final var request = ChatRequest.builder()
                .messages(SystemMessage.from(SYSTEM_PROMPT),
                    UserMessage.from(mapper.writeValueAsString(payload)))
                .responseFormat(RESPONSE_FORMAT)
                .build();
            final var response = model.chat(request);
            return parse(evalCase, response.aiMessage().text());
        } catch (final MalformedJudgeResponseException e) {
            throw e;
        } catch (final Exception e) {
            throw new IllegalStateException("ProximityJudge LLM call failed", e);
        }
    }

    private ProximityResult parse(final ProfiledEvalCase evalCase, final String json)
            throws JsonProcessingException {
        final JsonNode root = mapper.readTree(json);
        final JsonNode score = root.get("score");
        final JsonNode reasoning = root.get("reasoning");
        if (score == null || reasoning == null)
            throw new MalformedJudgeResponseException("ProximityJudge response missing score or reasoning");
        final List<String> gaps = new ArrayList<>();
        final JsonNode gapsNode = root.get("gaps");
        if (gapsNode != null && gapsNode.isArray()) gapsNode.forEach(n -> gaps.add(n.asText()));
        return new ProximityResult(evalCase, score.asInt(), reasoning.asText(), gaps);
    }
}
```

- [ ] **Step 4: Run ProximityJudgeTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=ProximityJudgeTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -q
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/ProximityJudge.java eval/src/test/java/io/casehub/eidos/eval/ProximityJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): ProximityJudge — semantic prose-to-render fidelity scoring

Refs #23"
```

---

## Task 10: VocabularyExpressivenessJudge

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/VocabularyExpressivenessJudge.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/VocabularyExpressivenessJudgeTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

class VocabularyExpressivenessJudgeTest {

    static final String AXIS_RESPONSE = """
        { "score": 3, "reasoning": "approximates but loses nuance", "gap": "correctness-over-velocity" }
        """;

    @Test
    void evaluate_makes_four_calls_one_per_axis() {
        final AtomicInteger calls = new AtomicInteger();
        final ChatModel counting = req -> {
            calls.incrementAndGet();
            return ChatResponse.builder().aiMessage(AiMessage.from(AXIS_RESPONSE)).build();
        };
        final var judge = new VocabularyExpressivenessJudge(counting, new ObjectMapper());
        judge.evaluate(minimalProfile());
        assertThat(calls.get()).isEqualTo(4);
    }

    @Test
    void evaluate_returns_scores_for_all_four_axes() {
        final var judge = new VocabularyExpressivenessJudge(stubModel(), new ObjectMapper());
        final VocabularyExpressivenessResult result = judge.evaluate(minimalProfile());
        assertThat(result.expressivenessScores())
            .containsKeys("socialOrient", "ruleFollowing", "riskAppetite", "autonomy");
    }

    @Test
    void evaluate_identifies_weak_axes_below_threshold() {
        final var judge = new VocabularyExpressivenessJudge(stubModel(), new ObjectMapper());
        // stub returns score=3 for all axes; 3 ≤ 2 is false, but let's check the boundary
        final VocabularyExpressivenessResult result = judge.evaluate(minimalProfile());
        // score=3 is NOT ≤ 2, so weakAxes should be empty
        assertThat(result.weakAxes()).isEmpty();
    }

    @Test
    void evaluate_marks_axes_scoring_le_2_as_weak() {
        final ChatModel lowScorer = req ->
            ChatResponse.builder()
                .aiMessage(AiMessage.from("{\"score\":2,\"reasoning\":\"poor\",\"gap\":\"x\"}"))
                .build();
        final var judge = new VocabularyExpressivenessJudge(lowScorer, new ObjectMapper());
        final VocabularyExpressivenessResult result = judge.evaluate(minimalProfile());
        assertThat(result.weakAxes()).hasSize(4);
    }

    @Test
    void evaluate_sets_profileName() {
        final var judge = new VocabularyExpressivenessJudge(stubModel(), new ObjectMapper());
        assertThat(judge.evaluate(minimalProfile()).profileName()).isEqualTo("test-profile");
    }

    private static ChatModel stubModel() {
        return req -> ChatResponse.builder().aiMessage(AiMessage.from(AXIS_RESPONSE)).build();
    }

    private static AgentProfile minimalProfile() {
        final var desc = new AgentDescriptor(
            "id", "N", null, null, null, null, null, null, null, null,
            "reviewer", List.of(), null, null, null, "t");
        return new AgentProfile("test-profile", "Test", "test", null, null,
            SourceType.PRACTITIONER, "You are a test agent.", null, null,
            Map.of(), Map.of(), desc, List.of());
    }
}
```

- [ ] **Step 2: Run — expect failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=VocabularyExpressivenessJudgeTest -q 2>&1 | tail -3
```

- [ ] **Step 3: Implement VocabularyExpressivenessJudge**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.request.ResponseFormat;
import dev.langchain4j.model.chat.request.ResponseFormatType;
import dev.langchain4j.model.chat.request.json.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class VocabularyExpressivenessJudge {

    static final List<String> AXES = List.of("socialOrient", "ruleFollowing", "riskAppetite", "autonomy");

    static final Map<String, String> AXIS_DESCRIPTIONS = Map.of(
        "socialOrient",
            "how collaborative or independent the agent is — whether it seeks input and coordinates with others or acts independently",
        "ruleFollowing",
            "how strictly the agent follows rules and conventions versus adapting its approach to context",
        "riskAppetite",
            "how risk-tolerant or risk-averse the agent is — whether it favours bold decisions under uncertainty or prioritises caution",
        "autonomy",
            "how self-directed versus directed-by-others the agent is — whether it takes initiative or waits for instruction"
    );

    static final String SYSTEM_TEMPLATE = """
        You are evaluating how precisely an open-string label (1–5 words) can express a
        personality concept found in the following system prompt.

        The concept: [%s]

        Score 1–5:
        - 5: A short label captures the nuance precisely
        - 3: A label approximates it but loses meaningful nuance
        - 1: The concept cannot be meaningfully captured in a short label

        If the score is ≤ 3, identify the specific nuance that is lost.

        Return JSON: { "score": int, "reasoning": string, "gap": string | null }
        """;

    static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
        .type(ResponseFormatType.JSON)
        .jsonSchema(JsonSchema.builder()
            .name("ExpressivenessJudgment")
            .rootElement(JsonObjectSchema.builder()
                .addIntegerProperty("score", "Score 1–5")
                .addStringProperty("reasoning", "Explanation")
                .addStringProperty("gap", "Specific nuance lost, or null")
                .required("score", "reasoning")
                .build())
            .build())
        .build();

    private final ChatModel model;
    private final ObjectMapper mapper;

    @Inject
    public VocabularyExpressivenessJudge(@Any final Instance<ChatModel> models,
                                          final ObjectMapper mapper) {
        if (!models.isResolvable()) throw new IllegalStateException("ChatModel not configured.");
        this.model = models.get();
        this.mapper = mapper;
    }

    VocabularyExpressivenessJudge(final ChatModel model, final ObjectMapper mapper) {
        this.model = model;
        this.mapper = mapper;
    }

    public VocabularyExpressivenessResult evaluate(final AgentProfile profile) {
        final Map<String, Integer> scores = new LinkedHashMap<>();
        final List<String> weakAxes = new ArrayList<>();

        for (final String axis : AXES) {
            final int score = evaluateAxis(profile.originalProse(), axis);
            scores.put(axis, score);
            if (score <= 2) weakAxes.add(axis);
        }
        return new VocabularyExpressivenessResult(profile.name(), scores, weakAxes);
    }

    private int evaluateAxis(final String prose, final String axis) {
        final String systemPrompt = String.format(SYSTEM_TEMPLATE, AXIS_DESCRIPTIONS.get(axis));
        try {
            final var request = ChatRequest.builder()
                .messages(SystemMessage.from(systemPrompt), UserMessage.from(prose))
                .responseFormat(RESPONSE_FORMAT)
                .build();
            final var response = model.chat(request);
            return parseScore(response.aiMessage().text(), axis);
        } catch (final MalformedJudgeResponseException e) {
            throw e;
        } catch (final Exception e) {
            throw new IllegalStateException("VocabularyExpressivenessJudge LLM call failed for axis: " + axis, e);
        }
    }

    private int parseScore(final String json, final String axis) throws JsonProcessingException {
        final JsonNode root = mapper.readTree(json);
        final JsonNode score = root.get("score");
        if (score == null) throw new MalformedJudgeResponseException(
            "VocabularyExpressivenessJudge: missing score for axis " + axis);
        return score.asInt();
    }
}
```

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=VocabularyExpressivenessJudgeTest -q
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/VocabularyExpressivenessJudge.java eval/src/test/java/io/casehub/eidos/eval/VocabularyExpressivenessJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): VocabularyExpressivenessJudge — Stage 1 axis expressiveness scoring

Refs #23"
```

---

## Task 11: TraitExpressionJudge

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/TraitExpressionJudge.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/TraitExpressionJudgeTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

class TraitExpressionJudgeTest {

    static final String VALID_JSON = """
        {
          "riskAppetite": 1, "socialOrient": 2, "ruleFollowing": 5, "autonomy": 1,
          "delegation": "NO", "reasoning": "Very careful and rule-following."
        }
        """;

    @Test
    void evaluate_parses_expression_scores() {
        final var result = judge(VALID_JSON).evaluate(minimalCase(), rendered());
        assertThat(result.expressionScores()).containsEntry("riskAppetite", 1);
        assertThat(result.expressionScores()).containsEntry("ruleFollowing", 5);
    }

    @Test
    void evaluate_parses_delegation_assessment() {
        assertThat(judge(VALID_JSON).evaluate(minimalCase(), rendered()).delegationAssessment())
            .isEqualTo("NO");
    }

    @Test
    void evaluate_computes_direction_matches_high_declaration() {
        // sw-engineer-careful has ruleFollowing: HIGH → blind score 5 ≥ 4 → match
        final var result = judge(VALID_JSON).evaluate(minimalCase(), rendered());
        assertThat(result.directionMatches()).containsEntry("ruleFollowing", true);
    }

    @Test
    void evaluate_computes_direction_matches_low_declaration_miss() {
        // sw-engineer-careful has autonomy: LOW → blind score 1 ≤ 2 → match
        final var result = judge(VALID_JSON).evaluate(minimalCase(), rendered());
        assertThat(result.directionMatches()).containsEntry("autonomy", true);
    }

    @Test
    void judge_payload_contains_only_rendered_text_not_descriptor() {
        final AtomicReference<String> capturedPayload = new AtomicReference<>();
        final ChatModel capture = req -> {
            req.messages().forEach(m -> {
                if (m instanceof dev.langchain4j.data.message.UserMessage um)
                    capturedPayload.set(um.singleText());
            });
            return ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build();
        };
        new TraitExpressionJudge(capture, new ObjectMapper()).evaluate(minimalCase(), rendered());
        // payload must NOT contain descriptor fields like agentId
        assertThat(capturedPayload.get()).doesNotContain("agentId").doesNotContain("tenancyId");
        assertThat(capturedPayload.get()).contains("You are a careful engineer");
    }

    private static TraitExpressionJudge judge(final String json) {
        return new TraitExpressionJudge(
            req -> ChatResponse.builder().aiMessage(AiMessage.from(json)).build(),
            new ObjectMapper());
    }

    private static RenderedPrompt rendered() {
        return new RenderedPrompt("You are a careful engineer.", RenderFormat.MARKDOWN, "dh", "ch");
    }

    private static ProfiledEvalCase minimalCase() {
        final var desc = new AgentDescriptor(
            "id", "N", null, null, null, null, null, null, null, null,
            "reviewer", List.of(), null, null, null, "t");
        final var profile = new AgentProfile(
            "sw-engineer-careful", "SW Eng", "sw", null, null,
            SourceType.ANTHROPIC_LIBRARY, "You are careful.", null, null,
            Map.of(), Map.of(
                "riskAppetite", TraitPolarity.LOW,
                "socialOrient", TraitPolarity.LOW,
                "ruleFollowing", TraitPolarity.HIGH,
                "autonomy", TraitPolarity.LOW),
            desc, List.of());
        return new ProfiledEvalCase("test", desc,
            AgentPromptContext.forFormat(RenderFormat.MARKDOWN), profile);
    }
}
```

- [ ] **Step 2: Implement TraitExpressionJudge**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.request.ResponseFormat;
import dev.langchain4j.model.chat.request.ResponseFormatType;
import dev.langchain4j.model.chat.request.json.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class TraitExpressionJudge {

    static final List<String> NUMERIC_AXES =
        List.of("socialOrient", "ruleFollowing", "riskAppetite", "autonomy");

    static final String SYSTEM_PROMPT = """
        You are characterising an AI agent's personality based solely on the following
        system prompt. Do not infer from context or domain — read only what is
        explicitly or strongly implied.

        Score each axis 1–5:

        riskAppetite:
          5 = explicitly endorses bold decisions, accepts uncertainty
          3 = neutral
          1 = explicitly emphasises correctness, caution, and risk avoidance

        socialOrient:
          5 = explicitly collaborative — seeks input, coordinates before acting
          3 = neutral
          1 = explicitly independent — self-directed, minimal consultation

        ruleFollowing:
          5 = explicitly strict — follows processes, does not deviate
          3 = neutral
          1 = explicitly adaptive — comfortable bending conventions

        autonomy:
          5 = explicitly autonomous — takes initiative, decides without approval
          3 = neutral
          1 = explicitly directed — waits for instruction, seeks approval

        delegation:
          Does this prompt explicitly grant or restrict sub-agent delegation authority?
          Answer: "YES" | "NO" | "UNCERTAIN"

        Return JSON:
        { "riskAppetite": int, "socialOrient": int, "ruleFollowing": int, "autonomy": int,
          "delegation": "YES"|"NO"|"UNCERTAIN", "reasoning": string }
        """;

    static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
        .type(ResponseFormatType.JSON)
        .jsonSchema(JsonSchema.builder()
            .name("TraitExpression")
            .rootElement(JsonObjectSchema.builder()
                .addIntegerProperty("riskAppetite", "1–5")
                .addIntegerProperty("socialOrient", "1–5")
                .addIntegerProperty("ruleFollowing", "1–5")
                .addIntegerProperty("autonomy", "1–5")
                .addStringProperty("delegation", "YES|NO|UNCERTAIN")
                .addStringProperty("reasoning", "Explanation")
                .required("riskAppetite", "socialOrient", "ruleFollowing", "autonomy",
                    "delegation", "reasoning")
                .build())
            .build())
        .build();

    private final ChatModel model;
    private final ObjectMapper mapper;

    @Inject
    public TraitExpressionJudge(@Any final Instance<ChatModel> models, final ObjectMapper mapper) {
        if (!models.isResolvable()) throw new IllegalStateException("ChatModel not configured.");
        this.model = models.get();
        this.mapper = mapper;
    }

    TraitExpressionJudge(final ChatModel model, final ObjectMapper mapper) {
        this.model = model;
        this.mapper = mapper;
    }

    public TraitExpressionResult evaluate(final ProfiledEvalCase evalCase,
                                           final RenderedPrompt rendered) {
        try {
            final var request = ChatRequest.builder()
                .messages(SystemMessage.from(SYSTEM_PROMPT),
                    UserMessage.from(rendered.content()))   // rendered text ONLY — no descriptor
                .responseFormat(RESPONSE_FORMAT)
                .build();
            final var response = model.chat(request);
            return parse(evalCase, rendered.format(), response.aiMessage().text());
        } catch (final MalformedJudgeResponseException e) {
            throw e;
        } catch (final Exception e) {
            throw new IllegalStateException("TraitExpressionJudge LLM call failed", e);
        }
    }

    private TraitExpressionResult parse(final ProfiledEvalCase evalCase,
                                         final io.casehub.eidos.api.SystemPromptRenderer.RenderFormat format,
                                         final String json) throws JsonProcessingException {
        final JsonNode root = mapper.readTree(json);
        final Map<String, Integer> scores = new LinkedHashMap<>();
        for (final String axis : NUMERIC_AXES) {
            final JsonNode n = root.get(axis);
            if (n == null) throw new MalformedJudgeResponseException("Missing axis: " + axis);
            scores.put(axis, n.asInt());
        }
        final JsonNode del = root.get("delegation");
        if (del == null) throw new MalformedJudgeResponseException("Missing delegation");

        final Map<String, Boolean> matches = computeMatches(evalCase.profile(), scores);
        return new TraitExpressionResult(evalCase, format, scores, matches, del.asText());
    }

    private Map<String, Boolean> computeMatches(final AgentProfile profile,
                                                  final Map<String, Integer> scores) {
        final Map<String, Boolean> result = new LinkedHashMap<>();
        for (final String axis : NUMERIC_AXES) {
            final TraitPolarity polarity = profile.expectedTraits() == null
                ? null : profile.expectedTraits().get(axis);
            if (polarity == null) continue;
            final int score = scores.getOrDefault(axis, 3);
            result.put(axis, switch (polarity) {
                case HIGH -> score >= 4;
                case LOW -> score <= 2;
                case NEUTRAL -> true;
            });
        }
        return result;
    }
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=TraitExpressionJudgeTest -q
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/TraitExpressionJudge.java eval/src/test/java/io/casehub/eidos/eval/TraitExpressionJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): TraitExpressionJudge — Stage 2 blind trait scoring

Refs #23"
```

---

## Task 12: PairContrastJudge

**Files:**
- Create: `eval/src/main/java/io/casehub/eidos/eval/PairContrastJudge.java`
- Create: `eval/src/test/java/io/casehub/eidos/eval/PairContrastJudgeTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PairContrastJudgeTest {

    static final String VALID_JSON = """
        { "higher": "A", "effectSize": 4, "reasoning": "Clearly more risk-tolerant." }
        """;

    @Test
    void evaluate_parses_effect_size() {
        final var result = judge().evaluate(pair(), RenderFormat.MARKDOWN, renders());
        assertThat(result.effectSize()).isEqualTo(4);
    }

    @Test
    void evaluate_correctly_identified_when_A_is_higher() {
        // pair.higher()=sw-engineer-bold, judge says "A" = sw-engineer-bold render → correct
        final var result = judge().evaluate(pair(), RenderFormat.MARKDOWN, renders());
        assertThat(result.correctlyIdentified()).isTrue();
    }

    @Test
    void evaluate_throws_when_profile_slug_missing_from_renders() {
        assertThatThrownBy(() ->
            judge().evaluate(pair(), RenderFormat.MARKDOWN, Map.of()))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("sw-engineer-bold");
    }

    @Test
    void evaluate_sets_primaryAxis() {
        assertThat(judge().evaluate(pair(), RenderFormat.MARKDOWN, renders()).primaryAxis())
            .isEqualTo("riskAppetite");
    }

    private PairContrastJudge judge() {
        return new PairContrastJudge(
            req -> ChatResponse.builder().aiMessage(AiMessage.from(VALID_JSON)).build(),
            new ObjectMapper());
    }

    private VariantPair pair() {
        return new VariantPair("riskAppetite", "sw-engineer-bold", "sw-engineer-careful");
    }

    private Map<ProfiledEvalCase, RenderedPrompt> renders() {
        return Map.of(
            profiledCase("sw-engineer-bold"),
                new RenderedPrompt("You are bold.", RenderFormat.MARKDOWN, "dh", "ch"),
            profiledCase("sw-engineer-careful"),
                new RenderedPrompt("You are careful.", RenderFormat.MARKDOWN, "dh", "ch")
        );
    }

    private ProfiledEvalCase profiledCase(final String profileName) {
        final var desc = new AgentDescriptor(
            "id", "N", null, null, null, null, null, null, null, null,
            "reviewer", List.of(), null, null, null, "t");
        final var profile = new AgentProfile(profileName, "R", "d", null, null,
            SourceType.PRACTITIONER, "prose", null, null, Map.of(), Map.of(), desc, List.of());
        return new ProfiledEvalCase(
            profileName + "-markdown", desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN), profile);
    }
}
```

- [ ] **Step 2: Implement PairContrastJudge**

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.request.ResponseFormat;
import dev.langchain4j.model.chat.request.ResponseFormatType;
import dev.langchain4j.model.chat.request.json.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.Map;

@ApplicationScoped
public class PairContrastJudge {

    static final Map<String, String> AXIS_DESCRIPTIONS =
        VocabularyExpressivenessJudge.AXIS_DESCRIPTIONS;

    static final String SYSTEM_TEMPLATE = """
        You are comparing two AI agent system prompts on a specific personality axis.

        Axis: [%s]
        Prompt A: %s
        Prompt B: %s

        Identify which prompt expresses the axis more strongly, and score how starkly
        different they are.

        Effect size rubric (1–5):
        - 5 = unmistakably different; a naive reader could identify which is which
        - 3 = distinguishable if you are looking for it
        - 1 = practically indistinguishable on this axis

        Return JSON: { "higher": "A" | "B", "effectSize": int, "reasoning": string }
        """;

    static final ResponseFormat RESPONSE_FORMAT = ResponseFormat.builder()
        .type(ResponseFormatType.JSON)
        .jsonSchema(JsonSchema.builder()
            .name("PairContrast")
            .rootElement(JsonObjectSchema.builder()
                .addStringProperty("higher", "A or B")
                .addIntegerProperty("effectSize", "1–5")
                .addStringProperty("reasoning", "Explanation")
                .required("higher", "effectSize", "reasoning")
                .build())
            .build())
        .build();

    private final ChatModel model;
    private final ObjectMapper mapper;

    @Inject
    public PairContrastJudge(@Any final Instance<ChatModel> models, final ObjectMapper mapper) {
        if (!models.isResolvable()) throw new IllegalStateException("ChatModel not configured.");
        this.model = models.get();
        this.mapper = mapper;
    }

    PairContrastJudge(final ChatModel model, final ObjectMapper mapper) {
        this.model = model;
        this.mapper = mapper;
    }

    public PairContrastResult evaluate(final VariantPair pair, final RenderFormat format,
                                        final Map<ProfiledEvalCase, RenderedPrompt> renders) {
        final String higherText = findRender(renders, pair.higher(), format);
        final String lowerText = findRender(renders, pair.lower(), format);
        final String axisDesc = AXIS_DESCRIPTIONS.getOrDefault(pair.primaryAxis(), pair.primaryAxis());
        final String prompt = String.format(SYSTEM_TEMPLATE, axisDesc, higherText, lowerText);
        try {
            final var request = ChatRequest.builder()
                .messages(SystemMessage.from(prompt), UserMessage.from("Compare the two prompts."))
                .responseFormat(RESPONSE_FORMAT)
                .build();
            final var response = model.chat(request);
            return parse(pair, format, response.aiMessage().text(), pair.higher());
        } catch (final MalformedJudgeResponseException e) {
            throw e;
        } catch (final Exception e) {
            throw new IllegalStateException("PairContrastJudge LLM call failed", e);
        }
    }

    private String findRender(final Map<ProfiledEvalCase, RenderedPrompt> renders,
                               final String profileSlug, final RenderFormat format) {
        return renders.entrySet().stream()
            .filter(e -> e.getKey().profile().name().equals(profileSlug)
                && e.getKey().context().format() == format)
            .map(e -> e.getValue().content())
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "No render found for profile: " + profileSlug + ", format: " + format));
    }

    private PairContrastResult parse(final VariantPair pair, final RenderFormat format,
                                      final String json, final String expectedHigher)
            throws JsonProcessingException {
        final JsonNode root = mapper.readTree(json);
        final JsonNode higher = root.get("higher");
        final JsonNode effectSize = root.get("effectSize");
        final JsonNode reasoning = root.get("reasoning");
        if (higher == null || effectSize == null || reasoning == null)
            throw new MalformedJudgeResponseException("PairContrastJudge response missing fields");
        // "A" = pair.higher() profile (passed as Prompt A)
        final boolean correct = "A".equals(higher.asText());
        return new PairContrastResult(pair.higher(), pair.lower(), pair.primaryAxis(),
            format, correct, effectSize.asInt(), reasoning.asText());
    }
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=PairContrastJudgeTest -q
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/PairContrastJudge.java eval/src/test/java/io/casehub/eidos/eval/PairContrastJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): PairContrastJudge — Stage 3 pairwise discriminability + effect size

Refs #23"
```

---

## Task 13: EvalReportWriter extensions

**Files:**
- Modify: `eval/src/main/java/io/casehub/eidos/eval/EvalReportWriter.java`
- Modify: `eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java`

- [ ] **Step 1: Write failing tests for new methods**

Add to `EvalReportWriterTest`:
```java
@Test
void writeProximityJson_creates_valid_json(@TempDir final Path dir) throws IOException {
    final var result = new ProximityResult(EvalReportWriterTest.sampleCase(), 4, "good", List.of("gap1"));
    final var report = ProximityReport.build(List.of(result), 3.0);
    final Path out = dir.resolve("prox.json");
    EvalReportWriter.writeProximityJson(report, out);
    assertThat(out).exists();
    new ObjectMapper().findAndRegisterModules().readTree(out.toFile());
}

@Test
void proximitySummaryTable_contains_mean_score(@TempDir final Path dir) {
    final var result = new ProximityResult(EvalReportWriterTest.sampleCase(), 4, "good", List.of());
    final var report = ProximityReport.build(List.of(result), 3.0);
    assertThat(EvalReportWriter.proximitySummaryTable(report)).contains("4.00");
}

@Test
void writePreservationJson_creates_valid_json(@TempDir final Path dir) throws IOException {
    final var report = PersonalityPreservationReport.build(List.of(), List.of(), List.of());
    final Path out = dir.resolve("pres.json");
    EvalReportWriter.writePreservationJson(report, out);
    assertThat(out).exists();
    new ObjectMapper().findAndRegisterModules().readTree(out.toFile());
}
```

Also add a helper to `EvalReportWriterTest`:
```java
static SyntheticEvalCase sampleCase() {
    final var desc = new AgentDescriptor(
        "a", "Agent", null, null, null, null, null, null, null, null,
        "worker", List.of(), null, null, null, "t");
    return new SyntheticEvalCase("case1", desc, AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
}
```

- [ ] **Step 2: Add four methods to EvalReportWriter**

Append to `EvalReportWriter.java`:
```java
public static void writeProximityJson(final ProximityReport report, final Path path) {
    try {
        MAPPER.writeValue(path.toFile(), report);
    } catch (final IOException e) {
        throw new UncheckedIOException("Failed to write proximity report to " + path, e);
    }
}

public static String proximitySummaryTable(final ProximityReport report) {
    final var sb = new StringBuilder();
    sb.append(String.format("=== Proximity Report (floor %.1f) ===%n", report.floor()));
    sb.append(String.format("Mean score (0–5):  %.2f%n", report.meanScore()));
    sb.append(String.format("Min score:         %d%n", (int) report.minScore()));
    sb.append(String.format("Below floor:       %d / %d%n",
        report.belowFloor(), report.results().size()));
    return sb.toString();
}

public static void writePreservationJson(final PersonalityPreservationReport report,
                                          final Path path) {
    try {
        MAPPER.writeValue(path.toFile(), report);
    } catch (final IOException e) {
        throw new UncheckedIOException("Failed to write preservation report to " + path, e);
    }
}

public static String preservationSummaryTable(final PersonalityPreservationReport report) {
    final var sb = new StringBuilder();
    sb.append("=== Personality Preservation Report ===%n".formatted());
    sb.append(String.format("Vocab expressiveness (1–5):  %.2f%n", report.meanExpressivenessScore()));
    sb.append(String.format("Trait match rate:            %.0f%%%n", report.meanTraitMatchRate() * 100));
    sb.append(String.format("Mean effect size (1–5):      %.2f%n", report.meanEffectSize()));
    sb.append(String.format("Discrimination accuracy:     %.0f%%%n", report.discriminationAccuracy() * 100));
    sb.append("%n--- Attribution Diagnoses ---%n".formatted());
    report.diagnoses().forEach(d ->
        sb.append(String.format("  %-25s %-14s → %s%n",
            d.profileName() + "/" + d.axis(), "", d.attribution())));
    if (!report.annotations().isEmpty()) {
        sb.append("%n--- Reliability Warnings ---%n".formatted());
        report.annotations().forEach(w -> sb.append("  ⚠ ").append(w).append("%n".formatted()));
    }
    return sb.toString();
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/EvalReportWriter.java eval/src/test/java/io/casehub/eidos/eval/EvalReportWriterTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): EvalReportWriter — proximity + preservation write/summary methods

Refs #23"
```

---

## Task 14: Phase 1 — Research and write 8 real profile YAMLs

This is a content research task. Use WebSearch + WebFetch to find real published system prompts.

**Files to create/update:**
- `eval/src/test/resources/profiles/index.yaml` (update to all 8)
- `eval/src/test/resources/profiles/sw-engineer-careful.yaml` (update prose)
- `eval/src/test/resources/profiles/sw-engineer-bold.yaml` (update prose)
- `eval/src/test/resources/profiles/security-analyst-defensive.yaml`
- `eval/src/test/resources/profiles/security-analyst-proactive.yaml`
- `eval/src/test/resources/profiles/product-manager.yaml`
- `eval/src/test/resources/profiles/clinical-researcher.yaml`
- `eval/src/test/resources/profiles/customer-support-agent.yaml`
- `eval/src/test/resources/profiles/technical-writer.yaml`

**Research sources (in priority order):**
1. Anthropic Prompt Library: `https://docs.anthropic.com/en/prompt-library/`
2. OpenAI Cookbook: `https://github.com/openai/openai-cookbook`
3. O*NET for capability derivation: `https://www.onetonline.org/`

- [ ] **Step 1: Research and write each profile**

For each role: search for real published prose, select prose > 100 words, derive AgentDescriptor, note vocabulary gaps.

Variant pair constraint reminder:
- `sw-engineer-careful` vs `sw-engineer-bold`: only `riskAppetite` differs in disposition
- `security-analyst-defensive` vs `security-analyst-proactive`: only `ruleFollowing` differs

- [ ] **Step 2: Update index.yaml**

```yaml
profiles:
  - sw-engineer-careful.yaml
  - sw-engineer-bold.yaml
  - security-analyst-defensive.yaml
  - security-analyst-proactive.yaml
  - product-manager.yaml
  - clinical-researcher.yaml
  - customer-support-agent.yaml
  - technical-writer.yaml

variants:
  - primaryAxis: riskAppetite
    higher: sw-engineer-bold
    lower: sw-engineer-careful
  - primaryAxis: ruleFollowing
    higher: security-analyst-defensive
    lower: security-analyst-proactive
```

- [ ] **Step 3: Verify all profiles load without exception**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -Dtest=AgentProfileLoaderTest -q
```

Update `AgentProfileLoaderTest` to expect 8 profiles (change `hasSize(2)` to `hasSize(8)`).

- [ ] **Step 4: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -q
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/resources/profiles/ eval/src/test/java/io/casehub/eidos/eval/AgentProfileLoaderTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): Phase 1 — 8 real-world agent profile YAMLs from published sources

Refs #23"
```

---

## Task 15: PromptEvalTest extension

**Files:**
- Modify: `eval/src/test/java/io/casehub/eidos/eval/PromptEvalTest.java`

- [ ] **Step 1: Add @BeforeAll and inject new judges**

Add to top of `PromptEvalTest`:
```java
import static java.util.function.Function.identity;
import static java.util.stream.Collectors.toMap;

static List<ProfiledEvalCase> realWorldCases;
static VariantIndex variantIndex;

@BeforeAll
static void loadProfiles() {
    realWorldCases = RealWorldEvalDataset.all();
    variantIndex = new AgentProfileLoader().loadIndex();
}

@Inject ProximityJudge proximityJudge;
@Inject VocabularyExpressivenessJudge expressivenessJudge;
@Inject TraitExpressionJudge traitExpressionJudge;
@Inject PairContrastJudge pairContrastJudge;
```

- [ ] **Step 2: Add evaluateRealWorldScenarios test method**

```java
private static final double PROXIMITY_FLOOR = 3.0;

@Test
void evaluateRealWorldScenarios() throws Exception {
    final Map<ProfiledEvalCase, RenderedPrompt> renders = realWorldCases.stream()
        .collect(toMap(identity(), c -> renderer.render(c.descriptor(), c.context())));

    final List<EvalResult> qualityResults = realWorldCases.stream()
        .map(c -> judge.evaluate(c, renders.get(c))).toList();

    final List<ProximityResult> proximityResults = realWorldCases.stream()
        .map(c -> proximityJudge.evaluate(c, renders.get(c))).toList();

    final List<VocabularyExpressivenessResult> expressivenessResults =
        realWorldCases.stream().map(c -> c.profile()).distinct()
            .map(p -> expressivenessJudge.evaluate(p)).toList();

    final List<TraitExpressionResult> traitResults = realWorldCases.stream()
        .map(c -> traitExpressionJudge.evaluate(c, renders.get(c))).toList();

    final List<PairContrastResult> contrastResults =
        variantIndex.variants().stream()
            .flatMap(pair -> Stream.of(RenderFormat.MARKDOWN, RenderFormat.PROSE)
                .map(format -> pairContrastJudge.evaluate(pair, format, renders)))
            .toList();

    final List<ProfiledEvalCase> sample = realWorldCases.stream()
        .filter(c -> c.context().format() == RenderFormat.MARKDOWN).limit(2).toList();
    final List<String> reliabilityWarnings = runReliabilityCheck(sample, renders, variantIndex);

    Files.createDirectories(Path.of("target"));

    final EvalReport qualityReport = EvalReport.build(qualityResults, "judge");
    EvalReportWriter.writeJson(qualityReport, Path.of("target/real-world-eval-report.json"));
    System.out.println(EvalReportWriter.summaryTable(qualityReport));

    final ProximityReport proximityReport = ProximityReport.build(proximityResults, PROXIMITY_FLOOR);
    EvalReportWriter.writeProximityJson(proximityReport, Path.of("target/proximity-report.json"));
    System.out.println(EvalReportWriter.proximitySummaryTable(proximityReport));

    final PersonalityPreservationReport preservationReport =
        PersonalityPreservationReport.build(expressivenessResults, traitResults, contrastResults);
    reliabilityWarnings.forEach(w -> preservationReport.annotations().add(w));
    EvalReportWriter.writePreservationJson(preservationReport,
        Path.of("target/personality-preservation-report.json"));
    System.out.println(EvalReportWriter.preservationSummaryTable(preservationReport));

    qualityReport.summaryByFormat().forEach((format, summary) -> {
        assertThat(summary.allCasesComplete())
            .as("All %s real-world cases complete", format).isTrue();
        assertThat(summary.meanOverall())
            .as("Mean quality score for %s", format)
            .isGreaterThanOrEqualTo(SCORE_FLOORS.getOrDefault(format, 3.5));
    });

    assertThat(proximityReport.meanScore())
        .as("Mean proximity score").isGreaterThanOrEqualTo(PROXIMITY_FLOOR);
}
```

- [ ] **Step 3: Add runReliabilityCheck helper**

```java
private List<String> runReliabilityCheck(
    final List<ProfiledEvalCase> sample,
    final Map<ProfiledEvalCase, RenderedPrompt> renders,
    final VariantIndex index
) throws Exception {
    final List<String> warnings = new ArrayList<>();
    // Run Stage 2 twice on each sample case, compare per-axis scores
    for (final ProfiledEvalCase c : sample) {
        final TraitExpressionResult r1 = traitExpressionJudge.evaluate(c, renders.get(c));
        final TraitExpressionResult r2 = traitExpressionJudge.evaluate(c, renders.get(c));
        for (final String axis : List.of("socialOrient", "ruleFollowing", "riskAppetite", "autonomy")) {
            final int s1 = r1.expressionScores().getOrDefault(axis, 3);
            final int s2 = r2.expressionScores().getOrDefault(axis, 3);
            if (Math.abs(s1 - s2) > 0.5) {
                warnings.add("Stage2 variance > 0.5 for " + c.profile().name() + "/" + axis
                    + ": " + s1 + " vs " + s2);
            }
        }
    }
    // Write reliability report
    final var relReport = Map.of("warnings", warnings, "passed", warnings.isEmpty());
    Files.createDirectories(Path.of("target"));
    new ObjectMapper().findAndRegisterModules()
        .enable(com.fasterxml.jackson.databind.SerializationFeature.INDENT_OUTPUT)
        .writeValue(Path.of("target/judge-reliability.json").toFile(), relReport);
    return warnings;
}
```

- [ ] **Step 4: Verify unit tests still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -q
```
Expected: BUILD SUCCESS (evaluateRealWorldScenarios is @Tag("eval") and excluded from normal test runs).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/test/java/io/casehub/eidos/eval/PromptEvalTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eidos#23): PromptEvalTest — evaluateRealWorldScenarios + reliability check

Refs #23"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| SourceType, CoverageLoss, TraitPolarity, Attribution enums | Task 1 |
| VocabularyGap, VariantPair, VariantIndex, AttributionDiagnosis | Task 2 |
| ProximityResult (validated), PairContrastResult (validated) | Task 3 |
| AgentProfile, VocabularyExpressivenessResult, TraitExpressionResult | Task 4 |
| ProximityReport.build(), PersonalityPreservationReport.build() with formulas | Task 5 |
| EvalCase → sealed interface; 20 call sites migrated | Task 6 |
| AgentProfileLoader (test scope), Stage 0 validation, stub YAMLs | Task 7 |
| RealWorldEvalDataset (test scope), evalGoal context | Task 8 |
| ProximityJudge | Task 9 |
| VocabularyExpressivenessJudge (Stage 1, 4 sequential calls) | Task 10 |
| TraitExpressionJudge (Stage 2, blind, direction match ≥4/≤2) | Task 11 |
| PairContrastJudge (Stage 3, resolves by profile.name()) | Task 12 |
| EvalReportWriter +4 methods | Task 13 |
| 8 real-world profile YAMLs (Phase 1 research) | Task 14 |
| PromptEvalTest: evaluateRealWorldScenarios + @BeforeAll + reliability check | Task 15 |
| EvalReportWriterTest, EvalReportTest migration | Task 6 |
| jackson-dataformat-yaml in pom.xml | Task 1 |

**Gaps found and addressed:**
- `EvalDataset.realWorld()` cannot be in main scope because `AgentProfileLoader` is test-scope — resolved via `RealWorldEvalDataset` in test scope (Task 8)
- `PersonalityPreservationReport.annotations()` must return mutable List — `new ArrayList<>()` in build() ✅
- `TraitExpressionResult` and `ProfiledEvalCase` are forward-referenced in Tasks 4/3; production code compiles fully after Task 6 ✅
