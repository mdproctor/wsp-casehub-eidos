# eidos/agents @McpDomain Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #204 — feat: REST endpoint for AgentDescriptor — expose rich agent personality data
**Issue group:** #204

**Goal:** Expose AgentRegistry data via @McpDomain("eidos/agents"), generating REST, GraphQL, and MCP tool endpoints from a single API class.

**Architecture:** New `graphql/` Maven module containing `EidosAgentApi` — a `@McpDomain("eidos/agents")` class with two `@PlatformQuery` operations (`findAgents`, `getAgent`). The platform's `GraphQLResolverProcessor` APT generates REST and GraphQL resources at compile time; `GraphQLModelScanner` registers MCP tools at runtime. Visibility filtering strips PRIVATE goals/constraints.

**Tech Stack:** Java 21, Quarkus 3.32.2, @McpDomain (casehub-platform-api), casehub-platform-graphql-generator (APT)

## Global Constraints

- Platform annotations only: `io.casehub.platform.api.mcp.*` — NOT `jakarta.ws.rs.*`
- `@RestPath("")` for clean URLs — default generator behavior kebab-cases method names
- `CurrentPrincipal.tenancyId()` for tenancy — never a query parameter
- No mutations — registration stays programmatic
- All versions: `${casehub-platform-api.version}` = `0.4-SNAPSHOT`

---

## Batch 1: Module + API + Tests

### Task 1: Create graphql Maven module with EidosAgentApi

**Files:**
- Create: `graphql/pom.xml`
- Create: `graphql/src/main/java/io/casehub/eidos/graphql/EidosAgentApi.java`
- Create: `graphql/src/test/java/io/casehub/eidos/graphql/EidosAgentApiTest.java`
- Modify: `pom.xml` (parent — add `<module>graphql</module>`)

**Interfaces:**
- Consumes: `AgentRegistry.findById(agentId, tenancyId)`, `AgentRegistry.find(AgentQuery)`, `AgentMatch.descriptor()`, `AgentDescriptor.publicGoals()`, `AgentDescriptor.publicConstraints()`, `AgentDescriptor.toBuilder()`, `CurrentPrincipal.tenancyId()`
- Produces: `EidosAgentApi.findAgents(slot, capabilityName, goalName, taskDomain, maxDepth)` → `List<AgentDescriptor>`, `EidosAgentApi.getAgent(agentId)` → `AgentDescriptor`

- [ ] **Step 1: Create `graphql/pom.xml`**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-parent</artifactId>
    <version>0.4-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-eidos-graphql</artifactId>
  <name>CaseHub Eidos - GraphQL</name>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-api</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>

    <!-- CDI -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>make-index</id>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>${compiler-plugin.version}</version>
        <configuration>
          <annotationProcessorPaths>
            <path>
              <groupId>io.casehub</groupId>
              <artifactId>casehub-platform-graphql-generator</artifactId>
              <version>${casehub-platform-api.version}</version>
            </path>
            <path>
              <groupId>io.casehub</groupId>
              <artifactId>casehub-eidos-api</artifactId>
              <version>${project.version}</version>
            </path>
          </annotationProcessorPaths>
          <compilerArgs>
            <arg>-AdomainFilter=eidos/agents</arg>
          </compilerArgs>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Add module to parent pom.xml**

In `pom.xml` (root), add `<module>graphql</module>` after `<module>eval</module>` in the `<modules>` block.

- [ ] **Step 3: Create directory structure**

```bash
mkdir -p graphql/src/main/java/io/casehub/eidos/graphql
mkdir -p graphql/src/test/java/io/casehub/eidos/graphql
```

- [ ] **Step 4: Write the failing test**

Create `graphql/src/test/java/io/casehub/eidos/graphql/EidosAgentApiTest.java`:

```java
package io.casehub.eidos.graphql;

import io.casehub.eidos.api.*;
import io.casehub.platform.api.identity.CurrentPrincipal;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class EidosAgentApiTest {

    @Mock AgentRegistry registry;
    @Mock CurrentPrincipal principal;

    EidosAgentApi api;

    static final String TENANT = "test-tenant";

    @BeforeEach
    void setUp() {
        when(principal.tenancyId()).thenReturn(TENANT);
        api = new EidosAgentApi(registry, principal);
    }

    @Test
    void findAgents_noParams_listsAll() {
        var descriptor = testDescriptor("agent-1");
        when(registry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(descriptor, null)));

        var result = api.findAgents(null, null, null, null, null);

        assertThat(result).hasSize(1);
        assertThat(result.getFirst().agentId()).isEqualTo("agent-1");
    }

    @Test
    void findAgents_withSlot_passesQueryParam() {
        when(registry.find(any(AgentQuery.class)))
            .thenReturn(List.of());

        api.findAgents("planner", null, null, null, null);

        org.mockito.Mockito.verify(registry).find(
            org.mockito.ArgumentMatchers.argThat(q ->
                "planner".equals(q.slot()) && TENANT.equals(q.tenancyId())));
    }

    @Test
    void findAgents_filtersPrivateGoals() {
        var descriptor = AgentDescriptor.builder()
            .agentId("agent-1").name("Agent One").slot("tester").tenancyId(TENANT)
            .goals(List.of(
                new AgentGoal("public-goal", "visible", GoalPriority.PRIMARY,
                    Visibility.PUBLIC, List.of(), null),
                new AgentGoal("private-goal", "hidden", GoalPriority.SECONDARY,
                    Visibility.PRIVATE, List.of(), null)))
            .constraints(List.of(
                new AgentConstraint("public-constraint", "visible",
                    Visibility.PUBLIC, ConstraintSeverity.HARD),
                new AgentConstraint("private-constraint", "hidden",
                    Visibility.PRIVATE, ConstraintSeverity.SOFT)))
            .build();
        when(registry.find(any(AgentQuery.class)))
            .thenReturn(List.of(new AgentMatch(descriptor, null)));

        var result = api.findAgents(null, null, null, null, null);

        assertThat(result).hasSize(1);
        assertThat(result.getFirst().goals()).hasSize(1);
        assertThat(result.getFirst().goals().getFirst().name()).isEqualTo("public-goal");
        assertThat(result.getFirst().constraints()).hasSize(1);
        assertThat(result.getFirst().constraints().getFirst().name()).isEqualTo("public-constraint");
    }

    @Test
    void getAgent_found_returnsFiltered() {
        var descriptor = testDescriptor("agent-1");
        when(registry.findById("agent-1", TENANT))
            .thenReturn(Optional.of(descriptor));

        var result = api.getAgent("agent-1");

        assertThat(result).isNotNull();
        assertThat(result.agentId()).isEqualTo("agent-1");
    }

    @Test
    void getAgent_notFound_returnsNull() {
        when(registry.findById("unknown", TENANT))
            .thenReturn(Optional.empty());

        var result = api.getAgent("unknown");

        assertThat(result).isNull();
    }

    private AgentDescriptor testDescriptor(String id) {
        return AgentDescriptor.builder()
            .agentId(id).name("Test " + id).slot("tester").tenancyId(TENANT)
            .build();
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql`
Expected: Compilation error — `EidosAgentApi` class does not exist yet.

