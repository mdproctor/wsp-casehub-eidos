# Design: AgentRegistry.find() returns match metadata

**Issue:** casehubio/eidos#84
**Date:** 2026-07-03
**Status:** Approved

## Problem

Match metadata is computed and discarded at three layers:

1. `CapabilityResolver.resolve()` iterates capabilities, computes `MatchDegree` for each, selects the best, returns only `AgentCapability` — degree discarded.
2. `AgentRegistry.find()` uses matching as a boolean filter — InMemory checks `not None`, JPA uses vocabulary expansion in JPQL. Both discard why each agent matched.
3. Callers re-derive what the registry already knew — engine's `AgentCandidateFactory` calls `CapabilityResolver.resolve()` again to log which capability matched; `BehavioralSignalStore.record()` callers must call `resolve()` to obtain the declared capability name.

## Design

### New types (api/, Tier 1)

**`ResolvedCapability`** — result of `CapabilityResolver.resolve()`:

```java
public record ResolvedCapability(AgentCapability capability, MatchDegree degree) {}
```

**`AgentMatch`** — result of `AgentRegistry.find()`:

```java
public record AgentMatch(AgentDescriptor descriptor, ResolvedCapability resolvedCapability) {}
```

`resolvedCapability` is null when the query has no `capabilityName` (slot-only or `all()` queries). Both fields are non-null when `query.capabilityName() != null`.

### CapabilityResolver changes

`resolve()` return type changes from `AgentCapability` to `ResolvedCapability`:

```java
public static ResolvedCapability resolve(
    List<AgentCapability> capabilities, String capabilityTag, VocabularyRegistry registry)
```

Returns null when no match (same contract as today). The internal loop already computes `MatchDegree` — it now carries it in the return value instead of discarding it.

`match()` is unchanged — it already returns `MatchDegree`.

### MatchDegree becomes Comparable

`MatchDegree` extends `Comparable<MatchDegree>`. Ordering reflects OWLS-MX priority — this is definitional, not a policy choice:

- `Exact` (best)
- `Plugin(depth)` — lower depth is better; Plugin at any depth beats Specialization
- `Specialization(depth)` — lower depth is better
- `None` (worst)

### AgentRegistry.find() changes

Return type changes from `List<AgentDescriptor>` to `List<AgentMatch>`:

```java
List<AgentMatch> find(AgentQuery query);
```

When `query.capabilityName()` is non-null, results are ordered by match quality (best first). Agents at the same match degree are in unspecified relative order. When no capability in the query, ordering is unspecified.

Same change for `ReactiveAgentRegistry`:

```java
Uni<List<AgentMatch>> find(AgentQuery query);
```

### Implementation changes

**InMemoryAgentRegistry:** Replace boolean `matchesCapability()` with `CapabilityResolver.resolve()` per agent. Capture `ResolvedCapability`, build `AgentMatch`, sort by degree.

**JpaAgentRegistry:** JPQL vocabulary-expansion query stays as-is (DB-level filtering is efficient). Post-process results: for each returned entity, call `CapabilityResolver.resolve()` to compute match metadata. Build `AgentMatch`, sort by degree. The post-processing cost is negligible — result sets are typically single-digit, vocabulary lookups are in-memory.

**JpaReactiveAgentRegistry:** Same post-processing in reactive pipeline (`.map()` stage).

**InMemoryReactiveAgentRegistry:** Delegates to `InMemoryAgentRegistry` — automatic.

**DefaultCapabilityHealth.probe():** Extract `.capability()` from the `ResolvedCapability` returned by `resolve()`. Probe logic is otherwise unchanged — it doesn't need the degree for its own decisions.

### Test impact

42 callers of `find()` across eidos tests. All are mechanical updates: extract `.descriptor()` from `AgentMatch` or use the new match fields. The subsumption scenario test (`multiple_agents_ranked_by_match_degree`) can now assert ordering instead of just presence.

### Out of scope

- Engine `AgentCandidateFactory` — benefits from `ResolvedCapability` but is engine#632's concern.
- Adding match degree to `AgentCandidate` — engine-side change.
- Ranking strategies beyond OWLS-MX default ordering — dispatch-specific ranking is the engine's domain.

## Platform coherence

- `ResolvedCapability` and `AgentMatch` are pure-Java records in `api/` (Tier 1) — no JPA or Quarkus deps.
- Naming follows platform convention: domain-concept result types (`WorkerResult`, `DispatchResult`, etc.).
- Blocking/reactive parity maintained: both `AgentRegistry` and `ReactiveAgentRegistry` updated.
- `MatchDegree` is already documented as OWLS-MX based — `Comparable` is intrinsic to the type.
