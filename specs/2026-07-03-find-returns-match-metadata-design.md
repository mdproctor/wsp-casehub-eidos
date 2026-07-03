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

`resolvedCapability` is null when the query has no `capabilityName` (slot-only or `all()` queries). Non-null for all capability-bearing queries, including combined slot-and-capability. The nesting (vs issue #84's flat shape) exists because `ResolvedCapability` serves double duty as `resolve()`'s return type — a flat `AgentMatch` would require a one-off pair for resolve's return or duplicate fields.

### CapabilityResolver changes

`resolve()` return type changes from `AgentCapability` to `ResolvedCapability`:

```java
public static ResolvedCapability resolve(
    List<AgentCapability> capabilities, String capabilityTag, VocabularyRegistry registry)
```

Returns null when no match (same contract as today). The selection loop changes from tracking raw `int bestDepth` to tracking `MatchDegree bestDegree` and using `compareTo()`. This fixes the existing bug where Plugin and Specialization compete on the same depth variable — a Specialization(depth=1) incorrectly beats Plugin(depth=2) under the current logic. With Comparable-based selection, the instanceof cascade for Plugin/Specialization collapses to a single `compareTo()` check.

`match()` is unchanged — it already returns `MatchDegree`.

### MatchDegree becomes Comparable

`MatchDegree` extends `Comparable<MatchDegree>`. Ordering reflects OWLS-MX priority — this is definitional, not a policy choice:

- `Exact` (best)
- `Plugin(depth)` — lower depth is better; Plugin at any depth beats Specialization
- `Specialization(depth)` — lower depth is better
- `None` (worst)

Plugin ranks above Specialization because a Plugin match means the agent's declared capability subsumes the request — the agent is guaranteed to cover it. A Specialization match means the agent's capability is narrower than the request — it covers only a subset. This is standard OWLS-MX semantics, not a policy choice.

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

**InMemoryAgentRegistry:** Replace boolean `matchesCapability()` with `CapabilityResolver.resolve()` per agent. Capture `ResolvedCapability`, build `AgentMatch`, sort by degree. When `VocabularyRegistry` is unavailable (`Instance.isResolvable()` is false), fall back to exact name comparison and wrap matches as `ResolvedCapability(capability, new MatchDegree.Exact())`.

**JpaAgentRegistry:** JPQL vocabulary-expansion query stays as-is (DB-level filtering is efficient). Post-process results: for each returned entity, call `CapabilityResolver.resolve()` to compute match metadata. Build `AgentMatch`, sort by degree. The post-processing cost is negligible — result sets are typically single-digit, vocabulary lookups are in-memory.

**JpaReactiveAgentRegistry:** Same post-processing in reactive pipeline (`.map()` stage).

**InMemoryReactiveAgentRegistry:** Delegates to `InMemoryAgentRegistry` — automatic.

**DefaultCapabilityHealth.probe():** Extract `.capability()` from the `ResolvedCapability` returned by `resolve()`. Probe logic is otherwise unchanged — it doesn't need the degree for its own decisions.

**DefaultReactiveCapabilityHealth:** Pure delegation to `DefaultCapabilityHealth` — change propagates automatically. No code changes needed.

### Test impact

42 callers of `find()` across eidos tests. All are mechanical updates: extract `.descriptor()` from `AgentMatch` or use the new match fields. The subsumption scenario test (`multiple_agents_ranked_by_match_degree`) can now assert ordering instead of just presence.

### Out of scope

- Engine `AgentCandidateFactory` — benefits from `ResolvedCapability` but is engine#632's concern.
- Adding match degree to `AgentCandidate` — engine-side change (engine#638).
- Ranking strategies beyond OWLS-MX default ordering — dispatch-specific ranking is the engine's domain (engine#639).

## Platform coherence

- `ResolvedCapability` and `AgentMatch` are pure-Java records in `api/` (Tier 1) — no JPA or Quarkus deps.
- Naming follows platform convention: domain-concept result types (`WorkerResult`, `DispatchResult`, etc.).
- Blocking/reactive parity maintained: both `AgentRegistry` and `ReactiveAgentRegistry` updated.
- `MatchDegree` is already documented as OWLS-MX based — `Comparable` is intrinsic to the type.
