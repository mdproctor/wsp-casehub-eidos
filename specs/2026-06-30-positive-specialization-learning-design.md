# Positive Specialization Learning — Design Spec

**Issue:** eidos#70
**Date:** 2026-06-30
**Status:** Approved

## Summary

Make `CapabilitySpecializationStore` bidirectional — it currently only tracks DECLINE signals (negative specialization). This adds SUCCESS signals (positive specialization) through a signal-parameterized API redesign. The store captures evidence; consumption policy (how positive data affects routing or probing) is a strategy concern left to domain-specific implementations.

## Motivation

The current model is asymmetric: agents can only get worse over time (accumulating exclusions) but never better (no mechanism to surface domains where they consistently succeed). Positive specialization enables:

- **Preferential routing** — routing strategies can prefer agents with demonstrated domain proficiency
- **Proficiency discovery** — agents may excel in domains not listed in their declared `epistemicDomains`
- **Balanced signal** — positive evidence counterweights the exclusion model

Trust scoring in casehub-ledger is a separate concern — it provides continuous Bayesian confidence for routing weight. Specialization is categorical: exclude or include. They complement, not overlap.

## Design Decisions

### The store captures data; consumption is strategy

`DefaultCapabilityHealth.probe()` stays a negative-only filter. No new `CapabilityStatus` variants. No policy baked into the default probe about how positive evidence affects routing or overrides `EpistemicallyWeak`. Domains compose their own probe logic by displacing `DefaultCapabilityHealth` (`@DefaultBean`) and querying both signal types from the store.

### Signal-parameterized API over parallel methods

One set of methods discriminated by `SpecializationSignal` enum, rather than mirroring every method with a `recordSuccess`/`clearSuccesses`/etc. counterpart. Same key shape `(agentId, tenancyId, capabilityName, domain)`, same lifecycle (TTL, clear), same threshold model (count-based). DRY in implementations — one codepath, one table with a discriminator column.

### Per-signal TTL

DECLINE and SUCCESS evidence may age at different rates. Independently configurable via separate config properties, same default (30 days).

### Breaking change to every method signature

No backward compatibility shims. Every call site migrates from `recordDecline(...)` to `record(..., DECLINE)`. The migration is mechanical and the breakage forces every caller to be explicit about which signal they're recording.

## SPI Contract

### `SpecializationSignal`

```java
package io.casehub.eidos.api;

public enum SpecializationSignal {
    DECLINE,
    SUCCESS
}
```

Pure Java, Tier 1. Lives in `casehub-eidos-api`.

### `CapabilitySpecializationStore`

```java
package io.casehub.eidos.api;

import java.util.Map;

public interface CapabilitySpecializationStore {

    void record(String agentId, String tenancyId, String capabilityName,
                String domain, SpecializationSignal signal);

    void clear(String agentId, String tenancyId, String capabilityName,
               SpecializationSignal signal);

    Map<String, Integer> learned(String agentId, String tenancyId,
                                 String capabilityName, SpecializationSignal signal);

    int count(String agentId, String tenancyId, String capabilityName,
              String domain, SpecializationSignal signal);
}
```

- `clear()` clears all domains for one signal type. No domain-level clear. No `clearAll` convenience.
- `learned()` returns `domain -> count` for one signal type. Consumers that want both signals call twice.
- `count()` returns unexpired count for a specific `(agent, capability, domain, signal)` tuple.

## Implementations

### `NoOpCapabilitySpecializationStore`

Signature-only change. All methods remain no-ops. `@DefaultBean @ApplicationScoped`.

### `InMemoryCapabilitySpecializationStore`

Internal key gains signal dimension:

```java
private record StoreKey(String agentId, String tenancyId,
                         String capabilityName, SpecializationSignal signal) {}
```

Outer map keyed by `StoreKey`, inner map `domain -> Queue<Instant>`. Same TTL expiry logic. `@Alternative @Priority(1)` in `casehub-eidos-memory`.

### `JpaCapabilitySpecializationStore`

**Composite key gains `signalType`:**

