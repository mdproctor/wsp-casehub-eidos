# Practical Examples — Operational Features End-to-End

**Issue:** casehubio/eidos#82 (epic)
**Children:** #78 (Learned specialization), #79 (Degradation/recovery — CLOSED, implemented separately), #80 (Full probe pipeline), #81 (Cost-aware routing)
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

A code-review agent accumulates DECLINE signals scoped by task domain. When probed
for security-domain work, the agent is excluded (learned exclusion). When probed for
general code-review (no task domain), the agent remains Ready — the learned exclusion
check is skipped when `taskDomain` is null. SUCCESS signals on core competency are
recorded. Signal clearing restores eligibility.

**Key mechanism:** Learned exclusion is keyed on `(capabilityName, qualifier)` where
the qualifier is the task domain string. Both `security-code-review` and `code-review`
query tags resolve to the same declared capability `code-review` via subsumption. The
differentiator is `ProbeContext.taskDomain`, not the capability query tag. When
`taskDomain` is null, the probe pipeline skips the learned exclusion check entirely.

### Agent

Single agent `specialization-agent` in tenancy `specialization-lifecycle`:
- Capability: `code-review` grounded via `CasehubCapabilityTerm.URI`
- `epistemicDomains: {"java": 0.95}`

### Test Methods

Each test method is self-contained — uses a unique tenancy ID, registers its own
agent, records signals, and cleans up. No dependency on test execution order.

| Method | Qualifier for record() | ProbeContext taskDomain | What it demonstrates |
|--------|----------------------|------------------------|---------------------|
| `agent_starts_ready_for_security_domain` | — | `"security"` | Baseline: probe with `taskDomain="security"` returns `Ready` (no signals yet) |
| `decline_signals_below_threshold_still_ready` | `"security"` | `"security"` | Record 2 DECLINE signals with qualifier `"security"`; probe still returns `Ready` (below threshold of 3) |
| `third_decline_triggers_learned_exclusion` | `"security"` | `"security"` | Record 3 DECLINE signals; probe returns `Excluded(LEARNED, "security", 3)` |
| `null_task_domain_skips_learned_exclusion` | `"security"` | `null` | Same agent with 3 DECLINE signals for `"security"`, but probe with `taskDomain=null` returns `Ready` — learned exclusion check is skipped |
| `success_signals_recorded_on_core_capability` | `"java"` (SUCCESS) | — | Record SUCCESS signals with qualifier `"java"`; verify `count()` reflects them |
| `clear_resets_learned_exclusion` | `"security"` | `"security"` | Record 3 DECLINE, then `clear()` DECLINE signals; probe returns `Ready` again |

### APIs Exercised

- `BehavioralSignalStore.record(agentId, tenancyId, capabilityName, qualifier, signal)` — qualifier is the task domain string for DECLINE/SUCCESS
- `BehavioralSignalStore.learned()`, `.count()`, `.clear()`
- `CapabilityHealth.probe(descriptor, capabilityTag, ProbeContext.of(taskDomain))` — `taskDomain` must match the qualifier used in `record()`
- Subsumption: both `"security-code-review"` and `"code-review"` resolve to declared capability `"code-review"`

### TTL Note

The in-memory store uses `@ConfigProperty` TTL values (default 30 days). Real TTL
expiry can't be tested without time manipulation. The `clear()` test demonstrates the
reset path. Test javadoc notes that TTL expiry happens automatically in production.

---

## 2. FullProbeScenarioTest (#80)

**File:** `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/FullProbeScenarioTest.java`

**Registration:** Programmatic (Builder API).

### Scenario

A cast of eight agents, each crafted to trigger a specific probe outcome. Demonstrates
all six check categories in the probe pipeline, plus both behavioral violation modes
(per-dimension and aggregate). Each test method is self-contained with its own tenancy ID.

### Agents

Each agent uses a unique tenancy ID (e.g., `probe-degraded`, `probe-unavailable`) for
test isolation. All agents declare `code-review` capability grounded via `CasehubCapabilityTerm.URI`
unless noted otherwise.

