# Agent Descriptor Content Comparator

**Issue:** casehubio/eidos#60
**Date:** 2026-06-25
**Revised:** 2026-06-25 (R3)

## Purpose

Full field-by-field agent drift detection for casehub-ops' desiredstate reconciliation.
The existing `AgentDriftChecker` in ops compares sorted capability names only — it misses
drift in disposition, vocabularies, briefing, provider/model fields, and capability
sub-fields (qualityHint, epistemicDomains, excludedDomains, etc.).

This is a cross-repo change:
- **casehub-eidos-api** gains `AgentDescriptorComparator` — pure Java utility defining
  content equality for `AgentDescriptor` (all fields except identity keys `agentId` and
  `tenancyId`). Type owner defines content equality — follows the `AgentDescriptorValidator`
  precedent.
- **casehub-ops** gains `AgentNodeSpec.toDescriptor(tenancyId)` and an enriched
  `AgentDriftChecker` that calls the comparator.

No new Maven module. No CDI beans. No Flyway.

## Why not a bridge module

`AgentNodeSpec` already imports `AgentCapability`, `AgentDisposition`, and `DispositionAxis`
from `io.casehub.eidos.api`. The ops `AgentDriftChecker` already injects `AgentRegistry`.
The dependency path `ops-api → eidos-api` exists at compile time.

A bridge module would require CDI displacement to replace the existing checker.
`@DefaultBean` is incompatible with `Instance<NodeDriftChecker>` multi-bean injection
(suppressed by other implementations of the same interface regardless of nodeType).
`@Alternative @Priority` produces non-deterministic map collision in
`DeploymentActualStateAdapter`'s `HashMap<nodeType, checker>`. Both approaches are
fundamentally broken for this SPI's discovery pattern.

The comparison uses identical types on both sides. Co-evolution doesn't justify a separate
module — `AgentNodeSpec` already requires a cross-repo update when `AgentDescriptor` adds
a field. The comparator update rides the same PR.

## eidos-api: AgentDescriptorComparator

New class in `io.casehub.eidos.api`. Pure Java, zero CDI, zero framework deps.

### Signature

```java
public static ComparisonResult compare(AgentDescriptor desired, AgentDescriptor actual)
```

### Inner records

```java
public record ComparisonResult(List<FieldDrift> drifts) {
    public boolean matches() { return drifts.isEmpty(); }
}

public record FieldDrift(String field, String desiredValue, String actualValue) {}
```

`matches()` is derived from `drifts.isEmpty()` — no separate boolean field.

### Fields compared (16 of 18 record components)

Identity layer:
- `name`, `slot`, `version`, `provider`, `modelFamily`, `modelVersion`, `weightsFingerprint`

Vocabulary layer:
- `domainVocabulary`, `slotVocabulary`, `dispositionVocabulary`, `axisVocabularies`

Policy layer:
- `jurisdiction`, `dataHandlingPolicy`, `briefing`

Disposition layer:
- `socialOrient`, `ruleFollowing`, `riskAppetite`, `autonomy`, `conflictMode`, `delegation`

Capabilities (order-independent deep comparison):
- Match by `name`, then compare: `qualityHint`, `latencyHintP50Ms`, `costHint`,
  `inputTypes`, `outputTypes`, `tags`, `epistemicDomains`, `excludedDomains`
- Missing capabilities in either direction reported as drift

### Skipped fields (2 of 18)

- `agentId` — identity key, not content
- `tenancyId` — scoping key, not content

### Structural coverage constants

```java
static final int COMPARED_FIELD_COUNT = 16;              // AgentDescriptor: 18 - agentId - tenancyId
static final int COMPARED_CAPABILITY_FIELD_COUNT = 8;    // AgentCapability: 9 - name (match key)
static final int COMPARED_DISPOSITION_FIELD_COUNT = 6;   // AgentDisposition: all components
```

Anchor the reflection-based sync tests (see Testing section). One constant per compared
type — catches silent field additions at every level.

### Field path convention

| Case | Path format | Example |
|------|-------------|---------|
| Simple field | `<field>` | `slot`, `briefing`, `name` |
| Disposition axis | `disposition.<axis>` | `disposition.socialOrient` |
| Disposition delegation | `disposition.delegation` | |
| Map entry | `<field>[<key>]` | `axisVocabularies[SOCIAL_ORIENTATION]` |
| Capability present/absent | `capabilities[<name>]` | desiredValue: `"(present)"`, actualValue: `"(absent)"` |
| Capability sub-field | `capabilities[<name>].<field>` | `capabilities[code-review].qualityHint` |

**Collection comparison granularity:** `inputTypes`, `outputTypes`, `tags`,
`epistemicDomains`, `excludedDomains` are compared as wholes, not element-by-element.
The `desiredValue`/`actualValue` strings use `toString()` representation:
- `capabilities[code-review].epistemicDomains` → desired: `"{java=0.95, rust=0.42}"`, actual: `"{java=0.90}"`

