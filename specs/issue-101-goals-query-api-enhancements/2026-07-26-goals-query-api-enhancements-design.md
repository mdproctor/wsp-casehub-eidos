# Goals Query, API Enhancements, and Template Examples

**Date:** 2026-07-26
**Branch:** issue-101-goals-query-api-enhancements
**Covers:** #101, #102, #103, #105

## Overview

Four issues on one branch delivering: capability name uniqueness validation (#102), constraint severity (#103), goal-based querying and A2A awareness (#101), and descriptor template composition examples (#105). Plus a Flyway V7 conflict fix on main.

## Flyway V7 Conflict Fix

Two migration files on main share the V7 version number:
- `V7__descriptor_templates.sql`
- `V7__goals_constraints.sql`

**Fix:** Rename `V7__goals_constraints.sql` → `V8__goals_constraints.sql`. No installations exist — no checksum issues.

## #102 — AgentCapability Name Uniqueness

### Problem

Goals and constraints validate name uniqueness in `AgentDescriptor`'s compact constructor. Capabilities do not. `AgentDescriptorComparator.compareCapabilities()` uses `Collectors.toMap(AgentCapability::name, c -> c)` which throws `IllegalStateException` on duplicates — so duplicate capabilities silently register but crash at comparison time. The `agent_capability` DDL also lacks a `UNIQUE (descriptor_id, name)` constraint.

### Changes

**`AgentDescriptor` compact constructor** — add duplicate capability name check after the existing goals/constraints checks. Same pattern:

```java
if (capabilities.size() > 1) {
    long distinctNames = capabilities.stream().map(AgentCapability::name).distinct().count();
    if (distinctNames < capabilities.size()) {
        String dup = capabilities.stream().map(AgentCapability::name)
            .collect(Collectors.groupingBy(n -> n, Collectors.counting()))
            .entrySet().stream().filter(e -> e.getValue() > 1)
            .map(Map.Entry::getKey).findFirst().orElse("?");
        throw new AgentValidationException("capabilities", "duplicate capability name: " + dup);
    }
}
```

**Flyway V9** — `UNIQUE (descriptor_id, name)` constraint on `agent_capability`.

**Tests** — unit test in `api/` confirming validation throws on duplicate capability names.

## #103 — ConstraintSeverity

### Problem

`AgentConstraint` has no severity field. Hard constraints ("never reveal identity") and soft constraints ("prefer elaborate schemes") are structurally indistinguishable.

### New type

**`ConstraintSeverity`** — enum in `io.casehub.eidos.api`:

```java
public enum ConstraintSeverity { HARD, SOFT }
```

### AgentConstraint changes

Add `severity` as the fourth field:

```java
public record AgentConstraint(
    String name,
    String description,
    Visibility visibility,
    ConstraintSeverity severity
)
```

Compact constructor adds: `Objects.requireNonNull(severity, "constraint.severity must not be null")`.

### Comparator

`AgentDescriptorComparator` — add `severity` to constraint field comparison. Increment `CONSTRAINT_FIELDS` from 2 to 3.

### Renderer

**MARKDOWN/PROSE** — severity-discriminated language:
- `HARD` → "You MUST..." / "You must never..."
- `SOFT` → "You should..." / "Prefer..."

**A2A_CARD** — include `severity` field on each constraint object in the JSON structure.

### JPA

`severity VARCHAR(20) NOT NULL` column on `agent_constraint` in Flyway V9:

```sql
ALTER TABLE agent_constraint ADD COLUMN severity VARCHAR(20) NOT NULL DEFAULT 'HARD';
ALTER TABLE agent_constraint ALTER COLUMN severity DROP DEFAULT;
```

### YAML descriptors

Add `severity` field to the constraint schema in `ClasspathYamlDescriptorRegistrar`.

### Builder

No change — `AgentDescriptor.Builder.constraints()` takes `List<AgentConstraint>`. Severity is a constructor parameter on the record.

### Test impact

All test code constructing `AgentConstraint` must include the new `severity` parameter.

## #101 — Goal-Based Querying and Awareness

### Platform boundary analysis

Eidos owns structured identity and data access. Engine owns dispatch decisions and orchestration.

The five use cases from the #100 design spec map as follows:

| Use case | Owner | This branch |
|----------|-------|-------------|
| Goal-based querying | Eidos | Yes — `goalName` on `AgentQuery` |
| `hasGoal()` / `hasConstraint()` | Eidos | Yes — convenience methods |
| Cross-agent goal awareness | Eidos (renderer) | Yes — public goals/constraints in A2A_CARD |
| Goal-based routing | Engine | No — file engine issue |
| Goal-based termination | Engine | No — file engine issue |

Routing strategies and completion evaluation are engine concerns. Eidos provides the query primitives and identity data; engine makes dispatch decisions.

### 101a. `goalName` on `AgentQuery`

Add optional `goalName` field:

```java
public record AgentQuery(
    String slot,
    String capabilityName,
    String tenancyId,
    String taskDomain,
    String goalName
)
```

**Factory methods:**
- `byGoal(String goalName, String tenancyId)` — find agents with a specific declared goal
- Existing factories unchanged — `goalName` defaults to null

**JpaAgentRegistry.find():**
- When `goalName` non-null: `LEFT JOIN FETCH a.goals g` + `AND g.name = :goalName`
- No vocabulary subsumption — goals are identity-level standing objectives, not vocabulary-grounded terms. Exact name match only.

**InMemoryAgentRegistry:**
- Stream filter on `descriptor.goals()` matching by name

**Reactive registries:**
- Mirror blocking implementations

**AgentMatch:**
- When querying by `goalName`, `resolvedCapability` stays null. Goals are not capabilities — no match degree ranking. The query is a filter, not a ranked match.

### 101b. Convenience methods on `AgentDescriptor`

```java
public boolean hasGoal(String name) {
    return goals.stream().anyMatch(g -> g.name().equals(name));
}

public boolean hasConstraint(String name) {
    return constraints.stream().anyMatch(c -> c.name().equals(name));
}
```

### 101c. Public goals and constraints in A2A_CARD

Currently goals and constraints render in MARKDOWN/PROSE only. A2A_CARD should include them for machine-to-machine negotiation.

**A2A_CARD JSON structure additions:**

```json
{
  "goals": [
    {"name": "quality-review", "description": "...", "priority": "PRIMARY"}
  ],
  "constraints": [
    {"name": "no-pii-exposure", "description": "...", "severity": "HARD"}
  ]
}
```

Rules:
- Only `Visibility.PUBLIC` items — use existing `publicGoals()` and `publicConstraints()`
- `visibility` field excluded from the card (everything in the card is public by definition)
- Goals sorted by priority then name; constraints sorted alphabetically (same as MARKDOWN/PROSE)
- Add goals and constraints to the A2A_CARD hash payload (per `a2a-structural-assembly-hash-coverage` protocol)

### 101d. Engine follow-on issues

Create two issues on `casehubio/engine` after this branch lands:

1. **Goal-based routing** — engine routing strategies can use `descriptor.goals()` and `AgentQuery.byGoal()` for goal-aware agent dispatch
2. **Goal-based termination** — engine completion evaluation can compare agent standing goals against task outcomes using `GoalExpression`/`GoalBasedCompletion`

## #105 — Descriptor Template Examples

`DescriptorTemplateScenarioTest` already has 7 tests covering registration, parameters, substitution, format discrimination, and A2A exclusion. Three new scenarios:

### 105a. Multi-template ordering

Two templates with distinct content. A descriptor references them in a specific order. Verify rendered MARKDOWN/PROSE contains template A's content before template B's content — proving order in `List<TemplateRef>` is preserved in output.

### 105b. Shared templates across agents

Two agents reference the same template (`communication-style`) with different `args`:
- Agent A: `formality=professional, feedback_approach=collaborative`
- Agent B: `formality=casual, feedback_approach=direct`

Verify each agent's rendered output contains correctly substituted values with no cross-contamination.

### 105c. YAML-based template registration

A template defined in `META-INF/eidos/templates.yaml` (classpath registrar). Verify it resolves from `TemplateRegistry` and renders correctly when referenced by a descriptor. May use existing YAML templates if they already cover this path.

## Flyway Migration Plan

| Version | File | Content |
|---------|------|---------|
| V1–V6 | Unchanged | — |
| V7 | `V7__descriptor_templates.sql` | Unchanged content |
| V8 | `V8__goals_constraints.sql` | Renamed from V7, unchanged content |
| V9 | `V9__capability_uniqueness_constraint_severity.sql` | Capability UNIQUE constraint + constraint severity column |

V9 content:

```sql
ALTER TABLE agent_capability ADD CONSTRAINT uq_capability_name
    UNIQUE (descriptor_id, name);

ALTER TABLE agent_constraint ADD COLUMN severity VARCHAR(20) NOT NULL DEFAULT 'HARD';
ALTER TABLE agent_constraint ALTER COLUMN severity DROP DEFAULT;
```

## Files Touched

### api/
- `ConstraintSeverity.java` — new enum
- `AgentConstraint.java` — add `severity` field
- `AgentDescriptor.java` — capability name uniqueness check; `hasGoal()`; `hasConstraint()`
- `AgentQuery.java` — add `goalName` field + factory methods
- `AgentDescriptorComparator.java` — severity in constraint comparison
- `AgentDescriptorValidator.java` — no changes expected

### runtime/
- `JpaAgentRegistry.java` — goal-based query filter
- `JpaReactiveAgentRegistry.java` — mirror goal filter
- JPA entity + mapper — `severity` column mapping
- `EidosSystemPromptRenderer.java` — severity-discriminated constraint rendering; A2A_CARD goals/constraints
- `ClasspathYamlDescriptorRegistrar.java` — `severity` in YAML schema
- Flyway: rename V7→V8, new V9

### persistence-memory/
- `InMemoryAgentRegistry.java` — goal-based query filter

### examples/
- `DescriptorTemplateScenarioTest.java` — 3 new scenarios
- Supporting YAML/registrar fixtures as needed

### tests across all modules
- All `AgentConstraint` construction sites updated for `severity` parameter

## Not In Scope

- `constraintName` on `AgentQuery` — no dispatch scenario queries by constraint name; compliance-aware routing is engine policy
- Goal-based routing strategies — engine concern
- Goal-based termination logic — engine concern
- `goalsByPriority()` utility — callers can `stream().filter()` themselves
