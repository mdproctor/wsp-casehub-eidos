# Practical Examples — Operational Features End-to-End

**Issue:** casehubio/eidos#82 (epic)
**Children:** #78 (Learned specialization), #80 (Full probe pipeline), #81 (Cost-aware routing)
**Date:** 2026-08-04

## Overview

Three independent `@QuarkusTest` scenario tests in `examples/agent-scenarios/`,
each demonstrating an operational feature that differentiates eidos from a static
registry. Each test class is self-contained — one file = one reference.

All tests use `casehub-eidos-memory` (in-memory stores) and `casehub-eidos-vocab`
(CasehubCapabilityTerm for subsumption). No external dependencies.

---

## 1. LearnedSpecializationScenarioTest (#78)

**File:** `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/LearnedSpecializationScenarioTest.java`

**Registration:** Programmatic (Builder API).

### Scenario

A code-review agent accumulates DECLINE signals for security tasks, gets excluded
from security dispatch, but continues serving code-review tasks. SUCCESS signals on
core competency are recorded. Signal clearing restores eligibility.

### Agent

Single agent `specialization-agent` in tenancy `specialization-lifecycle`:
- Capability: `code-review` grounded via `CasehubCapabilityTerm.URI`
- `epistemicDomains: {"java": 0.95}`

Security-code-review is reachable via subsumption (CasehubCapabilityTerm hierarchy:
`security-code-review` specializes `code-review`).

### Test Methods

| Method | What it demonstrates |
|--------|---------------------|
| `agent_starts_ready_for_all_capabilities` | Baseline: probe returns `Ready` for both `code-review` and `security-code-review` (via subsumption) |
| `decline_signals_accumulate_toward_exclusion` | Record 2 DECLINE signals for security work; probe still returns `Ready` (below default threshold of 3) |
| `third_decline_triggers_learned_exclusion` | Record 3rd DECLINE; probe returns `Excluded(LEARNED)` for `security-code-review` |
| `core_capability_unaffected_by_specialization_exclusion` | Same agent, same moment: probe returns `Ready` for `code-review` |
| `success_signals_recorded_on_core_capability` | Record SUCCESS signals on `code-review`; verify `count()` reflects them |
| `clear_resets_learned_exclusion` | `clear()` DECLINE signals; probe returns `Ready` again for security work |

### APIs Exercised

- `BehavioralSignalStore.record()` with `BehavioralSignal.DECLINE` and `SUCCESS`
- `BehavioralSignalStore.learned()`, `.count()`, `.clear()`
- `CapabilityHealth.probe()` → `Excluded(LEARNED)` and `Ready`
- Subsumption interaction: exclusion applies to subsumption-resolved capability

### TTL Note

The in-memory store uses `@ConfigProperty` TTL values (default 30 days). Real TTL
expiry can't be tested without time manipulation. The `clear()` test demonstrates the
reset path. Test javadoc notes that TTL expiry happens automatically in production.

---

## 2. FullProbeScenarioTest (#80)

**File:** `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/FullProbeScenarioTest.java`

**Registration:** Programmatic (Builder API).

### Scenario

A cast of seven agents, each crafted to trigger a specific probe outcome. Demonstrates
the complete six-step decision cascade and its precedence.

### Agents (tenancy `probe-pipeline`)

| Agent ID | Setup | Expected Probe Result |
|----------|-------|-----------------------|
| `degraded-agent` | Active degradation via `AgentStateStore.record()` with future expiry | `Degraded(RATE_LIMITED)` |
| `undeclared-agent` | Declares `code-review` only, probed for `testing` (no subsumption path) | `Unavailable` |
| `domain-excluded-agent` | Declares `code-review` with `excludedDomains: {"rust"}`, probed with `taskDomain: "rust"` | `Excluded(DECLARED)` |
| `learned-excluded-agent` | Declares `code-review`, 3+ DECLINE signals recorded | `Excluded(LEARNED)` |
| `weak-agent` | Declares `code-review` with `epistemicDomains: {"rust": 0.15}`, probed with `taskDomain: "rust"` | `EpistemicallyWeak("rust", 0.15)` |
| `violated-agent` | Declares `code-review`, 3+ VIOLATED signals on a single `ComplianceDimension` | `BehavioralViolation(violations, PER_DIMENSION)` |
| `healthy-agent` | Declares `code-review`, no degradation/exclusion/weakness/violations | `Ready` |

### Test Methods

