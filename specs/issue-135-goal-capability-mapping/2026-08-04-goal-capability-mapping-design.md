# Goal-Capability Mapping — Design Spec

**Issue:** casehubio/eidos#135
**Downstream:** casehubio/engine#860 (GoalFailureRecorder per-goal discrimination)
**Date:** 2026-08-04

## Problem

engine#807 (goal abandonment) records DECLINE signals for ALL agent goals on each worker decline. This makes goal abandonment equivalent to agent-level exclusion — no per-goal discrimination. engine#860 needs a goal-to-capability mapping so the engine can filter goals by the capability that actually failed.

## Design

Add `List<String> capabilities` to `AgentGoal`. Each string references a declared `AgentCapability.name()` on the same descriptor.

### Semantics

- **Non-empty list:** goal is served by exactly those capabilities. Engine records FAILURE only when one of the listed capabilities fails.
- **Empty list:** cross-cutting goal — affected by any capability failure.
- **Null:** normalized to empty in compact constructor (same pattern as `AgentDescriptor.capabilities`).

### API — `AgentGoal` record

```java
public record AgentGoal(
        String name,
        String description,
        GoalPriority priority,
        Visibility visibility,
        List<String> capabilities
) {
    public AgentGoal {
        // existing validation unchanged
        capabilities = capabilities != null ? List.copyOf(capabilities) : List.of();
        AgentDescriptorValidator.validateItems("goal.capabilities", capabilities,
            AgentDescriptorValidator.MAX_CAPABILITY_NAME);
    }
}
```

### Registration-time cross-validation — `DescriptorCollector`

After existing template validation, verify every capability name in each goal's list matches a declared capability on the same descriptor:

```java
var capabilityNames = d.capabilities().stream()
    .map(AgentCapability::name).collect(Collectors.toSet());
for (var goal : d.goals()) {
    for (var capName : goal.capabilities()) {
        if (!capabilityNames.contains(capName)) {
            throw new IllegalStateException("Descriptor '" + d.agentId()
                + "', goal '" + goal.name()
                + "': references unknown capability '" + capName + "'");
        }
    }
}
```

Fails fast at startup. Same pattern as template ref validation.

### YAML — `ClasspathYamlDescriptorRegistrar`

`GoalConfig` gains `public List<String> capabilities`.

`toDescriptor` passes it through:

```java
new AgentGoal(g.name, g.description, g.priority, g.visibility, g.capabilities)
```

Example YAML:

```yaml
goals:
  - name: find-diamond
    description: Find the Doily Diamond
    priority: PRIMARY
    visibility: PUBLIC
    capabilities:
      - investigation
      - evidence-analysis
```

### JPA — entity, mapper, schema

**`AgentGoalEntity`:** add `@Column(name = "capabilities") String capabilities` — JSON-serialized list.

**`AgentDescriptorMapper`:**

```java
// toGoal
new AgentGoal(g.name, g.description, GoalPriority.valueOf(g.priority),
              Visibility.valueOf(g.visibility),
              readJson(g.capabilities, new TypeReference<List<String>>() {}));

// toGoalEntity
e.capabilities = writeJson(g.capabilities());
```

**Schema — `V8__goals_constraints.sql`** (no deployed instances, modify in place):

Add column to `agent_goal`:

```sql
capabilities   TEXT,
```

### Goal evolution — `DefaultGoalEvolution`

`AgentGoal` construction during promotion/demotion carries `capabilities` through:

```java
new AgentGoal(g.name(), g.description(), GoalPriority.PRIMARY,
              g.visibility(), g.capabilities())
```

### Rendering

`capabilities` is a structural engine mapping — not rendered in MARKDOWN/PROSE system prompts. Surfaced in A2A_CARD for machine consumers.

## Files Changed

| File | Change |
|------|--------|
| `api/.../AgentGoal.java` | Add `capabilities` field, validation in compact constructor |
| `api/.../AgentGoalTest.java` | Tests for new field: null→empty, validation, immutability |
| `runtime/.../DescriptorCollector.java` | Cross-validation: goal capabilities vs declared capabilities |
| `runtime/.../ClasspathYamlDescriptorRegistrar.java` | `GoalConfig.capabilities`, `toDescriptor` mapping |
| `runtime/.../jpa/AgentGoalEntity.java` | JSON column for capabilities |
| `runtime/.../jpa/AgentDescriptorMapper.java` | toGoal/toGoalEntity with JSON round-trip |
| `runtime/.../health/DefaultGoalEvolution.java` | Pass `capabilities` through in new AgentGoal construction |
| `runtime/.../db/eidos/migration/V8__goals_constraints.sql` | Add `capabilities TEXT` column |
| `runtime/.../renderer/EidosSystemPromptRenderer.java` | Surface in A2A_CARD only |
| Tests | YAML round-trip, cross-validation failure, JPA round-trip, evolution pass-through |
