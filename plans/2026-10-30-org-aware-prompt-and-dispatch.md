# Org-Aware Prompt Contribution and Dispatch — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #194 — feat: auto-inject org context into rendered prompts via PromptContributor
**Issue group:** #194, #195

**Goal:** Give agents awareness of their organizational position in prompts, and give platform consumers a standard dispatch contract for org-aware routing and delegation.

**Architecture:** Two complementary features in the org module layer. `OrgPromptContributor` implements the existing `PromptContributor` SPI to inject organizational context (unit membership, supervision, escalation, team peers) into agent prompts at `COGNITIVE` tier. `OrgAwareDispatch` defines a new SPI in `eidos-api` with a default implementation in `org-runtime` that walks escalation paths with health probing, validates delegation via relationship/membership checks, and resolves team membership from org units. Both follow the established CDI ladder: NoOp `@DefaultBean` in `EidosCoreProducer`, real `@ApplicationScoped` in `org-runtime` overrides when present.

**Tech Stack:** Java 21, Quarkus CDI (quarkus-arc), eidos API/SPI layer, org-api

## Global Constraints

- Java 21 source, Java 26 JVM
- `OrgAwareDispatch` interface in `eidos-api` — pure Java, zero framework deps
- NoOp implementations in `eidos-core` — zero CDI
- Real implementations in `org-runtime` — `@ApplicationScoped`, CDI injection
- `Instance<OrgRegistry>` pattern for optional org dependency
- All tests use `InMemoryOrgRegistry` from `org-memory`
- No new module dependencies from `runtime` → `org-api` (keep modules decoupled)

---

## Batch 1: OrgPromptContributor (#194)

### Task 1: OrgPromptContributor — auto-inject org context into prompts

**Files:**
- Create: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/prompt/OrgPromptContributor.java`
- Test: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/prompt/OrgPromptContributorTest.java`

**Interfaces:**
- Consumes: `PromptContributor` SPI (`api/.../spi/PromptContributor.java`), `PromptBlock.cognitive(tag, content)`, `OrgRegistry` (all methods: `unitsFor`, `supervisors`, `subordinates`, `escalationPath`, `membersOf`)
- Produces: `PromptBlock` at `PromptTier.COGNITIVE` with tag `"ORG_CONTEXT"` containing organizational position text. No downstream task depends on this — it's a leaf contributor discovered by `CdiPromptAssembler`.

- [ ] **Step 1: Write the failing test — no org registry produces no blocks**

```java
package io.casehub.eidos.org.runtime.prompt;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.RenderFormat;
import jakarta.enterprise.inject.Instance;
import io.casehub.eidos.org.api.OrgRegistry;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class OrgPromptContributorTest {

    @Test void noRegistryProducesNoBlocks() {
        @SuppressWarnings("unchecked")
        Instance<OrgRegistry> absent = mock(Instance.class);
        when(absent.isResolvable()).thenReturn(false);

        var contributor = new OrgPromptContributor(absent);
        var descriptor = AgentDescriptor.builder()
                .agentId("agent-1").name("Agent One").tenancyId("t1").build();
        var context = AgentPromptContext.builder().format(RenderFormat.MARKDOWN).build();

        assertThat(contributor.contribute(descriptor, context)).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=OrgPromptContributorTest#noRegistryProducesNoBlocks`
Expected: FAIL — class not found

- [ ] **Step 3: Add mockito test dependency to org-runtime pom.xml**

Add to `org-runtime/pom.xml` in `<dependencies>`:

```xml
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 4: Implement OrgPromptContributor — minimal for first test**

```java
package io.casehub.eidos.org.runtime.prompt;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentPromptContext;
import io.casehub.eidos.api.PromptBlock;
import io.casehub.eidos.api.spi.PromptContributor;
import io.casehub.eidos.org.api.OrgRegistry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.List;

@ApplicationScoped
public class OrgPromptContributor implements PromptContributor {

    private final Instance<OrgRegistry> registryInstance;

    @Inject
    public OrgPromptContributor(Instance<OrgRegistry> registryInstance) {
        this.registryInstance = registryInstance;
    }

