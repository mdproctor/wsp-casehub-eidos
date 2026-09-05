# Load-Aware Agent Selection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #151 — feat: load-aware agent selection — Overloaded probe in CapabilityHealth
**Issue group:** #151

**Goal:** Exclude overloaded agents from selection by adding a capacity-aware probe step to CapabilityHealth.

**Architecture:** CDI injection of `Instance<ActorCapacityView>` (platform-api) into `DefaultCapabilityHealth` (eidos-runtime). New `Overloaded` variant in sealed `CapabilityStatus` (eidos-api). Probe step positioned after Degraded, before Unavailable in the chain.

**Tech Stack:** Java 21, Quarkus 3.32.2, platform-api 0.2-SNAPSHOT (ActorCapacityView, CapacitySignal)

## Global Constraints

- eidos-api MUST NOT gain a dependency on platform-api (Tier 1 purity)
- `ActorCapacityView` accessed only from eidos-runtime (already has platform-api dep)
- Default capacity threshold: 0.8
- Config key: `casehub.eidos.health.capacity-threshold`
- `CapabilityStatus.Overloaded` is filtered out by selectors (excluded from selection pool)

---

## Batch 1: Overloaded probe step

### Task 1: CapabilityStatus.Overloaded variant + DefaultCapabilityHealth probe step

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java` — add `Overloaded` to sealed permits + new record
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java` — add `Instance<ActorCapacityView>` + `capacityThreshold` + probe step 2
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthOverloadedTest.java` — all capacity probe tests
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthDegradedTest.java:65` — update constructor call (add 2 new params)
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthExclusionTest.java:77` — update constructor call
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthBehavioralViolationTest.java:28-29` — update constructor call

**Interfaces:**
- Produces: `CapabilityStatus.Overloaded(double pressure, double threshold)` — consumed by Task 2 selectors
- Consumes: `ActorCapacityView.aggregatedPressure(String actorId)` → `CapacitySignal` (from platform-api, already on classpath)

- [ ] **Step 1: Add Overloaded variant to CapabilityStatus sealed interface**

In `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java`, update the sealed permits clause and add the record. The permits order follows the probe chain:

```java
    sealed interface CapabilityStatus permits
            CapabilityStatus.Degraded,
            CapabilityStatus.Overloaded,
            CapabilityStatus.Unavailable,
            CapabilityStatus.Excluded,
            CapabilityStatus.EpistemicallyWeak,
            CapabilityStatus.BehavioralViolation,
            CapabilityStatus.Ready {

        record Ready() implements CapabilityStatus {}
        record Degraded(DegradationReason reason, String detail) implements CapabilityStatus {}
        record Overloaded(double pressure, double threshold) implements CapabilityStatus {}
        record Unavailable(String reason) implements CapabilityStatus {}
        record EpistemicallyWeak(String domain, double confidence) implements CapabilityStatus {}
        record Excluded(String domain, ExclusionSource source, int declineCount) implements CapabilityStatus {}
        record BehavioralViolation(Map<String, Integer> violations, ViolationKind kind) implements CapabilityStatus {
            public enum ViolationKind { PER_DIMENSION, AGGREGATE }
        }

        enum ExclusionSource { DECLARED, LEARNED }
    }
```

- [ ] **Step 2: Update DefaultCapabilityHealth constructor — add capacityThreshold + Instance\<ActorCapacityView\>**

In `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`, add two new fields and constructor parameters. Add the import for `ActorCapacityView` and `CapacitySignal`:

```java
import io.casehub.platform.api.capacity.ActorCapacityView;

@DefaultBean
@ApplicationScoped
public class DefaultCapabilityHealth implements CapabilityHealth {

    private final double weakThreshold;
    private final double capacityThreshold;
    private final AgentStateStore stateStore;
    private final BehavioralSignalStore signalStore;
    private final Instance<PreferenceProvider> preferenceProviderInstance;
    private final Instance<ActorCapacityView> capacityViewInstance;
    private final VocabularyRegistry vocabularyRegistry;

    @Inject
    public DefaultCapabilityHealth(
            @ConfigProperty(name = "casehub.eidos.epistemic.weak-threshold", defaultValue = "0.3")
            final double weakThreshold,
            @ConfigProperty(name = "casehub.eidos.health.capacity-threshold", defaultValue = "0.8")
            final double capacityThreshold,
            final AgentStateStore stateStore,
            final BehavioralSignalStore signalStore,
            final Instance<PreferenceProvider> preferenceProviderInstance,
            final Instance<ActorCapacityView> capacityViewInstance,
            final VocabularyRegistry vocabularyRegistry) {
        this.weakThreshold = weakThreshold;
        this.capacityThreshold = capacityThreshold;
        this.stateStore = stateStore;
        this.signalStore = signalStore;
        this.preferenceProviderInstance = preferenceProviderInstance;
        this.capacityViewInstance = capacityViewInstance;
        this.vocabularyRegistry = vocabularyRegistry;
    }
```

