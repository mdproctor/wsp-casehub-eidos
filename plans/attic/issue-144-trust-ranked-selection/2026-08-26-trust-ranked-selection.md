# Trust-Ranked Agent Selection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #144 — Trust-ranked agent selection from capability matches
**Issue group:** #144

**Goal:** Add an `AgentSelector` SPI to eidos-api with a simple fallback in eidos-runtime and an engine-delegating bridge in a new eidos-routing module.

**Architecture:** Three-tier. SPI + types in eidos-api (pure Java). `SimpleAgentSelector` `@DefaultBean` in eidos-runtime — health filtering + optional trust ranking via `Instance<TrustScoreSource>`. `EngineAwareAgentSelector` `@Alternative @Priority(1)` in new eidos-routing module — converts `AgentMatch` → `AgentCandidate`, delegates to engine's `AgentRoutingStrategy`.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (`@DefaultBean`, `@Alternative`, `Instance<T>`), casehub-ledger-api (`TrustScoreSource`), casehub-engine-api (`AgentRoutingStrategy`, `AgentCandidate`, `RoutingResult`)

## Global Constraints

- Java 21 source level, Java 26 JVM
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use `mvn` not `./mvnw`
- All new types in `io.casehub.eidos.api` (api) or `io.casehub.eidos.runtime.selector` (runtime) or `io.casehub.eidos.routing` (routing)
- eidos-api is pure Java — no CDI, no Quarkus annotations
- eidos-routing depends on eidos-api + engine-api + ledger-api only (NOT eidos-runtime)
- `casehub-engine-api` version: `${casehub-engine-api.version}` — add to parent pom `<properties>` as `0.2-SNAPSHOT`
- IntelliJ MCP required for all .java file operations

---

## Batch 1: API Types (eidos-api)