## casehub-ops: Changes

### AgentNodeSpec.toDescriptor(String tenancyId)

New method on the `AgentNodeSpec` record in `casehub-ops-api`:

```java
public AgentDescriptor toDescriptor(String tenancyId) {
    return new AgentDescriptor(
            agentId, name, version, provider, modelFamily, modelVersion,
            weightsFingerprint, domainVocabulary, slotVocabulary,
            dispositionVocabulary, axisVocabularies, slot, capabilities,
            disposition, jurisdiction, dataHandlingPolicy, tenancyId, briefing);
}
```

Eliminates the duplicated 18-field mapping currently in `AgentProvisionHandler.provision()`.
After adding `toDescriptor`, `AgentProvisionHandler` becomes:

```java
AgentDescriptor descriptor = spec.toDescriptor(context.tenancyId());
agentRegistry.register(descriptor);
```

When `AgentDescriptor` adds a 19th field, `toDescriptor` fails to compile — one fix point
instead of two.

### Enriched AgentDriftChecker

Existing `AgentDriftChecker` in ops `deployment/` module. No annotation changes — stays
`@ApplicationScoped`. No CDI displacement needed.

**Revised `check()` logic:**
1. Cast to `AgentNodeSpec` — return `UNKNOWN` if wrong type (existing behaviour)
2. `registry.findById(agentSpec.agentId(), tenancyId)` — return `ABSENT` if empty
3. `AgentDescriptor desired = agentSpec.toDescriptor(tenancyId)`
4. `ComparisonResult result = AgentDescriptorComparator.compare(desired, actual)`
5. If `!result.matches()`: log each `FieldDrift` at DEBUG, return `DRIFTED`
6. If `result.matches()`: return `PRESENT`

**DEBUG log format:**
```
agent <agentId>: <field> drifted [<desiredValue> → <actualValue>]
```

## Testing

### In casehub-eidos (api module)

**AgentDescriptorComparatorTest** — pure unit tests, no CDI:

- Identical descriptors → `matches() == true`, `drifts` empty
- Each field category drifted independently (identity, vocabulary, policy, disposition)
- Capability drift: added, removed, sub-field changed, order-independent
- Nested drift: epistemicDomains map differs, excludedDomains set differs
- Null handling: null vs non-null in both directions (null-safe equals)
- Multiple simultaneous drifts → all reported in drifts list
- Field path format verified (path-style convention)

**Structural sync tests** — one per compared type:
```java
@Test
void comparatorCoversAllDescriptorComponents() {
    int total = AgentDescriptor.class.getRecordComponents().length;
    int skipped = 2; // agentId, tenancyId
    assertEquals(total - skipped, AgentDescriptorComparator.COMPARED_FIELD_COUNT);
}

@Test
void comparatorCoversAllCapabilityComponents() {
    int total = AgentCapability.class.getRecordComponents().length;
    int matchKey = 1; // name
    assertEquals(total - matchKey, AgentDescriptorComparator.COMPARED_CAPABILITY_FIELD_COUNT);
}

@Test
void comparatorCoversAllDispositionComponents() {
    assertEquals(
        AgentDisposition.class.getRecordComponents().length,
        AgentDescriptorComparator.COMPARED_DISPOSITION_FIELD_COUNT);
}
```

Catches silent field drift at every compared type boundary.

### In casehub-ops (deployment module)

Enhanced `AgentDriftCheckerTest` — existing test class gains new cases:

- Agent registered, all fields match → PRESENT
- Agent registered, non-capability field drifted (e.g. disposition, vocabulary) → DRIFTED
- Agent registered, capability sub-field drifted (e.g. qualityHint changed) → DRIFTED
- Verify DEBUG log output contains field path and values

`AgentProvisionHandler` tests updated to use `toDescriptor()`.

## Architectural note — tenancyId in AgentDescriptor

`tenancyId` is arguably a scoping key (WHERE the agent is deployed), not an identity
attribute (WHAT the agent is). Extracting it from `AgentDescriptor` into the registry/query
layer would enable `AgentNodeSpec` to compose `AgentDescriptor` directly (eliminating field
duplication) and simplify content equality to `descriptor.equals(registered)`.

This is a platform-wide rearchitect — filed as a separate issue, not bundled here. The
`toDescriptor(tenancyId)` approach works within current constraints.

## What this is not

- Not a reconciliation engine — reports drift, does not fix it
- Not a new module — no new Maven artifact, no CDI beans
- Not ops-aware — `providerConfigs` and future ops-domain fields are outside scope
- Not configurable — log verbosity controlled via standard Quarkus log category
  (`quarkus.log.category."io.casehub.ops.deployment.drift".level=DEBUG`)
