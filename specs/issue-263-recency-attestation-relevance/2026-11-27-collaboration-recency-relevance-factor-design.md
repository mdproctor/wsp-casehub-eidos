# Collaboration-Recency Attestation Relevance Factor

**Issue:** casehubio/eidos#263
**Date:** 2026-11-27
**Status:** Draft

## Problem

Attestation relevance currently considers org-proximity (structural closeness),
capability-overlap (shared declared capabilities), and independence (anti-collusion
cap). It does not consider whether the attestor recently worked alongside the
scored agent on the same task. Fresh shared context makes an attestor's judgment
more informed — they observed the agent's work first-hand.

## Design

A new enricher/factor pair following the established pattern: enricher writes
a numeric value to the `relevanceSnapshot` JSON at attestation write time,
factor reads it at trust computation time.

### CollaborationRecencyAttestationEnricher

`@ApplicationScoped` bean implementing `LedgerAttestationEnricher` in
`io.casehub.eidos.runtime.trust`.

**Dependencies:**
- `Instance<AgentGraphQuery>` — CDI instance for fail-open injection

**Algorithm:**

1. Check `Instance<AgentGraphQuery>` — if unsatisfied or the resolved bean
   is `NoOpAgentGraphQuery`, return without enrichment (fail-open)
2. Compute lookback boundary: `Instant since = attestation.occurredAt.minus(lookbackDays, ChronoUnit.DAYS)`
3. Retrieve task histories for both agents:
   - `agentHistory(attestation.attestorId, entry.tenancyId, since)`
   - `agentHistory(entry.actorId, entry.tenancyId, since)`
4. Extract non-null `externalRef` values from each history's task list
5. Intersect the two sets of `externalRef` values
6. If intersection is empty, return without writing (factor returns NEUTRAL)
7. For each shared `externalRef`, find the most recent relevant timestamp:
   take the later of each agent's most recent task on that ref. For each
   agent's task, use `endedAt` if non-null, otherwise `startedAt` (task
   still in progress). The "collaboration timestamp" for a shared ref is
   the minimum of the two agents' timestamps (the most recent moment both
   had been active on it).
8. Take the maximum collaboration timestamp across all shared refs (the
   most recent collaboration)
9. Compute: `daysSince = Duration.between(mostRecentCollaboration, attestation.occurredAt).toMillis() / 86_400_000.0`
10. Clamp to `max(0.0, daysSince)` (in-progress tasks may produce negative values)
11. Write `{"collaboration-recency-days": daysSince}` to the snapshot JSON
    using the standard merge pattern (deserialize existing → put key →
    serialize back)

**Configuration:**

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `casehub.eidos.trust.collaboration.lookback-days` | `int` | `90` | How far back the enricher searches task history |

**Error handling:** `JsonProcessingException` caught and logged at WARN
level (non-fatal, consistent with sibling enrichers). `AgentGraphQuery`
exceptions caught and logged at WARN (enricher must not block attestation
persistence).

**Name:** `"collaboration-recency"`

### CollaborationRecencyRelevanceFactor

`@ApplicationScoped` bean implementing `AttestationRelevanceFactor` in
`io.casehub.eidos.runtime.trust`.

**Algorithm:**

1. Read `"collaboration-recency-days"` from `ctx.snapshot()`
2. If absent, null, or non-numeric → return `RelevanceAssessment.NEUTRAL`
3. Compute lambda: `lambda = Math.log(2) / halfLifeDays`
4. Compute weight: `weight = baseline + (1.0 - baseline) * Math.exp(-lambda * days)`
5. Return `new RelevanceAssessment(weight, "collaboration-recency-days:" + days + ", weight:" + weight)`

**Configuration:**

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `casehub.eidos.trust.collaboration.half-life-days` | `double` | `7.0` | Exponential decay half-life |
| `casehub.eidos.trust.collaboration.baseline` | `double` | `0.5` | Minimum weight asymptote |

**Validation (constructor):**
- `halfLifeDays` must be > 0
- `baseline` must be in [0.0, 1.0]

**Name:** `"collaboration-recency"`

### CollaborationRecencyUtil

Static utility class in `io.casehub.eidos.runtime.trust`:

