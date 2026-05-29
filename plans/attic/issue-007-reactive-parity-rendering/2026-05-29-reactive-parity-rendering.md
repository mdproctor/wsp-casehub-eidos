# Reactive Parity, GEMINI Rendering, A2A Narratives — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver issues eidos#7 (reactive parity + JPA), eidos#14 (GEMINI rendering + class rename), and eidos#13 (A2A per-capability narratives) in three sequential commits.

**Architecture:** Three separate commit groups on branch `issue-007-reactive-parity-rendering`. Issue #7 adds two new reactive SPIs with bridge/JPA impls and corrects `DefaultReactiveCapabilityHealth`. Issue #14 renames `ClaudeMarkdownRenderer` → `EidosSystemPromptRenderer` via IntelliJ and implements proper GEMINI format assembly. Issue #13 adds a dedicated `A2ASemanticEnrichmentStep` (descriptor-only payload, separate schema) so A2A cards get per-capability LLM narratives without polluting the narrative enrichment path.

**Tech Stack:** Java 21, Quarkus 3.x, Hibernate ORM + Hibernate Reactive Panache, Mutiny, LangChain4j 1.x, JUnit 5, AssertJ, ArchUnit

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`  
**Single module:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api` (or `runtime`, `persistence-memory`)

---

## File Map

**New files — api module:**
- `api/src/main/java/io/casehub/eidos/api/ReactiveAgentStateStore.java`
- `api/src/main/java/io/casehub/eidos/api/ReactiveSystemPromptRenderer.java`
- `api/src/test/java/io/casehub/eidos/api/AgentStateStoreContractTest.java`
- `api/src/test/java/io/casehub/eidos/api/ReactiveAgentStateStoreContractTest.java`

**New files — runtime module:**
- `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveAgentStateStore.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateEntity.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStore.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStore.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateReactivePanacheRepo.java`
- `runtime/src/main/resources/db/eidos/migration/V2__agent_degradation_state.sql`
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/A2AEnrichment.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/A2ASemanticEnrichmentStep.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStoreTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStoreTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthDefaultProfileTest.java`

**New files — persistence-memory module:**
- `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStore.java`
- `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStoreTest.java`

**Modified — api module:**
- (none — SPIs are new files; no existing api file changes)

**Modified — runtime module:**
- `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealth.java` — remove `@IfBuildProperty`, add `@DefaultBean`
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRenderer.java` → renamed to `EidosSystemPromptRenderer.java` by IntelliJ
- `runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java` — update class name reference after rename (handled by IntelliJ)
- `runtime/src/test/java/io/casehub/eidos/runtime/registry/BlockingReactiveParityTest.java` — add StateStore + Renderer parity assertions
- `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthTest.java` — move `ReactiveTestProfile` import (IntelliJ handles)
- `runtime/src/test/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRendererTest.java` → renamed to `EidosSystemPromptRendererTest.java` + new/deleted tests
- `deployment/src/main/java/io/casehub/eidos/deployment/EidosProcessor.java` — add @BuildStep for native image resources

**Modified — persistence-memory module:**
- `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentStateStoreTest.java` — extend `AgentStateStoreContractTest`

---

## PART 1 — Issue #7: Reactive Parity + JPA

---

### Task 1: New reactive SPIs

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/ReactiveAgentStateStore.java`
- Create: `api/src/main/java/io/casehub/eidos/api/ReactiveSystemPromptRenderer.java`

- [ ] **Step 1: Write `ReactiveAgentStateStore`**

```java
// api/src/main/java/io/casehub/eidos/api/ReactiveAgentStateStore.java
package io.casehub.eidos.api;

import io.smallrye.mutiny.Uni;
import java.time.Instant;
import java.util.Optional;

public interface ReactiveAgentStateStore {
    Uni<Void>                        record(String agentId, DegradationReason reason, Instant expiresAt);
    Uni<Optional<DegradationReason>> query(String agentId);
    Uni<Void>                        clear(String agentId);
}
```

- [ ] **Step 2: Write `ReactiveSystemPromptRenderer`**

```java
// api/src/main/java/io/casehub/eidos/api/ReactiveSystemPromptRenderer.java
package io.casehub.eidos.api;

import io.smallrye.mutiny.Uni;

public interface ReactiveSystemPromptRenderer {
    Uni<SystemPromptRenderer.RenderedPrompt> render(AgentDescriptor descriptor, AgentPromptContext context);
}
```

- [ ] **Step 3: Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api
```

Expected: BUILD SUCCESS

---

### Task 2: BlockingReactiveParityTest — add StateStore and Renderer assertions

**Files:**
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registry/BlockingReactiveParityTest.java`

- [ ] **Step 1: Run existing parity tests to confirm they pass before changes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BlockingReactiveParityTest
```

Expected: 3 tests pass.

- [ ] **Step 2: Add imports and two new parity test pairs**

Add these imports at the top of the file (after the existing imports):
```java
import io.casehub.eidos.api.AgentStateStore;
import io.casehub.eidos.api.ReactiveAgentStateStore;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.casehub.eidos.api.ReactiveSystemPromptRenderer;
```

Add these four test methods inside the class:

```java
@Test
void state_store_reactive_has_same_method_names_as_blocking() {
    var blockingMethods = API_CLASSES.get(AgentStateStore.class)
        .getMethods().stream()
        .map(m -> m.getName())
        .collect(Collectors.toSet());

    var reactiveMethods = API_CLASSES.get(ReactiveAgentStateStore.class)
        .getMethods().stream()
        .map(m -> m.getName())
        .collect(Collectors.toSet());

    assertThat(reactiveMethods)
        .as("ReactiveAgentStateStore must have the same method names as AgentStateStore")
        .containsExactlyInAnyOrderElementsOf(blockingMethods);
}

@Test
void state_store_reactive_methods_return_uni() {
    ArchRule rule = methods()
        .that().areDeclaredIn(ReactiveAgentStateStore.class)
        .should().haveRawReturnType(assignableTo(Uni.class));
    rule.check(API_CLASSES);
}

@Test
void renderer_reactive_has_same_method_names_as_blocking() {
    var blockingMethods = API_CLASSES.get(SystemPromptRenderer.class)
        .getMethods().stream()
        .map(m -> m.getName())
        .collect(Collectors.toSet());

    var reactiveMethods = API_CLASSES.get(ReactiveSystemPromptRenderer.class)
        .getMethods().stream()
        .map(m -> m.getName())
        .collect(Collectors.toSet());

    assertThat(reactiveMethods)
        .as("ReactiveSystemPromptRenderer must have the same method names as SystemPromptRenderer")
        .containsExactlyInAnyOrderElementsOf(blockingMethods);
}

@Test
void renderer_reactive_methods_return_uni() {
    JavaClasses apiClasses = new ClassFileImporter().importPackages("io.casehub.eidos.api");
    ArchRule rule = methods()
        .that().areDeclaredIn(ReactiveSystemPromptRenderer.class)
        .should().haveRawReturnType(assignableTo(Uni.class));
    rule.check(apiClasses);
}
```

- [ ] **Step 3: Run parity tests — all 7 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BlockingReactiveParityTest
```

Expected: 7 tests pass.

---

### Task 3: Abstract contract tests

**Files:**
- Create: `api/src/test/java/io/casehub/eidos/api/AgentStateStoreContractTest.java`
- Create: `api/src/test/java/io/casehub/eidos/api/ReactiveAgentStateStoreContractTest.java`

- [ ] **Step 1: Write `AgentStateStoreContractTest`**

```java
// api/src/test/java/io/casehub/eidos/api/AgentStateStoreContractTest.java
package io.casehub.eidos.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.*;