    @Override
    public List<PromptBlock> contribute(AgentDescriptor descriptor, AgentPromptContext context) {
        if (!registryInstance.isResolvable()) {
            return List.of();
        }
        var registry = registryInstance.get();
        var agentId = descriptor.agentId();
        var tenancyId = descriptor.tenancyId();

        var sb = new StringBuilder();
        appendUnits(sb, registry, agentId, tenancyId);
        appendSupervisors(sb, registry, agentId, tenancyId);
        appendSubordinates(sb, registry, agentId, tenancyId);
        appendEscalationPath(sb, registry, agentId, tenancyId);
        appendTeamPeers(sb, registry, agentId, tenancyId);

        if (sb.isEmpty()) {
            return List.of();
        }
        return List.of(PromptBlock.cognitive("ORG_CONTEXT", sb.toString().strip()));
    }

    private void appendUnits(StringBuilder sb, OrgRegistry registry,
                             String agentId, String tenancyId) {
        var units = registry.unitsFor(agentId, tenancyId);
        for (var unit : units) {
            sb.append("You are a member of \"").append(unit.name()).append("\"");
            if (unit.kind() != null && !unit.kind().isBlank()) {
                sb.append(" (").append(unit.kind()).append(")");
            }
            sb.append(".\n");
        }
    }

    private void appendSupervisors(StringBuilder sb, OrgRegistry registry,
                                   String agentId, String tenancyId) {
        var supervisors = registry.supervisors(agentId, tenancyId);
        if (!supervisors.isEmpty()) {
            var names = supervisors.stream()
                    .map(r -> r.sourceAgentId())
                    .toList();
            sb.append("Your supervisor");
            if (names.size() == 1) {
                sb.append(" is ").append(names.getFirst());
            } else {
                sb.append("s are ").append(String.join(", ", names));
            }
            sb.append(".\n");
        }
    }

    private void appendSubordinates(StringBuilder sb, OrgRegistry registry,
                                    String agentId, String tenancyId) {
        var subordinates = registry.subordinates(agentId, tenancyId);
        if (!subordinates.isEmpty()) {
            var names = subordinates.stream()
                    .map(r -> r.targetAgentId())
                    .toList();
            sb.append("You can delegate work to: ")
              .append(String.join(", ", names)).append(".\n");
        }
    }

    private void appendEscalationPath(StringBuilder sb, OrgRegistry registry,
                                      String agentId, String tenancyId) {
        var path = registry.escalationPath(agentId, tenancyId);
        if (!path.isEmpty()) {
            var chain = path.stream()
                    .map(r -> r.targetAgentId())
                    .toList();
            sb.append("To escalate unresolved issues, contact: ")
              .append(String.join(" → ", chain)).append(".\n");
        }
    }

    private void appendTeamPeers(StringBuilder sb, OrgRegistry registry,
                                 String agentId, String tenancyId) {
        var units = registry.unitsFor(agentId, tenancyId);
        var peers = units.stream()
                .flatMap(u -> registry.membersOf(u.unitId(), tenancyId).stream())
                .map(m -> m.agentId())
                .filter(id -> !id.equals(agentId))
                .distinct()
                .toList();
        if (!peers.isEmpty()) {
            sb.append("Your team includes: ")
              .append(String.join(", ", peers)).append(".\n");
        }
    }
}
```

- [ ] **Step 5: Run first test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=OrgPromptContributorTest#noRegistryProducesNoBlocks`
Expected: PASS

- [ ] **Step 6: Write failing test — full org context rendering**

Add to `OrgPromptContributorTest`:

```java
    @Test void rendersFullOrgContext() {
        var registry = new InMemoryOrgRegistry();

        var org = OrgStructure.define("acme")
            .unit("eng").name("Engineering").kind("department")
                .member("alice", "lead")
                .member("bob", "dev")
                .member("carol", "dev")
                .add()
            .supervises("alice", "bob").add()
            .supervises("alice", "carol").add()
            .escalatesTo("bob", "alice").add()
            .escalatesTo("alice", "cto").add()
            .build();
        org.registerAll(registry);

        @SuppressWarnings("unchecked")
        Instance<OrgRegistry> present = mock(Instance.class);
        when(present.isResolvable()).thenReturn(true);
        when(present.get()).thenReturn(registry);

        var contributor = new OrgPromptContributor(present);
        var descriptor = AgentDescriptor.builder()
                .agentId("bob").name("Bob").tenancyId("acme").build();
        var context = AgentPromptContext.builder().format(RenderFormat.MARKDOWN).build();

        var blocks = contributor.contribute(descriptor, context);
        assertThat(blocks).hasSize(1);

        var block = blocks.getFirst();
        assertThat(block.tier()).isEqualTo(PromptTier.COGNITIVE);
        assertThat(block.tag()).isEqualTo("ORG_CONTEXT");

        var content = block.content();
        assertThat(content).contains("You are a member of \"Engineering\" (department).");
        assertThat(content).contains("Your supervisor is alice.");
        assertThat(content).contains("To escalate unresolved issues, contact: alice");
        assertThat(content).contains("Your team includes:");
        assertThat(content).contains("alice");
        assertThat(content).contains("carol");
        assertThat(content).doesNotContain("bob");
    }
```

