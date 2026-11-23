# Cross-Unit Evidence Threshold Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #266 — minimum cross-unit evidence threshold
**Issue group:** #266

**Goal:** Add a `CrossUnitEvidenceAssessor` SPI that evaluates whether a trust score's evidence pool includes sufficient cross-unit attestations, returning a typed diversity assessment alongside the score.

**Architecture:** New SPI in `eidos-api` (following `OrgAwareDispatch` pattern — org-aware SPIs with no org model type dependencies live in api). `@DefaultBean` NoOp in `eidos-core`. Implementation in `org-runtime`. Shared `OrgDistanceUtil` utility extracted from existing factors.

**Tech Stack:** Java 21, Quarkus CDI, MicroProfile Config

## Global Constraints

- Java 21 source (Java 26 JVM)
- Quarkus 3.32.2
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All string fields validated at compact constructor time
- NoOp `@DefaultBean` for backward compatibility
- No ledger dependencies in new types

---

## Batch 1: SPI + NoOp + Utility Extraction

### Task 1: OrgDistanceUtil — extract shared distance logic

**Files:**
- Create: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/OrgDistanceUtil.java`
- Create: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/OrgDistanceUtilTest.java`
- Modify: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/OrgProximityRelevanceFactor.java` — use `OrgDistanceUtil.SNAPSHOT_KEY` and `extractDistance()`
- Modify: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java` — use `OrgDistanceUtil.extractDistance()`

**Interfaces:**
- Produces: `OrgDistanceUtil.extractDistance(Map<String, Object>)` → `int` (-1 for unknown, 0+ for distance); `OrgDistanceUtil.SNAPSHOT_KEY` → `"org-distance"`

- [ ] **Step 1: Write OrgDistanceUtil tests**

```java
package io.casehub.eidos.org.runtime.trust;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class OrgDistanceUtilTest {

    @Test
    void null_snapshot_returns_negative_one() {
        assertThat(OrgDistanceUtil.extractDistance(null)).isEqualTo(-1);
    }

    @Test
    void missing_key_returns_negative_one() {
        assertThat(OrgDistanceUtil.extractDistance(Map.of("other", 5))).isEqualTo(-1);
    }

    @Test
    void non_number_value_returns_negative_one() {
        assertThat(OrgDistanceUtil.extractDistance(Map.of("org-distance", "two"))).isEqualTo(-1);
    }

    @Test
    void integer_distance_returned() {
        assertThat(OrgDistanceUtil.extractDistance(Map.of("org-distance", 3))).isEqualTo(3);
    }

    @Test
    void double_distance_truncated_to_int() {
        assertThat(OrgDistanceUtil.extractDistance(Map.of("org-distance", 2.7))).isEqualTo(2);
    }

    @Test
    void zero_distance_returned() {
        assertThat(OrgDistanceUtil.extractDistance(Map.of("org-distance", 0))).isEqualTo(0);
    }

    @Test
    void snapshot_key_matches_existing_enricher() {
        assertThat(OrgDistanceUtil.SNAPSHOT_KEY).isEqualTo("org-distance");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=OrgDistanceUtilTest`
Expected: compilation error — `OrgDistanceUtil` does not exist

- [ ] **Step 3: Implement OrgDistanceUtil**

Create `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/OrgDistanceUtil.java`:

```java
package io.casehub.eidos.org.runtime.trust;

import java.util.Map;

public final class OrgDistanceUtil {

    public static final String SNAPSHOT_KEY = "org-distance";

    public static int extractDistance(Map<String, Object> snapshot) {
        if (snapshot == null || !snapshot.containsKey(SNAPSHOT_KEY)) {
            return -1;
        }
        Object raw = snapshot.get(SNAPSHOT_KEY);
        if (!(raw instanceof Number)) {
            return -1;
        }
        return ((Number) raw).intValue();
    }

    private OrgDistanceUtil() {}
}
```

