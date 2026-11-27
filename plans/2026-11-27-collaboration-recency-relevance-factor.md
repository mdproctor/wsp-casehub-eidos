# Collaboration-Recency Attestation Relevance Factor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #263 — collaboration-recency attestation relevance factor
**Issue group:** #263

**Goal:** Add an enricher/factor pair that upweights attestors who recently
collaborated with the scored agent on the same work item.

**Architecture:** Enricher queries `AgentGraphQuery` for both agents' task
histories, intersects `externalRef` values to find shared work items, writes
`collaboration-recency-days` (elapsed days since most recent shared task) to
the attestation's `relevanceSnapshot` JSON. Factor reads the value and applies
exponential decay: `weight = baseline + (1-baseline) × e^(-λ × days)`.

**Tech Stack:** Java 21, Quarkus CDI, Jackson, JUnit 5, Mockito, AssertJ

## Global Constraints

- Java 21 source (Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn` (not `./mvnw`)
- All classes in `io.casehub.eidos.runtime.trust` package
- Snapshot key: `"collaboration-recency-days"` (double, days since last shared task)
- Weight range: `[0.0, 1.0]`; `RelevanceAssessment.NEUTRAL` (1.0) when no data
- Fail-open: enricher must never block attestation persistence
- Use `ide-tooling` for code navigation and editing

---

## Batch 1: Utility and Factor (read-side, no I/O dependencies)

### Task 1: CollaborationRecencyUtil and CollaborationRecencyRelevanceFactor

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyUtil.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyRelevanceFactor.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyUtilTest.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyRelevanceFactorTest.java`

**Interfaces:**
- Consumes: `AttestationRelevanceFactor` (ledger-api), `AttestationRelevanceContext` (ledger-api), `RelevanceAssessment` (ledger-api)
- Produces: `CollaborationRecencyUtil.SNAPSHOT_KEY` → `"collaboration-recency-days"`, `CollaborationRecencyUtil.extractDays(Map<String, Object>) → OptionalDouble`; `CollaborationRecencyRelevanceFactor.assess(AttestationRelevanceContext) → RelevanceAssessment`

- [ ] **Step 1: Write CollaborationRecencyUtil**

```java
package io.casehub.eidos.runtime.trust;

import java.util.Map;
import java.util.OptionalDouble;

public final class CollaborationRecencyUtil {

    public static final String SNAPSHOT_KEY = "collaboration-recency-days";

    public static OptionalDouble extractDays(Map<String, Object> snapshot) {
        if (snapshot == null || !snapshot.containsKey(SNAPSHOT_KEY)) {
            return OptionalDouble.empty();
        }
        Object raw = snapshot.get(SNAPSHOT_KEY);
        if (!(raw instanceof Number)) {
            return OptionalDouble.empty();
        }
        return OptionalDouble.of(((Number) raw).doubleValue());
    }

    private CollaborationRecencyUtil() {}
}
```

- [ ] **Step 2: Write CollaborationRecencyUtilTest**

```java
package io.casehub.eidos.runtime.trust;

import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class CollaborationRecencyUtilTest {

    @Test
    void null_snapshot_returns_empty() {
        assertThat(CollaborationRecencyUtil.extractDays(null)).isEmpty();
    }

    @Test
    void missing_key_returns_empty() {
        assertThat(CollaborationRecencyUtil.extractDays(Map.of("other", 42))).isEmpty();
    }

    @Test
    void non_numeric_value_returns_empty() {
        assertThat(CollaborationRecencyUtil.extractDays(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, "not-a-number"))).isEmpty();
    }

    @Test
    void integer_value_returns_as_double() {
        var result = CollaborationRecencyUtil.extractDays(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 7));
        assertThat(result).hasValue(7.0);
    }

    @Test
    void double_value_returned() {
        var result = CollaborationRecencyUtil.extractDays(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 3.5));
        assertThat(result).hasValue(3.5);
    }

    @Test
    void snapshot_key_value() {
        assertThat(CollaborationRecencyUtil.SNAPSHOT_KEY)
                .isEqualTo("collaboration-recency-days");
    }
}
```

- [ ] **Step 3: Run util tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CollaborationRecencyUtilTest`
Expected: 6 tests PASS

- [ ] **Step 4: Write CollaborationRecencyRelevanceFactor**

```java
package io.casehub.eidos.runtime.trust;

import io.casehub.ledger.api.spi.AttestationRelevanceContext;
import io.casehub.ledger.api.spi.AttestationRelevanceFactor;
import io.casehub.ledger.api.spi.RelevanceAssessment;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class CollaborationRecencyRelevanceFactor implements AttestationRelevanceFactor {

    private final double baseline;
    private final double lambda;

    public CollaborationRecencyRelevanceFactor(
            @ConfigProperty(name = "casehub.eidos.trust.collaboration.half-life-days",
                    defaultValue = "7.0")
            double halfLifeDays,
            @ConfigProperty(name = "casehub.eidos.trust.collaboration.baseline",
                    defaultValue = "0.5")
            double baseline) {
        if (halfLifeDays <= 0) {
            throw new IllegalArgumentException("half-life-days must be > 0, got: " + halfLifeDays);
        }
        if (Double.isNaN(baseline) || baseline < 0.0 || baseline > 1.0) {
            throw new IllegalArgumentException("baseline must be in [0.0, 1.0], got: " + baseline);
        }
        this.baseline = baseline;
        this.lambda = Math.log(2) / halfLifeDays;
    }

    @Override
    public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
        var optDays = CollaborationRecencyUtil.extractDays(ctx.snapshot());
        if (optDays.isEmpty()) {
            return RelevanceAssessment.NEUTRAL;
        }
        double days = optDays.getAsDouble();
        double weight = baseline + (1.0 - baseline) * Math.exp(-lambda * days);
        return new RelevanceAssessment(weight,
                CollaborationRecencyUtil.SNAPSHOT_KEY + ":" + days + ", weight:" + weight);
    }

    @Override
    public String name() {
        return "collaboration-recency";
    }
}
```

- [ ] **Step 5: Write CollaborationRecencyRelevanceFactorTest**

```java
package io.casehub.eidos.runtime.trust;