Additional imports needed:
```java
import io.casehub.eidos.api.PromptTier;
import io.casehub.eidos.org.api.OrgStructure;
import io.casehub.eidos.org.memory.InMemoryOrgRegistry;
```

- [ ] **Step 7: Run test to verify it passes (implementation already covers this)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=OrgPromptContributorTest#rendersFullOrgContext`
Expected: PASS — the implementation from Step 4 should handle this.

- [ ] **Step 8: Write failing test — agent with no org membership produces no blocks**

```java
    @Test void noMembershipProducesNoBlocks() {
        var registry = new InMemoryOrgRegistry();

        @SuppressWarnings("unchecked")
        Instance<OrgRegistry> present = mock(Instance.class);
        when(present.isResolvable()).thenReturn(true);
        when(present.get()).thenReturn(registry);

        var contributor = new OrgPromptContributor(present);
        var descriptor = AgentDescriptor.builder()
                .agentId("stranger").name("Stranger").tenancyId("acme").build();
        var context = AgentPromptContext.builder().format(RenderFormat.MARKDOWN).build();

        assertThat(contributor.contribute(descriptor, context)).isEmpty();
    }
```

- [ ] **Step 9: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=OrgPromptContributorTest#noMembershipProducesNoBlocks`
Expected: PASS

- [ ] **Step 10: Write failing test — supervisor with subordinates rendering**

```java
    @Test void supervisorSeesSubordinates() {
        var registry = new InMemoryOrgRegistry();

        var org = OrgStructure.define("acme")
            .unit("eng").name("Engineering").kind("team")
                .member("alice", "lead")
                .member("bob", "dev")
                .member("carol", "dev")
                .add()
            .supervises("alice", "bob").add()
            .supervises("alice", "carol").add()
            .build();
        org.registerAll(registry);

        @SuppressWarnings("unchecked")
        Instance<OrgRegistry> present = mock(Instance.class);
        when(present.isResolvable()).thenReturn(true);
        when(present.get()).thenReturn(registry);

        var contributor = new OrgPromptContributor(present);
        var descriptor = AgentDescriptor.builder()
                .agentId("alice").name("Alice").tenancyId("acme").build();
        var context = AgentPromptContext.builder().format(RenderFormat.MARKDOWN).build();

        var blocks = contributor.contribute(descriptor, context);
        assertThat(blocks).hasSize(1);
        var content = blocks.getFirst().content();
        assertThat(content).contains("You can delegate work to: bob, carol");
        assertThat(content).doesNotContain("Your supervisor");
    }
```

- [ ] **Step 11: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=OrgPromptContributorTest#supervisorSeesSubordinates`
Expected: PASS

- [ ] **Step 12: Run full test suite and commit**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime`
Expected: all tests PASS

```bash
git add org-runtime/pom.xml org-runtime/src/
git commit -m "feat(#194): OrgPromptContributor — auto-inject org context into prompts

Implements PromptContributor SPI in org-runtime. When OrgRegistry is
available, injects unit membership, supervision, subordinates,
escalation path, and team peers as a COGNITIVE-tier prompt block.
No-op when OrgRegistry absent.