```java
@Embeddable
public class CapabilitySpecializationId implements Serializable {
    String agentId;
    String tenancyId;
    String capabilityName;
    String domain;
    String signalType;  // "DECLINE" or "SUCCESS"
}
```

**Entity columns renamed** to be signal-agnostic:
- `decline_count` -> `signal_count`
- `last_declined` -> `last_recorded`

All JPQL queries gain `AND e.id.signalType = :signalType`. The `signal` parameter is converted via `signal.name()` for persistence and `SpecializationSignal.valueOf()` on read.

`@IfBuildProperty(name = "casehub.eidos.reactive.enabled", stringValue = "false", enableIfMissing = true)`.

## Schema

V5 migration — clean redesign (no existing installations):

```sql
DROP TABLE IF EXISTS capability_specialization;

CREATE TABLE capability_specialization (
    agent_id        VARCHAR(255) NOT NULL,
    tenancy_id      VARCHAR(255) NOT NULL,
    capability_name VARCHAR(100) NOT NULL,
    domain          VARCHAR(200) NOT NULL,
    signal_type     VARCHAR(20)  NOT NULL,
    signal_count    INT          NOT NULL DEFAULT 0,
    last_recorded   TIMESTAMP WITH TIME ZONE NOT NULL,
    expires_at      TIMESTAMP WITH TIME ZONE NOT NULL,
    PRIMARY KEY (agent_id, tenancy_id, capability_name, domain, signal_type)
);
```

`signal_type` is `VARCHAR(20)` — portable, no DDL change if a third signal type appears.

## Probe Pipeline

Mechanical rename only. `DefaultCapabilityHealth.probe()` step 4:

```java
// Before
final int count = specializationStore.declineCount(
    descriptor.agentId(), descriptor.tenancyId(), capabilityTag, context.taskDomain());

// After
final int count = specializationStore.count(
    descriptor.agentId(), descriptor.tenancyId(), capabilityTag,
    context.taskDomain(), SpecializationSignal.DECLINE);
```

Five-step probe order unchanged. No positive data queried. No new steps. No new statuses.

## Configuration

| Property | Default | Governs |
|----------|---------|---------|
| `casehub.eidos.specialization.decline-ttl-days` | 30 | Expiry of DECLINE records |
| `casehub.eidos.specialization.success-ttl-days` | 30 | Expiry of SUCCESS records |

`EidosPreferenceKeys.EXCLUDE_THRESHOLD` unchanged — still governs DECLINE-based exclusion in the default probe.

No new preference key for proficiency thresholds — that's a consumer concern.

## Testing

### Existing tests — mechanical migration

- `DefaultCapabilityHealthExclusionTest` — stub implements new SPI shape
- `DefaultCapabilityHealthDegradedTest` — stub implements new shape
- `InMemoryCapabilitySpecializationStoreTest` — renamed methods with `DECLINE` signal
- `JpaCapabilitySpecializationStoreTest` — renamed methods, new schema

### New tests — SUCCESS signal path

Added to existing test classes:

**`InMemoryCapabilitySpecializationStoreTest`:**
- `record_success_increments_count`
- `success_expires_after_ttl`
- `learned_returns_success_domains`
- `clear_success_does_not_affect_declines`
- `decline_and_success_coexist_independently`

**`JpaCapabilitySpecializationStoreTest`:**
- SUCCESS record/count/learned/clear through JPA
- Signal isolation — DECLINE and SUCCESS rows coexist
- TTL expiry on SUCCESS rows

**`DefaultCapabilityHealthExclusionTest`:**
- `success_data_does_not_affect_probe_result` — confirms probe is behaviorally unchanged

## Out of Scope

- Probe pipeline using positive evidence (strategy concern — future issue)
- New `CapabilityStatus.Proficient` variant (conflates filtering with ranking)
- Reactive `CapabilitySpecializationStore` variant (reactive health delegates to blocking)
- Query pre-filter changes in `JpaAgentRegistry.find()` (static `excludedDomains` only)
- Routing strategy integration in casehub-engine (consumer of the data — separate repo, separate issue)