public abstract class AgentStateStoreContractTest {

    protected abstract AgentStateStore store();

    @BeforeEach
    protected void resetStore() {
        // Subclasses may override to reset state before each test
    }

    @Test
    void query_returns_empty_when_no_record() {
        assertThat(store().query("contract-agent-1")).isEmpty();
    }

    @Test
    void record_and_query_returns_reason() {
        store().record("contract-agent-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        assertThat(store().query("contract-agent-1")).contains(DegradationReason.RATE_LIMITED);
    }

    @Test
    void query_returns_empty_after_ttl_expired() {
        store().record("contract-agent-1", DegradationReason.OVERLOADED, Instant.now().minusSeconds(1));
        assertThat(store().query("contract-agent-1")).isEmpty();
    }

    @Test
    void clear_removes_entry() {
        store().record("contract-agent-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        store().clear("contract-agent-1");
        assertThat(store().query("contract-agent-1")).isEmpty();
    }

    @Test
    void clear_on_absent_agent_does_not_throw() {
        assertThatCode(() -> store().clear("contract-nonexistent")).doesNotThrowAnyException();
    }

    @Test
    void record_overwrites_previous_state() {
        store().record("contract-agent-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        store().record("contract-agent-1", DegradationReason.CONTEXT_EXHAUSTED, Instant.now().plusSeconds(60));
        assertThat(store().query("contract-agent-1")).contains(DegradationReason.CONTEXT_EXHAUSTED);
    }

    @Test
    void different_agents_are_independent() {
        store().record("contract-agent-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60));
        assertThat(store().query("contract-agent-2")).isEmpty();
    }
}
```

- [ ] **Step 2: Write `ReactiveAgentStateStoreContractTest`**

```java
// api/src/test/java/io/casehub/eidos/api/ReactiveAgentStateStoreContractTest.java
package io.casehub.eidos.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import static org.assertj.core.api.Assertions.*;

public abstract class ReactiveAgentStateStoreContractTest {

    protected abstract ReactiveAgentStateStore store();

    @BeforeEach
    protected void resetStore() {}

    @Test
    void query_returns_empty_when_no_record() {
        var result = store().query("r-contract-agent-1").await().atMost(Duration.ofSeconds(5));
        assertThat(result).isEmpty();
    }

    @Test
    void record_and_query_returns_reason() {
        store().record("r-contract-agent-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60))
               .await().atMost(Duration.ofSeconds(5));
        var result = store().query("r-contract-agent-1").await().atMost(Duration.ofSeconds(5));
        assertThat(result).contains(DegradationReason.RATE_LIMITED);
    }

    @Test
    void query_returns_empty_after_ttl_expired() {
        store().record("r-contract-agent-1", DegradationReason.OVERLOADED, Instant.now().minusSeconds(1))
               .await().atMost(Duration.ofSeconds(5));
        var result = store().query("r-contract-agent-1").await().atMost(Duration.ofSeconds(5));
        assertThat(result).isEmpty();
    }

    @Test
    void clear_removes_entry() {
        store().record("r-contract-agent-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60))
               .await().atMost(Duration.ofSeconds(5));
        store().clear("r-contract-agent-1").await().atMost(Duration.ofSeconds(5));
        var result = store().query("r-contract-agent-1").await().atMost(Duration.ofSeconds(5));
        assertThat(result).isEmpty();
    }

    @Test
    void clear_on_absent_agent_does_not_throw() {
        assertThatCode(() ->
            store().clear("r-contract-nonexistent").await().atMost(Duration.ofSeconds(5))
        ).doesNotThrowAnyException();
    }

    @Test
    void record_overwrites_previous_state() {
        store().record("r-contract-agent-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60))
               .await().atMost(Duration.ofSeconds(5));
        store().record("r-contract-agent-1", DegradationReason.CONTEXT_EXHAUSTED, Instant.now().plusSeconds(60))
               .await().atMost(Duration.ofSeconds(5));
        var result = store().query("r-contract-agent-1").await().atMost(Duration.ofSeconds(5));
        assertThat(result).contains(DegradationReason.CONTEXT_EXHAUSTED);
    }

    @Test
    void different_agents_are_independent() {
        store().record("r-contract-agent-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60))
               .await().atMost(Duration.ofSeconds(5));
        var result = store().query("r-contract-agent-2").await().atMost(Duration.ofSeconds(5));
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 3: Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl api
```

Expected: BUILD SUCCESS

---

### Task 4: Fix DefaultReactiveCapabilityHealth + add default-profile test

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealth.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthDefaultProfileTest.java`

- [ ] **Step 1: Write the failing default-profile test first**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthDefaultProfileTest.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
class DefaultReactiveCapabilityHealthDefaultProfileTest {

    @Inject
    ReactiveCapabilityHealth health;

    @Test
    void reactive_health_is_injectable_under_default_profile() {
        var descriptor = new AgentDescriptor(
            "agent-1", "Agent", "1.0", "anthropic", "claude", "claude-3-7",
            null, null, null, null, "reviewer",
            List.of(new AgentCapability("code-review", 0.9, null, null,
                List.of(), List.of(), List.of(), Map.of())),
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, "default"
        );

        var status = health.probe(descriptor, "code-review", ProbeContext.of(null))
                           .await().indefinitely();

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }
}
```

- [ ] **Step 2: Run the new test — confirm it fails (bean not resolvable due to @IfBuildProperty)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultReactiveCapabilityHealthDefaultProfileTest
```

Expected: FAIL — `UnsatisfiedResolutionException` or `NullPointerException` on injection because `DefaultReactiveCapabilityHealth` is gated off under default profile.

- [ ] **Step 3: Fix `DefaultReactiveCapabilityHealth` — remove `@IfBuildProperty`, add `@DefaultBean`**

Replace the entire file:
```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealth.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@DefaultBean
@ApplicationScoped
public class DefaultReactiveCapabilityHealth implements ReactiveCapabilityHealth {

    @Inject
    CapabilityHealth delegate;

    @Override
    public Uni<CapabilityStatus> probe(AgentDescriptor descriptor, String capabilityTag,
                                       ProbeContext context) {
        return Uni.createFrom().item(() -> delegate.probe(descriptor, capabilityTag, context));
    }
}
```

- [ ] **Step 4: Run both reactive health tests — all must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="DefaultReactiveCapabilityHealthTest,DefaultReactiveCapabilityHealthDefaultProfileTest"
```

Expected: 5 tests pass (3 from ReactiveTestProfile + 1 default-profile test, plus the existing test counts).

---

### Task 5: DefaultReactiveAgentStateStore bridge

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveAgentStateStore.java`

- [ ] **Step 1: Write the class**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveAgentStateStore.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.AgentStateStore;
import io.casehub.eidos.api.DegradationReason;
import io.casehub.eidos.api.ReactiveAgentStateStore;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class DefaultReactiveAgentStateStore implements ReactiveAgentStateStore {

    @Inject
    AgentStateStore delegate;

    @Override
    public Uni<Void> record(final String agentId, final DegradationReason reason, final Instant expiresAt) {
        return Uni.createFrom().<Void>item(() -> {
            delegate.record(agentId, reason, expiresAt);
            return null;
        }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    @Override
    public Uni<Optional<DegradationReason>> query(final String agentId) {
        return Uni.createFrom()
                  .item(() -> delegate.query(agentId))
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    @Override
    public Uni<Void> clear(final String agentId) {
        return Uni.createFrom().<Void>item(() -> {
            delegate.clear(agentId);
            return null;
        }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: BUILD SUCCESS

---

### Task 6: DefaultReactiveSystemPromptRenderer bridge

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java`

- [ ] **Step 1: Write the class**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java
package io.casehub.eidos.runtime.renderer;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.ReactiveSystemPromptRenderer;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.casehub.eidos.api.SystemPromptRenderer.RenderedPrompt;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@DefaultBean
@ApplicationScoped
public class DefaultReactiveSystemPromptRenderer implements ReactiveSystemPromptRenderer {

    @Inject
    SystemPromptRenderer delegate;

    @Override
    public Uni<RenderedPrompt> render(final AgentDescriptor descriptor, final AgentPromptContext context) {
        return Uni.createFrom()
                  .item(() -> delegate.render(descriptor, context))
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: BUILD SUCCESS

---

### Task 7: JPA entity + V2 migration

**Files:**
- Create: `runtime/src/main/resources/db/eidos/migration/V2__agent_degradation_state.sql`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateEntity.java`

- [ ] **Step 1: Write the V2 migration SQL**

```sql
-- runtime/src/main/resources/db/eidos/migration/V2__agent_degradation_state.sql
CREATE TABLE agent_degradation_state (
    agent_id            VARCHAR(255)             NOT NULL PRIMARY KEY,
    degradation_reason  VARCHAR(50)              NOT NULL,
    expires_at          TIMESTAMP WITH TIME ZONE NOT NULL
);
```

`TIMESTAMP WITH TIME ZONE` ensures correct UTC storage regardless of DB session timezone. H2 (tests) supports this type in compatibility mode.

- [ ] **Step 2: Write `AgentDegradationStateEntity`**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateEntity.java
package io.casehub.eidos.runtime.health.jpa;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "agent_degradation_state")
class AgentDegradationStateEntity {

    @Id
    @Column(name = "agent_id")
    private String agentId;

    @Column(name = "degradation_reason", nullable = false)
    private String degradationReason;

    @Column(name = "expires_at", nullable = false, columnDefinition = "TIMESTAMP WITH TIME ZONE")
    private Instant expiresAt;

    protected AgentDegradationStateEntity() {}

    AgentDegradationStateEntity(final String agentId, final String degradationReason, final Instant expiresAt) {
        this.agentId = agentId;
        this.degradationReason = degradationReason;
        this.expiresAt = expiresAt;
    }

    String getAgentId()           { return agentId; }
    String getDegradationReason() { return degradationReason; }
    Instant getExpiresAt()        { return expiresAt; }
}
```

- [ ] **Step 3: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: BUILD SUCCESS

---

### Task 8: JpaAgentStateStore + test

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStore.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStoreTest.java`

- [ ] **Step 1: Write failing test first**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStoreTest.java
package io.casehub.eidos.runtime.health.jpa;

import io.casehub.eidos.api.AgentStateStore;
import io.casehub.eidos.api.AgentStateStoreContractTest;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;

@QuarkusTest
@TestTransaction
class JpaAgentStateStoreTest extends AgentStateStoreContractTest {

    @Inject
    AgentStateStore store;

    @Override
    protected AgentStateStore store() {
        return store;
    }
}
```

`@TestTransaction` at class level: every inherited test method runs in a transaction that is rolled back after, giving clean state for each test.

- [ ] **Step 2: Run test — confirm it fails (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentStateStoreTest
```

Expected: FAIL — compilation error or CDI ambiguity (JpaAgentStateStore doesn't exist yet).

- [ ] **Step 3: Write `JpaAgentStateStore`**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStore.java
package io.casehub.eidos.runtime.health.jpa;

import io.casehub.eidos.api.AgentStateStore;
import io.casehub.eidos.api.DegradationReason;
import io.quarkus.arc.properties.IfBuildProperty;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;
import java.time.Instant;
import java.util.Optional;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "false", enableIfMissing = true)
@ApplicationScoped
public class JpaAgentStateStore implements AgentStateStore {

    @Inject EntityManager em;

    @Override
    @Transactional
    public void record(final String agentId, final DegradationReason reason, final Instant expiresAt) {
        em.createQuery("DELETE FROM AgentDegradationStateEntity e WHERE e.agentId = :id")
          .setParameter("id", agentId)
          .executeUpdate();
        // flush + clear required: without these, Hibernate's first-level cache retains the old entity
        // and throws EntityExistsException on the subsequent persist.
        em.flush();
        em.clear();
        em.persist(new AgentDegradationStateEntity(agentId, reason.name(), expiresAt));
    }

    @Override
    @Transactional(TxType.SUPPORTS)
    public Optional<DegradationReason> query(final String agentId) {
        return em.createQuery(
                "SELECT e FROM AgentDegradationStateEntity e WHERE e.agentId = :id AND e.expiresAt > :now",
                AgentDegradationStateEntity.class)
            .setParameter("id", agentId)
            .setParameter("now", Instant.now())
            .getResultStream()
            .findFirst()
            .map(e -> DegradationReason.valueOf(e.getDegradationReason()));
    }

    @Override
    @Transactional
    public void clear(final String agentId) {
        em.createQuery("DELETE FROM AgentDegradationStateEntity e WHERE e.agentId = :id")
          .setParameter("id", agentId)
          .executeUpdate();
    }
}
```

- [ ] **Step 4: Run test — all 7 contract tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaAgentStateStoreTest
```

Expected: 7 tests pass.

---

### Task 9: JpaReactiveAgentStateStore + test

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateReactivePanacheRepo.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStore.java`
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStoreTest.java`

- [ ] **Step 1: Write failing test**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStoreTest.java
package io.casehub.eidos.runtime.health.jpa;

import io.casehub.eidos.api.DegradationReason;
import io.casehub.eidos.api.ReactiveAgentStateStore;
import io.casehub.eidos.runtime.registry.ReactiveTestProfile;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.vertx.RunOnVertxContext;
import io.quarkus.test.vertx.UniAsserter;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.*;

@QuarkusTest
@TestProfile(ReactiveTestProfile.class)
class JpaReactiveAgentStateStoreTest {

    @Inject
    ReactiveAgentStateStore store;

    @Test
    @RunOnVertxContext
    void record_and_query_returns_reason(UniAsserter asserter) {
        asserter
            .execute(() -> store.record("r-jpa-1", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60)))
            .assertThat(() -> store.query("r-jpa-1"), result ->
                assertThat(result).contains(DegradationReason.RATE_LIMITED));
    }

    @Test
    @RunOnVertxContext
    void query_returns_empty_after_ttl_expired(UniAsserter asserter) {
        asserter
            .execute(() -> store.record("r-jpa-2", DegradationReason.OVERLOADED, Instant.now().minusSeconds(1)))
            .assertThat(() -> store.query("r-jpa-2"), result ->
                assertThat(result).isEmpty());
    }

    @Test
    @RunOnVertxContext
    void clear_removes_entry(UniAsserter asserter) {
        asserter
            .execute(() -> store.record("r-jpa-3", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60)))
            .execute(() -> store.clear("r-jpa-3"))
            .assertThat(() -> store.query("r-jpa-3"), result ->
                assertThat(result).isEmpty());
    }

    @Test
    @RunOnVertxContext
    void record_overwrites_previous_state(UniAsserter asserter) {
        asserter
            .execute(() -> store.record("r-jpa-4", DegradationReason.RATE_LIMITED, Instant.now().plusSeconds(60)))
            .execute(() -> store.record("r-jpa-4", DegradationReason.CONTEXT_EXHAUSTED, Instant.now().plusSeconds(60)))
            .assertThat(() -> store.query("r-jpa-4"), result ->
                assertThat(result).contains(DegradationReason.CONTEXT_EXHAUSTED));
    }
}
```

Note: uses unique agent IDs per test method to avoid cross-test contamination (no rolled-back transaction in reactive mode).

- [ ] **Step 2: Run test — confirm FAIL (class missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaReactiveAgentStateStoreTest
```

Expected: FAIL

- [ ] **Step 3: Write `AgentDegradationStateReactivePanacheRepo`**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateReactivePanacheRepo.java
package io.casehub.eidos.runtime.health.jpa;

import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.hibernate.reactive.panache.PanacheRepositoryBase;
import jakarta.enterprise.context.ApplicationScoped;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "true")
@ApplicationScoped
class AgentDegradationStateReactivePanacheRepo
        implements PanacheRepositoryBase<AgentDegradationStateEntity, String> {
}
```

- [ ] **Step 4: Write `JpaReactiveAgentStateStore`**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStore.java
package io.casehub.eidos.runtime.health.jpa;

import io.casehub.eidos.api.DegradationReason;
import io.casehub.eidos.api.ReactiveAgentStateStore;
import io.quarkus.arc.properties.IfBuildProperty;
import io.quarkus.hibernate.reactive.panache.common.WithSession;
import io.quarkus.hibernate.reactive.panache.common.WithTransaction;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.Optional;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class JpaReactiveAgentStateStore implements ReactiveAgentStateStore {

    @Inject
    AgentDegradationStateReactivePanacheRepo repo;

    @Override
    @WithTransaction
    public Uni<Void> record(final String agentId, final DegradationReason reason, final Instant expiresAt) {
        return repo.delete("agentId", agentId)
                   .chain(() -> repo.persist(new AgentDegradationStateEntity(agentId, reason.name(), expiresAt)))
                   .replaceWithVoid();
    }

    @Override
    @WithSession
    public Uni<Optional<DegradationReason>> query(final String agentId) {
        return repo.find("agentId = ?1 AND expiresAt > ?2", agentId, Instant.now())
                   .firstResult()
                   .map(e -> Optional.ofNullable(e)
                                     .map(entity -> DegradationReason.valueOf(entity.getDegradationReason())));
    }

    @Override
    @WithTransaction
    public Uni<Void> clear(final String agentId) {
        return repo.delete("agentId", agentId).replaceWithVoid();
    }
}
```

- [ ] **Step 5: Run test — all 4 tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaReactiveAgentStateStoreTest
```

Expected: 4 tests pass.

---

### Task 10: InMemoryReactiveAgentStateStore + tests + contract extension

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStore.java`
- Create: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStoreTest.java`
- Modify: `persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentStateStoreTest.java`

- [ ] **Step 1: Write `InMemoryReactiveAgentStateStore`**

```java
// persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStore.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.DegradationReason;
import io.casehub.eidos.api.ReactiveAgentStateStore;
import io.smallrye.mutiny.Uni;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.Optional;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryReactiveAgentStateStore implements ReactiveAgentStateStore {

    @Inject InMemoryAgentStateStore delegate;

    @Override
    public Uni<Void> record(final String agentId, final DegradationReason reason, final Instant expiresAt) {
        return Uni.createFrom().<Void>item(() -> {
            delegate.record(agentId, reason, expiresAt);
            return null;
        });
    }

    @Override
    public Uni<Optional<DegradationReason>> query(final String agentId) {
        return Uni.createFrom().item(() -> delegate.query(agentId));
    }

    @Override
    public Uni<Void> clear(final String agentId) {
        return Uni.createFrom().<Void>item(() -> {
            delegate.clear(agentId);
            return null;
        });
    }
}
```

No `runSubscriptionOn` needed: delegates to in-memory ConcurrentHashMap operations (microsecond-level).

- [ ] **Step 2: Write `InMemoryReactiveAgentStateStoreTest` extending the contract**

```java
// persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStoreTest.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.ReactiveAgentStateStore;
import io.casehub.eidos.api.ReactiveAgentStateStoreContractTest;
import org.junit.jupiter.api.BeforeEach;

class InMemoryReactiveAgentStateStoreTest extends ReactiveAgentStateStoreContractTest {

    private InMemoryReactiveAgentStateStore reactiveStore;
    private InMemoryAgentStateStore blockingStore;

    @BeforeEach
    @Override
    protected void resetStore() {
        blockingStore = new InMemoryAgentStateStore();
        reactiveStore = new InMemoryReactiveAgentStateStore();
        // Wire delegate via field injection workaround for pure-Java tests:
        // Use reflection to set the private field, or add a package-private constructor.
    }

    @Override
    protected ReactiveAgentStateStore store() {
        return reactiveStore;
    }
}
```

Wait — `InMemoryReactiveAgentStateStore` uses `@Inject InMemoryAgentStateStore delegate` which CDI handles at runtime but not in pure-JUnit tests. We need a package-private constructor for testing. Update `InMemoryReactiveAgentStateStore` to add:

```java
// Add this constructor to InMemoryReactiveAgentStateStore (package-private, for tests):
InMemoryReactiveAgentStateStore(final InMemoryAgentStateStore delegate) {
    this.delegate = delegate;
}
```

And update `InMemoryReactiveAgentStateStoreTest`:

```java
// persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStoreTest.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.ReactiveAgentStateStore;
import io.casehub.eidos.api.ReactiveAgentStateStoreContractTest;
import org.junit.jupiter.api.BeforeEach;

class InMemoryReactiveAgentStateStoreTest extends ReactiveAgentStateStoreContractTest {

    private InMemoryReactiveAgentStateStore reactiveStore;

    @BeforeEach
    @Override
    protected void resetStore() {
        reactiveStore = new InMemoryReactiveAgentStateStore(new InMemoryAgentStateStore());
    }

    @Override
    protected ReactiveAgentStateStore store() {
        return reactiveStore;
    }
}
```

- [ ] **Step 3: Update `InMemoryAgentStateStoreTest` to extend `AgentStateStoreContractTest`**

Replace the entire `InMemoryAgentStateStoreTest.java` with a version that extends the contract and keeps its InMemory-specific tests (concurrency):

```java
// persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentStateStoreTest.java
package io.casehub.eidos.memory;

import io.casehub.eidos.api.AgentStateStore;
import io.casehub.eidos.api.AgentStateStoreContractTest;
import io.casehub.eidos.api.DegradationReason;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import static org.assertj.core.api.Assertions.*;

class InMemoryAgentStateStoreTest extends AgentStateStoreContractTest {

    private InMemoryAgentStateStore store;

    @BeforeEach
    @Override
    protected void resetStore() {
        store = new InMemoryAgentStateStore();
    }

    @Override
    protected AgentStateStore store() {
        return store;
    }

    @Test
    void concurrent_writes_do_not_corrupt_state() throws InterruptedException {
        final int threads = 8;
        final int recordsPerThread = 100;
        final var latch = new CountDownLatch(threads);
        final var errors = new AtomicInteger(0);
        final var executor = Executors.newFixedThreadPool(threads);

        for (int t = 0; t < threads; t++) {
            final int threadId = t;
            executor.submit(() -> {
                try {
                    for (int i = 0; i < recordsPerThread; i++) {
                        store.record("agent-" + threadId, DegradationReason.RATE_LIMITED,
                                Instant.now().plusSeconds(60));
                        store.query("agent-" + threadId);
                    }
                } catch (final Exception e) {
                    errors.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            });
        }

        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        executor.shutdownNow();
        assertThat(errors.get()).isZero();
        for (int t = 0; t < threads; t++) {
            assertThat(store.query("agent-" + t)).contains(DegradationReason.RATE_LIMITED);
        }
    }
}
```

- [ ] **Step 4: Run all persistence-memory tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory
```

Expected: All tests pass (7 contract + 1 concurrency in InMemoryAgentStateStoreTest; 7 in InMemoryReactiveAgentStateStoreTest; existing InMemoryAgentRegistryTest etc. unchanged).

---

### Task 11: EidosProcessor — register migration resources for native image

**Files:**
- Modify: `deployment/src/main/java/io/casehub/eidos/deployment/EidosProcessor.java`

- [ ] **Step 1: Add the @BuildStep**

```java
// deployment/src/main/java/io/casehub/eidos/deployment/EidosProcessor.java
package io.casehub.eidos.deployment;

import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;
import io.quarkus.deployment.builditem.nativeimage.NativeImageResourcePatternsBuildItem;

class EidosProcessor {

    private static final String FEATURE = "eidos";

    @BuildStep
    FeatureBuildItem feature(EidosBuildTimeConfig config) {
        return new FeatureBuildItem(FEATURE);
    }

    @BuildStep
    NativeImageResourcePatternsBuildItem nativeFlywayResources() {
        return new NativeImageResourcePatternsBuildItem("db/eidos/migration/.*\\.sql");
    }
}
```

- [ ] **Step 2: Compile deployment module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl deployment
```

Expected: BUILD SUCCESS

---

### Task 12: Run full suite + commit Issue #7

- [ ] **Step 1: Run all tests across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass.

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  api/src/main/java/io/casehub/eidos/api/ReactiveAgentStateStore.java \
  api/src/main/java/io/casehub/eidos/api/ReactiveSystemPromptRenderer.java \
  api/src/test/java/io/casehub/eidos/api/AgentStateStoreContractTest.java \
  api/src/test/java/io/casehub/eidos/api/ReactiveAgentStateStoreContractTest.java \
  runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveAgentStateStore.java \
  runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealth.java \
  runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateEntity.java \
  runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/AgentDegradationStateReactivePanacheRepo.java \
  runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStore.java \
  runtime/src/main/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStore.java \
  runtime/src/main/java/io/casehub/eidos/runtime/renderer/DefaultReactiveSystemPromptRenderer.java \
  runtime/src/main/resources/db/eidos/migration/V2__agent_degradation_state.sql \
  runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthDefaultProfileTest.java \
  runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaAgentStateStoreTest.java \
  runtime/src/test/java/io/casehub/eidos/runtime/health/jpa/JpaReactiveAgentStateStoreTest.java \
  runtime/src/test/java/io/casehub/eidos/runtime/registry/BlockingReactiveParityTest.java \
  deployment/src/main/java/io/casehub/eidos/deployment/EidosProcessor.java \
  persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStore.java \
  persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryAgentStateStoreTest.java \
  persistence-memory/src/test/java/io/casehub/eidos/memory/InMemoryReactiveAgentStateStoreTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "$(cat <<'EOF'
feat(eidos#7): reactive parity — ReactiveAgentStateStore, ReactiveSystemPromptRenderer, JPA impls

New SPIs: ReactiveAgentStateStore (exact reactive mirror of AgentStateStore),
ReactiveSystemPromptRenderer (prep for eidos#17 async rendering).

Bridge impls (@DefaultBean): DefaultReactiveAgentStateStore and
DefaultReactiveSystemPromptRenderer — both use runSubscriptionOn(workerPool)
to avoid blocking the Vert.x event loop thread (LLM calls and JPA calls
are 500ms+ blocking operations).

JPA impls (gated on casehub.eidos.reactive.enabled):
- JpaAgentStateStore (blocking, enableIfMissing=true)
- JpaReactiveAgentStateStore (Hibernate Reactive Panache)
Schema: V2__agent_degradation_state.sql with TIMESTAMP WITH TIME ZONE
(UTC-safe expiry comparison).

InMemory: InMemoryReactiveAgentStateStore (@Alternative @Priority(1)),
delegates to InMemoryAgentStateStore via CDI injection.

DefaultReactiveCapabilityHealth: removed incorrect @IfBuildProperty gate,
added @DefaultBean — now activates under all profiles. Default-profile test
added to verify activation.

Abstract contract tests: AgentStateStoreContractTest and
ReactiveAgentStateStoreContractTest in api module. InMemoryAgentStateStoreTest
now extends AgentStateStoreContractTest.

Closes #7
EOF
)"
```

---

## PART 2 — Issue #14: GEMINI Rendering + Class Rename

---

### Task 13: IntelliJ rename ClaudeMarkdownRenderer → EidosSystemPromptRenderer

**Files:**
- Rename: `ClaudeMarkdownRenderer.java` → `EidosSystemPromptRenderer.java` (and all references)

- [ ] **Step 1: Use IntelliJ MCP to rename the class**

The rename must go through IntelliJ to update all references atomically (SemanticEnrichmentStep references `ClaudeMarkdownRenderer.PROMPT_TEMPLATE`; the test file; all imports).

Use the ide_refactor_rename MCP tool:
```
file: runtime/src/main/java/io/casehub/eidos/runtime/renderer/ClaudeMarkdownRenderer.java
line: 32  (the class declaration)
column: 14
newName: EidosSystemPromptRenderer
```

- [ ] **Step 2: Verify the rename updated SemanticEnrichmentStep**

Check that `SemanticEnrichmentStep.java` now reads `EidosSystemPromptRenderer.PROMPT_TEMPLATE` at line ~57 (was `ClaudeMarkdownRenderer.PROMPT_TEMPLATE`).

```bash
grep "ClaudeMarkdownRenderer" /Users/mdproctor/claude/casehub/eidos/runtime/src/main/java/io/casehub/eidos/runtime/renderer/SemanticEnrichmentStep.java
```

Expected: no output (all references updated).

- [ ] **Step 3: Verify test file was renamed**

```bash
ls /Users/mdproctor/claude/casehub/eidos/runtime/src/test/java/io/casehub/eidos/runtime/renderer/
```

Expected: `EidosSystemPromptRendererTest.java` exists; `ClaudeMarkdownRendererTest.java` absent.

- [ ] **Step 4: Run all tests to confirm rename didn't break anything**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass.

---

### Task 14: Implement assembleGemini() + assembleGeminiStructural()

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java`

The current `assembleGemini()` delegates to `assembleClaudeMarkdown()` as a placeholder. Replace it with proper GEMINI format assembly. Also update `assemble()` to accept `Optional<A2AEnrichment>` for the upcoming issue #13 (not yet populated — passed as `Optional.empty()` until Task 18).

- [ ] **Step 1: Write failing tests for GEMINI behaviour first (Task 15 will run them)**

Do Task 15 Step 1 (write tests) before implementing, so the tests fail before the implementation lands.

- [ ] **Step 2: Add imports to `EidosSystemPromptRenderer`**

Add `import java.util.stream.Collectors;` if not present. It's already imported; verify with:
```bash
grep "Collectors" /Users/mdproctor/claude/casehub/eidos/runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java
```

- [ ] **Step 3: Replace `assembleGemini()` with correct implementation**

In `EidosSystemPromptRenderer.java`, replace the `assembleGemini()` method (currently a placeholder delegating to `assembleClaudeMarkdown`) with:

```java
private String assembleGemini(final Optional<SemanticEnrichment> enrichment,
                               final AgentDescriptor descriptor,
                               final AgentPromptContext context) {
    final var sb = new StringBuilder();

    if (enrichment.isPresent()) {
        final SemanticEnrichment e = enrichment.get();
        sb.append(e.identityNarrative()).append(" ").append(e.roleNarrative()).append("\n");
        sb.append("\n").append(e.capabilityNarrative()).append("\n");
        e.dispositionNarrative().ifPresent(d -> sb.append("\n").append(d).append("\n"));
        e.constraintNarrative().ifPresent(c -> sb.append("\n").append(c).append("\n"));
        e.goalNarrative().ifPresent(g -> sb.append("\n").append(g).append("\n"));
    } else {
        assembleGeminiStructural(sb, descriptor, context);
    }

    // Resources — label(uri) format, no space before paren (explicit delta from OPENAI_SYSTEM label (uri))
    if (!context.resources().isEmpty()) {
        sb.append("\nResources: ");
        final var resources = context.resources().stream()
                .map(r -> (r.label() != null ? r.label() : r.uri()) + "(" + r.uri() + ")")
                .collect(Collectors.joining(", "));
        sb.append(resources).append("\n");
    }

    if (context.situationalContext() != null) {
        sb.append("\n").append(context.situationalContext()).append("\n");
    }

    return sb.toString().trim();
}

private void assembleGeminiStructural(final StringBuilder sb,
                                       final AgentDescriptor descriptor,
                                       final AgentPromptContext context) {
    sb.append(descriptor.name());
    if (descriptor.slot() != null) sb.append(", ").append(descriptor.slot());
    sb.append(".");
    if (descriptor.version() != null) sb.append(" Version ").append(descriptor.version()).append(".");
    sb.append("\n");

    if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
        sb.append("\nCapabilities: ");
        final var names = descriptor.capabilities().stream()
                .map(AgentCapability::name)
                .collect(Collectors.joining(", "));
        sb.append(names).append(".\n");
    }

    if (descriptor.disposition() != null) {
        final AgentDisposition d = descriptor.disposition();
        sb.append("\nOperating style:");
        if (d.ruleFollowing() != null) sb.append(" ").append(d.ruleFollowing()).append(" rule-following.");
        if (d.autonomy() != null)      sb.append(" Autonomy: ").append(d.autonomy()).append(".");
        sb.append(" Can delegate: ").append(d.delegation() ? "yes" : "no").append(".\n");
    }

    context.goal().ifPresent(goal -> {
        sb.append("\nGoal: ").append(goal.description()).append(".\n");
        if (!goal.subGoals().isEmpty()) {
            sb.append("Sub-goals: ").append(String.join(", ", goal.subGoals())).append(".\n");
        }
    });
}
```

- [ ] **Step 4: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: BUILD SUCCESS

---

### Task 15: GEMINI tests + delete placeholder test

**Files:**
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`

- [ ] **Step 1: Write failing GEMINI tests (do this before Task 14 Step 3 if following TDD strictly)**

Add these test methods to `EidosSystemPromptRendererTest`:

```java
// ── GEMINI path (new tests) ───────────────────────────────────────────────

@Test
void gemini_structural_has_no_markdown_headers() {
    final var ctx = AgentPromptContext.forFormat(GEMINI);
    final var result = rendererStructural.render(fullDescriptor(), ctx);
    assertThat(result.content()).doesNotContain("#");
}

@Test
void gemini_enriched_has_no_markdown_headers() {
    final var ctx = AgentPromptContext.forFormat(GEMINI);
    final var result = rendererWithLlm.render(fullDescriptor(), ctx);
    assertThat(result.content()).doesNotContain("#");
}

@Test
void gemini_enriched_contains_identity_and_role_narrative() {
    final var ctx = AgentPromptContext.forFormat(GEMINI);
    final var result = rendererWithLlm.render(fullDescriptor(), ctx);
    // LLM_RESPONSE is the identityNarrative ("You are a code reviewer specialising in Java.")
    assertThat(result.content()).contains(LLM_RESPONSE);
    assertThat(result.content()).contains("Your role is to review code.");
}

@Test
void gemini_enriched_resources_format_uses_no_space_before_paren() {
    final var ctx = AgentPromptContext.forFormat(GEMINI)
            .withResources(List.of(new Resource("https://api.example.com", "API docs", "uri")));
    final var result = rendererWithLlm.render(fullDescriptor(), ctx);
    // GEMINI: "API docs(https://api.example.com)" — no space before paren
    assertThat(result.content()).contains("API docs(https://api.example.com)");
    assertThat(result.content()).doesNotContain("API docs (https://api.example.com)");
}
```

- [ ] **Step 2: Delete the placeholder test**

In `EidosSystemPromptRendererTest`, delete the entire method `gemini_structural_produces_same_content_as_claude_md_structural` (it tested placeholder behaviour that the fix deliberately removes).

- [ ] **Step 3: Run GEMINI tests — all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosSystemPromptRendererTest
```

Expected: All tests pass. Note: `gemini_structural_produces_same_content_as_claude_md_structural` is gone; all 4 new GEMINI tests pass.

- [ ] **Step 4: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass.

---

### Task 16: Commit Issue #14

- [ ] **Step 1: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "$(cat <<'EOF'
feat(eidos#14): rename ClaudeMarkdownRenderer → EidosSystemPromptRenderer; implement GEMINI format

Class renamed via IntelliJ refactoring — all references (SemanticEnrichmentStep,
test file, imports) updated atomically.

GEMINI format: assembleGemini() now produces format-specific prose output.
- Enriched path: narratives concatenated with blank-line separators, no headers
- Structural path: separate assembleGeminiStructural() method producing dense
  prose (same concern shape as OPENAI_SYSTEM structural, isolated for future
  format divergence)
- Resources format: label(uri) with no space before paren — explicit delta
  from OPENAI_SYSTEM label (uri)

Tests: gemini_structural_produces_same_content_as_claude_md_structural deleted
(tested placeholder behaviour). New tests: structural has no headers, enriched
has no headers, enriched passes narratives through, resources format delta.

Closes #14
EOF
)"
```

---

## PART 3 — Issue #13: A2A Per-Capability Narratives

---

### Task 17: A2AEnrichment record + A2ASemanticEnrichmentStep

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/A2AEnrichment.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/A2ASemanticEnrichmentStep.java`

- [ ] **Step 1: Write `A2AEnrichment` record**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/A2AEnrichment.java
package io.casehub.eidos.runtime.renderer;

import java.util.List;

record A2AEnrichment(List<CapabilityNarrative> capabilityNarratives) {
    record CapabilityNarrative(String name, String description) {}
}
```

- [ ] **Step 2: Write `A2ASemanticEnrichmentStep`**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/renderer/A2ASemanticEnrichmentStep.java
package io.casehub.eidos.runtime.renderer;

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
import dev.langchain4j.model.chat.request.json.JsonArraySchema;
import dev.langchain4j.model.chat.request.json.JsonObjectSchema;
import dev.langchain4j.model.chat.request.json.JsonSchema;
import org.jboss.logging.Logger;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

class A2ASemanticEnrichmentStep {

    private static final Logger log = Logger.getLogger(A2ASemanticEnrichmentStep.class);

    static final String A2A_PROMPT_TEMPLATE = """
            You are writing per-capability descriptions for an AI agent's A2A (agent-to-agent) card.

            Given the agent's capabilities in JSON, produce a JSON object with one prose description
            per declared capability. Write in second person, addressing the agent directly.

            RULES:
            - Copy the capability name exactly as given — do not paraphrase or change capitalisation.
            - Each description is 1-2 sentences. Second person ("You can...").
            - Plain prose. No markdown, no bullet points.
            - Return ONLY the JSON object. No explanation, no preamble, no code fences.
            - If no capabilities are declared, return {"capabilityNarratives": []}.""";

    static final ResponseFormat A2A_RESPONSE_FORMAT = ResponseFormat.builder()
            .type(ResponseFormatType.JSON)
            .jsonSchema(JsonSchema.builder()
                    .name("A2AEnrichment")
                    .rootElement(JsonObjectSchema.builder()
                            .addProperty("capabilityNarratives", JsonArraySchema.builder()
                                    .description("One entry per declared capability. Empty array [] if none.")
                                    .items(JsonObjectSchema.builder()
                                            .addStringProperty("name",
                                                    "Capability name — must match exactly as given.")
                                            .addStringProperty("description",
                                                    "1-2 sentences, second person, what this agent can do with this capability.")
                                            .required("name", "description")
                                            .build())
                                    .build())
                            .required("capabilityNarratives")
                            .build())
                    .build())
            .build();

    private final ObjectMapper mapper;

    A2ASemanticEnrichmentStep(final ObjectMapper mapper) {
        this.mapper = mapper;
    }

    Optional<A2AEnrichment> enrich(final ChatModel llm, final ObjectNode descriptorNode) {
        try {
            final ChatRequest request = ChatRequest.builder()
                    .messages(
                            SystemMessage.from(A2A_PROMPT_TEMPLATE),
                            UserMessage.from(mapper.writeValueAsString(descriptorNode))
                    )
                    .responseFormat(A2A_RESPONSE_FORMAT)
                    .build();

            final var response = llm.chat(request);
            return Optional.of(parse(response.aiMessage().text()));

        } catch (final Exception e) {
            log.warn("A2A enrichment failed (" + e.getMessage()
                    + "), falling back to structural A2A rendering");
            return Optional.empty();
        }
    }

    private A2AEnrichment parse(final String json) throws JsonProcessingException {
        final JsonNode node = mapper.readTree(json);
        final JsonNode narrativesNode = node.get("capabilityNarratives");
        if (narrativesNode == null || !narrativesNode.isArray()) {
            return new A2AEnrichment(List.of());
        }
        final List<A2AEnrichment.CapabilityNarrative> narratives = new ArrayList<>();
        for (final JsonNode item : narrativesNode) {
            final String name = item.path("name").asText(null);
            final String description = item.path("description").asText(null);
            if (name != null && !name.isBlank() && description != null && !description.isBlank()) {
                narratives.add(new A2AEnrichment.CapabilityNarrative(name, description));
            }
        }
        return new A2AEnrichment(List.copyOf(narratives));
    }
}
```

- [ ] **Step 3: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: BUILD SUCCESS. If `JsonArraySchema` import fails (not in LangChain4j version used), check what array schema API is available: `JsonObjectSchema.builder().addArrayProperty(...)` may be an alternative.

---

### Task 18: Update EidosSystemPromptRenderer — Stage 2b + assembleA2aCard

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRenderer.java`

These changes wire the A2A enrichment path into the renderer without touching the narrative path.

- [ ] **Step 1: Write failing A2A enrichment tests first (Task 19 Step 1 — do that before this step)**

- [ ] **Step 2: Add `a2aEnrichmentStep` field and initialize it in both constructors**

In `EidosSystemPromptRenderer`, add the field and constructor initialization alongside `enrichmentStep`:

```java
// New field (add alongside enrichmentStep):
private final A2ASemanticEnrichmentStep a2aEnrichmentStep;

// In CDI constructor (@Inject), add:
this.a2aEnrichmentStep = new A2ASemanticEnrichmentStep(mapper);

// In package-private test constructor, add:
this.a2aEnrichmentStep = new A2ASemanticEnrichmentStep(mapper);
```

- [ ] **Step 3: Update `render()` to add Stage 2b and pass `a2aEnrichment` to `assemble()`**

In the `render()` method, after the Stage 2a enrichment block, add Stage 2b:

```java
// Stage 2b: A2A enrichment — descriptor-only payload, separate schema
Optional<A2AEnrichment> a2aEnrichment = Optional.empty();
if (context.format() == RenderFormat.A2A_CARD && llm != null) {
    a2aEnrichment = a2aEnrichmentStep.enrich(llm, descriptorNode);
}
```

Update the `assemble()` call to pass `a2aEnrichment`:
```java
final String content = assemble(enrichment, a2aEnrichment, descriptor, context);
```

- [ ] **Step 4: Update `assemble()` signature and A2A routing**

```java
private String assemble(final Optional<SemanticEnrichment> enrichment,
                         final Optional<A2AEnrichment> a2aEnrichment,
                         final AgentDescriptor descriptor,
                         final AgentPromptContext context) {
    return switch (context.format()) {
        case CLAUDE_MD     -> assembleClaudeMarkdown(enrichment, descriptor, context);
        case OPENAI_SYSTEM -> assembleOpenAiSystem(enrichment, descriptor, context);
        case GEMINI        -> assembleGemini(enrichment, descriptor, context);
        case A2A_CARD      -> assembleA2aCard(a2aEnrichment, descriptor);
    };
}
```

- [ ] **Step 5: Replace `assembleA2aCard()` with enrichment-aware version**

Replace the existing `assembleA2aCard(AgentDescriptor descriptor)` with:

```java
private String assembleA2aCard(final Optional<A2AEnrichment> enrichment,
                                 final AgentDescriptor descriptor) {
    final ObjectNode card = mapper.createObjectNode();
    card.put("name", descriptor.name());
    card.put("agentId", descriptor.agentId());
    addIfPresent(card, "version", descriptor.version());

    if (descriptor.capabilities() != null && !descriptor.capabilities().isEmpty()) {
        final java.util.Map<String, String> descriptionByName = enrichment
            .map(e -> e.capabilityNarratives().stream()
                .collect(java.util.stream.Collectors.toMap(
                    A2AEnrichment.CapabilityNarrative::name,
                    A2AEnrichment.CapabilityNarrative::description,
                    (a, b) -> a)))
            .orElse(java.util.Map.of());

        final ArrayNode capsArray = card.putArray("capabilities");
        for (final AgentCapability cap : descriptor.capabilities()) {
            final ObjectNode capNode = capsArray.addObject();
            capNode.put("name", cap.name());
            if (cap.qualityHint() != null) capNode.put("qualityHint", cap.qualityHint());
            final String desc = descriptionByName.get(cap.name());
            if (desc != null) capNode.put("description", desc);
        }
    }

    try {
        return mapper.writeValueAsString(card);
    } catch (final com.fasterxml.jackson.core.JsonProcessingException ex) {
        throw new IllegalStateException("A2A card serialization failed", ex);
    }
}
```

- [ ] **Step 6: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```

Expected: BUILD SUCCESS

---

### Task 19: A2A tests + delete old test

**Files:**
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosSystemPromptRendererTest.java`

- [ ] **Step 1: Write the four new A2A tests (do this before Task 18 steps if following TDD)**

First, update the `LLM_JSON_RESPONSE` constant or add a separate A2A JSON mock. The existing `LLM_JSON_RESPONSE` is for narrative enrichment (SemanticEnrichment fields). For A2A tests, the mock LLM must return the A2A schema format. Add a constant:

```java
static final String A2A_LLM_JSON_RESPONSE =
    "{\"capabilityNarratives\":[{\"name\":\"code-review\","
    + "\"description\":\"You conduct thorough Java code reviews, checking for correctness and style.\"}]}";
```

The A2A test setup needs a renderer where the mock LLM returns A2A JSON for A2A format calls. Since the same `rendererWithLlm` is used for all formats and the mock always returns the same JSON, we need a separate mock or to make the mock format-aware. Use a format-aware mock in a helper method:

```java
static EidosSystemPromptRenderer rendererWithA2aLlm(final ObjectMapper mapper) {
    final ChatModel a2aLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            // Return A2A schema JSON regardless of the request details
            return ChatResponse.builder().aiMessage(AiMessage.from(A2A_LLM_JSON_RESPONSE)).build();
        }
    };
    return new EidosSystemPromptRenderer(a2aLlm, new CdiVocabularyRegistry(),
            new NoOpRenderedPromptCache(), mapper);
}
```

Add these four tests in the A2A section:

```java
@Test
void a2a_card_enriched_includes_capability_descriptions() {
    final var renderer = rendererWithA2aLlm(MAPPER);
    final var ctx = AgentPromptContext.forFormat(A2A_CARD);
    final var result = renderer.render(fullDescriptor(), ctx);
    assertThat(result.content()).contains("\"description\"");
    assertThat(result.content()).contains("You conduct thorough Java code reviews");
}

@Test
void a2a_card_structural_omits_descriptions() {
    final var ctx = AgentPromptContext.forFormat(A2A_CARD);
    final var result = rendererStructural.render(fullDescriptor(), ctx);
    assertThat(result.content()).doesNotContain("\"description\"");
    assertThat(result.content()).contains("\"name\"");
    assertThat(result.content()).contains("code-review");
}

@Test
void a2a_card_enriched_matches_capability_names() {
    final var renderer = rendererWithA2aLlm(MAPPER);
    final var ctx = AgentPromptContext.forFormat(A2A_CARD);
    final var result = renderer.render(fullDescriptor(), ctx);
    // The narrative for "code-review" is present; agentId and name are also present
    assertThat(result.content()).contains("\"code-review\"");
    assertThat(result.content()).contains("You conduct thorough Java code reviews");
}

@Test
void a2a_card_enriched_ignores_unmatched_narrative_names() {
    // LLM returns a narrative for a capability name not declared in the descriptor
    final ChatModel mismatchLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            return ChatResponse.builder().aiMessage(AiMessage.from(
                "{\"capabilityNarratives\":[{\"name\":\"nonexistent-cap\","
                + "\"description\":\"You do things that don't exist.\"}]}"
            )).build();
        }
    };
    final var renderer = new EidosSystemPromptRenderer(mismatchLlm, new CdiVocabularyRegistry(),
            new NoOpRenderedPromptCache(), MAPPER);
    final var ctx = AgentPromptContext.forFormat(A2A_CARD);

    // Should not throw; description field absent for "code-review" (no matching narrative)
    assertThatCode(() -> renderer.render(fullDescriptor(), ctx)).doesNotThrowAnyException();
    final var result = renderer.render(fullDescriptor(), ctx);
    assertThat(result.content()).contains("\"code-review\"");
    assertThat(result.content()).doesNotContain("You do things that don't exist.");
    assertThat(result.content()).doesNotContain("nonexistent-cap");
}
```

- [ ] **Step 2: Delete `a2a_card_skips_llm_even_when_llm_is_configured`**

Delete the entire test method `a2a_card_skips_llm_even_when_llm_is_configured` — A2A now invokes LLM.

- [ ] **Step 3: Also verify A2A payload excludes goal context**

Add one more test to confirm the A2A LLM call does not include goal context (descriptor-only payload):

```java
@Test
void a2a_card_llm_payload_excludes_goal_context() {
    final String[] capturedPayload = {""};
    final ChatModel capturingLlm = new ChatModel() {
        @Override
        public ChatResponse doChat(final ChatRequest request) {
            capturedPayload[0] = request.messages().stream()
                .filter(m -> m instanceof UserMessage)
                .map(m -> ((UserMessage) m).singleText())
                .reduce("", (a, b) -> a + b);
            return ChatResponse.builder().aiMessage(AiMessage.from(A2A_LLM_JSON_RESPONSE)).build();
        }
    };
    final var renderer = new EidosSystemPromptRenderer(capturingLlm, new CdiVocabularyRegistry(),
            new NoOpRenderedPromptCache(), MAPPER);
    renderer.render(fullDescriptor(), AgentPromptContext.forFormat(A2A_CARD)
            .withGoal(new GoalContext("Review PR #42", List.of(), "case-123")));

    // Goal must not appear in the A2A payload
    assertThat(capturedPayload[0]).doesNotContain("Review PR #42");
    assertThat(capturedPayload[0]).doesNotContain("case-123");
    // But capability names must be present (descriptor is included)
    assertThat(capturedPayload[0]).contains("code-review");
}
```

- [ ] **Step 4: Run all renderer tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EidosSystemPromptRendererTest
```

Expected: All tests pass. Old `a2a_card_skips_llm_even_when_llm_is_configured` is gone; 5 new A2A tests pass.

- [ ] **Step 5: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass.

---

### Task 20: Commit Issue #13

- [ ] **Step 1: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "$(cat <<'EOF'
feat(eidos#13): A2A per-capability narratives via dedicated A2ASemanticEnrichmentStep

Adds A2AEnrichment record and A2ASemanticEnrichmentStep — a separate enrichment
path for A2A_CARD format that:
- Uses a descriptor-only payload (no goal context — A2A cards are context-independent)
- Has its own focused schema requesting only capabilityNarratives (not all six
  narrative fields), avoiding token waste on every CLAUDE_MD/OPENAI_SYSTEM/GEMINI
  render
- Returns Optional.empty() on failure, gracefully degrading to structural A2A

EidosSystemPromptRenderer changes:
- render(): adds Stage 2b for A2A enrichment, parallel to narrative Stage 2a
- assemble(): now accepts Optional<A2AEnrichment> alongside Optional<SemanticEnrichment>
- assembleA2aCard(): matches LLM capability narratives to declared capabilities by
  name (exact string equality); mismatched/omitted names result in absent description
  field (silent omission, not failure)

Tests: a2a_card_skips_llm_even_when_llm_is_configured deleted. New tests:
enriched includes descriptions, structural omits descriptions, names match,
mismatched names ignored (no throw), goal excluded from A2A LLM payload.

Closes #13
EOF
)"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task covering it |
|---|---|
| ReactiveAgentStateStore SPI | Task 1 |
| ReactiveSystemPromptRenderer SPI | Task 1 |
| BlockingReactiveParityTest extensions | Task 2 |
| Abstract contract tests | Task 3 |
| DefaultReactiveCapabilityHealth fix | Task 4 |
| Default-profile test for DefaultReactiveCapabilityHealth | Task 4 |
| DefaultReactiveAgentStateStore with runSubscriptionOn | Task 5 |
| DefaultReactiveSystemPromptRenderer with runSubscriptionOn | Task 6 |
| V2 migration with TIMESTAMP WITH TIME ZONE | Task 7 |
| JpaAgentStateStore with flush/clear | Task 8 |
| JpaReactiveAgentStateStore | Task 9 |
| InMemoryReactiveAgentStateStore | Task 10 |
| InMemoryAgentStateStoreTest extends contract | Task 10 |
| EidosProcessor native image @BuildStep | Task 11 |
| Rename ClaudeMarkdownRenderer → EidosSystemPromptRenderer | Task 13 |
| GEMINI enriched path (prose, no headers) | Task 14 |
| GEMINI structural path (separate method) | Task 14 |
| GEMINI resources format delta documented + tested | Tasks 14+15 |
| gemini_structural_produces_same_content... deleted | Task 15 |
| A2AEnrichment record | Task 17 |
| A2ASemanticEnrichmentStep (descriptor-only, own schema) | Task 17 |
| render() Stage 2b A2A enrichment | Task 18 |
| assembleA2aCard() with Optional<A2AEnrichment> | Task 18 |
| a2a_card_skips_llm... deleted | Task 19 |
| A2A enriched tests (4 new) | Task 19 |
| A2A goal-excluded-from-payload test | Task 19 |
| Commit per issue | Tasks 12, 16, 20 |

All spec requirements covered.
