# Practical Examples Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #82 — epic: Practical examples — demonstrate operational features end-to-end
**Issue group:** #82, #78, #80, #81

**Goal:** Three independent @QuarkusTest scenario tests demonstrating learned specialization, full probe pipeline, and cost-aware routing.

**Architecture:** Each test is self-contained in `examples/agent-scenarios/`. Tests use `casehub-eidos-memory` (in-memory stores) and `casehub-eidos-vocab` (CasehubCapabilityTerm hierarchy). No new production code — examples only exercise existing APIs. Each test method uses a unique tenancy ID for isolation.

**Tech Stack:** Java 21, Quarkus, JUnit 5, AssertJ, casehub-eidos (runtime + memory + vocab)

## Global Constraints

- No new production code — examples exercise existing APIs only
- No new dependencies in the examples module POM
- No shared base class or helpers across the three tests
- Each test method uses a unique tenancy ID — no execution order dependency
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios`
- IntelliJ MCP required for code navigation and structural editing

---

### Task 1: LearnedSpecializationScenarioTest (#78)

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedSpecializationScenarioTest.java`

**Interfaces:**
- Consumes: `AgentRegistry.register()`, `AgentDescriptor.builder()`, `AgentCapability.builder()`, `BehavioralSignalStore.record()/learned()/count()/clear()`, `CapabilityHealth.probe()`, `CapabilityHealth.ProbeContext.of()`, `BehavioralSignal.DECLINE/SUCCESS`, `CasehubCapabilityTerm.URI`
- Produces: Nothing — terminal test class

- [ ] **Step 1: Write the test class with all six test methods**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.vocab.CasehubCapabilityTerm;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Demonstrates task-domain-scoped behavioral signals: DECLINE signals accumulate
 * per (capabilityName, qualifier) where qualifier is the task domain string.
 * Probing with a matching taskDomain triggers learned exclusion; probing with
 * null taskDomain skips the check entirely. Both "security-code-review" and
 * "code-review" query tags resolve to the same declared capability via subsumption.
 *
 * <p>TTL: in production, signals expire after a configurable TTL (default 30 days).
 * This test uses clear() to demonstrate the reset path — real TTL expiry happens
 * automatically in the store implementation.
 */
@QuarkusTest
class LearnedSpecializationScenarioTest {

    @Inject AgentRegistry registry;
    @Inject CapabilityHealth capabilityHealth;
    @Inject BehavioralSignalStore signalStore;

