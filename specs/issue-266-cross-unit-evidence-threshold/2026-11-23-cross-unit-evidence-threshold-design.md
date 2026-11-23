# Cross-Unit Evidence Threshold — Design Spec

**Issue:** casehubio/eidos#266
**Epic:** casehubio/eidos#261 (anti-gaming closure)

## Summary

Add a `CrossUnitEvidenceAssessor` SPI to `eidos-org-api` that evaluates whether a trust score's evidence pool includes sufficient cross-unit attestations. Trust scores computed entirely from same-unit attestations lack independent validation — they may be statistically confident but systematically biased. The assessor returns a typed `CrossUnitEvidenceAssessment` alongside the trust score so consumers (engine dispatch) can factor evidence diversity into routing decisions.

## Motivation

Mass-preserving normalisation in the trust computation pipeline prevents relevance factors from addressing this problem. A factor that uniformly discounts all attestations in a pure-same-unit batch produces a no-op after normalisation — the total evidence mass is preserved by design. Evidence diversity is a property of the evidence pool as a whole, not of individual attestation weights. It requires a separate signal path.

The parallel is `GraphDataSufficiency` in eidos-api: a typed assessment that accompanies query results without modifying them. Consumers decide what to do with the signal.

## Architecture

```
eidos-org-api
├── CrossUnitEvidenceAssessor.java       — SPI
├── CrossUnitEvidenceAssessment.java     — result record
└── EvidenceDiversityLevel.java          — enum: DIVERSE, HOMOGENEOUS, UNKNOWN

eidos-org-runtime (org-runtime/)
└── trust/
    ├── DefaultCrossUnitEvidenceAssessor.java  — @ApplicationScoped
    └── OrgDistanceUtil.java                   — shared extractDistance() utility

eidos-runtime (runtime/)
└── trust/
    └── NoOpCrossUnitEvidenceAssessor.java     — @DefaultBean, returns UNKNOWN
```

**Dependency direction:** No ledger changes. The SPI takes `List<Map<String, Object>>` — attestation relevance snapshots — with no dependency on ledger types. The engine extracts snapshots from `LedgerAttestation.relevanceSnapshot` and passes them.

## SPI Contract

### CrossUnitEvidenceAssessment

```java
package io.casehub.eidos.org.api;

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

### EvidenceDiversityLevel

```java
package io.casehub.eidos.org.api;

public enum EvidenceDiversityLevel {
    DIVERSE,       // cross-unit count >= threshold
    HOMOGENEOUS,   // cross-unit count < threshold, attestations exist
    UNKNOWN        // no snapshots or no attestations — eidos org not on classpath
}
```

### CrossUnitEvidenceAssessor

```java
package io.casehub.eidos.org.api;

import java.util.List;
import java.util.Map;

public interface CrossUnitEvidenceAssessor {

    CrossUnitEvidenceAssessment assess(List<Map<String, Object>> snapshots);
}
```

The input is attestation relevance snapshots — `List<Map<String, Object>>` where each map is the parsed `relevanceSnapshot` JSON from a `LedgerAttestation`. Null entries in the list (attestations with no snapshot) are treated as unknown.

## Implementation

### DefaultCrossUnitEvidenceAssessor

`@ApplicationScoped` in `eidos-org-runtime`. Reads `org-distance` from each snapshot via shared `OrgDistanceUtil.extractDistance()`.

```java
@ApplicationScoped
public class DefaultCrossUnitEvidenceAssessor implements CrossUnitEvidenceAssessor {

    private final int minCrossUnit;