- [ ] **Step 4: Run OrgDistanceUtil tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=OrgDistanceUtilTest`
Expected: all 7 tests PASS

- [ ] **Step 5: Refactor OrgProximityRelevanceFactor to use OrgDistanceUtil**

In `OrgProximityRelevanceFactor.java`:
- Remove `static final String SNAPSHOT_KEY = "org-distance";`
- Replace `snapshot.get(SNAPSHOT_KEY)` with `OrgDistanceUtil.extractDistance(snapshot)`
- Replace `snapshot.containsKey(SNAPSHOT_KEY)` checks with distance check: `int distance = OrgDistanceUtil.extractDistance(snapshot); if (distance < 0) return NEUTRAL;`
- Use `OrgDistanceUtil.SNAPSHOT_KEY` in `OrgDistanceAttestationEnricher` (it references `OrgProximityRelevanceFactor.SNAPSHOT_KEY`)

- [ ] **Step 6: Refactor IndependenceAwareRelevanceFactor to use OrgDistanceUtil**

In `IndependenceAwareRelevanceFactor.java`:
- Remove the private `extractDistance()` method
- Replace all calls to `extractDistance(ctx)` with `OrgDistanceUtil.extractDistance(ctx.snapshot())`

- [ ] **Step 7: Update OrgDistanceAttestationEnricher**

In `OrgDistanceAttestationEnricher.java`:
- Replace `OrgProximityRelevanceFactor.SNAPSHOT_KEY` with `OrgDistanceUtil.SNAPSHOT_KEY`

- [ ] **Step 8: Run all existing trust tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest="OrgProximityRelevanceFactorTest,IndependenceAwareRelevanceFactorTest,OrgDistanceAttestationEnricherTest,TrustBeansCdiTest"`
Expected: all tests PASS with unchanged behaviour

- [ ] **Step 9: Commit**

```bash
git add org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/OrgDistanceUtil.java org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/OrgDistanceUtilTest.java org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/OrgProximityRelevanceFactor.java org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/OrgDistanceAttestationEnricher.java
git commit -m "refactor(#266): extract OrgDistanceUtil from relevance factors

Shared extractDistance() utility and SNAPSHOT_KEY constant. Both
OrgProximityRelevanceFactor and IndependenceAwareRelevanceFactor now
delegate to OrgDistanceUtil instead of private copies.

Refs #266"
```

---

### Task 2: SPI types + NoOp default

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/EvidenceDiversityLevel.java`
- Create: `api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessment.java`
- Create: `api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessor.java`
- Create: `api/src/test/java/io/casehub/eidos/api/CrossUnitEvidenceAssessmentTest.java`
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessor.java`
- Create: `eidos-core/src/test/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessorTest.java`

**Interfaces:**
- Produces: `CrossUnitEvidenceAssessor.assess(List<Map<String, Object>>)` → `CrossUnitEvidenceAssessment`; `CrossUnitEvidenceAssessment.UNKNOWN` static constant; `EvidenceDiversityLevel` enum