Refs #194

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: OrgAwareDispatch (#195)

### Task 2: OrgAwareDispatch interface + NoOp + DefaultBean wiring

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/OrgAwareDispatch.java`
- Create: `eidos-core/src/main/java/io/casehub/eidos/core/org/NoOpOrgAwareDispatch.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java` — add `@DefaultBean` producer method
- Test: `api/src/test/java/io/casehub/eidos/api/OrgAwareDispatchTest.java`

**Interfaces:**
- Consumes: `AgentMatch` (`api/.../AgentMatch.java`), `ResolvedCapability` (`api/.../ResolvedCapability.java`)
- Produces: `OrgAwareDispatch` SPI — consumed by Task 3 (DefaultOrgAwareDispatch) and by platform consumers (engine, qhorus, claudony)

- [ ] **Step 1: Write the interface**

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Optional;

public interface OrgAwareDispatch {

    Optional<AgentMatch> escalate(String agentId, String capabilityName, String tenancyId);

    boolean canDelegateTo(String fromAgentId, String toAgentId, String tenancyId);

    List<String> teamMembers(String agentId, String tenancyId);
}
```

- [ ] **Step 2: Write the failing test for NoOp behavior**

```java
package io.casehub.eidos.api;

import io.casehub.eidos.core.org.NoOpOrgAwareDispatch;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class OrgAwareDispatchTest {

    private final OrgAwareDispatch dispatch = new NoOpOrgAwareDispatch();

    @Test void escalateReturnsEmpty() {
        assertThat(dispatch.escalate("agent-1", "code-review", "t1")).isEmpty();
    }

    @Test void canDelegateToReturnsTrue() {
        assertThat(dispatch.canDelegateTo("from", "to", "t1")).isTrue();
    }

    @Test void teamMembersReturnsEmpty() {
        assertThat(dispatch.teamMembers("agent-1", "t1")).isEmpty();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=OrgAwareDispatchTest`
Expected: FAIL — classes not found

- [ ] **Step 4: Implement NoOpOrgAwareDispatch**

```java
package io.casehub.eidos.core.org;

import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.OrgAwareDispatch;

import java.util.List;
import java.util.Optional;

public class NoOpOrgAwareDispatch implements OrgAwareDispatch {

    @Override
    public Optional<AgentMatch> escalate(String agentId, String capabilityName,
                                         String tenancyId) {
        return Optional.empty();
    }

    @Override
    public boolean canDelegateTo(String fromAgentId, String toAgentId,
                                 String tenancyId) {
        return true;
    }

    @Override
    public List<String> teamMembers(String agentId, String tenancyId) {
        return List.of();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=OrgAwareDispatchTest`
Expected: PASS

- [ ] **Step 6: Wire @DefaultBean in EidosCoreProducer**

Add to `EidosCoreProducer.java`:

```java
    @Produces
    @DefaultBean
    OrgAwareDispatch orgAwareDispatch() {
        return new NoOpOrgAwareDispatch();
    }
```

Import: `io.casehub.eidos.api.OrgAwareDispatch`, `io.casehub.eidos.core.org.NoOpOrgAwareDispatch`

- [ ] **Step 7: Run full build to verify wiring compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,eidos-core,runtime`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/ eidos-core/src/ runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java
git commit -m "feat(#195): OrgAwareDispatch SPI + NoOp default

Pure Java interface in eidos-api: escalate(), canDelegateTo(),
teamMembers(). NoOp in eidos-core returns permissive defaults
(empty escalation, allow all delegation, no team). Wired as
@DefaultBean in EidosCoreProducer.

Refs #195

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: DefaultOrgAwareDispatch — org-backed implementation

**Files:**
- Create: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/dispatch/DefaultOrgAwareDispatch.java`
- Test: `org-runtime/src/test/java/io/casehub/eidos/org/runtime/dispatch/DefaultOrgAwareDispatchTest.java`

**Interfaces:**
- Consumes: `OrgAwareDispatch` (from Task 2), `OrgRegistry` (org-api: `escalationPath`, `subordinates`, `unitsFor`, `membersOf`), `AgentRegistry.findById()` (eidos-api), `CapabilityHealth.probe()` (eidos-api), `AgentMatch` (eidos-api), `CapabilityHealth.CapabilityStatus.Ready`
- Produces: `@ApplicationScoped` CDI bean that overrides the `@DefaultBean` NoOp when org-runtime is on the classpath

- [ ] **Step 1: Write failing test — escalation finds first healthy agent**

```java
package io.casehub.eidos.org.runtime.dispatch;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.CapabilityHealth;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.org.api.OrgRegistry;
import io.casehub.eidos.org.api.OrgStructure;
import io.casehub.eidos.org.memory.InMemoryOrgRegistry;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class DefaultOrgAwareDispatchTest {

    private OrgRegistry orgRegistry;
    private AgentRegistry agentRegistry;
    private CapabilityHealth capabilityHealth;
    private DefaultOrgAwareDispatch dispatch;

    @BeforeEach void setUp() {
        orgRegistry = new InMemoryOrgRegistry();
        agentRegistry = mock(AgentRegistry.class);
        capabilityHealth = mock(CapabilityHealth.class);
        dispatch = new DefaultOrgAwareDispatch(orgRegistry, agentRegistry, capabilityHealth);
    }

    @Test void escalateFindsFirstHealthyAgent() {
        var org = OrgStructure.define("acme")
            .unit("support").name("Support").kind("team")
                .member("l1", "tier-1")
                .member("l2", "tier-2")
                .member("l3", "tier-3")
                .add()
            .escalatesTo("l1", "l2").add()
            .escalatesTo("l2", "l3").add()
            .build();
        org.registerAll(orgRegistry);

        var l2Descriptor = AgentDescriptor.builder()
                .agentId("l2").name("L2").tenancyId("acme").build();
        when(agentRegistry.findById("l2", "acme")).thenReturn(Optional.of(l2Descriptor));
        when(capabilityHealth.probe(eq(l2Descriptor), eq("issue-resolution"), any(ProbeContext.class)))
                .thenReturn(new CapabilityStatus.Ready());

        var result = dispatch.escalate("l1", "issue-resolution", "acme");

        assertThat(result).isPresent();
        assertThat(result.get().descriptor().agentId()).isEqualTo("l2");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#escalateFindsFirstHealthyAgent`
Expected: FAIL — class not found

- [ ] **Step 3: Implement DefaultOrgAwareDispatch**

```java
package io.casehub.eidos.org.runtime.dispatch;

import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.CapabilityHealth;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.OrgAwareDispatch;
import io.casehub.eidos.org.api.OrgRegistry;
import io.casehub.eidos.org.api.RelationshipKind;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;
import java.util.Optional;

@ApplicationScoped
public class DefaultOrgAwareDispatch implements OrgAwareDispatch {

    private final OrgRegistry orgRegistry;
    private final AgentRegistry agentRegistry;
    private final CapabilityHealth capabilityHealth;

    @Inject
    public DefaultOrgAwareDispatch(OrgRegistry orgRegistry,
                                   AgentRegistry agentRegistry,
                                   CapabilityHealth capabilityHealth) {
        this.orgRegistry = orgRegistry;
        this.agentRegistry = agentRegistry;
        this.capabilityHealth = capabilityHealth;
    }

    @Override
    public Optional<AgentMatch> escalate(String agentId, String capabilityName,
                                         String tenancyId) {
        var path = orgRegistry.escalationPath(agentId, tenancyId);
        for (var step : path) {
            var targetId = step.targetAgentId();
            var descriptor = agentRegistry.findById(targetId, tenancyId);
            if (descriptor.isEmpty()) {
                continue;
            }
            var status = capabilityHealth.probe(
                    descriptor.get(), capabilityName, ProbeContext.of(null));
            if (status instanceof CapabilityStatus.Ready) {
                return Optional.of(new AgentMatch(descriptor.get(), null));
            }
        }
        return Optional.empty();
    }

    @Override
    public boolean canDelegateTo(String fromAgentId, String toAgentId,
                                 String tenancyId) {
        var subordinates = orgRegistry.subordinates(fromAgentId, tenancyId);
        if (subordinates.stream().anyMatch(r -> r.targetAgentId().equals(toAgentId))) {
            return true;
        }
        var fromUnits = orgRegistry.unitsFor(fromAgentId, tenancyId);
        for (var unit : fromUnits) {
            var members = orgRegistry.membersOf(unit.unitId(), tenancyId);
            if (members.stream().anyMatch(m -> m.agentId().equals(toAgentId))) {
                return true;
            }
        }
        return false;
    }

    @Override
    public List<String> teamMembers(String agentId, String tenancyId) {
        return orgRegistry.unitsFor(agentId, tenancyId).stream()
                .flatMap(u -> orgRegistry.membersOf(u.unitId(), tenancyId).stream())
                .map(m -> m.agentId())
                .filter(id -> !id.equals(agentId))
                .distinct()
                .toList();
    }
}
```

- [ ] **Step 4: Run first test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#escalateFindsFirstHealthyAgent`
Expected: PASS

- [ ] **Step 5: Write failing test — escalation skips unhealthy agents**

Add to `DefaultOrgAwareDispatchTest`:

```java
    @Test void escalateSkipsUnhealthyAgent() {
        var org = OrgStructure.define("acme")
            .unit("support").name("Support").kind("team")
                .member("l1", "tier-1")
                .member("l2", "tier-2")
                .member("l3", "tier-3")
                .add()
            .escalatesTo("l1", "l2").add()
            .escalatesTo("l2", "l3").add()
            .build();
        org.registerAll(orgRegistry);

        var l2Descriptor = AgentDescriptor.builder()
                .agentId("l2").name("L2").tenancyId("acme").build();
        var l3Descriptor = AgentDescriptor.builder()
                .agentId("l3").name("L3").tenancyId("acme").build();

        when(agentRegistry.findById("l2", "acme")).thenReturn(Optional.of(l2Descriptor));
        when(agentRegistry.findById("l3", "acme")).thenReturn(Optional.of(l3Descriptor));

        when(capabilityHealth.probe(eq(l2Descriptor), eq("issue-resolution"), any(ProbeContext.class)))
                .thenReturn(new CapabilityStatus.Unavailable("offline"));
        when(capabilityHealth.probe(eq(l3Descriptor), eq("issue-resolution"), any(ProbeContext.class)))
                .thenReturn(new CapabilityStatus.Ready());

        var result = dispatch.escalate("l1", "issue-resolution", "acme");

        assertThat(result).isPresent();
        assertThat(result.get().descriptor().agentId()).isEqualTo("l3");
    }
```

- [ ] **Step 6: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#escalateSkipsUnhealthyAgent`
Expected: PASS

- [ ] **Step 7: Write failing test — escalation returns empty when all unhealthy**

```java
    @Test void escalateReturnsEmptyWhenAllUnhealthy() {
        var org = OrgStructure.define("acme")
            .unit("support").name("Support").kind("team")
                .member("l1", "tier-1")
                .member("l2", "tier-2")
                .add()
            .escalatesTo("l1", "l2").add()
            .build();
        org.registerAll(orgRegistry);

        var l2Descriptor = AgentDescriptor.builder()
                .agentId("l2").name("L2").tenancyId("acme").build();
        when(agentRegistry.findById("l2", "acme")).thenReturn(Optional.of(l2Descriptor));
        when(capabilityHealth.probe(eq(l2Descriptor), eq("review"), any(ProbeContext.class)))
                .thenReturn(new CapabilityStatus.Unavailable("down"));

        assertThat(dispatch.escalate("l1", "review", "acme")).isEmpty();
    }
```

- [ ] **Step 8: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#escalateReturnsEmptyWhenAllUnhealthy`
Expected: PASS

- [ ] **Step 9: Write failing test — delegation validation (subordinate)**

```java
    @Test void canDelegateToSubordinate() {
        var org = OrgStructure.define("acme")
            .unit("eng").name("Engineering").kind("team")
                .member("alice", "lead")
                .member("bob", "dev")
                .add()
            .supervises("alice", "bob").add()
            .build();
        org.registerAll(orgRegistry);

        assertThat(dispatch.canDelegateTo("alice", "bob", "acme")).isTrue();
    }
```

- [ ] **Step 10: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#canDelegateToSubordinate`
Expected: PASS

- [ ] **Step 11: Write failing test — delegation validation (same unit peer)**

```java
    @Test void canDelegateToSameUnitPeer() {
        var org = OrgStructure.define("acme")
            .unit("eng").name("Engineering").kind("team")
                .member("alice", "dev")
                .member("bob", "dev")
                .add()
            .build();
        org.registerAll(orgRegistry);

        assertThat(dispatch.canDelegateTo("alice", "bob", "acme")).isTrue();
    }
```

- [ ] **Step 12: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#canDelegateToSameUnitPeer`
Expected: PASS

- [ ] **Step 13: Write failing test — delegation denied for unrelated agents**

```java
    @Test void cannotDelegateToUnrelatedAgent() {
        var org = OrgStructure.define("acme")
            .unit("eng").name("Engineering").kind("team")
                .member("alice", "dev")
                .add()
            .unit("sales").name("Sales").kind("team")
                .member("bob", "rep")
                .add()
            .build();
        org.registerAll(orgRegistry);

        assertThat(dispatch.canDelegateTo("alice", "bob", "acme")).isFalse();
    }
```

- [ ] **Step 14: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#cannotDelegateToUnrelatedAgent`
Expected: PASS

- [ ] **Step 15: Write failing test — team members**

```java
    @Test void teamMembersExcludesSelf() {
        var org = OrgStructure.define("acme")
            .unit("eng").name("Engineering").kind("team")
                .member("alice", "lead")
                .member("bob", "dev")
                .member("carol", "dev")
                .add()
            .build();
        org.registerAll(orgRegistry);

        var members = dispatch.teamMembers("bob", "acme");
        assertThat(members).containsExactlyInAnyOrder("alice", "carol");
        assertThat(members).doesNotContain("bob");
    }
```

- [ ] **Step 16: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#teamMembersExcludesSelf`
Expected: PASS

- [ ] **Step 17: Write failing test — no escalation path returns empty**

```java
    @Test void escalateWithNoPathReturnsEmpty() {
        assertThat(dispatch.escalate("lone-agent", "review", "acme")).isEmpty();
    }
```

- [ ] **Step 18: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#escalateWithNoPathReturnsEmpty`
Expected: PASS

- [ ] **Step 19: Write failing test — team members across multiple units**

```java
    @Test void teamMembersAcrossMultipleUnits() {
        var org = OrgStructure.define("acme")
            .unit("eng").name("Engineering").kind("team")
                .member("alice", "dev")
                .member("bob", "dev")
                .add()
            .unit("review").name("Review Board").kind("committee")
                .member("alice", "reviewer")
                .member("carol", "reviewer")
                .add()
            .build();
        org.registerAll(orgRegistry);

        var members = dispatch.teamMembers("alice", "acme");
        assertThat(members).containsExactlyInAnyOrder("bob", "carol");
    }
```

- [ ] **Step 20: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=DefaultOrgAwareDispatchTest#teamMembersAcrossMultipleUnits`
Expected: PASS

- [ ] **Step 21: Add mockito test dependency to org-runtime pom.xml (if not already added in Task 1)**

Verify `mockito-core` test dependency exists in `org-runtime/pom.xml` (added in Task 1, Step 3).

- [ ] **Step 22: Run full test suite and commit**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,eidos-core,runtime,org-runtime`
Expected: all tests PASS

```bash
git add org-runtime/src/
git commit -m "feat(#195): DefaultOrgAwareDispatch — org-backed routing and delegation

Walks OrgRegistry.escalationPath() with CapabilityHealth probing for
escalation, validates delegation via subordinate/same-unit membership,
resolves team members from org unit membership. Overrides NoOp
@DefaultBean when org-runtime is on the classpath.

Refs #195

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 23: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and all tests pass

---

## References

- [PromptContributor SPI] — `api/src/main/java/io/casehub/eidos/api/spi/PromptContributor.java:10`
- [PromptBlock] — `api/src/main/java/io/casehub/eidos/api/PromptBlock.java:5`
- [CdiPromptAssembler] — `runtime/src/main/java/io/casehub/eidos/runtime/renderer/CdiPromptAssembler.java:18`
- [OrgRegistry] — `org-api/src/main/java/io/casehub/eidos/org/api/OrgRegistry.java:6`
- [AgentRelationship] — `org-api/src/main/java/io/casehub/eidos/org/api/AgentRelationship.java:5`
- [CapabilityHealth] — `api/src/main/java/io/casehub/eidos/api/CapabilityHealth.java:5`
- [AgentSelector] — `api/src/main/java/io/casehub/eidos/api/AgentSelector.java:5`
- [EidosCoreProducer] — `runtime/src/main/java/io/casehub/eidos/runtime/EidosCoreProducer.java:48`
- [CustomerSupportTest] — `examples/org-scenarios/src/test/java/io/casehub/eidos/org/examples/CustomerSupportTest.java` — org structure test patterns
- [GitHub #194] — auto-inject org context into rendered prompts via PromptContributor
- [GitHub #195] — OrgAwareDispatch — standard contract for org-aware routing and delegation