- [ ] **Step 6: Implement EidosAgentApi**

Create `graphql/src/main/java/io/casehub/eidos/graphql/EidosAgentApi.java`:

```java
package io.casehub.eidos.graphql;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.AgentQuery;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.QueryParam;
import io.casehub.platform.api.mcp.RestPath;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;

@McpDomain(value = "eidos/agents", app = "eidos",
    basePath = "/api/eidos/agents",
    summary = "Agent identity registry — query agent descriptors with rich personality, capability, and disposition data")
@ApplicationScoped
public class EidosAgentApi {

    private final AgentRegistry registry;
    private final CurrentPrincipal currentPrincipal;

    @Inject
    public EidosAgentApi(AgentRegistry registry, CurrentPrincipal currentPrincipal) {
        this.registry = registry;
        this.currentPrincipal = currentPrincipal;
    }

    @PlatformQuery("Find agents matching optional criteria — no params lists all")
    @RestPath("")
    public List<AgentDescriptor> findAgents(
            @QueryParam("slot") String slot,
            @QueryParam("capability") String capabilityName,
            @QueryParam("goal") String goalName,
            @QueryParam("domain") String taskDomain,
            @QueryParam("maxDepth") Integer maxDepth) {
        String tenancyId = currentPrincipal.tenancyId();
        var query = new AgentQuery(slot, capabilityName, tenancyId, taskDomain, goalName, maxDepth);
        return registry.find(query).stream()
                .map(AgentMatch::descriptor)
                .map(this::filterPrivate)
                .toList();
    }

    @PlatformQuery("Get a specific agent descriptor by ID")
    @RestPath("")
    public AgentDescriptor getAgent(@PathParam("agentId") String agentId) {
        return registry.findById(agentId, currentPrincipal.tenancyId())
                .map(this::filterPrivate)
                .orElse(null);
    }

    private AgentDescriptor filterPrivate(AgentDescriptor d) {
        if (d.goals().isEmpty() && d.constraints().isEmpty()) {
            return d;
        }
        return d.toBuilder()
                .goals(d.publicGoals())
                .constraints(d.publicConstraints())
                .build();
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql`
Expected: All 5 tests PASS.

- [ ] **Step 8: Verify APT generates REST resource**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl graphql`

Check generated sources exist:
```bash
find graphql/target/generated-sources -name "*Resource.java" -o -name "*Resolver.java"
```
Expected: At least one generated file (e.g., `EidosAgentsResource.java`).

- [ ] **Step 9: Run full build to verify nothing breaks**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 10: Commit**

```bash
git add graphql/ pom.xml
git commit -m "feat: add @McpDomain eidos/agents API with findAgents and getAgent

New casehub-eidos-graphql module exposes AgentRegistry data via the
platform's unified API framework. Two @PlatformQuery operations:
- findAgents: optional filters (slot, capability, goal, domain, maxDepth)
- getAgent: lookup by agentId

PRIVATE goals and constraints are filtered before serialization.
Tenancy resolved from CurrentPrincipal.

Refs #204"
```

## References

- [2026-10-20-eidos-agents-mcpdomain-design.md] — design spec this plan implements
- [api/AgentRegistry.java] — registry SPI (findById, find)
- [api/AgentDescriptor.java] — 27-field record with builder, publicGoals(), publicConstraints()
- [api/AgentQuery.java] — query criteria with factory methods
- [api/AgentMatch.java] — find() result wrapping descriptor
- [platform-api/mcp/McpDomain.java] — unified API annotation
- [platform-api/mcp/PlatformQuery.java] — read operation annotation
- [platform-api/mcp/RestPath.java] — REST path override annotation
- [platform-api/mcp/PathParam.java] — path parameter binding
- [platform-api/mcp/QueryParam.java] — query parameter binding
- [platform-api/identity/CurrentPrincipal.java:61] — tenancyId() method
- [platform/testing/FixedCurrentPrincipal.java] — test CurrentPrincipal with DEFAULT_TENANT_ID
- [connectors/graphql/ConnectorOperationsImpl.java] — reference @McpDomain implementation
- [GitHub #204] — focal issue
