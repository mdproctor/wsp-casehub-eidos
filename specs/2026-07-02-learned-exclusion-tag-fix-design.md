# Learned Exclusion Tag Fix — Design Spec

**Issue:** eidos#76
**Date:** 2026-07-02

## Problem

`DefaultCapabilityHealth.probe()` uses the requested capability tag for `CapabilitySpecializationStore.count()` lookups (line 72). When subsumption matching causes an agent to be matched against a tag different from its declared capability, DECLINE signals recorded under the declared name are invisible.

**Example:** Agent declares `clinical-documentation-review`. Query asks for `documentation`. Agent matched via Specialization(1). DECLINE lookup checks `documentation` — declines recorded against `clinical-documentation-review` are never found.

## Root Cause

The resolution from query tag to declared capability happens on line 56 (`findCapability()` returns the matched `AgentCapability`), but the store lookup on line 72 uses the raw query tag instead of the resolved `capability.name()`.

The same resolution logic is private to `DefaultCapabilityHealth` and duplicated in `InMemoryAgentRegistry.matchesCapability()`. The recording side (callers of `CapabilitySpecializationStore.record()`) faces the same mismatch risk — there are no production callers yet, so we're defining the contract.

## Design

Three changes, ordered by scope:

### 1. Fix probe() — use declared capability name

One-line change in `DefaultCapabilityHealth.probe()` line 72:

```java
// Before (bug — uses requested tag):
specializationStore.count(descriptor.agentId(), descriptor.tenancyId(),
    capabilityTag, context.taskDomain(), SpecializationSignal.DECLINE);

// After (fix — uses resolved declared name):
specializationStore.count(descriptor.agentId(), descriptor.tenancyId(),
    capability.name(), context.taskDomain(), SpecializationSignal.DECLINE);
```

### 2. Extract CapabilityResolver utility

New class `io.casehub.eidos.api.CapabilityResolver` in the api module (Tier 1, pure Java):

```java
public final class CapabilityResolver {
    private CapabilityResolver() {}

    public static AgentCapability resolve(List<AgentCapability> capabilities,
                                           String capabilityTag,
                                           VocabularyRegistry registry) { ... }
}
```

- Static method, no CDI. Takes a capability list, a query tag, and a VocabularyRegistry.
- Returns the best-matching `AgentCapability` or null.
- Logic: exact match first, then subsumption walk with best-depth selection (same as current `DefaultCapabilityHealth.findCapability()`).
- `DefaultCapabilityHealth` replaces private `findCapability()` with `CapabilityResolver.resolve()`.
- Recording callers (engine, when it integrates) use `CapabilityResolver.resolve()` to get the declared name before calling `store.record()`.

### 3. Clarify store SPI contract

Update `CapabilitySpecializationStore.record()` Javadoc to specify that `capabilityName` must be the agent's declared capability name (as returned by `AgentCapability.name()`), not a query/lookup term. Reference `CapabilityResolver` as the resolution mechanism for callers that have a query tag.

Similarly update `count()`, `learned()`, and `clear()` Javadoc — all take `capabilityName` and the same contract applies.

## Design Decisions

### CapabilityHealth stays read-only

PLATFORM.md convention: domain SPIs separate read from write (AgentGraphStore vs AgentGraphQuery). CapabilityHealth is domain logic. Adding `recordSignal()` would violate this convention. Recording goes through the store directly, with `CapabilityResolver` providing the correct declared name.

### No new SPI for recording

A `CapabilitySignalRecorder` SPI would wrap 3 lines of utility + store call. Engine already injects `VocabularyRegistry` (for subsumption-aware queries) and `CapabilitySpecializationStore` (for recording). The utility gives it the correct declared name. The wrapper SPI would be unnecessary complexity with no architectural benefit.

### Utility, not service

`CapabilityResolver` is a static utility, not a CDI bean. The resolution logic is pure — it takes inputs and returns a result. No state, no lifecycle, no injection needed.

## Out of Scope (captured as issues)

- **eidos#83** — Unify `InMemoryAgentRegistry.matchesCapability()` to delegate to `CapabilityResolver` (same logic, different scope — refactor)
- **eidos#84** — `AgentRegistry.find()` should return match metadata (matched capability, match degree) so callers don't need to re-resolve

## Test Plan

### Unit tests (runtime module, Mockito-based)

1. **Learned exclusion uses declared name under subsumption** — Agent declares `security-code-review` (grounded). Probe for `code-review` (parent). DECLINE signals recorded against `security-code-review`. Probe returns `Excluded(LEARNED)`.

2. **Learned exclusion invisible under wrong key** — Same setup but DECLINE signals recorded against `code-review` (the query tag). Probe returns `Ready` — the wrong key produces no match, confirming the fix works.

3. **Exact match still works unchanged** — Agent declares `code-review`. Probe for `code-review`. DECLINE signals against `code-review`. Probe returns `Excluded(LEARNED)`. (Regression guard.)

### CapabilityResolver unit tests (api module)

4. **Exact match preferred over subsumption** — List with both `code-review` and `security-code-review`. Resolve `security-code-review` returns the exact match.

5. **Subsumption match returns closest depth** — List with `code-review` (depth 2) and `security-code-review` (depth 1). Resolve `sast-review` returns `security-code-review`.

6. **Ungrounded capabilities use exact match only** — Capability without `capabilityVocabulary`. Resolve for a child term returns null.

7. **Null/empty capabilities list returns null** — Edge case guard.

### Integration test (examples module, @QuarkusTest)

8. **End-to-end: subsumption probe with learned exclusion** — Full CDI wiring. Register agent with grounded capability. Record DECLINE signals via store using `CapabilityResolver`-resolved name. Probe via subsumption query. Verify `Excluded(LEARNED)`.
