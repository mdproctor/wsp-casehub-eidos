# Design: AgentCapability description field (#77) + Degradation/recovery example (#79)

**Date:** 2026-07-04
**Branch:** issue-77-capability-desc-degradation-example
**Covers:** #77, #79

---

## Part 1 — AgentCapability description field (#77)

### Change

Add `String description` to `AgentCapability` — a nullable, human/LLM-readable description of what the capability does. Placed as the second record component (after `name`, before `capabilityVocabulary`) since it is semantically closest to `name`.

### Record signature

```java
public record AgentCapability(
    String name,
    String description,          // NEW
    String capabilityVocabulary,
    Double qualityHint,
    Long latencyHintP50Ms,
    String costHint,
    List<String> inputTypes,
    List<String> outputTypes,
    List<String> tags,
    Map<String, Double> epistemicDomains,
    Set<String> excludedDomains
)
```

Validated with `AgentDescriptorValidator.validateOptional("description", description, 500)`.

Builder: add `.description(String)` method. Update `build()` to pass through.

### Persistence

**JPA entity** (`AgentCapabilityEntity`): add `@Column(columnDefinition = "TEXT") String description`.

**Flyway migration** (`V6__capability_description.sql`):
```sql
ALTER TABLE agent_capability ADD COLUMN description TEXT;
```

**Mapper** (`AgentDescriptorMapper`): pass `description` through in both `toCapability` and `toCapabilityEntity`.

### YAML registrar

`CapabilityConfig`: add `public String description`.

`toDescriptor`: pass `description` through when constructing `AgentCapability` (the direct constructor call on line 97 gains the new argument in position 2).

### Rendering

**Protocol PP-20260611-228599 compliance:** `description` is human-readable text, not a numeric routing signal. It renders in all formats — same treatment as `inputTypes`/`outputTypes`.

**LLM payload** (`buildDescriptorPayload`): include `description` in the capability node for all formats (not gated behind `format == A2A_CARD`).

**MARKDOWN** (`assembleMarkdownCapabilities`): after `**name**`, append ` — description` when description is non-null.

**PROSE** (`assembleProse`): for each capability, append description in parentheses when present; capabilities without a description render name only. E.g. `"code-review (reviews code for quality and correctness), test-writing."`.

**A2A_CARD** (`assembleA2aCard`): declared `description` is the fallback. LLM-enriched `A2AEnrichment.CapabilityNarrative.description()` wins when available (current behaviour — enriched description already writes to the `description` field in the JSON). When no enrichment exists but declared description is present, use the declared description.

### Test updates

All call sites constructing `AgentCapability` directly (not via builder) need the new second argument. These include:

- `AgentCapabilityTest` — unit tests for validation
- `AgentDescriptorMapper.toCapability()` — adds `c.description`
- `ClasspathYamlDescriptorRegistrar.toDescriptor()` — adds `c.description`
- Example tests that use the direct constructor (most use the builder — check each)
- Eval YAML profiles — add optional `description` fields where appropriate

New tests:
- Validate `description` rejects strings > 500 chars
- Validate `description` passes through builder
- Verify MARKDOWN/PROSE/A2A_CARD rendering includes description when present
- Verify A2A_CARD prefers enriched description over declared

---

## Part 2 — Degradation and recovery example (#79)

### What

A `@QuarkusTest` in `examples/agent-scenarios` demonstrating the `AgentStateStore` TTL lifecycle — how an agent transitions between Ready and Degraded states and recovers.

### Design

The test injects the SPI interfaces (`AgentStateStore`, `CapabilityHealth`, `AgentRegistry`) — never a concrete store implementation. Whether the backing store is in-memory or JPA is a deployment configuration concern. In the test profile, `persistence-memory` provides the `@Alternative @Priority(1)` in-memory implementation.

### Scenario

```
DegradationAndRecoveryTest

Setup:
  Register one agent ("worker-1") with capability "data-processing"

Test 1 — probe_returns_ready_when_no_degradation:
  probe("data-processing") → Ready

Test 2 — probe_returns_degraded_after_recording:
  stateStore.record("worker-1", tenancyId, RATE_LIMITED, futureExpiry)
  probe("data-processing") → Degraded(RATE_LIMITED, ...)

Test 3 — probe_returns_ready_after_clear:
  stateStore.record("worker-1", tenancyId, RATE_LIMITED, futureExpiry)
  probe → Degraded
  stateStore.clear("worker-1", tenancyId)
  probe → Ready

Test 4 — probe_returns_ready_after_ttl_expires:
  stateStore.record("worker-1", tenancyId, CONTEXT_EXHAUSTED, pastExpiry)
  probe → Ready  (TTL already expired, store returns empty)

Test 5 — degradation_reasons_are_distinguishable:
  Record with OVERLOADED, probe → Degraded(OVERLOADED, ...)
  Clear, record with DOMAIN_MISMATCH, probe → Degraded(DOMAIN_MISMATCH, ...)
```

### Pattern

Follows existing examples (`MultiAgentTeamTest`): `@BeforeEach` registers agent(s), individual `@Test` methods demonstrate specific behaviours. Assertions use AssertJ pattern matching on the sealed `CapabilityStatus` hierarchy.

---

## Out of scope

- Changes to `ReactiveAgentStateStore` or reactive probe path (parity maintained but no new reactive-specific tests in examples)
- New eval profiles for description (description is optional — existing profiles continue to work with null)
- A2A hash coverage update for description (description is enrichment-derived in A2A path, not a direct descriptor field read — protocol PP a2a-structural-assembly-hash-coverage does not apply)
