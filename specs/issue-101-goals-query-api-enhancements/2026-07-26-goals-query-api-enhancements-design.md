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

**`AgentDescriptorValidator`** — add `MAX_CAPABILITIES = 20`. Goals have `MAX_GOALS = 10` and constraints have `MAX_CONSTRAINTS = 10`; capabilities are the primary matching dimension so a higher limit is appropriate, but unbounded is a validation gap.

**`AgentDescriptor` compact constructor** — add capability count limit check (same pattern as goals/constraints), then duplicate capability name check:

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

**MARKDOWN** — label prefix, paralleling the goals `[PRIMARY]`/`[SECONDARY]` pattern:
- `- **[HARD]** description`
- `- **[SOFT]** description`
- Sorted: HARD before SOFT, then alphabetically by name within each severity

**PROSE** — grouped by severity, paralleling the goals primary/secondary split:
- `Hard constraints: desc1. desc2.`
- If soft constraints present: ` Also: desc3. desc4.`
- Same sentence structure as the existing goals prose rendering

**A2A_CARD** — include `severity` field on each constraint object:
```json
{"name": "no-pii-exposure", "description": "...", "severity": "HARD"}
```

### Hash coverage

Add `severity` to constraint objects in `buildDescriptorPayload()` for all formats. Per `a2a-structural-assembly-hash-coverage` protocol (PP-20260613-608684), any field rendered in `assembleA2aCard()` must appear in `buildDescriptorPayload()` for A2A_CARD format. Since severity also affects MARKDOWN/PROSE rendering, including it in all formats ensures any severity change invalidates the cache across all render paths.

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
- When `goalName` non-null: `AND EXISTS (SELECT 1 FROM AgentGoalEntity g WHERE g.descriptor = a AND g.name = :goalName)`
- EXISTS subquery — avoids `MultipleBagFetchException`. The existing query already fetches `capabilities` as a bag (`JOIN FETCH a.capabilities c`); adding a second bag fetch for `goals` is rejected by Hibernate 6. Goals are loaded lazily by `AgentDescriptorMapper.toRecord()` — no fetch needed, just a filter.
- No vocabulary subsumption — goals are identity-level standing objectives, not vocabulary-grounded terms. Exact name match only.

**InMemoryAgentRegistry:**
- Stream filter on `descriptor.goals()` matching by name

**AgentMatch:**
- When querying by `goalName`, `resolvedCapability` stays null. Goals are not capabilities — no match degree ranking. The query is a filter, not a ranked match.
- Goal query result ordering is unspecified, following the existing contract for non-capability queries (e.g., slot-only queries).

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

Already implemented. `EidosRenderPipeline.assembleA2aCard()` renders `publicGoals()` and `publicConstraints()` into the A2A JSON. `buildDescriptorPayload()` includes them in the hash payload with visibility filtering for A2A_CARD format. No work needed.

The addition of `severity` to A2A constraint output is covered by §103.

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

No deployed instances — all schema changes rewrite base migration files (ARC42STORIES §7, CLAUDE.md).

| Version | File | Change |
|---------|------|--------|
| V1 | `V1__initial_schema.sql` | Add `UNIQUE (descriptor_id, name)` to `agent_capability` CREATE TABLE |
| V7 | `V7__descriptor_templates.sql` | Unchanged |
| V8 | `V8__goals_constraints.sql` | Add `severity VARCHAR(20) NOT NULL` to `agent_constraint` CREATE TABLE |

No V9 migration. Severity is part of the table design from the start. Capability uniqueness belongs in V1 where the table was created. Developers drop and recreate local databases on schema changes — this is the established convention.

## Files Touched

### api/
- `ConstraintSeverity.java` — new enum
- `AgentConstraint.java` — add `severity` field
- `AgentDescriptor.java` — capability count limit check; capability name uniqueness check; `hasGoal()`; `hasConstraint()`
- `AgentDescriptorValidator.java` — add `MAX_CAPABILITIES = 20`
- `AgentQuery.java` — add `goalName` field + factory methods
- `AgentDescriptorComparator.java` — severity in constraint comparison

### runtime/
- `JpaAgentRegistry.java` — goal-based EXISTS subquery filter
- `AgentDescriptorMapper.java` — severity mapping in `toConstraint()`/`toConstraintEntity()`
- `AgentConstraintEntity.java` — add `severity` field
- `AgentCapabilityEntity.java` — add `@UniqueConstraint(columnNames = {"descriptor_id", "name"})` to `@Table`
- `EidosRenderPipeline.java` — severity-discriminated constraint rendering (MARKDOWN, PROSE, A2A_CARD); severity in `buildDescriptorPayload()` hash payload
- `ClasspathYamlDescriptorRegistrar.java` — `severity` in `ConstraintConfig` and `toDescriptor()`
- Flyway: rewrite V1 (capability uniqueness), rewrite V8 (severity column)

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
