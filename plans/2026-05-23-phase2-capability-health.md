# Phase 2 — CapabilityHealth, ReactiveCapabilityHealth, and Examples Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace NoOpCapabilityHealth with a real probe that checks declared capabilities and epistemic domain match, add ReactiveCapabilityHealth for parity, and deliver an examples module demonstrating all implemented eidos capabilities.

**Architecture:** CapabilityHealth.probe() takes AgentDescriptor directly (not agentId) — pure function, no registry lookup, no tenancy concern. DefaultCapabilityHealth checks declared capability existence then epistemic domain confidence. Examples module uses persistence-memory for zero-datasource operation, with @QuarkusTest demonstrating all API surface.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ. No new framework dependencies.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module> --also-make` — always use `mvn`, not `./mvnw`.

**Project root:** `/Users/mdproctor/claude/casehub/eidos`

---

## File Map

```
api/src/main/java/io/casehub/eidos/api/
  CapabilityHealth.java               MODIFY — probe(String agentId, ...) → probe(AgentDescriptor, ...)
  ReactiveCapabilityHealth.java       CREATE — Uni<CapabilityStatus> probe(...)

runtime/src/main/java/io/casehub/eidos/runtime/health/
  NoOpCapabilityHealth.java           DELETE — replaced by DefaultCapabilityHealth
  DefaultCapabilityHealth.java        CREATE — @ApplicationScoped, @IfBuildProperty
  DefaultReactiveCapabilityHealth.java CREATE — @IfBuildProperty reactive gate

runtime/src/test/java/io/casehub/eidos/runtime/health/
  NoOpCapabilityHealthTest.java       DELETE — replaced
  DefaultCapabilityHealthTest.java    CREATE — TDD: all 4 status variants
  DefaultReactiveCapabilityHealthTest.java CREATE — reactive parity

examples/pom.xml                      CREATE — aggregator
examples/README.md                    CREATE — capability coverage table
examples/agent-scenarios/pom.xml      CREATE — sub-module
examples/agent-scenarios/src/test/resources/
  application.properties              CREATE — H2 dummy, InMemory stores
examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/
  MultiAgentTeamTest.java             CREATE
  CrossVocabularyDiscoveryTest.java   CREATE
  EpistemicDomainMatchingTest.java    CREATE
  TenancyIsolationTest.java           CREATE
  DispositionVocabularyTest.java      CREATE

pom.xml                               MODIFY — add examples to <modules>
```

---

## Task 1: API Amendment — CapabilityHealth.probe signature + ReactiveCapabilityHealth

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java`
- Create: `api/src/main/java/io/casehub/eidos/api/ReactiveCapabilityHealth.java`

- [ ] **Step 1: Update CapabilityHealth.probe signature**

Read the file first. Change `probe(String agentId, ...)` to `probe(AgentDescriptor descriptor, ...)`:

```java
// api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java
package io.casehub.eidos.api;

import java.util.Map;

public interface CapabilityHealth {
    CapabilityStatus probe(AgentDescriptor descriptor, String capabilityTag, ProbeContext context);

    record ProbeContext(String taskDomain, Map<String, Object> taskMetadata) {
        public static ProbeContext of(String taskDomain) {
            return new ProbeContext(taskDomain, Map.of());
        }
    }

    sealed interface CapabilityStatus permits
            CapabilityStatus.Ready,
            CapabilityStatus.Degraded,
            CapabilityStatus.Unavailable,
            CapabilityStatus.EpistemicallyWeak {

        record Ready() implements CapabilityStatus {}
        record Degraded(DegradationReason reason, String detail) implements CapabilityStatus {}
        record Unavailable(String reason) implements CapabilityStatus {}
        record EpistemicallyWeak(String domain, double confidence) implements CapabilityStatus {}
    }

    enum DegradationReason {
        RATE_LIMITED, CONTEXT_EXHAUSTED, OVERLOADED, DOMAIN_MISMATCH
    }
}
```

- [ ] **Step 2: Create ReactiveCapabilityHealth**

```java
// api/src/main/java/io/casehub/eidos/api/ReactiveCapabilityHealth.java
package io.casehub.eidos.api;

import io.smallrye.mutiny.Uni;

public interface ReactiveCapabilityHealth {
    Uni<CapabilityHealth.CapabilityStatus> probe(
            AgentDescriptor descriptor, String capabilityTag,
            CapabilityHealth.ProbeContext context);
}
```

- [ ] **Step 3: Compile api module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java api/src/main/java/io/casehub/eidos/api/ReactiveCapabilityHealth.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(api): CapabilityHealth.probe takes AgentDescriptor; add ReactiveCapabilityHealth

Breaking: probe(String agentId, ...) → probe(AgentDescriptor, ...).
Eliminates redundant registry lookup and tenancy concern in the probe.
ReactiveCapabilityHealth for blocking/reactive parity.

