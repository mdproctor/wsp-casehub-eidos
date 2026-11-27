# Anti-Gaming Observability Dashboard Signals Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #267 — anti-gaming observability dashboard signals
**Issue group:** #267

**Goal:** Surface per-agent concentration metrics from the anti-gaming countermeasures via a recording SPI and in-memory accumulator, so operators can monitor independence cap activation rates, evidence diversity, and attestor concentration.

**Architecture:** Factor-side recording into a `TrustObservabilityCollector` SPI (eidos-api). Sealed `TrustObservabilitySignal` with `IndependenceSignal` and `DiversitySignal` variants. Counter-based `InMemoryTrustObservabilityCollector` (persistence-memory). `CrossUnitEvidenceAssessor` SPI signature changed to accept context. NoOp POJO in eidos-core, wired via `EidosCoreProducer`.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (`@Alternative @Priority(1)`), `java.util.concurrent` atomics

## Global Constraints

- Java 21 source (Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn clean install` (not `./mvnw`)
- Pre-release stage: no backward-compatibility shims
- Module tiers: api (Tier 1, pure Java), eidos-core (zero CDI), runtime (CDI wiring), persistence-memory (in-memory @Alternative)
- eidos-core has zero CDI annotations — NoOp POJOs only
- `EidosCoreProducer` in runtime wires NoOps via `@Produces @DefaultBean`
- `DiversitySignal` wraps `CrossUnitEvidenceAssessment` (no field duplication)
- Counters are monotonic — no `clear()` method
- `avgSameUnitShare` is an approximation under concurrent writes (documented, acceptable)

---

## Batch 1: API types and SPI

### Task 1: TrustObservabilitySignal sealed interface and signal records

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/TrustObservabilitySignal.java`
- Test: `api/src/test/java/io/casehub/eidos/api/TrustObservabilitySignalTest.java`

**Interfaces:**
- Consumes: `CrossUnitEvidenceAssessment` (existing in api), `EvidenceDiversityLevel` (existing in api)
- Produces: `TrustObservabilitySignal` sealed interface with `IndependenceSignal` and `DiversitySignal` records

- [ ] **Step 1: Write failing test for IndependenceSignal construction and validation**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class TrustObservabilitySignalTest {

    @Test
    void independenceSignalStoresAllFields() {
        var signal = new TrustObservabilitySignal.IndependenceSignal(
                "tenant-1", "agent-1", "code-review", "behavioral", Instant.now(),
                5, 3, 0.625, true, 0.72);
        assertEquals("tenant-1", signal.tenancyId());
        assertEquals("agent-1", signal.scoredAgentId());
        assertEquals("code-review", signal.capabilityTag());
        assertEquals("behavioral", signal.trustDimension());
        assertEquals(5, signal.sameUnitCount());
        assertEquals(3, signal.crossUnitCount());
        assertEquals(0.625, signal.sameUnitShare(), 0.001);
        assertTrue(signal.capActivated());
        assertEquals(0.72, signal.appliedWeight(), 0.001);
    }

    @Test
    void independenceSignalRejectsNullTenancyId() {
        assertThrows(NullPointerException.class, () ->
                new TrustObservabilitySignal.IndependenceSignal(
                        null, "agent-1", "cap", "dim", Instant.now(),
                        1, 1, 0.5, false, 1.0));
    }

    @Test
    void independenceSignalRejectsNullScoredAgentId() {
        assertThrows(NullPointerException.class, () ->
                new TrustObservabilitySignal.IndependenceSignal(
                        "tenant-1", null, "cap", "dim", Instant.now(),
                        1, 1, 0.5, false, 1.0));
    }

    @Test
    void independenceSignalRejectsNegativeCounts() {
        assertThrows(IllegalArgumentException.class, () ->
                new TrustObservabilitySignal.IndependenceSignal(
                        "tenant-1", "agent-1", "cap", "dim", Instant.now(),
                        -1, 1, 0.5, false, 1.0));
    }

    @Test
    void independenceSignalRejectsNaNShare() {
        assertThrows(IllegalArgumentException.class, () ->
                new TrustObservabilitySignal.IndependenceSignal(
                        "tenant-1", "agent-1", "cap", "dim", Instant.now(),
                        1, 1, Double.NaN, false, 1.0));
    }

    @Test
    void independenceSignalRejectsNaNWeight() {
        assertThrows(IllegalArgumentException.class, () ->
                new TrustObservabilitySignal.IndependenceSignal(
                        "tenant-1", "agent-1", "cap", "dim", Instant.now(),
                        1, 1, 0.5, false, Double.NaN));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=TrustObservabilitySignalTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — class does not exist

- [ ] **Step 3: Write TrustObservabilitySignal**

```java
package io.casehub.eidos.api;

import java.time.Instant;
import java.util.Objects;

public sealed interface TrustObservabilitySignal {

    String tenancyId();
    String scoredAgentId();
    Instant occurredAt();

    record IndependenceSignal(String tenancyId, String scoredAgentId,
            String capabilityTag, String trustDimension, Instant occurredAt,
            int sameUnitCount, int crossUnitCount, double sameUnitShare,
            boolean capActivated, double appliedWeight)
            implements TrustObservabilitySignal {

        public IndependenceSignal {
            Objects.requireNonNull(tenancyId, "tenancyId must not be null");
            Objects.requireNonNull(scoredAgentId, "scoredAgentId must not be null");
            Objects.requireNonNull(occurredAt, "occurredAt must not be null");
            if (sameUnitCount < 0) throw new IllegalArgumentException("sameUnitCount must be >= 0");
            if (crossUnitCount < 0) throw new IllegalArgumentException("crossUnitCount must be >= 0");
            if (Double.isNaN(sameUnitShare) || sameUnitShare < 0.0 || sameUnitShare > 1.0) {
                throw new IllegalArgumentException("sameUnitShare must be in [0.0, 1.0], got: " + sameUnitShare);
            }
            if (Double.isNaN(appliedWeight) || appliedWeight < 0.0 || appliedWeight > 1.0) {
                throw new IllegalArgumentException("appliedWeight must be in [0.0, 1.0], got: " + appliedWeight);
            }
        }
    }

    record DiversitySignal(String tenancyId, String scoredAgentId,
            Instant occurredAt, CrossUnitEvidenceAssessment assessment)
            implements TrustObservabilitySignal {

        public DiversitySignal {
            Objects.requireNonNull(tenancyId, "tenancyId must not be null");
            Objects.requireNonNull(scoredAgentId, "scoredAgentId must not be null");
            Objects.requireNonNull(occurredAt, "occurredAt must not be null");
            Objects.requireNonNull(assessment, "assessment must not be null");
        }
    }
}
```

- [ ] **Step 4: Add DiversitySignal tests**

Append to the test class:

```java
@Test
void diversitySignalWrapsAssessment() {
    var assessment = new CrossUnitEvidenceAssessment(3, 2, 1,
            EvidenceDiversityLevel.DIVERSE, List.of());
    var signal = new TrustObservabilitySignal.DiversitySignal(
            "tenant-1", "agent-1", Instant.now(), assessment);
    assertEquals("tenant-1", signal.tenancyId());
    assertEquals("agent-1", signal.scoredAgentId());
    assertSame(assessment, signal.assessment());
    assertEquals(EvidenceDiversityLevel.DIVERSE, signal.assessment().level());
}

@Test
void diversitySignalRejectsNullAssessment() {
    assertThrows(NullPointerException.class, () ->
            new TrustObservabilitySignal.DiversitySignal(
                    "tenant-1", "agent-1", Instant.now(), null));
}
```

- [ ] **Step 5: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=TrustObservabilitySignalTest`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/TrustObservabilitySignal.java api/src/test/java/io/casehub/eidos/api/TrustObservabilitySignalTest.java
git commit -m "feat(#267): add TrustObservabilitySignal sealed interface with IndependenceSignal and DiversitySignal"
```

### Task 2: Snapshot types and TrustObservabilityCollector SPI

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/AgentObservabilitySnapshot.java`
- Create: `api/src/main/java/io/casehub/eidos/api/TrustObservabilitySnapshot.java`
- Create: `api/src/main/java/io/casehub/eidos/api/TrustObservabilityCollector.java`
- Test: `api/src/test/java/io/casehub/eidos/api/TrustObservabilitySnapshotTest.java`

**Interfaces:**
- Consumes: `EvidenceDiversityLevel` (existing in api)
- Produces: `AgentObservabilitySnapshot`, `TrustObservabilitySnapshot`, `TrustObservabilityCollector` SPI

- [ ] **Step 1: Write failing tests for snapshot types**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class TrustObservabilitySnapshotTest {

    @Test
    void agentSnapshotEmptyConstant() {
        var empty = AgentObservabilitySnapshot.EMPTY;
        assertEquals(0, empty.independenceComputationCount());
        assertEquals(0, empty.independenceCapActivations());
        assertEquals(0.0, empty.avgSameUnitShare());
        assertEquals(0, empty.diversityComputationCount());
        assertTrue(empty.diversityDistribution().isEmpty());
    }

    @Test
    void agentSnapshotStoresFields() {
        var dist = Map.of(EvidenceDiversityLevel.DIVERSE, 5L,
                EvidenceDiversityLevel.HOMOGENEOUS, 2L);
        var snap = new AgentObservabilitySnapshot(10, 3, 0.65, 7, dist);
        assertEquals(10, snap.independenceComputationCount());
        assertEquals(3, snap.independenceCapActivations());
        assertEquals(0.65, snap.avgSameUnitShare(), 0.001);
        assertEquals(7, snap.diversityComputationCount());
        assertEquals(5L, snap.diversityDistribution().get(EvidenceDiversityLevel.DIVERSE));
    }

    @Test
    void agentSnapshotDiversityDistributionIsImmutable() {
        var dist = new java.util.HashMap<>(Map.of(EvidenceDiversityLevel.DIVERSE, 5L));
        var snap = new AgentObservabilitySnapshot(1, 0, 0.5, 1, dist);
        assertThrows(UnsupportedOperationException.class, () ->
                snap.diversityDistribution().put(EvidenceDiversityLevel.UNKNOWN, 1L));
    }

    @Test
    void trustSnapshotEmptyConstant() {
        var empty = TrustObservabilitySnapshot.EMPTY;
        assertEquals(0, empty.independenceComputationCount());
        assertTrue(empty.perAgent().isEmpty());
    }

    @Test
    void trustSnapshotPerAgentIsImmutable() {
        assertThrows(UnsupportedOperationException.class, () ->
                TrustObservabilitySnapshot.EMPTY.perAgent()
                        .put("agent-1", AgentObservabilitySnapshot.EMPTY));
    }

    @Test
    void trustSnapshotRejectsNegativeCounts() {
        assertThrows(IllegalArgumentException.class, () ->
                new TrustObservabilitySnapshot(-1, 0, 0.0, 0, Map.of(), Map.of()));
    }

    @Test
    void agentSnapshotRejectsNaNAvg() {
        assertThrows(IllegalArgumentException.class, () ->
                new AgentObservabilitySnapshot(1, 0, Double.NaN, 0, Map.of()));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=TrustObservabilitySnapshotTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure

- [ ] **Step 3: Write AgentObservabilitySnapshot**

```java
package io.casehub.eidos.api;

import java.util.Map;
import java.util.Objects;

public record AgentObservabilitySnapshot(
        long independenceComputationCount,
        long independenceCapActivations,
        double avgSameUnitShare,
        long diversityComputationCount,
        Map<EvidenceDiversityLevel, Long> diversityDistribution) {

    public static final AgentObservabilitySnapshot EMPTY =
            new AgentObservabilitySnapshot(0, 0, 0.0, 0, Map.of());

    public AgentObservabilitySnapshot {
        Objects.requireNonNull(diversityDistribution, "diversityDistribution must not be null");
        if (independenceComputationCount < 0) throw new IllegalArgumentException("independenceComputationCount must be >= 0");
        if (independenceCapActivations < 0) throw new IllegalArgumentException("independenceCapActivations must be >= 0");
        if (diversityComputationCount < 0) throw new IllegalArgumentException("diversityComputationCount must be >= 0");
        if (Double.isNaN(avgSameUnitShare)) throw new IllegalArgumentException("avgSameUnitShare must not be NaN");
        diversityDistribution = Map.copyOf(diversityDistribution);
    }
}
```

- [ ] **Step 4: Write TrustObservabilitySnapshot**

```java
package io.casehub.eidos.api;

import java.util.Map;
import java.util.Objects;

public record TrustObservabilitySnapshot(
        long independenceComputationCount,
        long independenceCapActivations,
        double avgSameUnitShare,
        long diversityComputationCount,
        Map<EvidenceDiversityLevel, Long> diversityDistribution,
        Map<String, AgentObservabilitySnapshot> perAgent) {

    public static final TrustObservabilitySnapshot EMPTY =
            new TrustObservabilitySnapshot(0, 0, 0.0, 0, Map.of(), Map.of());

    public TrustObservabilitySnapshot {
        Objects.requireNonNull(diversityDistribution, "diversityDistribution must not be null");
        Objects.requireNonNull(perAgent, "perAgent must not be null");
        if (independenceComputationCount < 0) throw new IllegalArgumentException("independenceComputationCount must be >= 0");
        if (independenceCapActivations < 0) throw new IllegalArgumentException("independenceCapActivations must be >= 0");
        if (diversityComputationCount < 0) throw new IllegalArgumentException("diversityComputationCount must be >= 0");
        if (Double.isNaN(avgSameUnitShare)) throw new IllegalArgumentException("avgSameUnitShare must not be NaN");
        diversityDistribution = Map.copyOf(diversityDistribution);
        perAgent = Map.copyOf(perAgent);
    }
}
```

- [ ] **Step 5: Write TrustObservabilityCollector SPI**

```java
package io.casehub.eidos.api;

public interface TrustObservabilityCollector {

    void record(TrustObservabilitySignal signal);

    TrustObservabilitySnapshot snapshot(String tenancyId);

    AgentObservabilitySnapshot snapshot(String tenancyId, String agentId);
}
```

Note: the agent-level query returns `AgentObservabilitySnapshot` directly (no recursive nesting).

- [ ] **Step 6: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=TrustObservabilitySnapshotTest`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/AgentObservabilitySnapshot.java api/src/main/java/io/casehub/eidos/api/TrustObservabilitySnapshot.java api/src/main/java/io/casehub/eidos/api/TrustObservabilityCollector.java api/src/test/java/io/casehub/eidos/api/TrustObservabilitySnapshotTest.java
git commit -m "feat(#267): add snapshot types and TrustObservabilityCollector SPI"
```

### Task 3: Change CrossUnitEvidenceAssessor SPI and update implementations

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessor.java`
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessor.java`
- Modify: `eidos-core/src/test/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessorTest.java`
- Modify: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessor.java`
- Modify: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessorTest.java`

**Interfaces:**
- Consumes: `CrossUnitEvidenceAssessment` (existing)
- Produces: Updated `CrossUnitEvidenceAssessor.assess(String tenancyId, String scoredAgentId, List<Map<String, Object>> snapshots)`

- [ ] **Step 1: Update the SPI interface**

Change `CrossUnitEvidenceAssessor.java`:

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Map;

public interface CrossUnitEvidenceAssessor {

    CrossUnitEvidenceAssessment assess(String tenancyId, String scoredAgentId,
            List<Map<String, Object>> snapshots);
}
```

- [ ] **Step 2: Update NoOpCrossUnitEvidenceAssessor**

```java
package io.casehub.eidos.core.trust;

import io.casehub.eidos.api.CrossUnitEvidenceAssessment;
import io.casehub.eidos.api.CrossUnitEvidenceAssessor;

import java.util.List;
import java.util.Map;

public class NoOpCrossUnitEvidenceAssessor implements CrossUnitEvidenceAssessor {

    @Override
    public CrossUnitEvidenceAssessment assess(String tenancyId, String scoredAgentId,
            List<Map<String, Object>> snapshots) {
        return CrossUnitEvidenceAssessment.UNKNOWN;
    }
}
```

- [ ] **Step 3: Update NoOpCrossUnitEvidenceAssessorTest**

Update test calls to pass the new parameters (tenancyId, scoredAgentId). The test verifies it still returns UNKNOWN regardless of context.

```java
@Test
void returnsUnknown() {
    var assessor = new NoOpCrossUnitEvidenceAssessor();
    assertEquals(CrossUnitEvidenceAssessment.UNKNOWN,
            assessor.assess("tenant-1", "agent-1", List.of()));
}
```

- [ ] **Step 4: Update DefaultCrossUnitEvidenceAssessor signature**

Change `assess()` signature to accept `String tenancyId, String scoredAgentId`. The implementation ignores these parameters for now — signal recording is added in Batch 3.

- [ ] **Step 5: Update DefaultCrossUnitEvidenceAssessorTest**

Update all test calls to pass tenancyId and scoredAgentId. Example:

```java
// Before:
assessor.assess(snapshots)
// After:
assessor.assess("tenant-1", "agent-1", snapshots)
```

- [ ] **Step 6: Run tests for affected modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,eidos-core,org-runtime`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessor.java eidos-core/src/main/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessor.java eidos-core/src/test/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessorTest.java org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessor.java org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessorTest.java
git commit -m "feat(#267): change CrossUnitEvidenceAssessor SPI to accept tenancyId and scoredAgentId"
```

## Batch 2: NoOp and in-memory implementations

### Task 4: NoOp collector and EidosCoreProducer wiring

**Files:**
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/trust/NoOpTrustObservabilityCollector.java`
- Create: `eidos-core/src/test/java/io/casehub/eidos/core/trust/NoOpTrustObservabilityCollectorTest.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java`

**Interfaces:**
- Consumes: `TrustObservabilityCollector`, `TrustObservabilitySignal`, `TrustObservabilitySnapshot`, `AgentObservabilitySnapshot`
- Produces: `NoOpTrustObservabilityCollector` (plain POJO), `@Produces @DefaultBean` wiring

- [ ] **Step 1: Write failing test for NoOp**

```java
package io.casehub.eidos.core.trust;

import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class NoOpTrustObservabilityCollectorTest {

    private final NoOpTrustObservabilityCollector collector = new NoOpTrustObservabilityCollector();

    @Test
    void recordDoesNotThrow() {
        var signal = new TrustObservabilitySignal.IndependenceSignal(
                "t", "a", "cap", "dim", Instant.now(), 1, 1, 0.5, false, 1.0);
        assertDoesNotThrow(() -> collector.record(signal));
    }

    @Test
    void tenancySnapshotReturnsEmpty() {
        assertEquals(TrustObservabilitySnapshot.EMPTY, collector.snapshot("tenant-1"));
    }

    @Test
    void agentSnapshotReturnsEmpty() {
        assertEquals(AgentObservabilitySnapshot.EMPTY, collector.snapshot("tenant-1", "agent-1"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core -Dtest=NoOpTrustObservabilityCollectorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure

- [ ] **Step 3: Write NoOpTrustObservabilityCollector**

```java
package io.casehub.eidos.core.trust;

import io.casehub.eidos.api.AgentObservabilitySnapshot;
import io.casehub.eidos.api.TrustObservabilityCollector;
import io.casehub.eidos.api.TrustObservabilitySignal;
import io.casehub.eidos.api.TrustObservabilitySnapshot;

public class NoOpTrustObservabilityCollector implements TrustObservabilityCollector {

    @Override
    public void record(TrustObservabilitySignal signal) {
    }

    @Override
    public TrustObservabilitySnapshot snapshot(String tenancyId) {
        return TrustObservabilitySnapshot.EMPTY;
    }

    @Override
    public AgentObservabilitySnapshot snapshot(String tenancyId, String agentId) {
        return AgentObservabilitySnapshot.EMPTY;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core -Dtest=NoOpTrustObservabilityCollectorTest`
Expected: PASS

- [ ] **Step 5: Add producer method to EidosCoreProducer**

Add to `EidosCoreProducer.java` (after the existing `crossUnitEvidenceAssessor()` method):

```java
@Produces
@DefaultBean
TrustObservabilityCollector trustObservabilityCollector() {
    return new NoOpTrustObservabilityCollector();
}
```

Add imports for `TrustObservabilityCollector` and `NoOpTrustObservabilityCollector`.

- [ ] **Step 6: Run full build to verify wiring**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eidos-core,runtime`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add eidos-core/src/main/java/io/casehub/eidos/core/trust/NoOpTrustObservabilityCollector.java eidos-core/src/test/java/io/casehub/eidos/core/trust/NoOpTrustObservabilityCollectorTest.java runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java
git commit -m "feat(#267): add NoOpTrustObservabilityCollector with EidosCoreProducer wiring"
```

### Task 5: InMemoryTrustObservabilityCollector

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryTrustObservabilityCollector.java`
- Create: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryTrustObservabilityCollectorTest.java`

**Interfaces:**
- Consumes: `TrustObservabilityCollector`, `TrustObservabilitySignal`, `TrustObservabilitySnapshot`, `AgentObservabilitySnapshot`, `EvidenceDiversityLevel`, `CrossUnitEvidenceAssessment`
- Produces: `InMemoryTrustObservabilityCollector` (`@Alternative @Priority(1)`)

- [ ] **Step 1: Write failing tests for record + snapshot round-trip**

```java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class InMemoryTrustObservabilityCollectorTest {

    private InMemoryTrustObservabilityCollector collector;

    @BeforeEach
    void setUp() {
        collector = new InMemoryTrustObservabilityCollector();
    }

    @Test
    void emptySnapshotWhenNoSignals() {
        var snap = collector.snapshot("tenant-1");
        assertEquals(0, snap.independenceComputationCount());
        assertEquals(0, snap.independenceCapActivations());
        assertEquals(0.0, snap.avgSameUnitShare());
        assertEquals(0, snap.diversityComputationCount());
        assertTrue(snap.perAgent().isEmpty());
    }

    @Test
    void emptyAgentSnapshotWhenNoSignals() {
        var snap = collector.snapshot("tenant-1", "agent-1");
        assertEquals(AgentObservabilitySnapshot.EMPTY, snap);
    }

    @Test
    void independenceSignalAccumulatesCorrectly() {
        collector.record(new TrustObservabilitySignal.IndependenceSignal(
                "t1", "a1", "cap", "dim", Instant.now(), 4, 2, 0.667, true, 0.72));
        collector.record(new TrustObservabilitySignal.IndependenceSignal(
                "t1", "a1", "cap", "dim", Instant.now(), 2, 4, 0.333, false, 1.0));

        var snap = collector.snapshot("t1");
        assertEquals(2, snap.independenceComputationCount());
        assertEquals(1, snap.independenceCapActivations());
        assertEquals(0.5, snap.avgSameUnitShare(), 0.01);
    }

    @Test
    void diversitySignalAccumulatesCorrectly() {
        collector.record(new TrustObservabilitySignal.DiversitySignal(
                "t1", "a1", Instant.now(),
                new CrossUnitEvidenceAssessment(3, 2, 0, EvidenceDiversityLevel.DIVERSE, List.of())));
        collector.record(new TrustObservabilitySignal.DiversitySignal(
                "t1", "a1", Instant.now(),
                new CrossUnitEvidenceAssessment(5, 0, 1, EvidenceDiversityLevel.HOMOGENEOUS, List.of())));

        var snap = collector.snapshot("t1");
        assertEquals(2, snap.diversityComputationCount());
        assertEquals(1L, snap.diversityDistribution().get(EvidenceDiversityLevel.DIVERSE));
        assertEquals(1L, snap.diversityDistribution().get(EvidenceDiversityLevel.HOMOGENEOUS));
    }

    @Test
    void perAgentBreakdownInTenancySnapshot() {
        collector.record(new TrustObservabilitySignal.IndependenceSignal(
                "t1", "a1", "cap", "dim", Instant.now(), 4, 2, 0.667, true, 0.72));
        collector.record(new TrustObservabilitySignal.IndependenceSignal(
                "t1", "a2", "cap", "dim", Instant.now(), 1, 5, 0.167, false, 1.0));

        var snap = collector.snapshot("t1");
        assertEquals(2, snap.independenceComputationCount());
        assertEquals(2, snap.perAgent().size());
        assertTrue(snap.perAgent().containsKey("a1"));
        assertTrue(snap.perAgent().containsKey("a2"));
        assertEquals(1, snap.perAgent().get("a1").independenceCapActivations());
        assertEquals(0, snap.perAgent().get("a2").independenceCapActivations());
    }

    @Test
    void agentSnapshotReturnsAgentOnly() {
        collector.record(new TrustObservabilitySignal.IndependenceSignal(
                "t1", "a1", "cap", "dim", Instant.now(), 4, 2, 0.667, true, 0.72));
        collector.record(new TrustObservabilitySignal.IndependenceSignal(
                "t1", "a2", "cap", "dim", Instant.now(), 1, 5, 0.167, false, 1.0));

        var a1 = collector.snapshot("t1", "a1");
        assertEquals(1, a1.independenceComputationCount());
        assertEquals(1, a1.independenceCapActivations());

        var a2 = collector.snapshot("t1", "a2");
        assertEquals(1, a2.independenceComputationCount());
        assertEquals(0, a2.independenceCapActivations());
    }

    @Test
    void differentTenanciesAreIsolated() {
        collector.record(new TrustObservabilitySignal.IndependenceSignal(
                "t1", "a1", "cap", "dim", Instant.now(), 4, 2, 0.667, true, 0.72));
        collector.record(new TrustObservabilitySignal.IndependenceSignal(
                "t2", "a1", "cap", "dim", Instant.now(), 1, 5, 0.167, false, 1.0));

        assertEquals(1, collector.snapshot("t1").independenceComputationCount());
        assertEquals(1, collector.snapshot("t2").independenceComputationCount());
        assertEquals(1, collector.snapshot("t1").independenceCapActivations());
        assertEquals(0, collector.snapshot("t2").independenceCapActivations());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryTrustObservabilityCollectorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure

- [ ] **Step 3: Write InMemoryTrustObservabilityCollector**

```java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.*;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import java.util.EnumMap;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.DoubleAdder;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryTrustObservabilityCollector implements TrustObservabilityCollector {

    private final ConcurrentHashMap<String, ConcurrentHashMap<String, AgentAccumulator>> store =
            new ConcurrentHashMap<>();

    @Override
    public void record(TrustObservabilitySignal signal) {
        var tenancyMap = store.computeIfAbsent(signal.tenancyId(), k -> new ConcurrentHashMap<>());
        var acc = tenancyMap.computeIfAbsent(signal.scoredAgentId(), k -> new AgentAccumulator());

        switch (signal) {
            case TrustObservabilitySignal.IndependenceSignal s -> {
                acc.independenceCount.incrementAndGet();
                if (s.capActivated()) acc.capActivations.incrementAndGet();
                acc.sameUnitShareSum.add(s.sameUnitShare());
            }
            case TrustObservabilitySignal.DiversitySignal s -> {
                acc.diversityCount.incrementAndGet();
                acc.diversityCounts
                        .computeIfAbsent(s.assessment().level(), k -> new AtomicLong())
                        .incrementAndGet();
            }
        }
    }

    @Override
    public TrustObservabilitySnapshot snapshot(String tenancyId) {
        var tenancyMap = store.get(tenancyId);
        if (tenancyMap == null) return TrustObservabilitySnapshot.EMPTY;

        long totalIndependence = 0, totalCap = 0, totalDiversity = 0;
        double totalShare = 0.0;
        Map<EvidenceDiversityLevel, Long> totalDiversityDist = new EnumMap<>(EvidenceDiversityLevel.class);
        Map<String, AgentObservabilitySnapshot> perAgent = new LinkedHashMap<>();

        for (var entry : tenancyMap.entrySet()) {
            var agentSnap = buildAgentSnapshot(entry.getValue());
            perAgent.put(entry.getKey(), agentSnap);

            totalIndependence += agentSnap.independenceComputationCount();
            totalCap += agentSnap.independenceCapActivations();
            totalDiversity += agentSnap.diversityComputationCount();
            totalShare += agentSnap.avgSameUnitShare() * agentSnap.independenceComputationCount();

            agentSnap.diversityDistribution().forEach((level, count) ->
                    totalDiversityDist.merge(level, count, Long::sum));
        }

        double avgShare = totalIndependence > 0 ? totalShare / totalIndependence : 0.0;

        return new TrustObservabilitySnapshot(
                totalIndependence, totalCap, avgShare,
                totalDiversity, totalDiversityDist, perAgent);
    }

    @Override
    public AgentObservabilitySnapshot snapshot(String tenancyId, String agentId) {
        var tenancyMap = store.get(tenancyId);
        if (tenancyMap == null) return AgentObservabilitySnapshot.EMPTY;
        var acc = tenancyMap.get(agentId);
        if (acc == null) return AgentObservabilitySnapshot.EMPTY;
        return buildAgentSnapshot(acc);
    }

    private AgentObservabilitySnapshot buildAgentSnapshot(AgentAccumulator acc) {
        long indCount = acc.independenceCount.get();
        long capCount = acc.capActivations.get();
        double shareSum = acc.sameUnitShareSum.sum();
        long divCount = acc.diversityCount.get();

        double avgShare = indCount > 0 ? shareSum / indCount : 0.0;

        Map<EvidenceDiversityLevel, Long> dist = new EnumMap<>(EvidenceDiversityLevel.class);
        acc.diversityCounts.forEach((level, count) -> {
            long val = count.get();
            if (val > 0) dist.put(level, val);
        });

        return new AgentObservabilitySnapshot(indCount, capCount, avgShare, divCount, dist);
    }

    private static class AgentAccumulator {
        final AtomicLong independenceCount = new AtomicLong();
        final AtomicLong capActivations = new AtomicLong();
        final DoubleAdder sameUnitShareSum = new DoubleAdder();
        final AtomicLong diversityCount = new AtomicLong();
        final ConcurrentHashMap<EvidenceDiversityLevel, AtomicLong> diversityCounts = new ConcurrentHashMap<>();
    }
}
```

- [ ] **Step 4: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryTrustObservabilityCollectorTest`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryTrustObservabilityCollector.java persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryTrustObservabilityCollectorTest.java
git commit -m "feat(#267): add InMemoryTrustObservabilityCollector with counter-based accumulators"
```

## Batch 3: Factor instrumentation

### Task 6: Instrument IndependenceAwareRelevanceFactor

**Files:**
- Modify: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java`
- Modify: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactorTest.java`

**Interfaces:**
- Consumes: `TrustObservabilityCollector`, `TrustObservabilitySignal.IndependenceSignal`, `AttestationRelevanceContext`
- Produces: Records `IndependenceSignal` after `assessBatch()`

- [ ] **Step 1: Write failing test for signal recording**

Add to `IndependenceAwareRelevanceFactorTest.java`:

```java
@Test
void assessBatchRecordsIndependenceSignal() {
    var collector = new InMemoryTrustObservabilityCollector();
    var factor = new IndependenceAwareRelevanceFactor(0.6, collector);

    // 3 same-unit, 1 cross-unit → 75% > 60% cap → cap activates
    var ctxs = List.of(
            context("att-1", "attestor-1", "scored-1", "tenant-1", "cap-1", "behavioral", 1),
            context("att-2", "attestor-2", "scored-1", "tenant-1", "cap-1", "behavioral", 1),
            context("att-3", "attestor-3", "scored-1", "tenant-1", "cap-1", "behavioral", 1),
            context("att-4", "attestor-4", "scored-1", "tenant-1", "cap-1", "behavioral", 2));

    factor.assessBatch(ctxs);

    var snap = collector.snapshot("tenant-1", "scored-1");
    assertEquals(1, snap.independenceComputationCount());
    assertEquals(1, snap.independenceCapActivations());
    assertEquals(0.75, snap.avgSameUnitShare(), 0.01);
}

@Test
void assessBatchRecordsSignalWhenCapNotActivated() {
    var collector = new InMemoryTrustObservabilityCollector();
    var factor = new IndependenceAwareRelevanceFactor(0.6, collector);

    // 1 same-unit, 3 cross-unit → 25% < 60% cap → no activation
    var ctxs = List.of(
            context("att-1", "attestor-1", "scored-1", "tenant-1", "cap-1", "behavioral", 1),
            context("att-2", "attestor-2", "scored-1", "tenant-1", "cap-1", "behavioral", 2),
            context("att-3", "attestor-3", "scored-1", "tenant-1", "cap-1", "behavioral", 2),
            context("att-4", "attestor-4", "scored-1", "tenant-1", "cap-1", "behavioral", 3));

    factor.assessBatch(ctxs);

    var snap = collector.snapshot("tenant-1", "scored-1");
    assertEquals(1, snap.independenceComputationCount());
    assertEquals(0, snap.independenceCapActivations());
    assertEquals(0.25, snap.avgSameUnitShare(), 0.01);
}
```

The `context()` helper method creates an `AttestationRelevanceContext` with scoredActorId, tenancyId, capabilityTag, trustDimension, and org-distance in the snapshot. If this helper doesn't exist yet, create it in the test class:

```java
private static AttestationRelevanceContext context(String attId, String attestorId,
        String scoredActorId, String tenancyId, String capabilityTag,
        String trustDimension, int orgDistance) {
    return new AttestationRelevanceContext(
            UUID.fromString("00000000-0000-0000-0000-" + attId.hashCode()),
            attestorId, ActorType.AGENT, null,
            scoredActorId, tenancyId, capabilityTag, trustDimension,
            Instant.now(), Map.of("org-distance", orgDistance));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=IndependenceAwareRelevanceFactorTest#assessBatchRecordsIndependenceSignal`
Expected: Compilation failure (constructor doesn't accept collector)

- [ ] **Step 3: Modify IndependenceAwareRelevanceFactor to inject and record**

Add `TrustObservabilityCollector` as a constructor parameter. After computing same/cross unit counts and the cap decision in `assessBatch()`, record:

```java
private final TrustObservabilityCollector observabilityCollector;

public IndependenceAwareRelevanceFactor(
        @ConfigProperty(name = "casehub.eidos.trust.independence.max-same-unit-share",
                        defaultValue = "0.6")
        double maxSameUnitShare,
        TrustObservabilityCollector observabilityCollector) {
    // existing validation...
    this.observabilityCollector = observabilityCollector;
}
```

At the end of `assessBatch()`, before returning `result`, add:

```java
if (!ctxs.isEmpty()) {
    var first = ctxs.getFirst();
    int knownCount = sameCount + crossCount;
    double share = knownCount > 0 ? (double) sameCount / knownCount : 0.0;
    boolean activated = share > maxSameUnitShare && crossCount > 0 && sameCount > 0;
    double appliedWeight = activated ? capped.weight() : 1.0;
    observabilityCollector.record(new TrustObservabilitySignal.IndependenceSignal(
            first.tenancyId(), first.scoredActorId(),
            first.capabilityTag(), first.trustDimension(), first.occurredAt(),
            sameCount, crossCount, share, activated, appliedWeight));
}
```

Note: `sameCount` and `crossCount` are already computed local variables. The `capped` variable holds the downweighted `RelevanceAssessment` — extract `.weight()` from it. When cap doesn't activate, `appliedWeight` is 1.0.

- [ ] **Step 4: Update existing tests to pass collector**

All existing test constructors need updating: `new IndependenceAwareRelevanceFactor(0.6)` → `new IndependenceAwareRelevanceFactor(0.6, new NoOpTrustObservabilityCollector())`.

- [ ] **Step 5: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=IndependenceAwareRelevanceFactorTest`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactorTest.java
git commit -m "feat(#267): instrument IndependenceAwareRelevanceFactor to record observability signals"
```

### Task 7: Instrument DefaultCrossUnitEvidenceAssessor

**Files:**
- Modify: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessor.java`
- Modify: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessorTest.java`

**Interfaces:**
- Consumes: `TrustObservabilityCollector`, `TrustObservabilitySignal.DiversitySignal`, `CrossUnitEvidenceAssessment`
- Produces: Records `DiversitySignal` after `assess()`

- [ ] **Step 1: Write failing test for signal recording**

Add to `DefaultCrossUnitEvidenceAssessorTest.java`:

```java
@Test
void assessRecordsDiversitySignal() {
    var collector = new InMemoryTrustObservabilityCollector();
    var assessor = new DefaultCrossUnitEvidenceAssessor(2, collector);

    var snapshots = List.of(
            Map.<String, Object>of("org-distance", 1),
            Map.<String, Object>of("org-distance", 2),
            Map.<String, Object>of("org-distance", 3));

    assessor.assess("tenant-1", "agent-1", snapshots);

    var snap = collector.snapshot("tenant-1", "agent-1");
    assertEquals(1, snap.diversityComputationCount());
    assertEquals(1L, snap.diversityDistribution().get(EvidenceDiversityLevel.DIVERSE));
}

@Test
void assessRecordsHomogeneousSignal() {
    var collector = new InMemoryTrustObservabilityCollector();
    var assessor = new DefaultCrossUnitEvidenceAssessor(2, collector);

    var snapshots = List.of(
            Map.<String, Object>of("org-distance", 1),
            Map.<String, Object>of("org-distance", 1));

    assessor.assess("tenant-1", "agent-1", snapshots);

    var snap = collector.snapshot("tenant-1", "agent-1");
    assertEquals(1, snap.diversityComputationCount());
    assertEquals(1L, snap.diversityDistribution().get(EvidenceDiversityLevel.HOMOGENEOUS));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultCrossUnitEvidenceAssessorTest#assessRecordsDiversitySignal`
Expected: Compilation failure (constructor doesn't accept collector)

- [ ] **Step 3: Add collector injection and recording**

```java
private final TrustObservabilityCollector observabilityCollector;

public DefaultCrossUnitEvidenceAssessor(
        @ConfigProperty(name = "casehub.eidos.trust.diversity.min-cross-unit",
                        defaultValue = "2")
        int minCrossUnit,
        TrustObservabilityCollector observabilityCollector) {
    // existing validation...
    this.observabilityCollector = observabilityCollector;
}

@Override
public CrossUnitEvidenceAssessment assess(String tenancyId, String scoredAgentId,
        List<Map<String, Object>> snapshots) {
    // ... existing assessment logic unchanged ...

    // Record signal after computing assessment
    observabilityCollector.record(new TrustObservabilitySignal.DiversitySignal(
            tenancyId, scoredAgentId, Instant.now(), result));

    return result;
}
```

Where `result` is the `CrossUnitEvidenceAssessment` that the method already computes and returns. Capture it in a local variable before the return statement.

- [ ] **Step 4: Update existing tests to pass collector**

All existing test constructors: `new DefaultCrossUnitEvidenceAssessor(2)` → `new DefaultCrossUnitEvidenceAssessor(2, new NoOpTrustObservabilityCollector())`.

- [ ] **Step 5: Run all tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultCrossUnitEvidenceAssessorTest`
Expected: All tests PASS

- [ ] **Step 6: Run full build to verify everything compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All modules pass. Any test that constructs `IndependenceAwareRelevanceFactor` or `DefaultCrossUnitEvidenceAssessor` directly may fail — check CDI test classes in `org-runtime` and `examples/` and update constructor calls.

- [ ] **Step 7: Commit**

```bash
git add org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessor.java org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessorTest.java
git commit -m "feat(#267): instrument DefaultCrossUnitEvidenceAssessor to record diversity signals"
```

## References

- [2026-11-27-anti-gaming-observability-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessor.java` — SPI being changed
- `api/src/main/java/io/casehub/eidos/api/CrossUnitEvidenceAssessment.java` — reused by DiversitySignal
- `eidos-core/src/main/java/io/casehub/eidos/core/trust/NoOpCrossUnitEvidenceAssessor.java` — NoOp pattern
- `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java:192-196` — producer wiring pattern
- `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryBehavioralSignalStore.java` — in-memory @Alternative pattern
- `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java` — factor to instrument
- `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/DefaultCrossUnitEvidenceAssessor.java` — assessor to instrument
- GitHub #267 — focal issue
- GitHub #261 — anti-gaming closure epic