### Task 1: AgentSelector SPI + SelectionContext + EscalationKind + AgentSelection

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/AgentSelector.java`
- Create: `api/src/main/java/io/casehub/eidos/api/SelectionContext.java`
- Create: `api/src/main/java/io/casehub/eidos/api/EscalationKind.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentSelection.java`
- Test: `api/src/test/java/io/casehub/eidos/api/SelectionContextTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentSelectionTest.java`

**Interfaces:**
- Produces: `AgentSelector.select(List<AgentMatch>, SelectionContext) → AgentSelection`
- Produces: `SelectionContext(String tenancyId, String capabilityName, String taskDomain)` with `of(tenancyId, capabilityName)` and `of(tenancyId, capabilityName, taskDomain)` factories
- Produces: `EscalationKind { BORDERLINE_STALEMATE, NO_QUALIFIED_AGENT }`
- Produces: `AgentSelection.Selected(AgentDescriptor, @Nullable ResolvedCapability, double, String)`, `AgentSelection.NoneQualified(String)`, `AgentSelection.Escalated(String, EscalationKind, String)`

- [ ] **Step 1: Write SelectionContext test**

```java
// api/src/test/java/io/casehub/eidos/api/SelectionContextTest.java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SelectionContextTest {

    @Test
    void nullTenancyIdRejected() {
        assertThrows(NullPointerException.class,
            () -> new SelectionContext(null, "cap", null));
    }

    @Test
    void twoArgFactoryMatchesRecordOrder() {
        var ctx = SelectionContext.of("tenant-1", "code-review");
        assertEquals("tenant-1", ctx.tenancyId());
        assertEquals("code-review", ctx.capabilityName());
        assertNull(ctx.taskDomain());
    }

    @Test
    void threeArgFactoryMatchesRecordOrder() {
        var ctx = SelectionContext.of("tenant-1", "code-review", "java");
        assertEquals("tenant-1", ctx.tenancyId());
        assertEquals("code-review", ctx.capabilityName());
        assertEquals("java", ctx.taskDomain());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=SelectionContextTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `SelectionContext` does not exist

- [ ] **Step 3: Create EscalationKind enum**

Use `ide_create_file` to create `api/src/main/java/io/casehub/eidos/api/EscalationKind.java`:

```java
package io.casehub.eidos.api;

public enum EscalationKind {
    BORDERLINE_STALEMATE,
    NO_QUALIFIED_AGENT
}
```

- [ ] **Step 4: Create SelectionContext record**

Use `ide_create_file` to create `api/src/main/java/io/casehub/eidos/api/SelectionContext.java`:

```java
package io.casehub.eidos.api;

import java.util.Objects;

public record SelectionContext(
    String tenancyId,
    String capabilityName,
    String taskDomain
) {
    public SelectionContext {
        Objects.requireNonNull(tenancyId, "tenancyId");
    }

    public static SelectionContext of(String tenancyId, String capabilityName) {
        return new SelectionContext(tenancyId, capabilityName, null);
    }

    public static SelectionContext of(String tenancyId, String capabilityName, String taskDomain) {
        return new SelectionContext(tenancyId, capabilityName, taskDomain);
    }
}
```

- [ ] **Step 5: Create AgentSelection sealed interface**

Use `ide_create_file` to create `api/src/main/java/io/casehub/eidos/api/AgentSelection.java`:

```java
package io.casehub.eidos.api;

import jakarta.annotation.Nullable;

public sealed interface AgentSelection {

    record Selected(
        AgentDescriptor agent,
        @Nullable ResolvedCapability resolvedCapability,
        double trustScore,
        String reason
    ) implements AgentSelection {}

    record NoneQualified(String reason) implements AgentSelection {}

    record Escalated(
        String capabilityName,
        EscalationKind kind,
        String reason
    ) implements AgentSelection {}
}
```

- [ ] **Step 6: Create AgentSelector SPI interface**

Use `ide_create_file` to create `api/src/main/java/io/casehub/eidos/api/AgentSelector.java`:

```java
package io.casehub.eidos.api;

import java.util.List;

public interface AgentSelector {
    AgentSelection select(List<AgentMatch> candidates, SelectionContext context);
}
```

- [ ] **Step 7: Write AgentSelection test**

```java
// api/src/test/java/io/casehub/eidos/api/AgentSelectionTest.java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AgentSelectionTest {

    @Test
    void selectedCarriesAllFields() {
        var descriptor = AgentDescriptor.builder()
            .agentId("agent-1").name("Agent One").tenancyId("t1").build();
        var resolved = new ResolvedCapability(
            AgentCapability.builder().name("code-review").build(),
            new MatchDegree.Exact());
        var sel = new AgentSelection.Selected(descriptor, resolved, 0.85, "highest trust");

        assertEquals("agent-1", sel.agent().agentId());
        assertEquals(0.85, sel.trustScore());
        assertNotNull(sel.resolvedCapability());
        assertEquals("highest trust", sel.reason());
    }

    @Test
    void selectedAllowsNullResolvedCapability() {
        var descriptor = AgentDescriptor.builder()
            .agentId("agent-1").name("Agent One").tenancyId("t1").build();
        var sel = new AgentSelection.Selected(descriptor, null, 0.5, "slot query");
        assertNull(sel.resolvedCapability());
    }

    @Test
    void noneQualifiedCarriesReason() {
        var nq = new AgentSelection.NoneQualified("all unhealthy");
        assertEquals("all unhealthy", nq.reason());
    }

    @Test
    void escalatedCarriesKindAndReason() {
        var esc = new AgentSelection.Escalated("code-review",
            EscalationKind.BORDERLINE_STALEMATE, "all candidates borderline");
        assertEquals("code-review", esc.capabilityName());
        assertEquals(EscalationKind.BORDERLINE_STALEMATE, esc.kind());
    }

    @Test
    void patternMatchingExhaustive() {
        AgentSelection selection = new AgentSelection.NoneQualified("test");
        String result = switch (selection) {
            case AgentSelection.Selected s -> "selected: " + s.agent().agentId();
            case AgentSelection.NoneQualified nq -> "none: " + nq.reason();
            case AgentSelection.Escalated e -> "escalated: " + e.kind();
        };
        assertEquals("none: test", result);
    }
}
```

- [ ] **Step 8: Run all api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: all tests PASS including SelectionContextTest and AgentSelectionTest

- [ ] **Step 9: Run ide_diagnostics to verify no compile errors**

Run: `ide_diagnostics` with `project_path=/Users/mdproctor/claude/casehub/eidos`
Expected: no errors in the new files

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentSelector.java api/src/main/java/io/casehub/eidos/api/SelectionContext.java api/src/main/java/io/casehub/eidos/api/EscalationKind.java api/src/main/java/io/casehub/eidos/api/AgentSelection.java api/src/test/java/io/casehub/eidos/api/SelectionContextTest.java api/src/test/java/io/casehub/eidos/api/AgentSelectionTest.java
```

Commit message: `feat(#144): add AgentSelector SPI, SelectionContext, AgentSelection, EscalationKind to eidos-api`

---

## Batch 2: SimpleAgentSelector (eidos-runtime)

### Task 2: SimpleAgentSelector @DefaultBean

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/selector/SimpleAgentSelector.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/selector/SimpleAgentSelectorTest.java`

**Interfaces:**
- Consumes: `AgentSelector.select(List<AgentMatch>, SelectionContext) → AgentSelection` (from Task 1)
- Consumes: `CapabilityHealth.probe(AgentDescriptor, String, ProbeContext) → CapabilityStatus` (existing eidos-api SPI)
- Consumes: `TrustScoreSource.capabilityScore(String, String) → OptionalDouble`, `TrustScoreSource.globalScore(String) → OptionalDouble` (from casehub-ledger-api)
- Produces: `SimpleAgentSelector` — `@DefaultBean @ApplicationScoped` implementing `AgentSelector`

- [ ] **Step 1: Write test — empty candidate list returns NoneQualified**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/selector/SimpleAgentSelectorTest.java
package io.casehub.eidos.runtime.selector;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.ledger.api.spi.TrustScoreSource;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class SimpleAgentSelectorTest {

    private CapabilityHealth healthMock;
    private TrustScoreSource trustSourceMock;
    private Instance<TrustScoreSource> trustSourceInstance;
    private SimpleAgentSelector selector;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        healthMock = mock(CapabilityHealth.class);
        trustSourceMock = mock(TrustScoreSource.class);
        trustSourceInstance = mock(Instance.class);
        when(trustSourceInstance.isResolvable()).thenReturn(true);
        when(trustSourceInstance.get()).thenReturn(trustSourceMock);
        selector = new SimpleAgentSelector(healthMock, trustSourceInstance, 0.0, 0.5);
    }

    @Test
    void emptyListReturnsNoneQualified() {
        var ctx = SelectionContext.of("t1", "cap");
        var result = selector.select(List.of(), ctx);
        assertInstanceOf(AgentSelection.NoneQualified.class, result);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SimpleAgentSelectorTest#emptyListReturnsNoneQualified`
Expected: compilation failure — `SimpleAgentSelector` does not exist

- [ ] **Step 3: Create SimpleAgentSelector with empty/health/trust/rank logic**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/eidos/runtime/selector/SimpleAgentSelector.java`:

```java
package io.casehub.eidos.runtime.selector;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.*;
import java.util.stream.Collectors;

@DefaultBean
@ApplicationScoped
public class SimpleAgentSelector implements AgentSelector {

    private final CapabilityHealth capabilityHealth;
    private final Instance<TrustScoreSource> trustSourceInstance;

    @ConfigProperty(name = "casehub.eidos.selector.trust-threshold", defaultValue = "0.0")
    double trustThreshold;

    @ConfigProperty(name = "casehub.eidos.selector.bootstrap-default-score", defaultValue = "0.5")
    double bootstrapDefaultScore;

    @Inject
    public SimpleAgentSelector(CapabilityHealth capabilityHealth,
                               Instance<TrustScoreSource> trustSourceInstance) {
        this.capabilityHealth = capabilityHealth;
        this.trustSourceInstance = trustSourceInstance;
    }

    // package-private constructor for testing
    SimpleAgentSelector(CapabilityHealth capabilityHealth,
                        Instance<TrustScoreSource> trustSourceInstance,
                        double trustThreshold,
                        double bootstrapDefaultScore) {
        this.capabilityHealth = capabilityHealth;
        this.trustSourceInstance = trustSourceInstance;
        this.trustThreshold = trustThreshold;
        this.bootstrapDefaultScore = bootstrapDefaultScore;
    }

    @Override
    public AgentSelection select(List<AgentMatch> candidates, SelectionContext context) {
        if (candidates.isEmpty()) {
            return new AgentSelection.NoneQualified("no candidates");
        }

        var healthy = filterHealthy(candidates, context);
        if (healthy.isEmpty()) {
            return new AgentSelection.NoneQualified(
                "all %d candidates unhealthy".formatted(candidates.size()));
        }

        var scored = scoreAndFilter(healthy, context);
        if (scored.isEmpty()) {
            return new AgentSelection.NoneQualified(
                "all %d candidates below trust threshold %.2f".formatted(
                    healthy.size(), trustThreshold));
        }

        scored.sort(Comparator
            .comparingDouble(ScoredMatch::score).reversed()
            .thenComparing(sm -> sm.match().resolvedCapability() != null
                ? sm.match().resolvedCapability().degree()
                : new MatchDegree.None())
            .thenComparing(sm -> sm.match().descriptor().agentId()));

        var best = scored.getFirst();
        return new AgentSelection.Selected(
            best.match().descriptor(),
            best.match().resolvedCapability(),
            best.score(),
            "highest trust score (simple selector)");
    }

    private List<AgentMatch> filterHealthy(List<AgentMatch> candidates,
                                            SelectionContext context) {
        var result = new ArrayList<AgentMatch>(candidates.size());
        for (var match : candidates) {
            var capTag = match.resolvedCapability() != null
                ? match.resolvedCapability().capability().name()
                : context.capabilityName();
            if (capTag == null) {
                result.add(match);
                continue;
            }
            var status = capabilityHealth.probe(
                match.descriptor(), capTag,
                ProbeContext.of(context.taskDomain()));
            if (status instanceof CapabilityStatus.Ready
                || status instanceof CapabilityStatus.Degraded
                || status instanceof CapabilityStatus.EpistemicallyWeak
                || status instanceof CapabilityStatus.BehavioralViolation) {
                result.add(match);
            }
        }
        return result;
    }

    private List<ScoredMatch> scoreAndFilter(List<AgentMatch> healthy,
                                              SelectionContext context) {
        var result = new ArrayList<ScoredMatch>(healthy.size());
        for (var match : healthy) {
            double score = resolveTrustScore(
                match.descriptor().agentId(), context.capabilityName());
            if (score >= trustThreshold) {
                result.add(new ScoredMatch(match, score));
            }
        }
        return result;
    }

    private double resolveTrustScore(String agentId, String capabilityName) {
        if (!trustSourceInstance.isResolvable()) {
            return 0.0;
        }
        var source = trustSourceInstance.get();
        var capScore = capabilityName != null
            ? source.capabilityScore(agentId, capabilityName)
            : OptionalDouble.empty();
        if (capScore.isPresent()) {
            return capScore.getAsDouble();
        }
        var globalScore = source.globalScore(agentId);
        return globalScore.orElse(bootstrapDefaultScore);
    }

    private record ScoredMatch(AgentMatch match, double score) {}
}
```

- [ ] **Step 4: Run first test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SimpleAgentSelectorTest#emptyListReturnsNoneQualified`
Expected: PASS

- [ ] **Step 5: Add remaining tests to SimpleAgentSelectorTest**

Add these test methods to the existing test class:

```java
@Test
void allUnhealthyReturnsNoneQualified() {
    var match = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), eq("cap-1"), any()))
        .thenReturn(new CapabilityStatus.Unavailable("down"));
    var result = selector.select(List.of(match), SelectionContext.of("t1", "cap-1"));
    assertInstanceOf(AgentSelection.NoneQualified.class, result);
}

@Test
void singleHealthyCandidateSelected() {
    var match = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), eq("cap-1"), any()))
        .thenReturn(new CapabilityStatus.Ready());
    when(trustSourceMock.capabilityScore("agent-1", "cap-1"))
        .thenReturn(OptionalDouble.of(0.8));
    var result = selector.select(List.of(match), SelectionContext.of("t1", "cap-1"));
    assertInstanceOf(AgentSelection.Selected.class, result);
    var selected = (AgentSelection.Selected) result;
    assertEquals("agent-1", selected.agent().agentId());
    assertEquals(0.8, selected.trustScore());
}

@Test
void highestTrustScoreWins() {
    var m1 = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
    var m2 = matchWith("agent-2", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), any(), any()))
        .thenReturn(new CapabilityStatus.Ready());
    when(trustSourceMock.capabilityScore("agent-1", "cap-1"))
        .thenReturn(OptionalDouble.of(0.6));
    when(trustSourceMock.capabilityScore("agent-2", "cap-1"))
        .thenReturn(OptionalDouble.of(0.9));
    var result = selector.select(List.of(m1, m2), SelectionContext.of("t1", "cap-1"));
    var selected = (AgentSelection.Selected) result;
    assertEquals("agent-2", selected.agent().agentId());
}

@Test
void tieBreakByMatchDegree() {
    var m1 = matchWith("agent-1", "cap-1", new MatchDegree.Plugin(1));
    var m2 = matchWith("agent-2", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), any(), any()))
        .thenReturn(new CapabilityStatus.Ready());
    when(trustSourceMock.capabilityScore(any(), eq("cap-1")))
        .thenReturn(OptionalDouble.of(0.8));
    var result = selector.select(List.of(m1, m2), SelectionContext.of("t1", "cap-1"));
    var selected = (AgentSelection.Selected) result;
    assertEquals("agent-2", selected.agent().agentId());
}

@Test
void capabilityToGlobalFallback() {
    var match = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), any(), any()))
        .thenReturn(new CapabilityStatus.Ready());
    when(trustSourceMock.capabilityScore("agent-1", "cap-1"))
        .thenReturn(OptionalDouble.empty());
    when(trustSourceMock.globalScore("agent-1"))
        .thenReturn(OptionalDouble.of(0.85));
    var result = selector.select(List.of(match), SelectionContext.of("t1", "cap-1"));
    var selected = (AgentSelection.Selected) result;
    assertEquals(0.85, selected.trustScore());
}

@Test
void trueBootstrapUsesDefaultScore() {
    var match = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), any(), any()))
        .thenReturn(new CapabilityStatus.Ready());
    when(trustSourceMock.capabilityScore(any(), any()))
        .thenReturn(OptionalDouble.empty());
    when(trustSourceMock.globalScore(any()))
        .thenReturn(OptionalDouble.empty());
    var result = selector.select(List.of(match), SelectionContext.of("t1", "cap-1"));
    var selected = (AgentSelection.Selected) result;
    assertEquals(0.5, selected.trustScore());
}

@Test
void noTrustSourceFallsBackToHealthOnlyMode() {
    when(trustSourceInstance.isResolvable()).thenReturn(false);
    selector = new SimpleAgentSelector(healthMock, trustSourceInstance, 0.0, 0.5);
    var match = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), any(), any()))
        .thenReturn(new CapabilityStatus.Ready());
    var result = selector.select(List.of(match), SelectionContext.of("t1", "cap-1"));
    var selected = (AgentSelection.Selected) result;
    assertEquals(0.0, selected.trustScore());
}

@Test
void thresholdFiltersLowScoreCandidates() {
    selector = new SimpleAgentSelector(healthMock, trustSourceInstance, 0.7, 0.5);
    var match = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), any(), any()))
        .thenReturn(new CapabilityStatus.Ready());
    when(trustSourceMock.capabilityScore("agent-1", "cap-1"))
        .thenReturn(OptionalDouble.of(0.3));
    var result = selector.select(List.of(match), SelectionContext.of("t1", "cap-1"));
    assertInstanceOf(AgentSelection.NoneQualified.class, result);
}

@Test
void neverReturnsEscalated() {
    selector = new SimpleAgentSelector(healthMock, trustSourceInstance, 0.99, 0.5);
    var m1 = matchWith("agent-1", "cap-1", new MatchDegree.Exact());
    when(healthMock.probe(any(), any(), any()))
        .thenReturn(new CapabilityStatus.Ready());
    when(trustSourceMock.capabilityScore(any(), any()))
        .thenReturn(OptionalDouble.of(0.5));
    var result = selector.select(List.of(m1), SelectionContext.of("t1", "cap-1"));
    assertNotInstanceOf(AgentSelection.Escalated.class, result);
    assertInstanceOf(AgentSelection.NoneQualified.class, result);
}

// Helper
private AgentMatch matchWith(String agentId, String capName, MatchDegree degree) {
    var descriptor = AgentDescriptor.builder()
        .agentId(agentId).name(agentId).tenancyId("t1")
        .capabilities(List.of(AgentCapability.builder().name(capName).build()))
        .build();
    var resolved = new ResolvedCapability(
        AgentCapability.builder().name(capName).build(), degree);
    return new AgentMatch(descriptor, resolved);
}
```

- [ ] **Step 6: Run all SimpleAgentSelector tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SimpleAgentSelectorTest`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/main/java/io/casehub/eidos/runtime/selector/SimpleAgentSelector.java runtime/src/test/java/io/casehub/eidos/runtime/selector/SimpleAgentSelectorTest.java
```

Commit message: `feat(#144): add SimpleAgentSelector @DefaultBean with health filtering and trust ranking`

---

## Batch 3: Engine Bridge (eidos-routing)

### Task 3: Create eidos-routing module + EngineAwareAgentSelector

**Files:**
- Create: `routing/pom.xml`
- Modify: `pom.xml` (parent — add `<module>routing</module>` and `casehub-engine-api.version` property)
- Create: `routing/src/main/java/io/casehub/eidos/routing/EngineAwareAgentSelector.java`
- Test: `routing/src/test/java/io/casehub/eidos/routing/EngineAwareAgentSelectorTest.java`

**Interfaces:**
- Consumes: `AgentSelector` SPI (from Task 1)
- Consumes: `AgentRoutingStrategy.select(AgentRoutingContext, List<AgentCandidate>) → RoutingResult` (from engine-api)
- Consumes: `CapabilityHealth.probe(AgentDescriptor, String, ProbeContext) → CapabilityStatus` (from eidos-api)
- Consumes: `TrustScoreSource.capabilityScore(String, String) → OptionalDouble` (from ledger-api)
- Produces: `EngineAwareAgentSelector` — `@Alternative @Priority(1)` implementing `AgentSelector`

- [ ] **Step 1: Add engine-api version property and routing module to parent pom**

Use `ide_edit_member` on the parent pom to add:
- In `<properties>`: `<casehub-engine-api.version>0.2-SNAPSHOT</casehub-engine-api.version>`
- In `<modules>`: `<module>routing</module>` (after `graph`)
- In `<dependencyManagement><dependencies>`: managed entry for `casehub-engine-api`

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-api</artifactId>
    <version>${casehub-engine-api.version}</version>
</dependency>
```

- [ ] **Step 2: Create routing/pom.xml**

Use `Write` (new file) to create `routing/pom.xml`:

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
    <relativePath>../pom.xml</relativePath>
  </parent>

  <artifactId>casehub-eidos-routing</artifactId>
  <name>CaseHub Eidos - Routing Bridge</name>
  <description>Engine-aware agent selection — bridges eidos AgentMatch discovery to engine AgentRoutingStrategy</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger-api</artifactId>
    </dependency>

    <!-- CDI -->
    <dependency>
      <groupId>jakarta.enterprise</groupId>
      <artifactId>jakarta.enterprise.cdi-api</artifactId>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>jakarta.annotation</groupId>
      <artifactId>jakarta.annotation-api</artifactId>
      <scope>provided</scope>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 3: Write EngineAwareAgentSelector test — selected result mapping**

```java
// routing/src/test/java/io/casehub/eidos/routing/EngineAwareAgentSelectorTest.java
package io.casehub.eidos.routing;

import io.casehub.api.spi.routing.*;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.ledger.api.spi.TrustScoreSource;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class EngineAwareAgentSelectorTest {

    private CapabilityHealth healthMock;
    private AgentRoutingStrategy strategyMock;
    private Instance<AgentRoutingStrategy> strategyInstance;
    private TrustScoreSource trustSourceMock;
    private Instance<TrustScoreSource> trustSourceInstance;
    private EngineAwareAgentSelector selector;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        healthMock = mock(CapabilityHealth.class);
        strategyMock = mock(AgentRoutingStrategy.class);
        strategyInstance = mock(Instance.class);
        when(strategyInstance.stream()).thenReturn(java.util.stream.Stream.of(strategyMock));
        trustSourceMock = mock(TrustScoreSource.class);
        trustSourceInstance = mock(Instance.class);
        when(trustSourceInstance.isResolvable()).thenReturn(true);
        when(trustSourceInstance.get()).thenReturn(trustSourceMock);
        selector = new EngineAwareAgentSelector(healthMock, strategyInstance, trustSourceInstance);
    }

    @Test
    void selectedResultMapsCorrectly() {
        var match = matchWith("agent-1", "code-review", new MatchDegree.Exact());
        when(healthMock.probe(any(), any(), any()))
            .thenReturn(new CapabilityStatus.Ready());
        when(strategyMock.select(any(), anyList()))
            .thenReturn(RoutingResult.assigned("agent-1", "trust qualified"));
        when(trustSourceMock.capabilityScore("agent-1", "code-review"))
            .thenReturn(OptionalDouble.of(0.9));

        var result = selector.select(List.of(match),
            SelectionContext.of("t1", "code-review"));

        assertInstanceOf(AgentSelection.Selected.class, result);
        var selected = (AgentSelection.Selected) result;
        assertEquals("agent-1", selected.agent().agentId());
        assertEquals(0.9, selected.trustScore());
    }

    @Test
    void unresolvableMapsToNoneQualified() {
        var match = matchWith("agent-1", "code-review", new MatchDegree.Exact());
        when(healthMock.probe(any(), any(), any()))
            .thenReturn(new CapabilityStatus.Ready());
        when(strategyMock.select(any(), anyList()))
            .thenReturn(RoutingResult.unresolvable("no viable candidates"));

        var result = selector.select(List.of(match),
            SelectionContext.of("t1", "code-review"));

        assertInstanceOf(AgentSelection.NoneQualified.class, result);
    }

    @Test
    void escalatedMapsWithKind() {
        var match = matchWith("agent-1", "code-review", new MatchDegree.Exact());
        when(healthMock.probe(any(), any(), any()))
            .thenReturn(new CapabilityStatus.Ready());
        when(strategyMock.select(any(), anyList()))
            .thenReturn(RoutingResult.escalate("code-review",
                EscalationReason.BORDERLINE_STALEMATE, "borderline"));

        var result = selector.select(List.of(match),
            SelectionContext.of("t1", "code-review"));

        assertInstanceOf(AgentSelection.Escalated.class, result);
        var esc = (AgentSelection.Escalated) result;
        assertEquals(EscalationKind.BORDERLINE_STALEMATE, esc.kind());
    }

    @Test
    void unavailableCandidatesFilteredBeforeDelegation() {
        var m1 = matchWith("agent-1", "code-review", new MatchDegree.Exact());
        var m2 = matchWith("agent-2", "code-review", new MatchDegree.Exact());
        when(healthMock.probe(argThat(d -> d.agentId().equals("agent-1")), any(), any()))
            .thenReturn(new CapabilityStatus.Unavailable("down"));
        when(healthMock.probe(argThat(d -> d.agentId().equals("agent-2")), any(), any()))
            .thenReturn(new CapabilityStatus.Ready());
        when(strategyMock.select(any(), anyList()))
            .thenReturn(RoutingResult.assigned("agent-2", "only viable"));
        when(trustSourceMock.capabilityScore("agent-2", "code-review"))
            .thenReturn(OptionalDouble.of(0.7));

        var result = selector.select(List.of(m1, m2),
            SelectionContext.of("t1", "code-review"));

        assertInstanceOf(AgentSelection.Selected.class, result);
        // Verify strategy was called with only 1 candidate (agent-2)
        verify(strategyMock).select(any(), argThat(list -> list.size() == 1));
    }

    @Test
    void allFilteredReturnsNoneQualified() {
        var match = matchWith("agent-1", "code-review", new MatchDegree.Exact());
        when(healthMock.probe(any(), any(), any()))
            .thenReturn(new CapabilityStatus.Unavailable("down"));

        var result = selector.select(List.of(match),
            SelectionContext.of("t1", "code-review"));

        assertInstanceOf(AgentSelection.NoneQualified.class, result);
        verifyNoInteractions(strategyMock);
    }

    private AgentMatch matchWith(String agentId, String capName, MatchDegree degree) {
        var descriptor = AgentDescriptor.builder()
            .agentId(agentId).name(agentId).tenancyId("t1")
            .capabilities(List.of(AgentCapability.builder().name(capName).build()))
            .build();
        var resolved = new ResolvedCapability(
            AgentCapability.builder().name(capName).build(), degree);
        return new AgentMatch(descriptor, resolved);
    }
}
```

- [ ] **Step 4: Create EngineAwareAgentSelector**

Use `ide_create_file` to create `routing/src/main/java/io/casehub/eidos/routing/EngineAwareAgentSelector.java`:

```java
package io.casehub.eidos.routing;

import io.casehub.api.spi.routing.*;
import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.ledger.api.spi.TrustScoreSource;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.*;
import java.util.stream.Collectors;

@Alternative
@Priority(1)
@ApplicationScoped
public class EngineAwareAgentSelector implements AgentSelector {

    private final CapabilityHealth capabilityHealth;
    private final Instance<AgentRoutingStrategy> routingStrategies;
    private final Instance<TrustScoreSource> trustSourceInstance;

    @Inject
    public EngineAwareAgentSelector(CapabilityHealth capabilityHealth,
                                     Instance<AgentRoutingStrategy> routingStrategies,
                                     Instance<TrustScoreSource> trustSourceInstance) {
        this.capabilityHealth = capabilityHealth;
        this.routingStrategies = routingStrategies;
        this.trustSourceInstance = trustSourceInstance;
    }

    @Override
    public AgentSelection select(List<AgentMatch> candidates, SelectionContext context) {
        if (candidates.isEmpty()) {
            return new AgentSelection.NoneQualified("no candidates");
        }

        var converted = convertAndFilter(candidates, context);
        if (converted.isEmpty()) {
            return new AgentSelection.NoneQualified(
                "all %d candidates unhealthy".formatted(candidates.size()));
        }

        var strategy = resolveStrategy();
        var routingContext = toRoutingContext(context);
        var result = strategy.select(routingContext, converted.stream()
            .map(CandidateMapping::candidate).toList());

        return toAgentSelection(result, converted, context);
    }

    private List<CandidateMapping> convertAndFilter(List<AgentMatch> matches,
                                                      SelectionContext context) {
        var result = new ArrayList<CandidateMapping>(matches.size());
        for (var match : matches) {
            var descriptor = match.descriptor();
            var capTag = match.resolvedCapability() != null
                ? match.resolvedCapability().capability().name()
                : context.capabilityName();
            var status = capTag != null
                ? capabilityHealth.probe(descriptor, capTag,
                    ProbeContext.of(context.taskDomain()))
                : new CapabilityStatus.Ready();
            var health = mapHealth(status);
            if (health == null) {
                continue;
            }
            var violations = status instanceof CapabilityStatus.BehavioralViolation bv
                ? bv.violations() : null;
            var candidate = new AgentCandidate(
                descriptor.agentId(),
                descriptor.capabilities().stream()
                    .map(AgentCapability::name).collect(Collectors.toSet()),
                0,
                health,
                descriptor,
                match.resolvedCapability() != null
                    ? match.resolvedCapability().degree() : null,
                violations);
            result.add(new CandidateMapping(candidate, match));
        }
        return result;
    }

    private AgentHealth mapHealth(CapabilityStatus status) {
        return switch (status) {
            case CapabilityStatus.Ready r -> AgentHealth.READY;
            case CapabilityStatus.Degraded d -> AgentHealth.DEGRADED;
            case CapabilityStatus.EpistemicallyWeak ew -> AgentHealth.EPISTEMICALLY_WEAK;
            case CapabilityStatus.BehavioralViolation bv -> AgentHealth.BEHAVIORAL_VIOLATION;
            case CapabilityStatus.Unavailable u -> null;
            case CapabilityStatus.Excluded ex -> null;
        };
    }

    private AgentRoutingContext toRoutingContext(SelectionContext context) {
        return new AgentRoutingContext(
            null,
            context.capabilityName(),
            null,
            context.tenancyId(),
            List.of(),
            null,
            null);
    }

    private AgentRoutingStrategy resolveStrategy() {
        return routingStrategies.stream()
            .max(Comparator.comparingInt(s -> {
                var p = s.getClass().getAnnotation(Priority.class);
                return p != null ? p.value() : 0;
            }))
            .orElseThrow(() -> new IllegalStateException(
                "No AgentRoutingStrategy found — eidos-routing requires engine on the classpath"));
    }

    private AgentSelection toAgentSelection(RoutingResult result,
                                              List<CandidateMapping> mappings,
                                              SelectionContext context) {
        return switch (result) {
            case RoutingResult.Selected sel -> {
                var assignment = sel.single();
                var mapping = mappings.stream()
                    .filter(m -> m.candidate().workerId().equals(assignment.executorId()))
                    .findFirst()
                    .orElseThrow(() -> new IllegalStateException(
                        "Strategy returned unknown executorId: " + assignment.executorId()));
                var trustScore = lookupTrustScore(assignment.executorId(),
                    context.capabilityName());
                yield new AgentSelection.Selected(
                    mapping.match().descriptor(),
                    mapping.match().resolvedCapability(),
                    trustScore,
                    assignment.reason() != null ? assignment.reason() : "engine selected");
            }
            case RoutingResult.Unresolvable u ->
                new AgentSelection.NoneQualified(u.reason());
            case RoutingResult.Escalated e ->
                new AgentSelection.Escalated(
                    e.capabilityName() != null ? e.capabilityName() : context.capabilityName(),
                    mapEscalationKind(e.escalationReason()),
                    e.reason());
        };
    }

    private EscalationKind mapEscalationKind(EscalationReason reason) {
        return switch (reason) {
            case BORDERLINE_STALEMATE -> EscalationKind.BORDERLINE_STALEMATE;
            default -> EscalationKind.NO_QUALIFIED_AGENT;
        };
    }

    private double lookupTrustScore(String agentId, String capabilityName) {
        if (!trustSourceInstance.isResolvable()) {
            return 0.0;
        }
        var source = trustSourceInstance.get();
        var capScore = capabilityName != null
            ? source.capabilityScore(agentId, capabilityName)
            : OptionalDouble.empty();
        if (capScore.isPresent()) {
            return capScore.getAsDouble();
        }
        return source.globalScore(agentId).orElse(0.0);
    }

    private record CandidateMapping(AgentCandidate candidate, AgentMatch match) {}
}
```

- [ ] **Step 5: Run routing tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl routing`
Expected: all PASS

- [ ] **Step 6: Run full build to verify no breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl api,runtime,routing`
Expected: BUILD SUCCESS

- [ ] **Step 7: Run ide_diagnostics on routing module**

Run: `ide_diagnostics` with `project_path=/Users/mdproctor/claude/casehub/eidos`
Expected: no errors in routing module

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add routing/ pom.xml
```

Commit message: `feat(#144): add eidos-routing module with EngineAwareAgentSelector bridge`

---

## Batch 4: Docs + Full Verification

### Task 4: Update CLAUDE.md, consumer guide, capability-ownership.md

**Files:**
- Modify: `CLAUDE.md` (project — add routing module to structure and maven coordinates)
- Modify: `docs/guides/consumer-guide.md` (add AgentSelector section)
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/platform/capability-ownership.md` (distinguish discovery+selection from case-routing)

**Interfaces:**
- Consumes: all types from Tasks 1-3

- [ ] **Step 1: Update CLAUDE.md — add routing module to project structure**

Add `casehub-eidos-routing` to the Maven Coordinates table and the Project Structure tree:

Maven Coordinates:
| Routing artifactId | `casehub-eidos-routing` |
| Routing package | `io.casehub.eidos.routing` |

Project Structure: add after `eval/`:
```
├── routing/                             — casehub-eidos-routing: engine-aware agent selection bridge
│   └── src/main/java/io/casehub/eidos/routing/
│       └── EngineAwareAgentSelector.java — @Alternative @Priority(1); converts AgentMatch→AgentCandidate, delegates to AgentRoutingStrategy
```

Add to "What This Project Is" section:
- **AgentSelector** — SPI for selecting the best agent from `AgentRegistry.find()` results; `SimpleAgentSelector` `@DefaultBean` in runtime (health + optional trust ranking via `Instance<TrustScoreSource>`); `EngineAwareAgentSelector` `@Alternative @Priority(1)` in eidos-routing (bridges to engine's `AgentRoutingStrategy` trust maturity model)

Add to Modules to Depend On in consumer guide:
| `casehub-eidos-routing` | Optional — engine-aware selection | `EngineAwareAgentSelector` `@Alternative @Priority(1)`. Bridges `AgentMatch` → `AgentCandidate` → `AgentRoutingStrategy`. Requires `casehub-engine-api` on classpath. Displaces `SimpleAgentSelector` via CDI priority. |

- [ ] **Step 2: Update capability-ownership.md**

Add a clarifying note to the "Agent routing / selection" row:

```
| Agent routing / selection | `casehub-engine-api` | `AgentRoutingStrategy` SPI; CDI priority resolution in `WorkOrchestrator`. Discovery-layer selection proxy: `AgentSelector` SPI in `casehub-eidos-api` takes `AgentRegistry.find()` results and delegates to engine's routing via `casehub-eidos-routing` bridge. `SimpleAgentSelector` `@DefaultBean` provides basic trust ranking when engine is not on the classpath. |
```

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: all tests PASS across all modules

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add CLAUDE.md docs/
git -C /Users/mdproctor/claude/casehub/parent add docs/platform/capability-ownership.md
```

Commit eidos: `docs(#144): add AgentSelector to CLAUDE.md and consumer guide`
Commit parent: `docs: clarify agent selection ownership — eidos discovery proxy + engine routing`

---

## References

- [2026-08-26-trust-ranked-selection-design.md](/Users/mdproctor/claude/public/casehub/eidos/specs/issue-144-trust-ranked-selection/2026-08-26-trust-ranked-selection-design.md) — design spec this plan implements
- [AgentRoutingStrategy.java](/Users/mdproctor/claude/casehub/engine/api/src/main/java/io/casehub/api/spi/routing/AgentRoutingStrategy.java) — engine SPI
- [TrustCandidateClassifier.java](/Users/mdproctor/claude/casehub/engine/ledger/src/main/java/io/casehub/ledger/routing/TrustCandidateClassifier.java) — trust maturity model
- [TrustScoreSource.java](/Users/mdproctor/claude/casehub/ledger/api/src/main/java/io/casehub/ledger/api/spi/TrustScoreSource.java) — ledger trust score SPI
- [AgentCandidate.java](/Users/mdproctor/claude/casehub/engine/api/src/main/java/io/casehub/api/spi/routing/AgentCandidate.java) — engine candidate record
- [capability-ownership.md](/Users/mdproctor/claude/casehub/parent/docs/platform/capability-ownership.md) — platform capability ownership
- GitHub #144 — Trust-ranked agent selection from capability matches