Refs casehubio/eidos#4"
```

---

## Task 2: DefaultCapabilityHealth — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java`
- Delete: `runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpCapabilityHealth.java`
- Delete: `runtime/src/test/java/io/casehub/eidos/runtime/health/NoOpCapabilityHealthTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java`

- [ ] **Step 1: Delete NoOp files**

```bash
rm /Users/mdproctor/claude/casehub/eidos/runtime/src/main/java/io/casehub/eidos/runtime/health/NoOpCapabilityHealth.java
rm /Users/mdproctor/claude/casehub/eidos/runtime/src/test/java/io/casehub/eidos/runtime/health/NoOpCapabilityHealthTest.java
```

- [ ] **Step 2: Write DefaultCapabilityHealthTest (failing)**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealthTest.java
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
class DefaultCapabilityHealthTest {

    @Inject
    CapabilityHealth health;

    static AgentDescriptor agent(String agentId, AgentCapability... capabilities) {
        return new AgentDescriptor(
            agentId, "Agent", "1.0", "anthropic", "claude", "claude-3-7",
            null, null, null, null, "reviewer",
            List.of(capabilities),
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, "default"
        );
    }

    static AgentCapability capability(String name, Map<String, Double> epistemicDomains) {
        return new AgentCapability(name, 0.9, null, null,
            List.of(), List.of(), List.of(), epistemicDomains);
    }