import io.casehub.ledger.api.spi.AttestationRelevanceContext;
import io.casehub.ledger.api.spi.RelevanceAssessment;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.data.Offset.offset;

class CollaborationRecencyRelevanceFactorTest {

    private static final double DEFAULT_HALF_LIFE = 7.0;
    private static final double DEFAULT_BASELINE = 0.5;

    private CollaborationRecencyRelevanceFactor factor() {
        return new CollaborationRecencyRelevanceFactor(DEFAULT_HALF_LIFE, DEFAULT_BASELINE);
    }

    private AttestationRelevanceContext ctx(Map<String, Object> snapshot) {
        return new AttestationRelevanceContext(
                UUID.randomUUID(), "attestor", ActorType.AGENT, null,
                "scored", "tenant", "cap", null, Instant.now(), snapshot);
    }

    @Test
    void null_snapshot_returns_neutral() {
        assertThat(factor().assess(ctx(null))).isEqualTo(RelevanceAssessment.NEUTRAL);
    }

    @Test
    void missing_key_returns_neutral() {
        assertThat(factor().assess(ctx(Map.of("other", 42))))
                .isEqualTo(RelevanceAssessment.NEUTRAL);
    }

    @Test
    void zero_days_returns_weight_one() {
        var result = factor().assess(ctx(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 0.0)));
        assertThat(result.weight()).isCloseTo(1.0, offset(0.001));
    }

    @Test
    void half_life_days_returns_midpoint() {
        var result = factor().assess(ctx(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 7.0)));
        // baseline + (1-baseline) * 0.5 = 0.5 + 0.25 = 0.75
        assertThat(result.weight()).isCloseTo(0.75, offset(0.01));
    }

    @Test
    void large_days_approaches_baseline() {
        var result = factor().assess(ctx(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 90.0)));
        assertThat(result.weight()).isCloseTo(DEFAULT_BASELINE, offset(0.01));
    }

    @Test
    void custom_half_life() {
        var custom = new CollaborationRecencyRelevanceFactor(14.0, DEFAULT_BASELINE);
        var result = custom.assess(ctx(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 14.0)));
        assertThat(result.weight()).isCloseTo(0.75, offset(0.01));
    }

    @Test
    void custom_baseline() {
        var custom = new CollaborationRecencyRelevanceFactor(DEFAULT_HALF_LIFE, 0.3);
        var atZero = custom.assess(ctx(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 0.0)));
        assertThat(atZero.weight()).isCloseTo(1.0, offset(0.001));

        var atLarge = custom.assess(ctx(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 90.0)));
        assertThat(atLarge.weight()).isCloseTo(0.3, offset(0.01));
    }

    @Test
    void reason_contains_key_and_weight() {
        var result = factor().assess(ctx(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, 3.0)));
        assertThat(result.reason()).contains("collaboration-recency-days:3.0");
        assertThat(result.reason()).contains("weight:");
    }

    @Test
    void name_returns_collaboration_recency() {
        assertThat(factor().name()).isEqualTo("collaboration-recency");
    }

    @Test
    void half_life_zero_throws() {
        assertThatThrownBy(() -> new CollaborationRecencyRelevanceFactor(0.0, DEFAULT_BASELINE))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void half_life_negative_throws() {
        assertThatThrownBy(() -> new CollaborationRecencyRelevanceFactor(-1.0, DEFAULT_BASELINE))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void baseline_negative_throws() {
        assertThatThrownBy(() -> new CollaborationRecencyRelevanceFactor(DEFAULT_HALF_LIFE, -0.1))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void baseline_above_one_throws() {
        assertThatThrownBy(() -> new CollaborationRecencyRelevanceFactor(DEFAULT_HALF_LIFE, 1.1))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void non_numeric_snapshot_value_returns_neutral() {
        assertThat(factor().assess(ctx(
                Map.of(CollaborationRecencyUtil.SNAPSHOT_KEY, "bad"))))
                .isEqualTo(RelevanceAssessment.NEUTRAL);
    }
}
```

- [ ] **Step 6: Run factor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CollaborationRecencyRelevanceFactorTest`
Expected: 13 tests PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyUtil.java \
       runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyRelevanceFactor.java \
       runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyUtilTest.java \
       runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyRelevanceFactorTest.java
git commit -m "feat(#263): add CollaborationRecencyUtil and CollaborationRecencyRelevanceFactor

Exponential decay with configurable half-life (7d) and baseline (0.5).
Reads collaboration-recency-days from attestation relevance snapshot.

Refs #263"
```

---

## Batch 2: Enricher (write-side, queries AgentGraphQuery)

### Task 2: CollaborationRecencyAttestationEnricher

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyAttestationEnricher.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyAttestationEnricherTest.java`

**Interfaces:**
- Consumes: `LedgerAttestationEnricher` (ledger-api), `AgentGraphQuery` (eidos-api), `AgentTaskHistory` (eidos-api), `AgentTask` (eidos-api), `CollaborationRecencyUtil.SNAPSHOT_KEY` (from Task 1)
- Produces: Writes `{"collaboration-recency-days": N}` to `LedgerAttestation.relevanceSnapshot` JSON

- [ ] **Step 1: Write CollaborationRecencyAttestationEnricher**

```java
package io.casehub.eidos.runtime.trust;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.eidos.api.AgentGraphQuery;
import io.casehub.eidos.api.AgentTask;
import io.casehub.eidos.api.AgentTaskHistory;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.spi.LedgerAttestationEnricher;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;
import java.util.Set;
import java.util.stream.Collectors;

@ApplicationScoped
public class CollaborationRecencyAttestationEnricher implements LedgerAttestationEnricher {

    private static final Logger LOG = Logger.getLogger(CollaborationRecencyAttestationEnricher.class);
    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final TypeReference<Map<String, Object>> MAP_TYPE = new TypeReference<>() {};
    private static final double MILLIS_PER_DAY = 86_400_000.0;

    private final Instance<AgentGraphQuery> graphQueryInstance;
    private final int lookbackDays;

    @Inject
    public CollaborationRecencyAttestationEnricher(
            Instance<AgentGraphQuery> graphQueryInstance,
            @ConfigProperty(name = "casehub.eidos.trust.collaboration.lookback-days",
                    defaultValue = "90")
            int lookbackDays) {
        this.graphQueryInstance = graphQueryInstance;
        this.lookbackDays = lookbackDays;
    }

    @Override
    public void enrich(LedgerAttestation attestation, LedgerEntry entry) {
        if (graphQueryInstance.isUnsatisfied()) {
            return;
        }

        AgentGraphQuery graphQuery;
        try {
            graphQuery = graphQueryInstance.get();
        } catch (Exception e) {
            LOG.warnf("Failed to resolve AgentGraphQuery: %s", e.getMessage());
            return;
        }

        try {
            Instant since = attestation.occurredAt != null
                    ? attestation.occurredAt.minus(lookbackDays, ChronoUnit.DAYS)
                    : Instant.now().minus(lookbackDays, ChronoUnit.DAYS);

            AgentTaskHistory attestorHistory = graphQuery.agentHistory(
                    attestation.attestorId, entry.tenancyId, since);
            AgentTaskHistory scoredHistory = graphQuery.agentHistory(
                    entry.actorId, entry.tenancyId, since);

            Set<String> attestorRefs = externalRefs(attestorHistory);
            Set<String> scoredRefs = externalRefs(scoredHistory);

            attestorRefs.retainAll(scoredRefs);
            if (attestorRefs.isEmpty()) {
                return;
            }

            Instant mostRecent = findMostRecentCollaboration(
                    attestorRefs, attestorHistory, scoredHistory);
            if (mostRecent == null) {
                return;
            }

            Instant reference = attestation.occurredAt != null
                    ? attestation.occurredAt : Instant.now();
            double daysSince = Math.max(0.0,
                    Duration.between(mostRecent, reference).toMillis() / MILLIS_PER_DAY);

            Map<String, Object> snapshot = attestation.relevanceSnapshot != null
                    ? MAPPER.readValue(attestation.relevanceSnapshot, MAP_TYPE)
                    : new LinkedHashMap<>();
            snapshot.put(CollaborationRecencyUtil.SNAPSHOT_KEY, daysSince);
            attestation.relevanceSnapshot = MAPPER.writeValueAsString(snapshot);

        } catch (JsonProcessingException e) {
            LOG.warnf("Failed to write collaboration-recency to relevance snapshot: %s",
                    e.getMessage());
        } catch (Exception e) {
            LOG.warnf("Collaboration-recency enrichment failed: %s", e.getMessage());
        }
    }

    @Override
    public String name() {
        return "collaboration-recency";
    }

    private static Set<String> externalRefs(AgentTaskHistory history) {
        return history.tasks().stream()
                .map(AgentTask::externalRef)
                .filter(Objects::nonNull)
                .collect(Collectors.toSet());
    }

    static Instant findMostRecentCollaboration(
            Set<String> sharedRefs,
            AgentTaskHistory attestorHistory,
            AgentTaskHistory scoredHistory) {

        Instant mostRecent = null;
        for (String ref : sharedRefs) {
            Instant attestorTime = mostRecentTaskTime(attestorHistory, ref);
            Instant scoredTime = mostRecentTaskTime(scoredHistory, ref);
            if (attestorTime == null || scoredTime == null) continue;

            // Collaboration timestamp = min of the two (latest point both participated)
            Instant collabTime = attestorTime.isBefore(scoredTime)
                    ? attestorTime : scoredTime;
            if (mostRecent == null || collabTime.isAfter(mostRecent)) {
                mostRecent = collabTime;
            }
        }
        return mostRecent;
    }

    private static Instant mostRecentTaskTime(AgentTaskHistory history, String ref) {
        Instant latest = null;
        for (AgentTask task : history.tasks()) {
            if (!ref.equals(task.externalRef())) continue;
            Instant taskTime = task.endedAt() != null ? task.endedAt() : task.startedAt();
            if (taskTime != null && (latest == null || taskTime.isAfter(latest))) {
                latest = taskTime;
            }
        }
        return latest;
    }
}
```

- [ ] **Step 2: Write CollaborationRecencyAttestationEnricherTest**

```java
package io.casehub.eidos.runtime.trust;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.eidos.api.*;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.data.Offset.offset;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class CollaborationRecencyAttestationEnricherTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Instant NOW = Instant.parse("2026-11-27T12:00:00Z");

    private AgentGraphQuery graphQuery;
    private Instance<AgentGraphQuery> graphQueryInstance;
    private CollaborationRecencyAttestationEnricher enricher;

    @SuppressWarnings("unchecked")
    @BeforeEach
    void setUp() {
        graphQuery = mock(AgentGraphQuery.class);
        graphQueryInstance = mock(Instance.class);
        when(graphQueryInstance.isUnsatisfied()).thenReturn(false);
        when(graphQueryInstance.get()).thenReturn(graphQuery);
        enricher = new CollaborationRecencyAttestationEnricher(graphQueryInstance, 90);
    }

    private LedgerAttestation attestation(String attestorId) {
        var a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.attestorId = attestorId;
        a.attestorType = ActorType.AGENT;
        a.occurredAt = NOW;
        a.relevanceSnapshot = null;
        return a;
    }

    private LedgerEntry entry(String actorId, String tenancyId) {
        return new TestLedgerEntry(actorId, tenancyId);
    }

    private AgentTaskHistory history(String agentId, String tenancyId, List<AgentTask> tasks) {
        return new AgentTaskHistory(agentId, tenancyId, tasks, List.of(), List.of(), List.of(),
                GraphDataSufficiency.empty(List.of()));
    }

    private AgentTask task(String agentId, String tenancyId, String externalRef,
                           Instant startedAt, Instant endedAt) {
        return new AgentTask("task-" + UUID.randomUUID(), agentId, tenancyId,
                "cap", null, null, externalRef, startedAt, endedAt);
    }

    @Test
    void name_returns_collaboration_recency() {
        assertThat(enricher.name()).isEqualTo("collaboration-recency");
    }

    @Test
    void shared_ref_writes_days_to_snapshot() throws Exception {
        Instant threeDaysAgo = NOW.minus(3, ChronoUnit.DAYS);
        when(graphQuery.agentHistory(eq("attestor-1"), eq("t1"), any()))
                .thenReturn(history("attestor-1", "t1", List.of(
                        task("attestor-1", "t1", "CASE-100", threeDaysAgo.minus(1, ChronoUnit.HOURS), threeDaysAgo))));
        when(graphQuery.agentHistory(eq("scored-1"), eq("t1"), any()))
                .thenReturn(history("scored-1", "t1", List.of(
                        task("scored-1", "t1", "CASE-100", threeDaysAgo.minus(2, ChronoUnit.HOURS), threeDaysAgo))));

        var att = attestation("attestor-1");
        enricher.enrich(att, entry("scored-1", "t1"));

        assertThat(att.relevanceSnapshot).isNotNull();
        Map<String, Object> snapshot = MAPPER.readValue(att.relevanceSnapshot, new TypeReference<>() {});
        double days = ((Number) snapshot.get(CollaborationRecencyUtil.SNAPSHOT_KEY)).doubleValue();
        assertThat(days).isCloseTo(3.0, offset(0.1));
    }

    @Test
    void no_shared_ref_leaves_snapshot_null() {
        when(graphQuery.agentHistory(eq("attestor-1"), eq("t1"), any()))
                .thenReturn(history("attestor-1", "t1", List.of(
                        task("attestor-1", "t1", "CASE-100", NOW.minus(1, ChronoUnit.DAYS), NOW))));
        when(graphQuery.agentHistory(eq("scored-1"), eq("t1"), any()))
                .thenReturn(history("scored-1", "t1", List.of(
                        task("scored-1", "t1", "CASE-200", NOW.minus(1, ChronoUnit.DAYS), NOW))));

        var att = attestation("attestor-1");
        enricher.enrich(att, entry("scored-1", "t1"));

        assertThat(att.relevanceSnapshot).isNull();
    }

    @Test
    void unsatisfied_instance_leaves_snapshot_null() {
        @SuppressWarnings("unchecked")
        Instance<AgentGraphQuery> unsatisfied = mock(Instance.class);
        when(unsatisfied.isUnsatisfied()).thenReturn(true);
        var e = new CollaborationRecencyAttestationEnricher(unsatisfied, 90);

        var att = attestation("attestor-1");
        e.enrich(att, entry("scored-1", "t1"));

        assertThat(att.relevanceSnapshot).isNull();
    }

    @Test
    void in_progress_task_uses_started_at() throws Exception {
        Instant startedAt = NOW.minus(1, ChronoUnit.DAYS);
        when(graphQuery.agentHistory(eq("attestor-1"), eq("t1"), any()))
                .thenReturn(history("attestor-1", "t1", List.of(
                        task("attestor-1", "t1", "CASE-100", startedAt, null))));
        when(graphQuery.agentHistory(eq("scored-1"), eq("t1"), any()))
                .thenReturn(history("scored-1", "t1", List.of(
                        task("scored-1", "t1", "CASE-100", startedAt, null))));

        var att = attestation("attestor-1");
        enricher.enrich(att, entry("scored-1", "t1"));

        Map<String, Object> snapshot = MAPPER.readValue(att.relevanceSnapshot, new TypeReference<>() {});
        double days = ((Number) snapshot.get(CollaborationRecencyUtil.SNAPSHOT_KEY)).doubleValue();
        assertThat(days).isCloseTo(1.0, offset(0.1));
    }

    @Test
    void multiple_shared_refs_uses_most_recent() throws Exception {
        Instant tenDaysAgo = NOW.minus(10, ChronoUnit.DAYS);
        Instant twoDaysAgo = NOW.minus(2, ChronoUnit.DAYS);
        when(graphQuery.agentHistory(eq("attestor-1"), eq("t1"), any()))
                .thenReturn(history("attestor-1", "t1", List.of(
                        task("attestor-1", "t1", "CASE-OLD", tenDaysAgo.minus(1, ChronoUnit.HOURS), tenDaysAgo),
                        task("attestor-1", "t1", "CASE-NEW", twoDaysAgo.minus(1, ChronoUnit.HOURS), twoDaysAgo))));
        when(graphQuery.agentHistory(eq("scored-1"), eq("t1"), any()))
                .thenReturn(history("scored-1", "t1", List.of(
                        task("scored-1", "t1", "CASE-OLD", tenDaysAgo.minus(2, ChronoUnit.HOURS), tenDaysAgo),
                        task("scored-1", "t1", "CASE-NEW", twoDaysAgo.minus(2, ChronoUnit.HOURS), twoDaysAgo))));

        var att = attestation("attestor-1");
        enricher.enrich(att, entry("scored-1", "t1"));

        Map<String, Object> snapshot = MAPPER.readValue(att.relevanceSnapshot, new TypeReference<>() {});
        double days = ((Number) snapshot.get(CollaborationRecencyUtil.SNAPSHOT_KEY)).doubleValue();
        assertThat(days).isCloseTo(2.0, offset(0.1));
    }

    @Test
    void merges_into_existing_snapshot() throws Exception {
        Instant oneDayAgo = NOW.minus(1, ChronoUnit.DAYS);
        when(graphQuery.agentHistory(eq("attestor-1"), eq("t1"), any()))
                .thenReturn(history("attestor-1", "t1", List.of(
                        task("attestor-1", "t1", "CASE-100", oneDayAgo, oneDayAgo))));
        when(graphQuery.agentHistory(eq("scored-1"), eq("t1"), any()))
                .thenReturn(history("scored-1", "t1", List.of(
                        task("scored-1", "t1", "CASE-100", oneDayAgo, oneDayAgo))));

        var att = attestation("attestor-1");
        att.relevanceSnapshot = "{\"org-distance\": 2}";
        enricher.enrich(att, entry("scored-1", "t1"));

        Map<String, Object> snapshot = MAPPER.readValue(att.relevanceSnapshot, new TypeReference<>() {});
        assertThat(snapshot).containsEntry("org-distance", 2);
        assertThat(snapshot).containsKey(CollaborationRecencyUtil.SNAPSHOT_KEY);
    }

    @Test
    void empty_histories_leaves_snapshot_null() {
        when(graphQuery.agentHistory(eq("attestor-1"), eq("t1"), any()))
                .thenReturn(history("attestor-1", "t1", List.of()));
        when(graphQuery.agentHistory(eq("scored-1"), eq("t1"), any()))
                .thenReturn(history("scored-1", "t1", List.of()));

        var att = attestation("attestor-1");
        enricher.enrich(att, entry("scored-1", "t1"));

        assertThat(att.relevanceSnapshot).isNull();
    }

    @Test
    void graph_query_exception_logs_and_continues() {
        when(graphQuery.agentHistory(eq("attestor-1"), eq("t1"), any()))
                .thenThrow(new RuntimeException("connection refused"));

        var att = attestation("attestor-1");
        enricher.enrich(att, entry("scored-1", "t1"));

        assertThat(att.relevanceSnapshot).isNull();
    }

    @Test
    void round_trip_enricher_to_factor() throws Exception {
        Instant threeDaysAgo = NOW.minus(3, ChronoUnit.DAYS);
        when(graphQuery.agentHistory(eq("attestor-1"), eq("t1"), any()))
                .thenReturn(history("attestor-1", "t1", List.of(
                        task("attestor-1", "t1", "CASE-100", threeDaysAgo.minus(1, ChronoUnit.HOURS), threeDaysAgo))));
        when(graphQuery.agentHistory(eq("scored-1"), eq("t1"), any()))
                .thenReturn(history("scored-1", "t1", List.of(
                        task("scored-1", "t1", "CASE-100", threeDaysAgo.minus(1, ChronoUnit.HOURS), threeDaysAgo))));

        var att = attestation("attestor-1");
        enricher.enrich(att, entry("scored-1", "t1"));

        Map<String, Object> snapshot = MAPPER.readValue(att.relevanceSnapshot, new TypeReference<>() {});
        var factor = new CollaborationRecencyRelevanceFactor(7.0, 0.5);
        var ctx = new io.casehub.ledger.api.spi.AttestationRelevanceContext(
                att.id, att.attestorId, att.attestorType, att.attestorRole,
                "scored-1", "t1", "cap", null, NOW, snapshot);
        var result = factor.assess(ctx);
        assertThat(result.weight()).isBetween(0.5, 1.0);
        assertThat(result.reason()).contains("collaboration-recency-days:");
    }

    static class TestLedgerEntry extends LedgerEntry {
        TestLedgerEntry(String actorId, String tenancyId) {
            this.actorId = actorId;
            this.tenancyId = tenancyId;
        }
    }
}
```

- [ ] **Step 3: Run enricher tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=CollaborationRecencyAttestationEnricherTest`
Expected: 9 tests PASS

- [ ] **Step 4: Run all collaboration-recency tests together**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="CollaborationRecency*"`
Expected: All 28 tests PASS

- [ ] **Step 5: Run full runtime module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All existing tests still PASS (no regressions)

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyAttestationEnricher.java \
       runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyAttestationEnricherTest.java
git commit -m "feat(#263): add CollaborationRecencyAttestationEnricher

Queries AgentGraphQuery for shared externalRef task history between
attestor and scored agent. Writes collaboration-recency-days to the
attestation relevance snapshot. Fail-open when graph module absent.

Refs #263"
```

---

## Batch 3: Documentation update

### Task 3: ARC42STORIES attestation relevance pipeline update

**Files:**
- Modify: `ARC42STORIES.MD` — Attestation Relevance Pipeline table (around line 150)

**Interfaces:**
- Consumes: None (documentation only)
- Produces: Updated architecture documentation

- [ ] **Step 1: Add collaboration-recency rows to the relevance pipeline table**

Find the table in the `Attestation Relevance Pipeline` section (after the
`IndependenceAwareRelevanceFactor` row). Add two new rows:

```markdown
| `CollaborationRecencyAttestationEnricher` | `casehub-eidos` | `LedgerAttestationEnricher` | Write-time: queries `AgentGraphQuery` for shared `externalRef` task history between attestor and scored agent; writes `{"collaboration-recency-days": N}` to snapshot; fail-open when graph module absent |
| `CollaborationRecencyRelevanceFactor` | `casehub-eidos` | `AttestationRelevanceFactor` | Read-time: maps `collaboration-recency-days` via exponential decay (configurable half-life default 7 days, baseline 0.5); accuracy-biased — recent collaborators receive higher weight |
```

- [ ] **Step 2: Add beans to the component table**

Find the component table section listing `OrgProximityRelevanceFactor`,
`IndependenceAwareRelevanceFactor`, etc. Add:

```markdown
| `CollaborationRecencyAttestationEnricher` | `@ApplicationScoped` bean implementing `LedgerAttestationEnricher`; write-time enricher that queries `AgentGraphQuery` for shared `externalRef` task history between attestor and scored agent within a configurable lookback window (default 90 days); writes `{"collaboration-recency-days": N}` to the attestation's `relevanceSnapshot` JSON; fail-open via `Instance<AgentGraphQuery>` — returns without enrichment when graph module absent |
| `CollaborationRecencyRelevanceFactor` | `@ApplicationScoped` bean implementing `AttestationRelevanceFactor`; reads `"collaboration-recency-days"` from attestation snapshot and maps via exponential decay with configurable half-life (default 7 days) and baseline (default 0.5); accuracy-biased — recent collaborators receive higher weight; returns NEUTRAL when no data |
```

- [ ] **Step 3: Commit**

```bash
git add ARC42STORIES.MD
git commit -m "docs(#263): add collaboration-recency to attestation relevance pipeline

Refs #263"
```

---

## References

- [2026-11-27-collaboration-recency-relevance-factor-design.md] — design spec this plan implements
- [OrgDistanceAttestationEnricher.java] — sibling enricher pattern
- [OrgProximityRelevanceFactor.java] — sibling factor pattern
- [CapabilityOverlapAttestationEnricher.java] — sibling enricher in runtime module
- [CapabilityOverlapRelevanceFactor.java] — sibling factor with baseline pattern
- [AgentGraphQuery.java:8] — agentHistory() method
- [AgentTask.java:12] — externalRef field
- [AgentTaskHistory.java] — history record with tasks list
- [ARC42STORIES.MD:136-158] — attestation relevance pipeline section
- [GitHub #263] — focal issue