| Method | What it demonstrates |
|--------|---------------------|
| `step1_degradation_overrides_everything` | Degraded agent returns `Degraded` |
| `step2_undeclared_capability_returns_unavailable` | Agent lacks capability, returns `Unavailable` |
| `step3_declared_domain_exclusion` | `excludedDomains` match, returns `Excluded(DECLARED)` |
| `step4_learned_domain_exclusion` | DECLINE signals above threshold, returns `Excluded(LEARNED)` |
| `step5_epistemic_weakness_below_confidence` | Low epistemic confidence, returns `EpistemicallyWeak` |
| `step6_behavioral_violation_per_dimension` | VIOLATED signals exceed per-dimension threshold, returns `BehavioralViolation(PER_DIMENSION)` |
| `step7_healthy_agent_passes_all_checks` | Clean agent, returns `Ready` |
| `precedence_degraded_agent_with_exclusion_returns_degraded` | Agent with both degradation AND exclusion returns `Degraded` (earlier step wins) |
| `probe_context_task_domain_vs_capability_tag` | Demonstrates correct `ProbeContext` usage: `taskDomain` is the subject domain ("rust"), not the capability name ("code-review"). Garden gotcha GE-20260523-fa7407. |

### APIs Exercised

- `CapabilityHealth.probe()` → all six `CapabilityStatus` variants
- `AgentStateStore.record()` for degradation
- `BehavioralSignalStore.record()` with `BehavioralSignal.DECLINE` and `VIOLATED`
- `ComplianceDimension` qualifier keys
- `ProbeContext` with `taskDomain`
- `ExclusionSource.DECLARED` vs `LEARNED`
- `BehavioralViolation.ViolationKind.PER_DIMENSION`

---

## 3. CostAwareRoutingScenarioTest (#81)

**File:** `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/CostAwareRoutingScenarioTest.java`

**Registration:** YAML (`META-INF/eidos/descriptors.yaml` in test resources).

### Scenario

A fleet of three code-review agents with different quality/latency/cost profiles.
Consumer queries by capability, gets all three (one via subsumption), and uses
routing signals inline to make dispatch decisions.

### Agents (tenancy `cost-routing`)

| Agent ID | Capability | Quality | Latency (p50ms) | Cost | Epistemic | Excluded |
|----------|-----------|---------|-----------------|------|-----------|----------|
| `premium-reviewer` | `code-review` (grounded) | 0.95 | 30000 | `premium` | java: 0.95, rust: 0.85 | — |
| `standard-reviewer` | `code-review` (grounded) | 0.70 | 2000 | `standard` | java: 0.80, python: 0.90 | — |
| `security-specialist` | `security-code-review` (grounded) | 0.90 | 5000 | `premium` | java: 0.95 | rust |

**YAML file:** `examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml`

This file already exists with four DraftHouse agents (tenancy `drafthouse`). The three
cost-routing agents (tenancy `cost-routing`) are appended to the same `descriptors:` list.
Tenancy isolation prevents interference — `cost-routing` queries never see DraftHouse agents.

### Test Methods

**Discovery:**

| Method | What it demonstrates |
|--------|---------------------|
| `all_three_found_by_foundation_capability` | `find(code-review)` returns all three; security-specialist via subsumption |
| `matches_ordered_by_match_degree` | Exact matches rank before Plugin/Specialization |
| `each_match_carries_routing_signals` | `resolvedCapability()` populated; routing signals accessible |

**Domain filtering:**

| Method | What it demonstrates |
|--------|---------------------|
| `task_domain_excludes_declared_exclusions` | Query with `taskDomain: "rust"` excludes security-specialist |
| `task_domain_without_exclusion_returns_all` | Query with `taskDomain: "java"` returns all three |

**Consumer-side ranking (inline logic):**

| Method | What it demonstrates |
|--------|---------------------|
| `rank_by_quality_for_critical_review` | Sort by `qualityHint` desc → premium-reviewer first |
| `rank_by_latency_for_time_sensitive_task` | Filter `latencyHintP50Ms <= 5000`, sort asc → standard-reviewer wins |
| `filter_by_cost_tier` | Filter `costHint.equals("standard")` → only standard-reviewer |
| `combined_ranking_quality_within_latency_budget` | Filter latency ≤ 10000, sort quality desc → security-specialist beats standard-reviewer |

**A2A_CARD rendering:**

| Method | What it demonstrates |
|--------|---------------------|
| `a2a_card_surfaces_routing_signals` | Render as A2A_CARD; JSON contains `qualityHint`, `latencyHintP50Ms`, `costHint`, `epistemicDomains` |

### APIs Exercised

- `AgentRegistry.find(AgentQuery)` with capability and taskDomain
- `AgentMatch.resolvedCapability()` and `MatchDegree`
- `AgentCapability` routing signals: `qualityHint`, `latencyHintP50Ms`, `costHint`
- `epistemicDomains` and `excludedDomains`
- `SystemPromptRenderer.render()` with `RenderFormat.A2A_CARD`
- `ClasspathYamlDescriptorRegistrar` (YAML-driven registration)

---

## What's NOT in scope

- No new production code — examples only exercise existing APIs
- No new dependencies in the examples module POM (already has runtime, memory, vocab, H2, quarkus-junit, assertj)
- No shared base class or helper utilities across the three tests
- No LLM-enriched rendering (examples module has no ChatModel)
