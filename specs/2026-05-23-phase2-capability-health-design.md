# Phase 2 Design — CapabilityHealth, ReactiveCapabilityHealth, and Examples Module

**Branch:** `issue-4-phase-2-capability-health`
**Issue:** casehubio/eidos#4
**Date:** 2026-05-23

---

## Summary

Phase 2 replaces `NoOpCapabilityHealth` with a real probe that checks declared capabilities and epistemic domain match. It adds `ReactiveCapabilityHealth` for blocking/reactive parity, and delivers an examples module demonstrating all implemented eidos capabilities with a coverage table showing what's built and what's coming.

---

## API Amendments (`casehub-eidos-api`)

### CapabilityHealth — probe takes AgentDescriptor, not agentId

Breaking change to the probe signature. The caller already has the descriptor from its registry query — passing it directly eliminates a redundant lookup and sidesteps the tenancy question (the registry enforces tenancy; the probe shouldn't need to).

```java
public interface CapabilityHealth {
    CapabilityStatus probe(AgentDescriptor descriptor, String capabilityTag, ProbeContext context);
    // ... sealed CapabilityStatus, DegradationReason, ProbeContext unchanged
}
```

### ReactiveCapabilityHealth (new interface)

Blocking/reactive parity per `reactive-service-build-gating` protocol.

```java
public interface ReactiveCapabilityHealth {
    Uni<CapabilityStatus> probe(AgentDescriptor descriptor, String capabilityTag,
                                CapabilityHealth.ProbeContext context);
}
```

Uses `CapabilityHealth.CapabilityStatus` and `CapabilityHealth.ProbeContext` — the nested types stay on the blocking interface (Tier 1 pure Java, no Mutiny dependency on nested types).

---

## DefaultCapabilityHealth (`runtime/`)

`@ApplicationScoped` (not `@DefaultBean` — this is the real implementation, Tier 2 in the CDI ladder). Gated with `@IfBuildProperty(name="casehub.eidos.reactive.enabled", stringValue="false", enableIfMissing=true)` to suppress in reactive profile.

### Probe logic

1. **Capability declared?** Scan `descriptor.capabilities()` for `name.equals(capabilityTag)`. Not found → `Unavailable("Capability '<tag>' not declared")`.

2. **Epistemic domain match?** If `context.taskDomain()` is non-null and the capability has `epistemicDomains`:
   - Look up `epistemicDomains.get(taskDomain)`
   - Confidence < `casehub.eidos.epistemic.weak-threshold` (default `0.3`) → `EpistemicallyWeak(domain, confidence)`
   - Domain not in map or confidence above threshold → `Ready`

3. **Runtime state** — returns `Ready`. Rate limits, context exhaustion, and overload require infrastructure not yet available (tracked in casehubio/eidos#5).

### Configuration

| Property | Default | Purpose |
|---|---|---|
| `casehub.eidos.epistemic.weak-threshold` | `0.3` | Below this confidence, probe returns `EpistemicallyWeak` |

---

## DefaultReactiveCapabilityHealth (`runtime/`)

`@IfBuildProperty(name="casehub.eidos.reactive.enabled", stringValue="true") @ApplicationScoped`. Delegates to `DefaultCapabilityHealth` via `Uni.createFrom().item(...)` — the probe is pure computation with no I/O.

---

## Examples Module

### Structure

```
examples/
├── pom.xml                          # Aggregator
├── README.md                        # Capability coverage table
└── agent-scenarios/
    ├── pom.xml
    └── src/test/java/io/casehub/eidos/examples/
        ├── MultiAgentTeamTest.java
        ├── CrossVocabularyDiscoveryTest.java
        ├── EpistemicDomainMatchingTest.java
        ├── TenancyIsolationTest.java
        └── DispositionVocabularyTest.java
```

### Dependencies (agent-scenarios)

| Dependency | Scope | Purpose |
|---|---|---|
| `casehub-eidos` (runtime) | compile | CdiVocabularyRegistry, DefaultCapabilityHealth |
| `casehub-eidos-memory` | compile | InMemoryAgentRegistry — zero datasource |
| `casehub-eidos-vocab` | compile | SVO, Conscientiousness, CasehubSlot |
| `quarkus-jdbc-h2` | test | Dummy datasource for Quarkus/Hibernate boot |
| `quarkus-junit5` | test | Test framework |

`application.properties`: H2 dummy datasource, `quarkus.flyway.migrate-at-start=false`, `quarkus.datasource.reactive=false`.

### Examples

**MultiAgentTeamTest** — register planner/reviewer/executor agents with different slots and capabilities using CasehubSlot vocabulary. Query by slot, by capability, by slot+capability, by all. Verify correct agents returned.
*Covers:* AgentDescriptor creation, AgentRegistry CRUD, all AgentQuery factories, SVO + CasehubSlot vocabularies.

**CrossVocabularyDiscoveryTest** — register agents using SVO vocabulary and CasehubSlot vocabulary. Use `equivalentValues()` to discover SVO "evaluator" = CasehubSlot "reviewer". Resolve terms by alias. Verify bidirectional cross-references.
*Covers:* VocabularyRegistry.find, resolve, equivalentValues; SVO ↔ CasehubSlot exactMatches.

**EpistemicDomainMatchingTest** — register agent with `{"java": 0.95, "rust": 0.2}` on code-review capability. Probe with `taskDomain="java"` → Ready. Probe with `taskDomain="rust"` → EpistemicallyWeak. Probe with `taskDomain="python"` (not in map) → Ready. Probe non-declared capability → Unavailable.
*Covers:* All four CapabilityStatus variants, epistemic domain resolution, weak-threshold config.

**TenancyIsolationTest** — register agents in tenant-a and tenant-b. Query from each tenant. Verify complete isolation — tenant-a never sees tenant-b agents, even with `AgentQuery.all()`.
*Covers:* Unconditional tenancy filtering, AgentQuery.all.

**DispositionVocabularyTest** — register agents with different disposition values (strict rule-following, flexible, autonomous). Resolve each value via the Conscientiousness vocabulary to verify label, description, and aliases for all four disposition axes.
*Covers:* Conscientiousness vocabulary all 12 terms, VocabularyRegistry.resolve for disposition values.

### Coverage Table (README.md)

| Capability | Example | Status |
|---|---|---|
| AgentDescriptor creation | Multi-agent team | ✅ |
| AgentRegistry.register | Multi-agent team | ✅ |
| AgentRegistry.findById | Multi-agent team | ✅ |
| AgentQuery.bySlot | Multi-agent team | ✅ |
| AgentQuery.byCapability | Multi-agent team | ✅ |
| AgentQuery.bySlotAndCapability | Multi-agent team | ✅ |
| AgentQuery.all | Tenancy isolation | ✅ |
| Tenancy isolation | Tenancy isolation | ✅ |
| VocabularyRegistry.find | Cross-vocabulary | ✅ |
| VocabularyRegistry.resolve | Cross-vocabulary, Disposition | ✅ |
| VocabularyRegistry.equivalentValues | Cross-vocabulary | ✅ |
| SVO vocabulary | Multi-agent team, Cross-vocabulary | ✅ |
| CasehubSlot vocabulary | Multi-agent team, Cross-vocabulary | ✅ |
| Conscientiousness vocabulary | Disposition | ✅ |
| CapabilityHealth.probe → Ready | Epistemic domain | ✅ |
| CapabilityHealth.probe → Unavailable | Epistemic domain | ✅ |
| CapabilityHealth.probe → EpistemicallyWeak | Epistemic domain | ✅ |
| CapabilityHealth.probe → Degraded | — | Phase 3 (runtime state infrastructure) |
| ReactiveCapabilityHealth | — | Implemented but not example'd (build-gated) |
| SystemPromptRenderer | — | Phase 3 (casehubio/eidos#5) |
| Knowledge graph | — | Phase 4 |

---

## Module Changes Summary

| Module | Change |
|---|---|
| `api/` | CapabilityHealth.probe signature: `String agentId` → `AgentDescriptor descriptor`; new `ReactiveCapabilityHealth` interface |
| `runtime/` | Replace `NoOpCapabilityHealth` with `DefaultCapabilityHealth`; add `DefaultReactiveCapabilityHealth`; both build-gated |
| `persistence-memory/` | No changes |
| `vocab/` | No changes |
| `deployment/` | No changes |
| `examples/` | **NEW** — aggregator + `agent-scenarios` sub-module |
| Parent `pom.xml` | Add `examples` to `<modules>` |

---

## Flyway

No migrations needed. Phase 2 adds no schema changes.
Update `design/.meta`: `flyway-next-v: none`.

---

## Platform Coherence Final Check

| Check | Status |
|---|---|
| Already exists elsewhere? | No |
| Correct repo? | Yes — eidos SPI + eidos examples |
| Consolidation opportunity? | None |
| CDI priority ladder | ✅ DefaultCapabilityHealth is @ApplicationScoped (Tier 2); consumers override with @Alternative @Priority(1) |
| Reactive build gating | ✅ @IfBuildProperty on reactive variant; enableIfMissing on blocking |
| Module naming | ✅ examples/agent-scenarios follows convention |
| Examples pattern | ✅ matches qhorus: sub-modules, README coverage table, InMemory stores |
| No-conditional-tenancy-filtering | ✅ not applicable — probe takes descriptor directly, no registry query |
| Trust maturity model | ✅ not applicable — trust routing is an engine/application concern, not a probe concern |
| PLATFORM.md update | After implementation — update CapabilityHealth entry |

## Deferred Items (tracked)

| Item | Issue |
|---|---|
| Engine dispatch integration | casehubio/engine#341 |
| Phase 3: runtime state probing + SystemPromptRenderer | casehubio/eidos#5 |
| Minor findings from Phase 1 review | casehubio/eidos#2, #3 |