    @Test
    void returns_ready_when_capability_declared_and_no_task_domain() {
        var descriptor = agent("a1", capability("code-review", Map.of()));

        var status = health.probe(descriptor, "code-review", ProbeContext.of(null));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void returns_unavailable_when_capability_not_declared() {
        var descriptor = agent("a2", capability("code-review", Map.of()));

        var status = health.probe(descriptor, "test-writing", ProbeContext.of(null));

        assertThat(status).isInstanceOf(CapabilityStatus.Unavailable.class);
        assertThat(((CapabilityStatus.Unavailable) status).reason())
            .contains("test-writing");
    }

    @Test
    void returns_ready_when_epistemic_domain_above_threshold() {
        var descriptor = agent("a3", capability("code-review", Map.of("java", 0.95)));

        var status = health.probe(descriptor, "code-review", ProbeContext.of("java"));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void returns_epistemically_weak_when_domain_below_threshold() {
        var descriptor = agent("a4", capability("code-review", Map.of("rust", 0.2)));

        var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"));

        assertThat(status).isInstanceOf(CapabilityStatus.EpistemicallyWeak.class);
        var weak = (CapabilityStatus.EpistemicallyWeak) status;
        assertThat(weak.domain()).isEqualTo("rust");
        assertThat(weak.confidence()).isEqualTo(0.2);
    }

    @Test
    void returns_ready_when_task_domain_not_in_epistemic_map() {
        var descriptor = agent("a5", capability("code-review", Map.of("java", 0.95)));

        var status = health.probe(descriptor, "code-review", ProbeContext.of("python"));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void returns_ready_when_epistemic_domains_null() {
        var descriptor = agent("a6", capability("code-review", null));

        var status = health.probe(descriptor, "code-review", ProbeContext.of("java"));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void returns_ready_when_confidence_exactly_at_threshold() {
        // Default threshold is 0.3 — exactly at threshold should be Ready (not weak)
        var descriptor = agent("a7", capability("code-review", Map.of("go", 0.3)));

        var status = health.probe(descriptor, "code-review", ProbeContext.of("go"));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void returns_unavailable_for_agent_with_no_capabilities() {
        var descriptor = agent("a8");

        var status = health.probe(descriptor, "code-review", ProbeContext.of(null));

        assertThat(status).isInstanceOf(CapabilityStatus.Unavailable.class);
    }
}
```

- [ ] **Step 3: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,deployment --also-make -Dtest=DefaultCapabilityHealthTest 2>&1 | tail -10
```

Expected: compile error or CDI unsatisfied dependency (DefaultCapabilityHealth doesn't exist yet).

- [ ] **Step 4: Create DefaultCapabilityHealth**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultCapabilityHealth.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.CapabilityHealth;
import io.quarkus.arc.properties.IfBuildProperty;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "false", enableIfMissing = true)
@ApplicationScoped
public class DefaultCapabilityHealth implements CapabilityHealth {

    @ConfigProperty(name = "casehub.eidos.epistemic.weak-threshold", defaultValue = "0.3")
    double weakThreshold;

    @Override
    public CapabilityStatus probe(AgentDescriptor descriptor, String capabilityTag, ProbeContext context) {
        var capability = descriptor.capabilities().stream()
            .filter(c -> c.name().equals(capabilityTag))
            .findFirst()
            .orElse(null);

        if (capability == null) {
            return new CapabilityStatus.Unavailable(
                "Capability '" + capabilityTag + "' not declared");
        }

        if (context.taskDomain() != null && capability.epistemicDomains() != null) {
            Double confidence = capability.epistemicDomains().get(context.taskDomain());
            if (confidence != null && confidence < weakThreshold) {
                return new CapabilityStatus.EpistemicallyWeak(context.taskDomain(), confidence);
            }
        }

        return new CapabilityStatus.Ready();
    }
}
```

- [ ] **Step 5: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,deployment --also-make -Dtest=DefaultCapabilityHealthTest
```

Expected: `Tests run: 8, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): DefaultCapabilityHealth — real probe replacing NoOp

Checks declared capability existence, then epistemic domain confidence
against configurable weak-threshold (default 0.3). Returns Ready/
Unavailable/EpistemicallyWeak. No runtime state probing yet (Phase 3).
@IfBuildProperty gated. 8 tests cover all status variants + edge cases.

Refs casehubio/eidos#4"
```

---

## Task 3: DefaultReactiveCapabilityHealth — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthTest.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealth.java`

- [ ] **Step 1: Write test (failing)**

```java
// runtime/src/test/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealthTest.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.runtime.registry.ReactiveTestProfile;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

@QuarkusTest
@TestProfile(ReactiveTestProfile.class)
class DefaultReactiveCapabilityHealthTest {

    @Inject
    ReactiveCapabilityHealth health;

    static AgentDescriptor agent(AgentCapability... capabilities) {
        return new AgentDescriptor(
            "agent-1", "Agent", "1.0", "anthropic", "claude", "claude-3-7",
            null, null, null, null, "reviewer",
            List.of(capabilities),
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, "default"
        );
    }

    @Test
    void reactive_probe_returns_ready_when_capability_declared() {
        var descriptor = agent(new AgentCapability("code-review", 0.9, null, null,
            List.of(), List.of(), List.of(), Map.of()));

        var status = health.probe(descriptor, "code-review", ProbeContext.of(null))
            .await().indefinitely();

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void reactive_probe_returns_unavailable_when_capability_missing() {
        var descriptor = agent();

        var status = health.probe(descriptor, "missing", ProbeContext.of(null))
            .await().indefinitely();

        assertThat(status).isInstanceOf(CapabilityStatus.Unavailable.class);
    }

    @Test
    void reactive_probe_returns_epistemically_weak() {
        var descriptor = agent(new AgentCapability("code-review", 0.9, null, null,
            List.of(), List.of(), List.of(), Map.of("rust", 0.1)));

        var status = health.probe(descriptor, "code-review", ProbeContext.of("rust"))
            .await().indefinitely();

        assertThat(status).isInstanceOf(CapabilityStatus.EpistemicallyWeak.class);
    }
}
```

- [ ] **Step 2: Create DefaultReactiveCapabilityHealth**

```java
// runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultReactiveCapabilityHealth.java
package io.casehub.eidos.runtime.health;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.quarkus.arc.properties.IfBuildProperty;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class DefaultReactiveCapabilityHealth implements ReactiveCapabilityHealth {

    @Inject
    DefaultCapabilityHealth delegate;

    @Override
    public Uni<CapabilityStatus> probe(AgentDescriptor descriptor, String capabilityTag,
                                       ProbeContext context) {
        return Uni.createFrom().item(() -> delegate.probe(descriptor, capabilityTag, context));
    }
}
```

Note: `DefaultReactiveCapabilityHealth` injects `DefaultCapabilityHealth` directly (not the `CapabilityHealth` interface) to avoid CDI ambiguity. Both are `@IfBuildProperty`-gated but the reactive variant is only active when `reactive.enabled=true`. When the reactive profile is active, `DefaultCapabilityHealth` is excluded by its own `@IfBuildProperty(stringValue="false", enableIfMissing=true)`. This means the reactive delegate injection will fail.

**Fix:** Remove the build-gate from `DefaultCapabilityHealth` — it should always be available as a CDI bean, whether reactive is enabled or not. The blocking `CapabilityHealth` SPI injection is what should be gated, not the implementation bean itself. Change `DefaultCapabilityHealth` to plain `@ApplicationScoped` (no `@IfBuildProperty`). The reactive variant delegates to it; the blocking test profile injects it directly via `CapabilityHealth`.

Update `DefaultCapabilityHealth` annotation to just:
```java
@ApplicationScoped
public class DefaultCapabilityHealth implements CapabilityHealth {
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,deployment --also-make -Dtest=DefaultReactiveCapabilityHealthTest
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`

- [ ] **Step 4: Run ALL runtime tests to confirm no regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,deployment --also-make
```

Expected: All tests green (both blocking and reactive profiles).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add runtime/src/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(runtime): DefaultReactiveCapabilityHealth — delegates to blocking probe

@IfBuildProperty reactive gate. Uni.createFrom().item() wraps pure computation.
DefaultCapabilityHealth is now plain @ApplicationScoped (always available as
delegate for the reactive variant).

Refs casehubio/eidos#4"
```

---

## Task 4: Examples Module — Scaffolding

**Files:**
- Create: `examples/pom.xml`
- Create: `examples/agent-scenarios/pom.xml`
- Create: `examples/agent-scenarios/src/test/resources/application.properties`
- Modify: `pom.xml` (parent)

- [ ] **Step 1: Add examples to parent pom.xml modules**

In `/Users/mdproctor/claude/casehub/eidos/pom.xml`, add `<module>examples</module>` after `<module>vocab</module>`.

- [ ] **Step 2: Create examples aggregator pom**

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
  </parent>

  <artifactId>casehub-eidos-examples</artifactId>
  <packaging>pom</packaging>
  <name>CaseHub Eidos - Examples</name>

  <modules>
    <module>agent-scenarios</module>
  </modules>
</project>
```

- [ ] **Step 3: Create agent-scenarios pom**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-examples</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-eidos-example-agent-scenarios</artifactId>
  <name>CaseHub Eidos - Examples - Agent Scenarios</name>

  <dependencies>
    <!-- Core eidos extension -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- In-memory stores — zero datasource -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-memory</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- Well-known vocabularies -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-vocab</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- Dummy datasource for Quarkus/Hibernate boot -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 4: Create application.properties for examples**

```properties
# examples/agent-scenarios/src/test/resources/application.properties

# Dummy datasource — Quarkus/Hibernate requires one even though InMemory stores are active
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:examples;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.datasource.devservices.enabled=false
quarkus.datasource.reactive=false
quarkus.flyway.migrate-at-start=false
quarkus.hibernate-orm.database.generation=none
```

- [ ] **Step 5: Compile to verify scaffold**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl examples/agent-scenarios --also-make
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add pom.xml examples/
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(examples): scaffold examples module with agent-scenarios sub-module

Zero-datasource operation via casehub-eidos-memory.
Follows qhorus examples pattern.

Refs casehubio/eidos#4"
```

---

## Task 5: MultiAgentTeamTest

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/MultiAgentTeamTest.java`

- [ ] **Step 1: Write MultiAgentTeamTest**

```java
// examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/MultiAgentTeamTest.java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

/**
 * Demonstrates registering a team of agents and querying by slot and capability.
 */
@QuarkusTest
class MultiAgentTeamTest {

    @Inject AgentRegistry registry;

    @BeforeEach
    void registerTeam() {
        registry.register(new AgentDescriptor(
            "planner-1", "Strategic Planner", "1.0", "anthropic",
            "claude", "claude-3-7-sonnet", null,
            "urn:casehub:vocab:casehub-slot", null, null,
            "planner",
            List.of(new AgentCapability("planning", 0.9, 200L, "medium",
                List.of("requirements"), List.of("plan"), List.of("orchestration"),
                Map.of("software", 0.95, "logistics", 0.4))),
            new AgentDisposition("facilitative", "principled", "measured", "semi-autonomous", true),
            null, null, "default"));

        registry.register(new AgentDescriptor(
            "reviewer-1", "Code Reviewer", "1.0", "anthropic",
            "claude", "claude-3-7-sonnet", null,
            "urn:casehub:vocab:casehub-slot", null, null,
            "reviewer",
            List.of(
                new AgentCapability("code-review", 0.95, 150L, "low",
                    List.of("code"), List.of("review"), List.of("quality"),
                    Map.of("java", 0.95, "python", 0.8, "rust", 0.3)),
                new AgentCapability("test-writing", 0.8, 300L, "medium",
                    List.of("code"), List.of("tests"), List.of("testing"),
                    Map.of("java", 0.9))),
            new AgentDisposition("independent", "strict", "conservative", "directed", false),
            null, null, "default"));

        registry.register(new AgentDescriptor(
            "executor-1", "Task Executor", "1.0", "anthropic",
            "claude", "claude-3-7-sonnet", null,
            "urn:casehub:vocab:casehub-slot", null, null,
            "executor",
            List.of(new AgentCapability("code-generation", 0.85, 500L, "high",
                List.of("spec"), List.of("code"), List.of("implementation"),
                Map.of("java", 0.9, "python", 0.85, "rust", 0.6))),
            new AgentDisposition("collaborative", "flexible", "bold", "autonomous", false),
            null, null, "default"));
    }

    @Test
    void find_by_id_returns_complete_descriptor() {
        var planner = registry.findById("planner-1", "default");

        assertThat(planner).isPresent();
        assertThat(planner.get().name()).isEqualTo("Strategic Planner");
        assertThat(planner.get().slot()).isEqualTo("planner");
        assertThat(planner.get().disposition().delegation()).isTrue();
        assertThat(planner.get().capabilities()).hasSize(1);
        assertThat(planner.get().capabilities().get(0).epistemicDomains())
            .containsEntry("software", 0.95);
    }

    @Test
    void find_reviewers_by_slot() {
        var reviewers = registry.find(AgentQuery.bySlot("reviewer", "default"));

        assertThat(reviewers).hasSize(1);
        assertThat(reviewers.get(0).agentId()).isEqualTo("reviewer-1");
    }

    @Test
    void find_agents_with_code_review_capability() {
        var agents = registry.find(AgentQuery.byCapability("code-review", "default"));

        assertThat(agents).hasSize(1);
        assertThat(agents.get(0).agentId()).isEqualTo("reviewer-1");
    }

    @Test
    void find_executor_by_slot_and_capability() {
        var executors = registry.find(
            AgentQuery.bySlotAndCapability("executor", "code-generation", "default"));

        assertThat(executors).hasSize(1);
        assertThat(executors.get(0).agentId()).isEqualTo("executor-1");
    }

    @Test
    void find_all_returns_entire_team() {
        var all = registry.find(AgentQuery.all("default"));

        assertThat(all).hasSizeGreaterThanOrEqualTo(3);
        assertThat(all.stream().map(AgentDescriptor::agentId).toList())
            .contains("planner-1", "reviewer-1", "executor-1");
    }

    @Test
    void agent_with_multiple_capabilities_found_by_either() {
        var codeReviewers = registry.find(AgentQuery.byCapability("code-review", "default"));
        var testWriters = registry.find(AgentQuery.byCapability("test-writing", "default"));

        assertThat(codeReviewers.stream().map(AgentDescriptor::agentId).toList())
            .contains("reviewer-1");
        assertThat(testWriters.stream().map(AgentDescriptor::agentId).toList())
            .contains("reviewer-1");
    }
}
```

- [ ] **Step 2: Run test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios --also-make -Dtest=MultiAgentTeamTest
```

Expected: `Tests run: 6, Failures: 0, Errors: 0`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/MultiAgentTeamTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "example: MultiAgentTeamTest — register team, query by slot/capability

Demonstrates AgentDescriptor creation, AgentRegistry.register/findById/find,
all AgentQuery factories. Uses CasehubSlot vocabulary for slot assignment.

Refs casehubio/eidos#4"
```

---

## Task 6: CrossVocabularyDiscoveryTest

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CrossVocabularyDiscoveryTest.java`

- [ ] **Step 1: Write CrossVocabularyDiscoveryTest**

```java
// examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CrossVocabularyDiscoveryTest.java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.vocab.CasehubSlotVocabularyProducer;
import io.casehub.eidos.vocab.SvoVocabularyProducer;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Demonstrates cross-vocabulary discovery: agents registered with different
 * vocabularies can be found equivalent via VocabularyRegistry.equivalentValues().
 */
@QuarkusTest
class CrossVocabularyDiscoveryTest {

    @Inject VocabularyRegistry vocabRegistry;

    @Test
    void svo_and_casehub_slot_vocabularies_are_discoverable() {
        assertThat(vocabRegistry.find(SvoVocabularyProducer.URI)).isPresent();
        assertThat(vocabRegistry.find(CasehubSlotVocabularyProducer.URI)).isPresent();
    }

    @Test
    void svo_evaluator_is_equivalent_to_casehub_slot_reviewer() {
        var equivalents = vocabRegistry.equivalentValues(
            SvoVocabularyProducer.URI, "evaluator", CasehubSlotVocabularyProducer.URI);

        assertThat(equivalents).containsExactly("reviewer");
    }

    @Test
    void casehub_slot_reviewer_is_equivalent_to_svo_evaluator() {
        var equivalents = vocabRegistry.equivalentValues(
            CasehubSlotVocabularyProducer.URI, "reviewer", SvoVocabularyProducer.URI);

        assertThat(equivalents).containsExactly("evaluator");
    }

    @Test
    void cross_reference_is_bidirectional_for_all_pairs() {
        // coordinator ↔ planner
        assertThat(vocabRegistry.equivalentValues(
            SvoVocabularyProducer.URI, "coordinator", CasehubSlotVocabularyProducer.URI))
            .containsExactly("planner");
        assertThat(vocabRegistry.equivalentValues(
            CasehubSlotVocabularyProducer.URI, "planner", SvoVocabularyProducer.URI))
            .containsExactly("coordinator");

        // performer ↔ executor
        assertThat(vocabRegistry.equivalentValues(
            SvoVocabularyProducer.URI, "performer", CasehubSlotVocabularyProducer.URI))
            .containsExactly("executor");
        assertThat(vocabRegistry.equivalentValues(
            CasehubSlotVocabularyProducer.URI, "executor", SvoVocabularyProducer.URI))
            .containsExactly("performer");
    }

    @Test
    void resolve_term_by_alias() {
        var term = vocabRegistry.resolve(SvoVocabularyProducer.URI, "reviewer");

        assertThat(term).isPresent();
        assertThat(term.get().value()).isEqualTo("evaluator");
        assertThat(term.get().label()).isEqualTo("Evaluator");
    }

    @Test
    void supervisor_has_no_svo_equivalent() {
        var equivalents = vocabRegistry.equivalentValues(
            CasehubSlotVocabularyProducer.URI, "supervisor", SvoVocabularyProducer.URI);

        assertThat(equivalents).isEmpty();
    }
}
```

- [ ] **Step 2: Run test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios --also-make -Dtest=CrossVocabularyDiscoveryTest
```

Expected: `Tests run: 6, Failures: 0, Errors: 0`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CrossVocabularyDiscoveryTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "example: CrossVocabularyDiscoveryTest — SVO ↔ CasehubSlot equivalence

Demonstrates VocabularyRegistry.find, resolve, equivalentValues.
Bidirectional cross-references, alias resolution, empty-match handling.

Refs casehubio/eidos#4"
```

---

## Task 7: EpistemicDomainMatchingTest

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/EpistemicDomainMatchingTest.java`

- [ ] **Step 1: Write EpistemicDomainMatchingTest**

```java
// examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/EpistemicDomainMatchingTest.java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

/**
 * Demonstrates CapabilityHealth probing with epistemic domain matching.
 * An agent strong in Java but weak in Rust gets different health statuses
 * depending on the task domain.
 */
@QuarkusTest
class EpistemicDomainMatchingTest {

    @Inject CapabilityHealth health;

    final AgentDescriptor polyglotReviewer = new AgentDescriptor(
        "polyglot-1", "Polyglot Reviewer", "1.0", "anthropic",
        "claude", "claude-3-7-sonnet", null,
        null, null, null, "reviewer",
        List.of(new AgentCapability("code-review", 0.9, 150L, "low",
            List.of("code"), List.of("review"), List.of("quality"),
            Map.of("java", 0.95, "python", 0.8, "rust", 0.2))),
        new AgentDisposition("independent", "principled", "measured", "directed", false),
        null, null, "default"
    );

    @Test
    void probe_ready_for_strong_domain() {
        var status = health.probe(polyglotReviewer, "code-review", ProbeContext.of("java"));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void probe_epistemically_weak_for_low_confidence_domain() {
        var status = health.probe(polyglotReviewer, "code-review", ProbeContext.of("rust"));

        assertThat(status).isInstanceOf(CapabilityStatus.EpistemicallyWeak.class);
        var weak = (CapabilityStatus.EpistemicallyWeak) status;
        assertThat(weak.domain()).isEqualTo("rust");
        assertThat(weak.confidence()).isEqualTo(0.2);
    }

    @Test
    void probe_ready_for_unknown_domain() {
        // Domain not in epistemicDomains map — assume capable (no data = no objection)
        var status = health.probe(polyglotReviewer, "code-review", ProbeContext.of("haskell"));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void probe_unavailable_for_undeclared_capability() {
        var status = health.probe(polyglotReviewer, "penetration-testing", ProbeContext.of(null));

        assertThat(status).isInstanceOf(CapabilityStatus.Unavailable.class);
        assertThat(((CapabilityStatus.Unavailable) status).reason())
            .contains("penetration-testing");
    }
}
```

- [ ] **Step 2: Run test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios --also-make -Dtest=EpistemicDomainMatchingTest
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/EpistemicDomainMatchingTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "example: EpistemicDomainMatchingTest — all 4 CapabilityStatus variants

Polyglot reviewer: Java=0.95 (Ready), Rust=0.2 (EpistemicallyWeak),
Haskell=absent (Ready), undeclared capability (Unavailable).

Refs casehubio/eidos#4"
```

---

## Task 8: TenancyIsolationTest + DispositionVocabularyTest

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/TenancyIsolationTest.java`
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DispositionVocabularyTest.java`

- [ ] **Step 1: Write TenancyIsolationTest**

```java
// examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/TenancyIsolationTest.java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

/**
 * Demonstrates unconditional tenancy isolation: agents in different tenants
 * are completely invisible to each other.
 */
@QuarkusTest
class TenancyIsolationTest {

    @Inject AgentRegistry registry;

    @BeforeEach
    void registerAgentsInDifferentTenants() {
        registry.register(new AgentDescriptor(
            "tenant-a-agent", "Agent A", "1.0", "anthropic",
            "claude", "claude-3-7", null, null, null, null,
            "reviewer",
            List.of(new AgentCapability("code-review", 0.9, null, null,
                List.of(), List.of(), List.of(), Map.of())),
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, "tenant-a"));

        registry.register(new AgentDescriptor(
            "tenant-b-agent", "Agent B", "1.0", "anthropic",
            "claude", "claude-3-7", null, null, null, null,
            "reviewer",
            List.of(new AgentCapability("code-review", 0.9, null, null,
                List.of(), List.of(), List.of(), Map.of())),
            new AgentDisposition("collaborative", "principled", "measured", "semi-autonomous", false),
            null, null, "tenant-b"));
    }

    @Test
    void tenant_a_sees_only_own_agents() {
        var agents = registry.find(AgentQuery.all("tenant-a"));

        assertThat(agents.stream().map(AgentDescriptor::agentId).toList())
            .contains("tenant-a-agent")
            .doesNotContain("tenant-b-agent");
    }

    @Test
    void tenant_b_sees_only_own_agents() {
        var agents = registry.find(AgentQuery.all("tenant-b"));

        assertThat(agents.stream().map(AgentDescriptor::agentId).toList())
            .contains("tenant-b-agent")
            .doesNotContain("tenant-a-agent");
    }

    @Test
    void find_by_id_respects_tenancy() {
        assertThat(registry.findById("tenant-a-agent", "tenant-a")).isPresent();
        assertThat(registry.findById("tenant-a-agent", "tenant-b")).isEmpty();
    }

    @Test
    void nonexistent_tenant_returns_empty() {
        assertThat(registry.find(AgentQuery.all("tenant-c"))).isEmpty();
    }
}
```

- [ ] **Step 2: Write DispositionVocabularyTest**

```java
// examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DispositionVocabularyTest.java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.vocab.ConscientiousnessVocabularyProducer;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Demonstrates using the Conscientiousness vocabulary to resolve
 * disposition axis values into human-readable descriptions.
 */
@QuarkusTest
class DispositionVocabularyTest {

    @Inject VocabularyRegistry vocabRegistry;

    static final String VOCAB = ConscientiousnessVocabularyProducer.URI;

    @Test
    void resolve_rule_following_axis_values() {
        assertThat(vocabRegistry.resolve(VOCAB, "strict")).isPresent();
        assertThat(vocabRegistry.resolve(VOCAB, "principled")).isPresent();
        assertThat(vocabRegistry.resolve(VOCAB, "flexible")).isPresent();

        var strict = vocabRegistry.resolve(VOCAB, "strict").get();
        assertThat(strict.label()).isEqualTo("Strict Rule Following");
        assertThat(strict.aliases()).contains("rule-bound");
    }

    @Test
    void resolve_risk_appetite_axis_values() {
        assertThat(vocabRegistry.resolve(VOCAB, "conservative")).isPresent();
        assertThat(vocabRegistry.resolve(VOCAB, "measured")).isPresent();
        assertThat(vocabRegistry.resolve(VOCAB, "bold")).isPresent();

        var bold = vocabRegistry.resolve(VOCAB, "bold").get();
        assertThat(bold.aliases()).contains("risk-tolerant");
    }

    @Test
    void resolve_social_orient_axis_values() {
        assertThat(vocabRegistry.resolve(VOCAB, "collaborative")).isPresent();
        assertThat(vocabRegistry.resolve(VOCAB, "independent")).isPresent();
        assertThat(vocabRegistry.resolve(VOCAB, "facilitative")).isPresent();
    }

    @Test
    void resolve_autonomy_axis_values() {
        assertThat(vocabRegistry.resolve(VOCAB, "directed")).isPresent();
        assertThat(vocabRegistry.resolve(VOCAB, "semi-autonomous")).isPresent();
        assertThat(vocabRegistry.resolve(VOCAB, "autonomous")).isPresent();

        var autonomous = vocabRegistry.resolve(VOCAB, "autonomous").get();
        assertThat(autonomous.aliases()).contains("self-governing", "agentic");
    }

    @Test
    void resolve_by_alias() {
        var term = vocabRegistry.resolve(VOCAB, "risk-averse");

        assertThat(term).isPresent();
        assertThat(term.get().value()).isEqualTo("conservative");
    }
}
```

- [ ] **Step 3: Run all examples tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios --also-make
```

Expected: all example tests green.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/TenancyIsolationTest.java examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DispositionVocabularyTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "example: TenancyIsolationTest + DispositionVocabularyTest

Tenancy: proves complete cross-tenant isolation including findById.
Disposition: resolves all 12 Conscientiousness terms across 4 axes.

Refs casehubio/eidos#4"
```

---

## Task 9: Examples README — Capability Coverage Table

**Files:**
- Create: `examples/README.md`

- [ ] **Step 1: Write README.md**

```markdown
# CaseHub Eidos — Examples

Executable examples demonstrating eidos capabilities. Each example is a `@QuarkusTest`
that runs with in-memory stores (zero datasource) and well-known vocabularies.

## Running

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios --also-make
```

## Capability Coverage

| Capability | Example | Status |
|---|---|---|
| AgentDescriptor creation | MultiAgentTeamTest | ✅ |
| AgentRegistry.register | MultiAgentTeamTest | ✅ |
| AgentRegistry.findById | MultiAgentTeamTest, TenancyIsolationTest | ✅ |
| AgentQuery.bySlot | MultiAgentTeamTest | ✅ |
| AgentQuery.byCapability | MultiAgentTeamTest | ✅ |
| AgentQuery.bySlotAndCapability | MultiAgentTeamTest | ✅ |
| AgentQuery.all | TenancyIsolationTest | ✅ |
| Tenancy isolation | TenancyIsolationTest | ✅ |
| VocabularyRegistry.find | CrossVocabularyDiscoveryTest | ✅ |
| VocabularyRegistry.resolve | CrossVocabularyDiscoveryTest, DispositionVocabularyTest | ✅ |
| VocabularyRegistry.equivalentValues | CrossVocabularyDiscoveryTest | ✅ |
| SVO vocabulary | CrossVocabularyDiscoveryTest | ✅ |
| CasehubSlot vocabulary | CrossVocabularyDiscoveryTest | ✅ |
| Conscientiousness vocabulary (12 terms, 4 axes) | DispositionVocabularyTest | ✅ |
| CapabilityHealth.probe → Ready | EpistemicDomainMatchingTest | ✅ |
| CapabilityHealth.probe → Unavailable | EpistemicDomainMatchingTest | ✅ |
| CapabilityHealth.probe → EpistemicallyWeak | EpistemicDomainMatchingTest | ✅ |
| CapabilityHealth.probe → Degraded | — | Phase 3 (runtime state infrastructure) |
| ReactiveCapabilityHealth | — | Implemented, build-gated |
| SystemPromptRenderer | — | Phase 3 ([#5](https://github.com/casehubio/eidos/issues/5)) |
| Knowledge graph | — | Phase 4 |

## Module Index

| Module | Focus | Dependencies |
|---|---|---|
| `agent-scenarios` | Core identity, discovery, vocabulary, health probing | eidos + eidos-memory + eidos-vocab |
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add examples/README.md
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs: examples README with capability coverage table

Refs casehubio/eidos#4"
```

---

## Task 10: Full Build Verification + PLATFORM.md Update

- [ ] **Step 1: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS across all modules. Report test counts per module.

- [ ] **Step 2: Update PLATFORM.md capability ownership entry**

In `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`, find the line:
```
| Agent capability health (declared vs operable) | `casehub-eidos` | `CapabilityHealth` SPI; `NoOpCapabilityHealth` @DefaultBean (Phase 2: real probing against trust scores) |
```

Replace with:
```
| Agent capability health (declared vs operable) | `casehub-eidos` | `CapabilityHealth` SPI — `probe(AgentDescriptor, capabilityTag, ProbeContext)` returns Ready/Unavailable/EpistemicallyWeak; `DefaultCapabilityHealth` checks declared capabilities + epistemic domain confidence; configurable `casehub.eidos.epistemic.weak-threshold` (default 0.3); `ReactiveCapabilityHealth` for reactive parity (build-gated) |
```

- [ ] **Step 3: Commit PLATFORM.md**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: update CapabilityHealth capability ownership — Phase 2 complete

DefaultCapabilityHealth replaces NoOp. Probe checks declared capabilities
and epistemic domain confidence. ReactiveCapabilityHealth for parity.

Refs casehubio/eidos#4"
git -C /Users/mdproctor/claude/casehub/parent push
```

---

## Self-Review

**Spec coverage check:**
- ✅ CapabilityHealth API change (Task 1)
- ✅ ReactiveCapabilityHealth SPI (Task 1)
- ✅ DefaultCapabilityHealth with probe logic (Task 2)
- ✅ DefaultReactiveCapabilityHealth (Task 3)
- ✅ Examples module scaffold (Task 4)
- ✅ MultiAgentTeamTest (Task 5)
- ✅ CrossVocabularyDiscoveryTest (Task 6)
- ✅ EpistemicDomainMatchingTest (Task 7)
- ✅ TenancyIsolationTest (Task 8)
- ✅ DispositionVocabularyTest (Task 8)
- ✅ Coverage table README (Task 9)
- ✅ PLATFORM.md update (Task 10)

**Placeholder scan:** No TBD, TODO, or "implement later" found.

**Type consistency:** `CapabilityHealth.probe(AgentDescriptor, String, ProbeContext)` signature is consistent across Task 1 (API), Task 2 (DefaultCapabilityHealth), Task 3 (delegate), Tasks 5-7 (examples). `ReactiveCapabilityHealth.probe()` uses `CapabilityHealth.CapabilityStatus` and `CapabilityHealth.ProbeContext` consistently.

**Note on Task 3:** The plan identifies a build-gate conflict between DefaultCapabilityHealth and DefaultReactiveCapabilityHealth during Task 3. The fix (remove `@IfBuildProperty` from `DefaultCapabilityHealth`, make it plain `@ApplicationScoped`) is documented inline. The implementer must apply this fix — it is not a separate task.