- [ ] **Step 3: Add overloaded probe step — after Degraded (step 1), before Unavailable (step 2)**

In the `probe()` method, insert after the existing degraded check and before the capabilities null check:

```java
        // Step 1: operational degradation takes priority
        final var degraded = stateStore.query(descriptor.agentId(), descriptor.tenancyId());
        if (degraded.isPresent()) {
            return new CapabilityStatus.Degraded(degraded.get(), "recorded at dispatch time");
        }

        // Step 2: capacity overload — live signal from ActorCapacityView
        if (capacityViewInstance.isResolvable()) {
            final var signal = capacityViewInstance.get()
                .aggregatedPressure(descriptor.agentId());
            if (signal != null && signal.pressure() >= capacityThreshold) {
                return new CapabilityStatus.Overloaded(signal.pressure(), capacityThreshold);
            }
        }

        // Step 3: capability not declared → unavailable
        if (descriptor.capabilities() == null || descriptor.capabilities().isEmpty()) {
```

- [ ] **Step 4: Update existing unit test constructor calls**

Three unit test files create `DefaultCapabilityHealth` directly. Each needs 2 new params: `capacityThreshold` (0.8) and an unsatisfied `Instance<ActorCapacityView>` mock.

In `DefaultCapabilityHealthDegradedTest.java`, update setUp():

```java
    @SuppressWarnings("unchecked")
    Instance<ActorCapacityView> capacityViewInstance;

    @BeforeEach
    void setUp() {
        stateStore = new StubStateStore();
        preferenceProviderInstance = org.mockito.Mockito.mock(Instance.class);
        org.mockito.Mockito.lenient().when(preferenceProviderInstance.isUnsatisfied()).thenReturn(true);
        capacityViewInstance = org.mockito.Mockito.mock(Instance.class);
        org.mockito.Mockito.lenient().when(capacityViewInstance.isResolvable()).thenReturn(false);
        mockVocabRegistry = org.mockito.Mockito.mock(VocabularyRegistry.class);
        health = new DefaultCapabilityHealth(0.3, 0.8, stateStore, new NoOpBehavioralSignalStore(),
                preferenceProviderInstance, capacityViewInstance, mockVocabRegistry);
    }
```

Add the import: `import io.casehub.platform.api.capacity.ActorCapacityView;`

In `DefaultCapabilityHealthExclusionTest.java`, update setUp():

```java
    @Mock
    @SuppressWarnings("unchecked")
    Instance<ActorCapacityView> capacityViewInstance;

    @BeforeEach
    void setUp() {
        stateStore = new StubStateStore();
        signalStore = new StubBehavioralSignalStore();
        lenient().when(preferenceProviderInstance.isUnsatisfied()).thenReturn(true);
        lenient().when(capacityViewInstance.isResolvable()).thenReturn(false);
        health = new DefaultCapabilityHealth(0.3, 0.8, stateStore, signalStore,
                preferenceProviderInstance, capacityViewInstance, mockVocabRegistry);
    }
```

Add the import: `import io.casehub.platform.api.capacity.ActorCapacityView;`

In `DefaultCapabilityHealthBehavioralViolationTest.java`, update setUp():

```java
    @BeforeEach
    void setUp() {
        signalStore = new StubBehavioralSignalStore();
        @SuppressWarnings("unchecked")
        Instance<PreferenceProvider> emptyProvider = mock(Instance.class);
        when(emptyProvider.isUnsatisfied()).thenReturn(true);
        @SuppressWarnings("unchecked")
        Instance<ActorCapacityView> noCapacity = mock(Instance.class);
        when(noCapacity.isResolvable()).thenReturn(false);
        health = new DefaultCapabilityHealth(0.3, 0.8, mock(AgentStateStore.class),
                signalStore, emptyProvider, noCapacity, new StubVocabularyRegistry());
    }
```

