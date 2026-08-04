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
        capabilities = capabilities != null
            ? List.copyOf(capabilities.stream().filter(Objects::nonNull).toList())
            : List.of();
        AgentDescriptorValidator.validateItems("goal.capabilities", capabilities,
            AgentDescriptorValidator.MAX_CAPABILITY_NAME);
        if (capabilities.size() != capabilities.stream().distinct().count()) {
            throw new AgentValidationException("goal.capabilities",
                "duplicate capability name in goal '" + name + "'");
        }
    }
}
```

Null elements are filtered before `List.copyOf` (avoids NPE). Duplicates are rejected explicitly.

### Cross-validation — `AgentDescriptor` compact constructor

Verify every capability name in each goal's capabilities list matches a declared `AgentCapability.name()` on the same descriptor. This validation is self-contained within the record (unlike template validation which needs an external `TemplateRegistry`):

```java
// In AgentDescriptor compact constructor, after goals normalization
var capabilityNames = capabilities.stream()
    .map(AgentCapability::name).collect(Collectors.toSet());
for (var goal : goals) {
    for (var capName : goal.capabilities()) {
        if (!capabilityNames.contains(capName)) {
            throw new AgentValidationException("goals",
                "goal '" + goal.name() + "' references unknown capability '" + capName + "'");
        }
    }
}
```

Exact string match against declared capability names. Subsumption matching is not applicable — this is a structural reference within a single descriptor, not a query.

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

Both construction sites (promotion and demotion) carry `capabilities` through:

```java
// Promotion (line ~98)
new AgentGoal(g.name(), g.description(), GoalPriority.PRIMARY,
              g.visibility(), g.capabilities())
// Demotion (line ~101)
new AgentGoal(g.name(), g.description(), GoalPriority.SECONDARY,
              g.visibility(), g.capabilities())
```

### Rendering

`capabilities` is a structural engine mapping — not rendered in MARKDOWN/PROSE system prompts. In A2A_CARD, goals are already serialized as JSON objects via Jackson; the `capabilities` field appears automatically as `"capabilities": ["cap-a", "cap-b"]` on each goal node. No renderer code change needed — Jackson auto-serialization handles it.

## Files Changed

| File | Change |
|------|--------|
| `api/.../AgentGoal.java` | Add `capabilities` field, validation in compact constructor |
| `api/.../AgentDescriptor.java` | Cross-validation in compact constructor: goal capabilities vs declared capabilities |
| `api/.../AgentGoalTest.java` | Tests for new field: null→empty, null-element filtering, duplicate rejection, immutability |
| `runtime/.../ClasspathYamlDescriptorRegistrar.java` | `GoalConfig.capabilities`, `toDescriptor` mapping |
| `runtime/.../jpa/AgentGoalEntity.java` | JSON column for capabilities |
| `runtime/.../jpa/AgentDescriptorMapper.java` | toGoal/toGoalEntity with JSON round-trip |
| `runtime/.../health/DefaultGoalEvolution.java` | Pass `capabilities` through in new AgentGoal construction |
| `runtime/.../db/eidos/migration/V8__goals_constraints.sql` | Add `capabilities TEXT` column |
| Tests | YAML round-trip, cross-validation failure, JPA round-trip, evolution pass-through, AgentDescriptor cross-validation |

## Review Findings Addressed

- **Validation placement:** moved cross-validation from `DescriptorCollector` to `AgentDescriptor` compact constructor (self-contained, no external registry needed)
- **Goal evolution construction sites:** both promotion and demotion paths documented
- **A2A_CARD rendering:** Jackson auto-serialization — no explicit renderer change needed
- **Duplicate capability names:** rejected in `AgentGoal` compact constructor
- **Null elements:** filtered before `List.copyOf`
- **Empty-list semantics:** intentionally cross-cutting (affected by any capability failure). Distinct from "not configured" — all goals start empty; authors add mappings when discrimination is needed