    private AgentDescriptor agent(String tenancyId) {
        return AgentDescriptor.builder()
                .agentId("specialization-agent")
                .tenancyId(tenancyId)
                .name("Code Review Specialist")
                .slot("reviewer")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("code-review")
                                .capabilityVocabulary(CasehubCapabilityTerm.URI)
                                .epistemicDomains(Map.of("java", 0.95))
                                .build()))
                .disposition(AgentDisposition.builder().build())
                .build();
    }

    @Test
    void agent_starts_ready_for_security_domain() {
        var tenancy = "spec-baseline";
        var desc = agent(tenancy);
        registry.register(desc);

        var status = capabilityHealth.probe(desc, "security-code-review",
                ProbeContext.of("security"));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void decline_signals_below_threshold_still_ready() {
        var tenancy = "spec-below-threshold";
        var desc = agent(tenancy);
        registry.register(desc);

        signalStore.record(desc.agentId(), tenancy, "code-review",
                "security", BehavioralSignal.DECLINE);
        signalStore.record(desc.agentId(), tenancy, "code-review",
                "security", BehavioralSignal.DECLINE);

        var status = capabilityHealth.probe(desc, "security-code-review",
                ProbeContext.of("security"));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
        assertThat(signalStore.count(desc.agentId(), tenancy, "code-review",
                "security", BehavioralSignal.DECLINE)).isEqualTo(2);
    }

    @Test
    void third_decline_triggers_learned_exclusion() {
        var tenancy = "spec-exclusion";
        var desc = agent(tenancy);
        registry.register(desc);

        for (int i = 0; i < 3; i++) {
            signalStore.record(desc.agentId(), tenancy, "code-review",
                    "security", BehavioralSignal.DECLINE);
        }

        var status = capabilityHealth.probe(desc, "security-code-review",
                ProbeContext.of("security"));

        assertThat(status).isInstanceOf(CapabilityStatus.Excluded.class);
        var excluded = (CapabilityStatus.Excluded) status;
        assertThat(excluded.source()).isEqualTo(CapabilityStatus.ExclusionSource.LEARNED);
        assertThat(excluded.domain()).isEqualTo("security");
        assertThat(excluded.declineCount()).isEqualTo(3);
    }

    @Test
    void null_task_domain_skips_learned_exclusion() {
        var tenancy = "spec-null-domain";
        var desc = agent(tenancy);
        registry.register(desc);

        for (int i = 0; i < 3; i++) {
            signalStore.record(desc.agentId(), tenancy, "code-review",
                    "security", BehavioralSignal.DECLINE);
        }

        // Probe with null taskDomain — learned exclusion check is skipped entirely
        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of(null));

        assertThat(status).isInstanceOf(CapabilityStatus.Ready.class);
    }

    @Test
    void success_signals_recorded_on_core_capability() {
        var tenancy = "spec-success";
        var desc = agent(tenancy);
        registry.register(desc);

        signalStore.record(desc.agentId(), tenancy, "code-review",
                "java", BehavioralSignal.SUCCESS);
        signalStore.record(desc.agentId(), tenancy, "code-review",
                "java", BehavioralSignal.SUCCESS);

        assertThat(signalStore.count(desc.agentId(), tenancy, "code-review",
                "java", BehavioralSignal.SUCCESS)).isEqualTo(2);

        var learned = signalStore.learned(desc.agentId(), tenancy,
                "code-review", BehavioralSignal.SUCCESS);
        assertThat(learned).containsEntry("java", 2);
    }

    @Test
    void clear_resets_learned_exclusion() {
        var tenancy = "spec-clear";
        var desc = agent(tenancy);
        registry.register(desc);

        for (int i = 0; i < 3; i++) {
            signalStore.record(desc.agentId(), tenancy, "code-review",
                    "security", BehavioralSignal.DECLINE);
        }

        // Verify excluded
        var excluded = capabilityHealth.probe(desc, "security-code-review",
                ProbeContext.of("security"));
        assertThat(excluded).isInstanceOf(CapabilityStatus.Excluded.class);

        // Clear and verify ready
        signalStore.clear(desc.agentId(), tenancy, "code-review",
                BehavioralSignal.DECLINE);

        var ready = capabilityHealth.probe(desc, "security-code-review",
                ProbeContext.of("security"));
        assertThat(ready).isInstanceOf(CapabilityStatus.Ready.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=LearnedSpecializationScenarioTest`
Expected: All 6 tests PASS

- [ ] **Step 3: Commit**

```bash
git add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedSpecializationScenarioTest.java
git commit -m "feat(#78): learned specialization lifecycle example

Demonstrates task-domain-scoped behavioral signals: DECLINE signals
accumulate per (capabilityName, qualifier), probe with matching
taskDomain triggers Excluded(LEARNED), null taskDomain skips check.

Refs #78"
```

---

### Task 2: FullProbeScenarioTest (#80)

**Files:**
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/FullProbeScenarioTest.java`

**Interfaces:**
- Consumes: `AgentRegistry.register()`, `AgentDescriptor.builder()`, `AgentCapability.builder()`, `CapabilityHealth.probe()`, `CapabilityHealth.ProbeContext.of()`, `AgentStateStore.record()`, `BehavioralSignalStore.record()`, `BehavioralSignal.DECLINE/VIOLATED`, `ComplianceDimension.LATENCY/DELEGATION/ESCALATION`, `DegradationReason.RATE_LIMITED`, `CasehubCapabilityTerm.URI`, all `CapabilityStatus` variants
- Produces: Nothing — terminal test class

- [ ] **Step 1: Write the test class with all ten test methods**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus;
import io.casehub.eidos.api.CapabilityHealth.CapabilityStatus.*;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.vocab.CasehubCapabilityTerm;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Complete probe pipeline reference — every CapabilityStatus variant in one test class.
 * Each test method is self-contained with its own tenancy ID.
 *
 * <p>Probe check order: Degraded → Unavailable → Excluded(DECLARED) →
 * Excluded(LEARNED) → EpistemicallyWeak → BehavioralViolation → Ready.
 */
@QuarkusTest
class FullProbeScenarioTest {

    @Inject AgentRegistry registry;
    @Inject CapabilityHealth capabilityHealth;
    @Inject AgentStateStore stateStore;
    @Inject BehavioralSignalStore signalStore;

    private AgentDescriptor codeReviewAgent(String agentId, String tenancyId) {
        return AgentDescriptor.builder()
                .agentId(agentId).tenancyId(tenancyId)
                .name(agentId).slot("reviewer")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("code-review")
                                .capabilityVocabulary(CasehubCapabilityTerm.URI)
                                .build()))
                .disposition(AgentDisposition.builder().build())
                .build();
    }

    @Test
    void degradation_overrides_everything() {
        var tenancy = "probe-degraded";
        var desc = codeReviewAgent("degraded-agent", tenancy);
        registry.register(desc);

        stateStore.record(desc.agentId(), tenancy,
                DegradationReason.RATE_LIMITED,
                Instant.now().plus(1, ChronoUnit.HOURS));

        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("java"));

        assertThat(status).isInstanceOf(Degraded.class);
        assertThat(((Degraded) status).reason())
                .isEqualTo(DegradationReason.RATE_LIMITED);
    }

    @Test
    void undeclared_capability_returns_unavailable() {
        var tenancy = "probe-unavailable";
        var desc = codeReviewAgent("undeclared-agent", tenancy);
        registry.register(desc);

        // Agent declares "code-review", probed for "testing" (no subsumption path)
        var status = capabilityHealth.probe(desc, "testing",
                ProbeContext.of(null));

        assertThat(status).isInstanceOf(Unavailable.class);
    }

    @Test
    void declared_domain_exclusion() {
        var tenancy = "probe-declared-excl";
        var desc = AgentDescriptor.builder()
                .agentId("domain-excluded-agent").tenancyId(tenancy)
                .name("Domain Excluded Agent").slot("reviewer")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("code-review")
                                .capabilityVocabulary(CasehubCapabilityTerm.URI)
                                .excludedDomains(Set.of("rust"))
                                .build()))
                .disposition(AgentDisposition.builder().build())
                .build();
        registry.register(desc);

        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("rust"));

        assertThat(status).isInstanceOf(Excluded.class);
        var excluded = (Excluded) status;
        assertThat(excluded.source()).isEqualTo(ExclusionSource.DECLARED);
        assertThat(excluded.domain()).isEqualTo("rust");
        assertThat(excluded.declineCount()).isZero();
    }

    @Test
    void learned_domain_exclusion() {
        var tenancy = "probe-learned-excl";
        var desc = codeReviewAgent("learned-excluded-agent", tenancy);
        registry.register(desc);

        for (int i = 0; i < 3; i++) {
            signalStore.record(desc.agentId(), tenancy, "code-review",
                    "rust", BehavioralSignal.DECLINE);
        }

        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("rust"));

        assertThat(status).isInstanceOf(Excluded.class);
        var excluded = (Excluded) status;
        assertThat(excluded.source()).isEqualTo(ExclusionSource.LEARNED);
        assertThat(excluded.domain()).isEqualTo("rust");
        assertThat(excluded.declineCount()).isEqualTo(3);
    }

    @Test
    void epistemic_weakness_below_confidence() {
        var tenancy = "probe-weak";
        var desc = AgentDescriptor.builder()
                .agentId("weak-agent").tenancyId(tenancy)
                .name("Weak Agent").slot("reviewer")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("code-review")
                                .capabilityVocabulary(CasehubCapabilityTerm.URI)
                                .epistemicDomains(Map.of("rust", 0.15))
                                .build()))
                .disposition(AgentDisposition.builder().build())
                .build();
        registry.register(desc);

        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("rust"));

        assertThat(status).isInstanceOf(EpistemicallyWeak.class);
        var weak = (EpistemicallyWeak) status;
        assertThat(weak.domain()).isEqualTo("rust");
        assertThat(weak.confidence()).isEqualTo(0.15);
    }

    @Test
    void behavioral_violation_per_dimension() {
        var tenancy = "probe-violated-pd";
        var desc = codeReviewAgent("violated-agent-pd", tenancy);
        registry.register(desc);

        for (int i = 0; i < 3; i++) {
            signalStore.record(desc.agentId(), tenancy, "code-review",
                    ComplianceDimension.LATENCY, BehavioralSignal.VIOLATED);
        }

        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("java"));

        assertThat(status).isInstanceOf(BehavioralViolation.class);
        var violation = (BehavioralViolation) status;
        assertThat(violation.kind())
                .isEqualTo(BehavioralViolation.ViolationKind.PER_DIMENSION);
        assertThat(violation.violations())
                .containsEntry(ComplianceDimension.LATENCY, 3);
    }

    @Test
    void behavioral_violation_aggregate() {
        var tenancy = "probe-violated-agg";
        var desc = codeReviewAgent("violated-agent-agg", tenancy);
        registry.register(desc);

        // 2 violations per dimension — none exceeds per-dimension threshold (3)
        // but total (6) exceeds aggregate threshold (5)
        for (int i = 0; i < 2; i++) {
            signalStore.record(desc.agentId(), tenancy, "code-review",
                    ComplianceDimension.LATENCY, BehavioralSignal.VIOLATED);
            signalStore.record(desc.agentId(), tenancy, "code-review",
                    ComplianceDimension.DELEGATION, BehavioralSignal.VIOLATED);
            signalStore.record(desc.agentId(), tenancy, "code-review",
                    ComplianceDimension.ESCALATION, BehavioralSignal.VIOLATED);
        }

        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("java"));

        assertThat(status).isInstanceOf(BehavioralViolation.class);
        var violation = (BehavioralViolation) status;
        assertThat(violation.kind())
                .isEqualTo(BehavioralViolation.ViolationKind.AGGREGATE);
        assertThat(violation.violations())
                .containsEntry(ComplianceDimension.LATENCY, 2)
                .containsEntry(ComplianceDimension.DELEGATION, 2)
                .containsEntry(ComplianceDimension.ESCALATION, 2);
    }

    @Test
    void healthy_agent_passes_all_checks() {
        var tenancy = "probe-healthy";
        var desc = codeReviewAgent("healthy-agent", tenancy);
        registry.register(desc);

        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("java"));

        assertThat(status).isInstanceOf(Ready.class);
    }

    @Test
    void precedence_degraded_beats_exclusion() {
        var tenancy = "probe-precedence";
        var desc = AgentDescriptor.builder()
                .agentId("precedence-agent").tenancyId(tenancy)
                .name("Precedence Agent").slot("reviewer")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("code-review")
                                .capabilityVocabulary(CasehubCapabilityTerm.URI)
                                .excludedDomains(Set.of("rust"))
                                .build()))
                .disposition(AgentDisposition.builder().build())
                .build();
        registry.register(desc);

        // Agent has BOTH degradation AND declared exclusion for "rust"
        stateStore.record(desc.agentId(), tenancy,
                DegradationReason.RATE_LIMITED,
                Instant.now().plus(1, ChronoUnit.HOURS));

        // Degradation is checked first — it wins
        var status = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("rust"));

        assertThat(status).isInstanceOf(Degraded.class);
    }

    /**
     * Garden gotcha GE-20260523-fa7407: taskDomain is the subject domain
     * ("rust"), not the capability name ("code-review"). Passing the
     * capability name as taskDomain causes epistemicDomains lookup to miss.
     */
    @Test
    void probe_context_task_domain_vs_capability_tag() {
        var tenancy = "probe-context";
        var desc = AgentDescriptor.builder()
                .agentId("context-agent").tenancyId(tenancy)
                .name("Context Agent").slot("reviewer")
                .capabilities(List.of(
                        AgentCapability.builder()
                                .name("code-review")
                                .capabilityVocabulary(CasehubCapabilityTerm.URI)
                                .epistemicDomains(Map.of("rust", 0.15))
                                .build()))
                .disposition(AgentDisposition.builder().build())
                .build();
        registry.register(desc);

        // CORRECT: taskDomain is the subject domain
        var correct = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("rust"));
        assertThat(correct).isInstanceOf(EpistemicallyWeak.class);

        // WRONG (but not an error): taskDomain = capability name
        // epistemicDomains doesn't have "code-review" → no weakness found → Ready
        var wrong = capabilityHealth.probe(desc, "code-review",
                ProbeContext.of("code-review"));
        assertThat(wrong).isInstanceOf(Ready.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=FullProbeScenarioTest`
Expected: All 10 tests PASS

- [ ] **Step 3: Commit**

```bash
git add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/FullProbeScenarioTest.java
git commit -m "feat(#80): full probe pipeline example — all CapabilityStatus variants

Eight agents demonstrating all six probe check categories plus both
behavioral violation modes (PER_DIMENSION and AGGREGATE). Includes
precedence test and ProbeContext gotcha (GE-20260523-fa7407).

Refs #80"
```

---

### Task 3: CostAwareRoutingScenarioTest (#81)

**Files:**
- Modify: `examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml`
- Create: `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CostAwareRoutingScenarioTest.java`

**Interfaces:**
- Consumes: `AgentRegistry.find(AgentQuery)`, `AgentQuery.byCapability()`, `AgentQuery.byCapabilityAndDomain()`, `AgentMatch.descriptor()`, `AgentMatch.resolvedCapability()`, `ResolvedCapability.capability()`, `ResolvedCapability.degree()`, `MatchDegree.Exact/Plugin/Specialization`, `AgentCapability.qualityHint()/latencyHintP50Ms()/costHint()`, `SystemPromptRenderer.render()`, `AgentPromptContext.forFormat()`, `RenderFormat.A2A_CARD`, `CasehubCapabilityTerm.URI`
- Produces: Nothing — terminal test class

- [ ] **Step 1: Append cost-routing agents to descriptors.yaml**

Append to the end of `examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml`:

```yaml

  - agentId: premium-reviewer
    name: Premium Code Reviewer
    slot: reviewer
    tenancyId: cost-routing
    capabilities:
      - name: code-review
        capabilityVocabulary: urn:casehub:vocab:capability
        qualityHint: 0.95
        latencyHintP50Ms: 30000
        costHint: premium
        epistemicDomains:
          java: 0.95
          rust: 0.85

  - agentId: standard-reviewer
    name: Standard Code Reviewer
    slot: reviewer
    tenancyId: cost-routing
    capabilities:
      - name: code-review
        capabilityVocabulary: urn:casehub:vocab:capability
        qualityHint: 0.70
        latencyHintP50Ms: 2000
        costHint: standard
        epistemicDomains:
          java: 0.80
          python: 0.90

  - agentId: security-specialist
    name: Security Code Review Specialist
    slot: security-reviewer
    tenancyId: cost-routing
    capabilities:
      - name: security-code-review
        capabilityVocabulary: urn:casehub:vocab:capability
        qualityHint: 0.90
        latencyHintP50Ms: 5000
        costHint: premium
        epistemicDomains:
          java: 0.95
        excludedDomains: [rust]
```

- [ ] **Step 2: Write the test class with all ten test methods**

```java
package io.casehub.eidos.examples;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.SystemPromptRenderer.RenderFormat;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Comparator;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Cost-aware multi-agent routing: three code-review agents with different
 * quality/latency/cost profiles. Consumer queries by capability, ranks by
 * routing signals, and filters by task domain — all inline, no framework.
 *
 * <p>Agents registered via META-INF/eidos/descriptors.yaml (YAML-driven).
 */
@QuarkusTest
class CostAwareRoutingScenarioTest {

    static final String TENANCY = "cost-routing";

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;

    // ── Discovery ────────────────────────────────────────────────────────

    @Test
    void all_three_found_by_foundation_capability() {
        var matches = registry.find(AgentQuery.byCapability("code-review", TENANCY));

        assertThat(matches).hasSize(3);
        assertThat(matches).extracting(m -> m.descriptor().agentId())
                .containsExactlyInAnyOrder(
                        "premium-reviewer", "standard-reviewer", "security-specialist");
    }

    @Test
    void matches_ordered_by_match_degree() {
        var matches = registry.find(AgentQuery.byCapability("code-review", TENANCY));

        // Exact matches (premium-reviewer, standard-reviewer) before
        // Specialization (security-specialist)
        var exactCount = matches.stream()
                .filter(m -> m.resolvedCapability().degree() instanceof MatchDegree.Exact)
                .count();
        assertThat(exactCount).isEqualTo(2);

        var last = matches.get(matches.size() - 1);
        assertThat(last.descriptor().agentId()).isEqualTo("security-specialist");
        assertThat(last.resolvedCapability().degree())
                .isInstanceOf(MatchDegree.Specialization.class);
    }

    @Test
    void each_match_carries_routing_signals() {
        var matches = registry.find(AgentQuery.byCapability("code-review", TENANCY));

        for (var match : matches) {
            assertThat(match.resolvedCapability()).isNotNull();
            var cap = match.resolvedCapability().capability();
            assertThat(cap.qualityHint()).isNotNull();
            assertThat(cap.latencyHintP50Ms()).isNotNull();
            assertThat(cap.costHint()).isNotNull();
        }
    }

    // ── Domain filtering ─────────────────────────────────────────────────

    @Test
    void task_domain_excludes_declared_exclusions() {
        var matches = registry.find(
                AgentQuery.byCapabilityAndDomain("code-review", "rust", TENANCY));

        // security-specialist declares excludedDomains: [rust]
        assertThat(matches).extracting(m -> m.descriptor().agentId())
                .doesNotContain("security-specialist")
                .contains("premium-reviewer", "standard-reviewer");
    }

    @Test
    void task_domain_without_exclusion_returns_all() {
        var matches = registry.find(
                AgentQuery.byCapabilityAndDomain("code-review", "java", TENANCY));

        assertThat(matches).hasSize(3);
    }

    // ── Consumer-side ranking (inline logic) ─────────────────────────────

    @Test
    void rank_by_quality_for_critical_review() {
        var matches = registry.find(AgentQuery.byCapability("code-review", TENANCY));

        var ranked = matches.stream()
                .sorted(Comparator.comparingDouble(
                        (AgentMatch m) -> m.resolvedCapability().capability().qualityHint())
                        .reversed())
                .toList();

        assertThat(ranked.get(0).descriptor().agentId()).isEqualTo("premium-reviewer");
        assertThat(ranked.get(ranked.size() - 1).descriptor().agentId())
                .isEqualTo("standard-reviewer");
    }

    @Test
    void rank_by_latency_for_time_sensitive_task() {
        var matches = registry.find(AgentQuery.byCapability("code-review", TENANCY));

        var fast = matches.stream()
                .filter(m -> m.resolvedCapability().capability().latencyHintP50Ms() <= 5000)
                .sorted(Comparator.comparingLong(
                        (AgentMatch m) -> m.resolvedCapability().capability().latencyHintP50Ms()))
                .toList();

        assertThat(fast).hasSize(2);
        assertThat(fast.get(0).descriptor().agentId()).isEqualTo("standard-reviewer");
    }

    @Test
    void filter_by_cost_tier() {
        var matches = registry.find(AgentQuery.byCapability("code-review", TENANCY));

        var standard = matches.stream()
                .filter(m -> "standard".equals(
                        m.resolvedCapability().capability().costHint()))
                .toList();

        assertThat(standard).hasSize(1);
        assertThat(standard.get(0).descriptor().agentId())
                .isEqualTo("standard-reviewer");
    }

    @Test
    void combined_ranking_quality_within_latency_budget() {
        var matches = registry.find(AgentQuery.byCapability("code-review", TENANCY));

        var withinBudget = matches.stream()
                .filter(m -> m.resolvedCapability().capability().latencyHintP50Ms() <= 10000)
                .sorted(Comparator.comparingDouble(
                        (AgentMatch m) -> m.resolvedCapability().capability().qualityHint())
                        .reversed())
                .toList();

        // security-specialist (0.90, 5000ms) beats standard-reviewer (0.70, 2000ms)
        assertThat(withinBudget).hasSize(2);
        assertThat(withinBudget.get(0).descriptor().agentId())
                .isEqualTo("security-specialist");
        assertThat(withinBudget.get(1).descriptor().agentId())
                .isEqualTo("standard-reviewer");
    }

    // ── A2A_CARD rendering ───────────────────────────────────────────────

    @Test
    void a2a_card_surfaces_routing_signals() {
        var desc = registry.findById("premium-reviewer", TENANCY).orElseThrow();
        var rendered = renderer.render(desc,
                AgentPromptContext.forFormat(RenderFormat.A2A_CARD));

        assertThat(rendered.content()).contains("\"qualityHint\"");
        assertThat(rendered.content()).contains("0.95");
        assertThat(rendered.content()).contains("\"latencyHintP50Ms\"");
        assertThat(rendered.content()).contains("30000");
        assertThat(rendered.content()).contains("\"costHint\"");
        assertThat(rendered.content()).contains("\"premium\"");
        assertThat(rendered.content()).contains("\"epistemicDomains\"");
    }
}
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios -Dtest=CostAwareRoutingScenarioTest`
Expected: All 10 tests PASS

- [ ] **Step 4: Run the full examples module to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/agent-scenarios`
Expected: All existing tests + all new tests PASS

- [ ] **Step 5: Commit**

```bash
git add examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml
git add examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CostAwareRoutingScenarioTest.java
git commit -m "feat(#81): cost-aware multi-agent routing example

Three YAML-registered agents with different quality/latency/cost
profiles. Demonstrates consumer-side ranking, taskDomain filtering,
subsumption discovery, and A2A_CARD routing signal rendering.

Refs #81"
```