Add the import: `import io.casehub.platform.api.capacity.ActorCapacityView;`

- [ ] **Step 5: Verify existing tests still pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime`
Expected: All existing tests PASS (the new Overloaded variant compiles; existing probe behavior unchanged because `capacityViewInstance.isResolvable()` returns false in all existing tests)

- [ ] **Step 6: Write the overloaded probe test class**

Create `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthOverloadedTest.java`:

```java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.platform.api.capacity.ActorCapacityView;
import io.casehub.platform.api.capacity.CapacitySignal;
import io.casehub.platform.api.preferences.PreferenceProvider;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class DefaultCapabilityHealthOverloadedTest {

    static class NoOpStateStore implements AgentStateStore {
        @Override public void record(String a, String t, DegradationReason r, Instant e) {}
        @Override public Optional<DegradationReason> query(String a, String t) { return Optional.empty(); }
        @Override public void clear(String a, String t) {}
    }

    static class NoOpSignalStore implements BehavioralSignalStore {
        @Override public void record(String a, String t, String c, String d, BehavioralSignal s) {}
        @Override public void clear(String a, String t, String c, BehavioralSignal s) {}
        @Override public Map<String, Integer> learned(String a, String t, String c, BehavioralSignal s) { return Map.of(); }
        @Override public int count(String a, String t, String c, String d, BehavioralSignal s) { return 0; }
    }

    ActorCapacityView capacityView;
    @SuppressWarnings("unchecked")
    Instance<ActorCapacityView> capacityViewInstance = mock(Instance.class);
    @SuppressWarnings("unchecked")
    Instance<PreferenceProvider> prefProvider = mock(Instance.class);
    DefaultCapabilityHealth health;

    @BeforeEach
    void setUp() {
        capacityView = mock(ActorCapacityView.class);
        capacityViewInstance = mock(Instance.class);
        when(capacityViewInstance.isResolvable()).thenReturn(true);
        when(capacityViewInstance.get()).thenReturn(capacityView);
        lenient().when(prefProvider.isUnsatisfied()).thenReturn(true);
        health = new DefaultCapabilityHealth(0.3, 0.8, new NoOpStateStore(),
                new NoOpSignalStore(), prefProvider, capacityViewInstance,
                mock(VocabularyRegistry.class));
    }

    static AgentDescriptor agent(String agentId, AgentCapability... capabilities) {
        return AgentDescriptor.builder()
            .agentId(agentId).name("Agent").version("1.0")
            .provider("anthropic").modelFamily("claude").modelVersion("claude-3-7")
            .slot("worker")
            .capabilities(List.of(capabilities))
            .disposition(AgentDisposition.builder()
                .socialOrient("collaborative").ruleFollowing("principled")
                .riskAppetite("measured").autonomy("semi-autonomous").build())
            .tenancyId("default").build();
    }

    @Test
    void overloaded_above_threshold_returns_overloaded() {
        when(capacityView.aggregatedPressure("agent-1"))
            .thenReturn(new CapacitySignal("agent-1", "test", 0.95, Instant.now()));
        var descriptor = agent("agent-1",
            AgentCapability.builder().name("code-review").build());
        var status = health.probe(descriptor, "code-review", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Overloaded.class);
        var overloaded = (CapabilityStatus.Overloaded) status;
        assertThat(overloaded.pressure()).isEqualTo(0.95);
        assertThat(overloaded.threshold()).isEqualTo(0.8);
    }

    @Test
    void overloaded_at_threshold_returns_overloaded() {
        when(capacityView.aggregatedPressure("agent-1"))
            .thenReturn(new CapacitySignal("agent-1", "test", 0.8, Instant.now()));
        var descriptor = agent("agent-1",
            AgentCapability.builder().name("code-review").build());
        var status = health.probe(descriptor, "code-review", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Overloaded.class);
    }

    @Test
    void below_threshold_returns_ready() {
        when(capacityView.aggregatedPressure("agent-1"))
            .thenReturn(new CapacitySignal("agent-1", "test", 0.5, Instant.now()));
        var descriptor = agent("agent-1",
            AgentCapability.builder().name("code-review").build());
        var status = health.probe(descriptor, "code-review", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void null_signal_falls_through_to_ready() {
        when(capacityView.aggregatedPressure("agent-1")).thenReturn(null);
        var descriptor = agent("agent-1",
            AgentCapability.builder().name("code-review").build());
        var status = health.probe(descriptor, "code-review", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void no_capacity_view_deployed_falls_through_to_ready() {
        @SuppressWarnings("unchecked")
        Instance<ActorCapacityView> noCapacity = mock(Instance.class);
        when(noCapacity.isResolvable()).thenReturn(false);
        var healthNoCapacity = new DefaultCapabilityHealth(0.3, 0.8, new NoOpStateStore(),
                new NoOpSignalStore(), prefProvider, noCapacity,
                mock(VocabularyRegistry.class));
        var descriptor = agent("agent-1",
            AgentCapability.builder().name("code-review").build());
        var status = healthNoCapacity.probe(descriptor, "code-review", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void custom_threshold_respected() {
        var customHealth = new DefaultCapabilityHealth(0.3, 0.6, new NoOpStateStore(),
                new NoOpSignalStore(), prefProvider, capacityViewInstance,
                mock(VocabularyRegistry.class));
        when(capacityView.aggregatedPressure("agent-1"))
            .thenReturn(new CapacitySignal("agent-1", "test", 0.7, Instant.now()));
        var descriptor = agent("agent-1",
            AgentCapability.builder().name("code-review").build());
        var status = customHealth.probe(descriptor, "code-review", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Overloaded.class);
        assertThat(((CapabilityStatus.Overloaded) status).threshold()).isEqualTo(0.6);
    }

    @Test
    void degraded_takes_precedence_over_overloaded() {
        var stateStore = new DefaultCapabilityHealthDegradedTest.StubStateStore();
        stateStore.record("agent-1", "default", DegradationReason.RATE_LIMITED,
            Instant.now().plusSeconds(60));
        when(capacityView.aggregatedPressure("agent-1"))
            .thenReturn(new CapacitySignal("agent-1", "test", 0.95, Instant.now()));
        var healthWithState = new DefaultCapabilityHealth(0.3, 0.8, stateStore,
                new NoOpSignalStore(), prefProvider, capacityViewInstance,
                mock(VocabularyRegistry.class));
        var descriptor = agent("agent-1",
            AgentCapability.builder().name("code-review").build());
        var status = healthWithState.probe(descriptor, "code-review", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Degraded.class);
    }

    @Test
    void overloaded_takes_precedence_over_unavailable() {
        when(capacityView.aggregatedPressure("agent-1"))
            .thenReturn(new CapacitySignal("agent-1", "test", 0.95, Instant.now()));
        var descriptor = agent("agent-1");
        var status = health.probe(descriptor, "missing-capability", ProbeContext.of(null));
        assertThat(status).isInstanceOf(CapabilityStatus.Overloaded.class);
    }
}
```

- [ ] **Step 7: Run new tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultCapabilityHealthOverloadedTest`
Expected: All 8 tests PASS

- [ ] **Step 8: Run full module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime`
Expected: All tests PASS (existing + new)

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java
git add runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java
git add runtime/src/test/java/io/casehub/eidos/runtime/health/
git commit -m "feat(#151): add Overloaded probe step to CapabilityHealth

Adds CapabilityStatus.Overloaded variant and capacity-aware probe step
in DefaultCapabilityHealth via CDI-injected Instance<ActorCapacityView>.
Positioned after Degraded, before Unavailable in the probe chain.

Refs #151"
```

---

### Task 2: Selector switch update + selector-level overloaded filtering tests

**Files:**
- Modify: `routing/src/main/java/io/casehub/eidos/routing/EngineAwareAgentSelector.java:103-111` — add `Overloaded → null` case
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/selector/SimpleAgentSelectorTest.java` — add overloaded filtering tests

**Interfaces:**
- Consumes: `CapabilityStatus.Overloaded(double pressure, double threshold)` from Task 1

- [ ] **Step 1: Verify compile fails — EngineAwareAgentSelector exhaustive switch is now incomplete**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl routing`
Expected: FAIL — exhaustive switch in `mapHealth()` missing `Overloaded` case

- [ ] **Step 2: Write failing test — overloaded agents filtered by SimpleAgentSelector**

Add to `SimpleAgentSelectorTest.java`:

```java
    @Test
    void overloadedAgentFilteredOut() {
        var match = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
        when(healthMock.probe(any(), eq("cap-1"), any()))
            .thenReturn(new CapabilityStatus.Overloaded(0.95, 0.8));
        when(trustSourceMock.capabilityScore("agent-1", "cap-1"))
            .thenReturn(OptionalDouble.of(0.9));
        var result = selector.select(List.of(match), SelectionContext.of("t1", "cap-1"));
        assertInstanceOf(AgentSelection.NoneQualified.class, result);
    }

    @Test
    void mixOfHealthyAndOverloadedSelectsHealthy() {
        var m1 = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
        var m2 = matchWith("agent-2", "cap-1", new MatchDegree.Exact());
        when(healthMock.probe(eq(m1.descriptor()), eq("cap-1"), any()))
            .thenReturn(new CapabilityStatus.Overloaded(0.9, 0.8));
        when(healthMock.probe(eq(m2.descriptor()), eq("cap-1"), any()))
            .thenReturn(new CapabilityStatus.Ready());
        when(trustSourceMock.capabilityScore("agent-2", "cap-1"))
            .thenReturn(OptionalDouble.of(0.7));
        var result = selector.select(List.of(m1, m2), SelectionContext.of("t1", "cap-1"));
        assertInstanceOf(AgentSelection.Selected.class, result);
        assertEquals("agent-2", ((AgentSelection.Selected) result).agent().agentId());
    }
```

- [ ] **Step 3: Run tests to verify they pass (SimpleAgentSelector already filters implicitly)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SimpleAgentSelectorTest`
Expected: PASS — `filterHealthy()` uses positive matching; `Overloaded` doesn't match any kept type

- [ ] **Step 4: Fix EngineAwareAgentSelector — add Overloaded case to mapHealth()**

In `routing/src/main/java/io/casehub/eidos/routing/EngineAwareAgentSelector.java`, update the `mapHealth()` switch:

```java
    private AgentHealth mapHealth(CapabilityStatus status) {
        return switch (status) {
            case CapabilityStatus.Ready r -> AgentHealth.READY;
            case CapabilityStatus.Degraded d -> AgentHealth.DEGRADED;
            case CapabilityStatus.EpistemicallyWeak ew -> AgentHealth.EPISTEMICALLY_WEAK;
            case CapabilityStatus.BehavioralViolation bv -> AgentHealth.BEHAVIORAL_VIOLATION;
            case CapabilityStatus.Overloaded o -> null;
            case CapabilityStatus.Unavailable u -> null;
            case CapabilityStatus.Excluded ex -> null;
        };
    }
```

- [ ] **Step 5: Verify routing module compiles and all tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl routing`
Expected: PASS

- [ ] **Step 6: Run full project test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All modules PASS

- [ ] **Step 7: Commit**

```bash
git add routing/src/main/java/io/casehub/eidos/routing/EngineAwareAgentSelector.java
git add runtime/src/test/java/io/casehub/eidos/runtime/selector/SimpleAgentSelectorTest.java
git commit -m "feat(#151): update selectors for Overloaded status filtering

EngineAwareAgentSelector.mapHealth() maps Overloaded → null (filter out).
SimpleAgentSelector implicit filtering verified with new tests.

Refs #151"
```

---

## References

- [2026-09-05-load-aware-selection-design.md] — design spec this plan implements
- [CapabilityHealth.java:1-33] — sealed CapabilityStatus interface (permits clause + variants)
- [DefaultCapabilityHealth.java:21-144] — probe chain implementation
- [SimpleAgentSelector.java:82-104] — positive health filter
- [EngineAwareAgentSelector.java:103-112] — exhaustive switch on CapabilityStatus
- [DefaultCapabilityHealthDegradedTest.java] — unit test pattern: direct constructor, StubStateStore
- [DefaultCapabilityHealthExclusionTest.java] — unit test pattern: @Mock Instance, StubBehavioralSignalStore
- [SimpleAgentSelectorTest.java] — unit test pattern: mock CapabilityHealth, mock Instance<TrustScoreSource>
- [ActorCapacityView.class] (platform-api) — aggregatedPressure(actorId) → CapacitySignal
- [CapacitySignal.class] (platform-api) — (actorId, source, pressure[0.0-1.0], timestamp)
- [GitHub #151] — focal issue
- [decisions.md] — D1: CDI injection rationale