- [ ] **Step 1: Write CrossUnitEvidenceAssessment tests**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.ArrayList;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class CrossUnitEvidenceAssessmentTest {

    @Test
    void valid_construction() {
        var assessment = new CrossUnitEvidenceAssessment(5, 3, 1,
                EvidenceDiversityLevel.DIVERSE, List.of());
        assertThat(assessment.sameUnitCount()).isEqualTo(5);
        assertThat(assessment.crossUnitCount()).isEqualTo(3);
        assertThat(assessment.unknownCount()).isEqualTo(1);
        assertThat(assessment.level()).isEqualTo(EvidenceDiversityLevel.DIVERSE);
        assertThat(assessment.warnings()).isEmpty();
    }

    @Test
    void negative_same_unit_count_rejected() {
        assertThatThrownBy(() -> new CrossUnitEvidenceAssessment(-1, 0, 0,
                EvidenceDiversityLevel.UNKNOWN, List.of()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void negative_cross_unit_count_rejected() {
        assertThatThrownBy(() -> new CrossUnitEvidenceAssessment(0, -1, 0,
                EvidenceDiversityLevel.UNKNOWN, List.of()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void negative_unknown_count_rejected() {
        assertThatThrownBy(() -> new CrossUnitEvidenceAssessment(0, 0, -1,
                EvidenceDiversityLevel.UNKNOWN, List.of()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void warnings_are_immutable() {
        var mutable = new ArrayList<>(List.of("warning"));
        var assessment = new CrossUnitEvidenceAssessment(1, 0, 0,
                EvidenceDiversityLevel.HOMOGENEOUS, mutable);
        assertThatThrownBy(() -> assessment.warnings().add("another"))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void unknown_constant_has_zero_counts() {
        var unknown = CrossUnitEvidenceAssessment.UNKNOWN;
        assertThat(unknown.sameUnitCount()).isEqualTo(0);
        assertThat(unknown.crossUnitCount()).isEqualTo(0);
        assertThat(unknown.unknownCount()).isEqualTo(0);
        assertThat(unknown.level()).isEqualTo(EvidenceDiversityLevel.UNKNOWN);
        assertThat(unknown.warnings()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CrossUnitEvidenceAssessmentTest`
Expected: compilation error — types do not exist

- [ ] **Step 3: Implement EvidenceDiversityLevel enum**

Create `api/src/main/java/io/casehub/eidos/api/EvidenceDiversityLevel.java`:

```java
package io.casehub.eidos.api;

public enum EvidenceDiversityLevel {
    DIVERSE,
    HOMOGENEOUS,
    UNKNOWN
}
```

- [ ] **Step 4: Implement CrossUnitEvidenceAssessment record**

Create `api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessment.java`:

```java
package io.casehub.eidos.api;

import java.util.List;

public record CrossUnitEvidenceAssessment(
        int sameUnitCount,
        int crossUnitCount,
        int unknownCount,
        EvidenceDiversityLevel level,
        List<String> warnings
) {
    public static final CrossUnitEvidenceAssessment UNKNOWN =
            new CrossUnitEvidenceAssessment(0, 0, 0, EvidenceDiversityLevel.UNKNOWN, List.of());

    public CrossUnitEvidenceAssessment {
        if (sameUnitCount < 0 || crossUnitCount < 0 || unknownCount < 0) {
            throw new IllegalArgumentException("Counts must be non-negative");
        }
        warnings = List.copyOf(warnings);
    }
}
```

- [ ] **Step 5: Implement CrossUnitEvidenceAssessor SPI**

Create `api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessor.java`:

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Map;

public interface CrossUnitEvidenceAssessor {

    CrossUnitEvidenceAssessment assess(List<Map<String, Object>> snapshots);
}
```

- [ ] **Step 6: Run CrossUnitEvidenceAssessment tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CrossUnitEvidenceAssessmentTest`
Expected: all 6 tests PASS

- [ ] **Step 7: Write NoOp test**

Create `eidos-core/src/test/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessorTest.java`:

```java
package io.casehub.eidos.core.trust;

import io.casehub.eidos.api.CrossUnitEvidenceAssessment;
import io.casehub.eidos.api.EvidenceDiversityLevel;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class NoOpCrossUnitEvidenceAssessorTest {

    private final NoOpCrossUnitEvidenceAssessor assessor = new NoOpCrossUnitEvidenceAssessor();

    @Test
    void returns_unknown_for_null() {
        var result = assessor.assess(null);
        assertThat(result).isEqualTo(CrossUnitEvidenceAssessment.UNKNOWN);
    }

    @Test
    void returns_unknown_for_populated_snapshots() {
        var result = assessor.assess(List.of(Map.of("org-distance", 1)));
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.UNKNOWN);
    }
}
```

- [ ] **Step 8: Implement NoOp**

Create `eidos-core/src/main/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessor.java`:

```java
package io.casehub.eidos.core.trust;

import io.casehub.eidos.api.CrossUnitEvidenceAssessment;
import io.casehub.eidos.api.CrossUnitEvidenceAssessor;

import java.util.List;
import java.util.Map;

public class NoOpCrossUnitEvidenceAssessor implements CrossUnitEvidenceAssessor {

    @Override
    public CrossUnitEvidenceAssessment assess(List<Map<String, Object>> snapshots) {
        return CrossUnitEvidenceAssessment.UNKNOWN;
    }
}
```

Note: `@DefaultBean` annotation is added when this is wired into a CDI context. The core module is framework-neutral (no CDI annotations). The CDI `@DefaultBean` producer lives in `runtime/` or `deployment/`.

- [ ] **Step 9: Run NoOp tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core -Dtest=NoOpCrossUnitEvidenceAssessorTest`
Expected: both tests PASS

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/EvidenceDiversityLevel.java api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessment.java api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessor.java api/src/test/java/io/casehub/eidos/api/CrossUnitEvidenceAssessmentTest.java eidos-core/src/main/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessor.java eidos-core/src/test/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessorTest.java
git commit -m "feat(#266): add CrossUnitEvidenceAssessor SPI + NoOp default

EvidenceDiversityLevel enum (DIVERSE, HOMOGENEOUS, UNKNOWN),
CrossUnitEvidenceAssessment record with compact constructor validation,
CrossUnitEvidenceAssessor SPI taking List<Map<String, Object>> snapshots.
NoOpCrossUnitEvidenceAssessor returns UNKNOWN for all inputs.

Refs #266"
```

---

## Batch 2: Default Implementation

### Task 3: DefaultCrossUnitEvidenceAssessor

**Files:**
- Create: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessor.java`
- Create: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessorTest.java`

**Interfaces:**
- Consumes: `OrgDistanceUtil.extractDistance(Map<String, Object>)` → `int` (from Task 1); `CrossUnitEvidenceAssessor` SPI (from Task 2)
- Produces: `DefaultCrossUnitEvidenceAssessor` `@ApplicationScoped` CDI bean

- [ ] **Step 1: Write DefaultCrossUnitEvidenceAssessor tests**

```java
package io.casehub.eidos.org.runtime.trust;

import io.casehub.eidos.api.EvidenceDiversityLevel;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class DefaultCrossUnitEvidenceAssessorTest {

    private final DefaultCrossUnitEvidenceAssessor assessor =
            new DefaultCrossUnitEvidenceAssessor(2);

    @Test
    void null_snapshots_returns_unknown() {
        var result = assessor.assess(null);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.UNKNOWN);
    }

    @Test
    void empty_snapshots_returns_unknown() {
        var result = assessor.assess(List.of());
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.UNKNOWN);
    }

    @Test
    void all_same_unit_returns_homogeneous() {
        var snapshots = List.of(
                Map.<String, Object>of("org-distance", 1),
                Map.<String, Object>of("org-distance", 1),
                Map.<String, Object>of("org-distance", 1));
        var result = assessor.assess(snapshots);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.HOMOGENEOUS);
        assertThat(result.sameUnitCount()).isEqualTo(3);
        assertThat(result.crossUnitCount()).isEqualTo(0);
        assertThat(result.warnings()).hasSize(1);
    }

    @Test
    void cross_unit_at_threshold_returns_diverse() {
        var snapshots = List.of(
                Map.<String, Object>of("org-distance", 1),
                Map.<String, Object>of("org-distance", 2),
                Map.<String, Object>of("org-distance", 3));
        var result = assessor.assess(snapshots);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.DIVERSE);
        assertThat(result.sameUnitCount()).isEqualTo(1);
        assertThat(result.crossUnitCount()).isEqualTo(2);
        assertThat(result.warnings()).isEmpty();
    }

    @Test
    void cross_unit_below_threshold_returns_homogeneous() {
        var snapshots = List.of(
                Map.<String, Object>of("org-distance", 1),
                Map.<String, Object>of("org-distance", 1),
                Map.<String, Object>of("org-distance", 2));
        var result = assessor.assess(snapshots);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.HOMOGENEOUS);
        assertThat(result.crossUnitCount()).isEqualTo(1);
    }

    @Test
    void all_unknown_snapshots_returns_unknown() {
        var snapshots = List.of(
                Map.<String, Object>of("other-key", "value"),
                Map.<String, Object>of("other-key", "value"));
        var result = assessor.assess(snapshots);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.UNKNOWN);
        assertThat(result.unknownCount()).isEqualTo(2);
    }

    @Test
    void mixed_known_and_unknown_uses_known_counts() {
        var snapshots = List.of(
                Map.<String, Object>of("org-distance", 1),
                Map.<String, Object>of("org-distance", 2),
                Map.<String, Object>of("org-distance", 3),
                Map.<String, Object>of("other-key", "value"));
        var result = assessor.assess(snapshots);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.DIVERSE);
        assertThat(result.unknownCount()).isEqualTo(1);
        assertThat(result.crossUnitCount()).isEqualTo(2);
    }

    @Test
    void null_entries_in_list_treated_as_unknown() {
        List<Map<String, Object>> snapshots = new java.util.ArrayList<>();
        snapshots.add(Map.of("org-distance", 1));
        snapshots.add(null);
        snapshots.add(Map.of("org-distance", 2));
        snapshots.add(Map.of("org-distance", 3));
        var result = assessor.assess(snapshots);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.DIVERSE);
        assertThat(result.unknownCount()).isEqualTo(1);
    }

    @Test
    void threshold_one_makes_single_cross_unit_sufficient() {
        var assessor1 = new DefaultCrossUnitEvidenceAssessor(1);
        var snapshots = List.of(
                Map.<String, Object>of("org-distance", 1),
                Map.<String, Object>of("org-distance", 2));
        var result = assessor1.assess(snapshots);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.DIVERSE);
    }

    @Test
    void invalid_threshold_rejected() {
        assertThatThrownBy(() -> new DefaultCrossUnitEvidenceAssessor(0))
                .isInstanceOf(IllegalArgumentException.class);
        assertThatThrownBy(() -> new DefaultCrossUnitEvidenceAssessor(-1))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void homogeneous_warning_includes_counts_and_threshold() {
        var snapshots = List.of(
                Map.<String, Object>of("org-distance", 1),
                Map.<String, Object>of("org-distance", 1));
        var result = assessor.assess(snapshots);
        assertThat(result.warnings().get(0))
                .contains("cross-unit: 0/2")
                .contains("threshold: 2");
    }

    @Test
    void all_cross_unit_returns_diverse() {
        var snapshots = List.of(
                Map.<String, Object>of("org-distance", 2),
                Map.<String, Object>of("org-distance", 3));
        var result = assessor.assess(snapshots);
        assertThat(result.level()).isEqualTo(EvidenceDiversityLevel.DIVERSE);
        assertThat(result.sameUnitCount()).isEqualTo(0);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultCrossUnitEvidenceAssessorTest`
Expected: compilation error — class does not exist

- [ ] **Step 3: Implement DefaultCrossUnitEvidenceAssessor**

Create `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessor.java`:

```java
package io.casehub.eidos.org.runtime.trust;

import io.casehub.eidos.api.CrossUnitEvidenceAssessment;
import io.casehub.eidos.api.CrossUnitEvidenceAssessor;
import io.casehub.eidos.api.EvidenceDiversityLevel;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class DefaultCrossUnitEvidenceAssessor implements CrossUnitEvidenceAssessor {

    private final int minCrossUnit;

    public DefaultCrossUnitEvidenceAssessor(
            @ConfigProperty(name = "casehub.eidos.trust.diversity.min-cross-unit",
                            defaultValue = "2")
            int minCrossUnit) {
        if (minCrossUnit < 1) {
            throw new IllegalArgumentException(
                    "minCrossUnit must be >= 1, got: " + minCrossUnit);
        }
        this.minCrossUnit = minCrossUnit;
    }

    @Override
    public CrossUnitEvidenceAssessment assess(List<Map<String, Object>> snapshots) {
        if (snapshots == null || snapshots.isEmpty()) {
            return CrossUnitEvidenceAssessment.UNKNOWN;
        }

        int sameUnit = 0, crossUnit = 0, unknown = 0;

        for (Map<String, Object> snapshot : snapshots) {
            int distance = OrgDistanceUtil.extractDistance(snapshot);
            if (distance == 1) {
                sameUnit++;
            } else if (distance > 1) {
                crossUnit++;
            } else {
                unknown++;
            }
        }

        if (sameUnit == 0 && crossUnit == 0) {
            return new CrossUnitEvidenceAssessment(0, 0, unknown,
                    EvidenceDiversityLevel.UNKNOWN, List.of());
        }

        EvidenceDiversityLevel level = crossUnit >= minCrossUnit
                ? EvidenceDiversityLevel.DIVERSE
                : EvidenceDiversityLevel.HOMOGENEOUS;

        List<String> warnings = new ArrayList<>();
        if (level == EvidenceDiversityLevel.HOMOGENEOUS) {
            int total = sameUnit + crossUnit;
            warnings.add("cross-unit: " + crossUnit + "/" + total
                    + " (threshold: " + minCrossUnit + ")");
        }

        return new CrossUnitEvidenceAssessment(sameUnit, crossUnit, unknown,
                level, warnings);
    }
}
```

- [ ] **Step 4: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultCrossUnitEvidenceAssessorTest`
Expected: all 12 tests PASS

- [ ] **Step 5: Run full module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,eidos-core,org-runtime`
Expected: all tests PASS across all three modules

- [ ] **Step 6: Commit**

```bash
git add org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessor.java org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessorTest.java
git commit -m "feat(#266): add DefaultCrossUnitEvidenceAssessor

@ApplicationScoped implementation reading org-distance from attestation
snapshots via OrgDistanceUtil. Configurable min-cross-unit threshold
(default 2). Returns DIVERSE/HOMOGENEOUS/UNKNOWN with warning details.

Closes #266"
```

## References

- [2026-11-23-cross-unit-evidence-threshold-design.md] — design spec this plan implements
- `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java` — existing extractDistance() logic to extract
- `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/OrgProximityRelevanceFactor.java` — existing SNAPSHOT_KEY to extract
- `api/src/main/java/io/casehub/eidos/api/GraphDataSufficiency.java` — parallel pattern
- `eidos-core/src/main/java/io/casehub/eidos/core/org/NoOpOrgAwareDispatch.java` — NoOp placement pattern
- GitHub #266 — focal issue
- GitHub #261 — parent epic