- `SNAPSHOT_KEY = "collaboration-recency-days"`
- `extractDays(Map<String, Object> snapshot)` → `OptionalDouble`
  (empty when absent or non-numeric)

### Decay behaviour

```
weight = baseline + (1 - baseline) × e^(-λ × days)
λ = ln(2) / halfLifeDays

With halfLifeDays=7, baseline=0.5:
  0 days:  1.00    (just collaborated)
  3 days:  0.87
  7 days:  0.75    (half-life)
 14 days:  0.63
 30 days:  0.53
 90 days:  0.50    (≈ baseline)
```

### Graceful degradation

| Condition | Behaviour |
|-----------|-----------|
| `AgentGraphQuery` not on classpath | Enricher skips → no snapshot key → NEUTRAL |
| `NoOpAgentGraphQuery` active | Empty history → no shared refs → NEUTRAL |
| No shared `externalRef` between agents | No snapshot key written → NEUTRAL |
| Snapshot key absent at factor read time | NEUTRAL |

No scenario produces an error or blocks attestation persistence.

### Module placement

Both classes in `runtime/src/main/java/io/casehub/eidos/runtime/trust/`,
alongside `CapabilityOverlapAttestationEnricher` and
`CapabilityOverlapRelevanceFactor`. No new module or dependency required —
`AgentGraphQuery` is in `eidos-api` which `runtime` already depends on.

### File inventory

**Production:**
- `runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyAttestationEnricher.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyRelevanceFactor.java`
- `runtime/src/main/java/io/casehub/eidos/runtime/trust/CollaborationRecencyUtil.java`

**Tests:**
- `runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyAttestationEnricherTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyRelevanceFactorTest.java`
- `runtime/src/test/java/io/casehub/eidos/runtime/trust/CollaborationRecencyUtilTest.java`

### Test scenarios

**Enricher tests:**
- Shared externalRef → writes correct days to snapshot
- No shared externalRef → snapshot unchanged
- AgentGraphQuery unsatisfied → snapshot unchanged
- In-progress shared task (endedAt=null) → uses startedAt, daysSince ≥ 0
- Multiple shared refs → uses most recent collaboration
- Merges into existing snapshot (preserves other keys)
- AgentGraphQuery throws → logs WARN, snapshot unchanged

**Factor tests:**
- Snapshot absent → NEUTRAL
- Snapshot key absent → NEUTRAL
- Non-numeric value → NEUTRAL
- 0 days → weight = 1.0
- Half-life days → weight ≈ baseline + (1-baseline) × 0.5
- Large days → weight ≈ baseline
- Custom half-life and baseline via constructor
- Validation: halfLifeDays ≤ 0 → exception
- Validation: baseline out of range → exception

**Round-trip test:**
- Enricher writes → factor reads → correct weight computed

### ARC42STORIES update

Add to the `Attestation Relevance Pipeline` section:

| Bean | Module | SPI | Description |
|------|--------|-----|-------------|
| `CollaborationRecencyAttestationEnricher` | `casehub-eidos` | `LedgerAttestationEnricher` | Write-time: queries `AgentGraphQuery` for shared `externalRef` task history between attestor and scored agent; writes `{"collaboration-recency-days": N}` to snapshot; fail-open when graph module absent |
| `CollaborationRecencyRelevanceFactor` | `casehub-eidos` | `AttestationRelevanceFactor` | Read-time: maps `collaboration-recency-days` via exponential decay (configurable half-life default 7 days, baseline 0.5); accuracy-biased — recent collaborators receive higher weight |

## References

- `AgentGraphQuery.java:8` — `agentHistory()` method providing task history with `externalRef`
- `AgentTask.java:12` — `externalRef` field: opaque work item reference
- `OrgDistanceAttestationEnricher.java` — sibling enricher pattern (write-time, snapshot merge, fail-open)
- `OrgProximityRelevanceFactor.java` — sibling factor pattern (read snapshot key, configurable weights)
- `CapabilityOverlapAttestationEnricher.java` — sibling enricher in runtime module (registry-based)
- `CapabilityOverlapRelevanceFactor.java` — sibling factor with baseline pattern
- `ARC42STORIES.MD:136-158` — Attestation Relevance Pipeline section
- casehubio/eidos#261 — anti-gaming closure epic (parent)
- casehubio/eidos#248 — OrgProximity implementation (precedent)
