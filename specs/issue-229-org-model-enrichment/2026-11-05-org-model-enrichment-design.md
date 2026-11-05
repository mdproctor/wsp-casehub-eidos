# OrganizationalUnit Model Enrichment — Design Spec

**Issue:** casehubio/eidos#229
**Branch:** issue-229-org-model-enrichment
**Date:** 2026-11-05

## Overview

Three model additions to org-api that close gaps between the self-organising society design spec and the current OrganizationalUnit model. All changes are additive — no existing fields or records are modified. No runtime behaviour — model fields only.

These additions unblock:
- OrgGoalCompiler enrichment (casehub-desiredstate#229)
- OrgPlanApprovalPolicy (casehub-desiredstate#227)
- OrgDriftPolicy refinement

## 1. Normative Obligations and Permissions on RoleSpec

### What

Add `List<String> obligations` and `List<String> permissions` to `RoleSpec`.

```java
public record RoleSpec(
    String roleName,
    String roleVocabulary,
    int minCardinality,
    int maxCardinality,
    Set<String> incompatibleWith,
    List<RolePolicy> policies,
    List<String> obligations,    // new
    List<String> permissions     // new
)
```

### Why

The spec (§2.2) requires each role to carry norm identifiers — strings like `"review-output"`, `"provision-agent"` that feed into agency-core's `NormDecision<String>` framework. At enforcement time, the society runtime wraps each identifier with source `"{groupId}:{roleName}"`, priority from `GroupPolicy.normativePriority`, and specificity from `GroupPolicy.normSpecificity`.

### Design rationale

Obligations and permissions are placed directly on `RoleSpec`, not on the existing `RolePolicy` record. The existing `RolePolicy(capabilityName, PolicyKind, expectations)` serves capability-based matching — a different concern with different consumers. Capability policies answer "can this agent fill this role?" Normative obligations answer "what must/may the role holder do?" Mixing them in one record conflates two concerns.

This follows eidos's existing pattern: `AgentDescriptor` puts goals and constraints as direct `List` fields, not wrapped in policy records.

### Validation

- Both lists are defensively copied and default to `List.of()` when null.
- No validation against agency-core's normative framework — norm identifiers are opaque strings until enforcement, consistent with eidos's open-string philosophy.
- Duplicate norm identifiers within a single list are not rejected — they are harmless (enforcement is idempotent). The same identifier appearing in both obligations and permissions is valid — a norm can be both obligated and permitted in different contexts.

### Existing RolePolicy unchanged

`RolePolicy(capabilityName, PolicyKind, expectations)` and `PolicyKind { ELIGIBILITY, OBLIGATION, PERMISSION }` remain as-is. The OBLIGATION/PERMISSION kinds on PolicyKind refer to capability-based constraints, not normative identifiers.

## 2. Dissolution Drain Timeout on GroupPolicy

### What

Add `Duration dissolutionDrainTimeout` to `GroupPolicy`.

```java
public record GroupPolicy(
    FormationRules formationRules,
    int normativePriority,
    NormSpecificity normSpecificity,
    Duration dissolutionDrainTimeout    // new
)
```

### Why

The spec (§4) says PENDING_DRAIN during dissolution has a configurable timeout, defaulting to 30 minutes. After timeout, `SocietyLifecycleManager` handles the stuck item. The specific post-timeout action (force-reassign, escalate, force-dissolve) is CBR-informed — a runtime concern, not a model concern.

### Validation

- Defaults to `Duration.ofMinutes(30)` when null.
- Must be positive (> 0). Zero-duration drain timeout would skip the drain phase entirely — if that's desired, the dissolution path should be DISSOLVING directly, not PENDING_DRAIN with zero wait.

### No strategy enum

The post-timeout strategy is not modeled. The spec explicitly delegates this to SocietyLifecycleManager's CBR-informed runtime logic. Adding a strategy enum now would be speculative — if declarative strategy control is needed later, it can be added without breaking changes.

## 3. Topology Family Classification

### What

Add two new types to org-api:

```java
public enum TopologyFamily {
    DISTRIBUTED,   // swarm, mesh
    CENTRALISED,   // hive
    INDEPENDENT    // many
}

@FunctionalInterface
public interface TopologyFamilyClassifier {
    Optional<TopologyFamily> classify(String topology);
}
```

### Why

The spec (§2.1) says topology transitions with medium selfDetermination (0.3–0.7) are auto-approved "within topology family" but require approval "across families." The model needs a way to classify topology strings into families so the runtime can make this determination.

### Design rationale

**This is a classification, not a policy.** It does not belong on GroupPolicy. The family of a topology value is intrinsic to the value itself, not a per-group configuration choice.

**SPI, not static mapping.** Topology values are open strings — domain apps can define custom topologies via vocabulary registrars. A static utility would hardcode the mapping for well-known values only. The `TopologyFamilyClassifier` SPI allows custom implementations that classify domain-specific topologies.

**Default implementation in org-runtime.** `WellKnownTopologyFamilyClassifier` (@DefaultBean) maps:
- `"swarm"`, `"mesh"` → `DISTRIBUTED`
- `"hive"` → `CENTRALISED`
- `"many"` → `INDEPENDENT`
- Anything else → `Optional.empty()`

**Safe default for unknowns.** `Optional.empty()` means the runtime treats the topology as having no family — any transition involving it is cross-family and requires approval. This is the conservative choice: unknown topologies get the most restrictive governance.

### No fields on OrganizationalUnit or GroupPolicy

The topology family is derived from the topology string via the classifier SPI. No new fields are needed on existing records. The runtime (SocietyLifecycleManager, OrgPlanApprovalPolicy) calls `classifier.classify(topology)` at transition time.

## Impact Summary

| File | Change |
|------|--------|
| `org-api/.../RoleSpec.java` | Add `obligations`, `permissions` fields |
| `org-api/.../GroupPolicy.java` | Add `dissolutionDrainTimeout` field |
| `org-api/.../TopologyFamily.java` | New enum |
| `org-api/.../TopologyFamilyClassifier.java` | New SPI interface |
| `org-runtime/.../WellKnownTopologyFamilyClassifier.java` | New @DefaultBean |
| `org-annotations/.../RoleSpecDef.java` | Add `obligations()`, `permissions()` annotation attributes |
| `org-annotations/.../GroupPolicyDef.java` | Add `dissolutionDrainTimeoutMinutes()` annotation attribute |
| `org-annotations/.../EidosOrgAnnotationsProcessor.java` | Extract new fields from Jandex |
| `org-annotations/.../EidosOrgAnnotationsRecorder.java` | Map new annotation values to model records |
| YAML deserializer tests | Update for new RoleSpec and GroupPolicy fields |
| Existing callers | Update `RoleSpec` and `GroupPolicy` construction sites across test fixtures |
| `OrganizationalUnit.Builder` | No changes needed |
| `OrgStructure` DSL | No changes needed (takes `List<RoleSpec>` — callers construct RoleSpec directly) |

## Non-Goals

- Implementing normative reasoning integration (agency-core)
- Adding runtime behaviour (SocietyLifecycleManager, OrgDriftPolicy)
- Changing `GroupInstance` (these are spec-side additions)
- Creating a topology vocabulary enum in eidos-vocab (future enhancement)
- Declarative dissolution strategy (CBR-informed at runtime)

## References

- Spec §2.1: topology transitions, selfDetermination thresholds
- Spec §2.2: role obligations/permissions, normative framework integration
- Spec §4: dissolution, PENDING_DRAIN timeout
- `org-api/src/main/java/io/casehub/eidos/org/api/RoleSpec.java` — current role model
- `org-api/src/main/java/io/casehub/eidos/org/api/RolePolicy.java` — capability-based policy (unchanged)
- `org-api/src/main/java/io/casehub/eidos/org/api/GroupPolicy.java` — current group policy
- `org-api/src/main/java/io/casehub/eidos/org/api/OrganizationalUnit.java` — topology fields
- `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — pattern precedent for direct list fields
- D1–D3 in `decisions.md`