| Agent ID | Setup | ProbeContext | Qualifier for record() | Expected Probe Result |
|----------|-------|-------------|----------------------|----------------------|
| `degraded-agent` | `AgentStateStore.record()` with `RATE_LIMITED`, future expiry | `of("java")` | — | `Degraded(RATE_LIMITED, ...)` |
| `undeclared-agent` | Declares `code-review` only, probed for `testing` | `of(null)` | — | `Unavailable("testing")` |
| `domain-excluded-agent` | `excludedDomains: {"rust"}` | `of("rust")` | — | `Excluded("rust", DECLARED, 0)` |
| `learned-excluded-agent` | 3 DECLINE signals recorded with qualifier `"rust"` | `of("rust")` | `"rust"` | `Excluded("rust", LEARNED, 3)` |
| `weak-agent` | `epistemicDomains: {"rust": 0.15}` | `of("rust")` | — | `EpistemicallyWeak("rust", 0.15)` |
| `violated-agent-pd` | 3 VIOLATED signals with qualifier `ComplianceDimension.LATENCY` | `of("java")` | `LATENCY` | `BehavioralViolation({LATENCY: 3}, PER_DIMENSION)` |
| `violated-agent-agg` | 2 VIOLATED on `LATENCY` + 2 on `DELEGATION` + 2 on `ESCALATION` (total 6 > aggregate threshold 5, none ≥ per-dimension threshold 3) | `of("java")` | `LATENCY`, `DELEGATION`, `ESCALATION` | `BehavioralViolation({LATENCY:2, DELEGATION:2, ESCALATION:2}, AGGREGATE)` |
| `healthy-agent` | No degradation/exclusion/weakness/violations | `of("java")` | — | `Ready` |

### Test Methods

| Method | What it demonstrates |
|--------|---------------------|
| `degradation_overrides_everything` | Degraded agent returns `Degraded` (checked first) |
| `undeclared_capability_returns_unavailable` | Agent lacks capability, returns `Unavailable` |
| `declared_domain_exclusion` | `excludedDomains` match, returns `Excluded(DECLARED)` |
| `learned_domain_exclusion` | DECLINE signals above threshold, returns `Excluded(LEARNED)` |
| `epistemic_weakness_below_confidence` | Low epistemic confidence, returns `EpistemicallyWeak` |
| `behavioral_violation_per_dimension` | VIOLATED signals on one dimension exceed threshold, returns `BehavioralViolation(PER_DIMENSION)` |
| `behavioral_violation_aggregate` | VIOLATED signals spread across dimensions exceed aggregate threshold (5), no single dimension exceeds per-dimension threshold (3), returns `BehavioralViolation(AGGREGATE)` |
| `healthy_agent_passes_all_checks` | Clean agent, returns `Ready` |
| `precedence_degraded_beats_exclusion` | Agent with both degradation AND exclusion returns `Degraded` (earlier step wins) |
| `probe_context_task_domain_vs_capability_tag` | Correct `ProbeContext` usage: `taskDomain` is the subject domain ("rust"), not the capability name ("code-review"). Garden gotcha GE-20260523-fa7407. |

### APIs Exercised

- `CapabilityHealth.probe()` → all seven `CapabilityStatus` variants (including both violation modes)
- `AgentStateStore.record()` for degradation
- `BehavioralSignalStore.record()` with `BehavioralSignal.DECLINE` (qualifier = task domain) and `VIOLATED` (qualifier = `ComplianceDimension` key)
- `ComplianceDimension.LATENCY`, `.DELEGATION`, `.ESCALATION` as qualifier keys
- `ProbeContext.of(taskDomain)` — `taskDomain` must match qualifier used in `record()`
- `ExclusionSource.DECLARED` vs `LEARNED`
- `BehavioralViolation.ViolationKind.PER_DIMENSION` vs `AGGREGATE`

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