    public DefaultCrossUnitEvidenceAssessor(
            @ConfigProperty(name = "casehub.eidos.trust.diversity.min-cross-unit",
                            defaultValue = "2")
            int minCrossUnit) {
        if (minCrossUnit < 1) {
            throw new IllegalArgumentException("minCrossUnit must be >= 1, got: " + minCrossUnit);
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

### OrgDistanceUtil

Shared static utility in `eidos-org-runtime`. Extracted from `IndependenceAwareRelevanceFactor.extractDistance()` — both the factor and the assessor use the same logic.

```java
public final class OrgDistanceUtil {

    static final String SNAPSHOT_KEY = "org-distance";

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

`IndependenceAwareRelevanceFactor` and `OrgProximityRelevanceFactor` are refactored to use `OrgDistanceUtil.extractDistance()` and `OrgDistanceUtil.SNAPSHOT_KEY` instead of their own private copies.

### NoOpCrossUnitEvidenceAssessor

`@DefaultBean @ApplicationScoped` in `eidos-runtime`. Returns UNKNOWN for all assessments — backward-compatible when eidos org module is not on the classpath.

```java
@DefaultBean
@ApplicationScoped
public class NoOpCrossUnitEvidenceAssessor implements CrossUnitEvidenceAssessor {

    @Override
    public CrossUnitEvidenceAssessment assess(List<Map<String, Object>> snapshots) {
        return CrossUnitEvidenceAssessment.UNKNOWN;
    }
}
```

## Engine Integration

The engine already builds attestation data for trust computation. After computing trust scores via `TrustScoreCalculator.computeAll()`, it extracts snapshots and calls the assessor:

```java
// Engine dispatch (illustrative — actual integration is engine's concern)
List<Map<String, Object>> snapshots = attestations.stream()
        .map(a -> parseSnapshot(a.relevanceSnapshot))
        .toList();
CrossUnitEvidenceAssessment diversity = assessor.assess(snapshots);
// Use diversity.level() alongside trustScore in routing decision
```

The assessment is per-capability: the engine calls `assess()` with the attestations for a specific capability's trust computation batch, not all attestations globally.

## Testing Strategy

### Unit Tests (pure Java, no CDI)

- `CrossUnitEvidenceAssessmentTest` — compact constructor validates non-negative counts, warnings immutable
- `OrgDistanceUtilTest` — extractDistance: null snapshot → -1, missing key → -1, non-Number → -1, valid distance → value
- `DefaultCrossUnitEvidenceAssessorTest`:
  - Empty/null snapshots → UNKNOWN
  - All same-unit (distance=1) → HOMOGENEOUS with warning
  - All cross-unit (distance>1) → DIVERSE (when >= threshold)
  - Mixed same+cross below threshold → HOMOGENEOUS
  - Mixed same+cross above threshold → DIVERSE
  - All unknown (no org-distance key) → UNKNOWN
  - Mixed known+unknown → level based on known counts only
  - Threshold=1 makes single cross-unit attestation sufficient
  - Invalid threshold (<1) → IllegalArgumentException
- `NoOpCrossUnitEvidenceAssessorTest` — always returns UNKNOWN

### Refactoring Tests

- `IndependenceAwareRelevanceFactor` — verify unchanged behavior after extractDistance() extraction
- `OrgProximityRelevanceFactor` — verify unchanged behavior after SNAPSHOT_KEY extraction

## Scope

**In scope:**
- `CrossUnitEvidenceAssessor` SPI in `eidos-org-api`
- `CrossUnitEvidenceAssessment` record and `EvidenceDiversityLevel` enum in `eidos-org-api`
- `DefaultCrossUnitEvidenceAssessor` in `eidos-org-runtime`
- `NoOpCrossUnitEvidenceAssessor` `@DefaultBean` in `eidos-runtime`
- `OrgDistanceUtil` shared utility in `eidos-org-runtime`
- Refactor `IndependenceAwareRelevanceFactor` and `OrgProximityRelevanceFactor` to use `OrgDistanceUtil`
- Unit tests for all new types and refactored usages

**Out of scope:**
- Engine integration (engine owns its dispatch logic — this provides the SPI)
- Dashboard/observability (#267 — separate issue)
- Persistence of diversity assessments (compute on demand)
- Ledger changes of any kind

## References

- `eidos/org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java` — existing partition logic and extractDistance()
- `eidos/org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/OrgProximityRelevanceFactor.java` — existing SNAPSHOT_KEY usage
- `eidos/api/src/main/java/io/casehub/eidos/api/GraphDataSufficiency.java` — parallel pattern for sufficiency assessments
- `ledger/ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreComputer.java` — mass-preserving normalisation (why a factor approach is a no-op)
- `ledger/ledger-core/src/main/java/io/casehub/ledger/core/trust/RelevanceComposer.java` — factor composition pipeline
- casehubio/eidos#261 — parent epic (anti-gaming closure)
- casehubio/eidos#257 — IndependenceAwareRelevanceFactor (same domain, provides counterweight)
- Attestation relevance design spec (`ledger/docs/specs/issue-231-org-scoped-attestation-weighting/`)
