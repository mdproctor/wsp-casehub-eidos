# Agent Organizational Model — Research & Design

**Status:** Design research — complements [`eidos.md`](eidos.md) (individual agent identity)  
**Started:** 2026-08-31  
**Research backing:** [`agent-description-ontology.md`](agent-description-ontology.md), [`eidos.md`](eidos.md)  
**Cross-repo context:** casehubio/engine#1017 (YAML-first deployment), casehubio/engine#1019 (casehub.yaml project descriptor), casehubio/casehub-ops#83 (casehub-application NodeSpec)

---

## The Problem

Eidos models individual agents comprehensively — identity, capabilities, disposition, goals, constraints — but has no relational structure. Agents are flat. There is no way to say "agent A supervises agent B," "these three agents form a review team," or "if this agent fails, escalate to that one."

The gap is visible in concrete systems. Gastown has a 4-level named supervision hierarchy (Boot → Deacon → Witness → Polecat) hardcoded in its architecture. CaseHub's engine has `RecoveryPolicy` SPI (not yet implemented) for failure recovery, but it is policy-driven, not relationship-driven. The `delegation` boolean on `AgentDisposition` is the only inter-agent signal, and per protocol PP-20260605-1000ad it means "can spawn sub-agents" — a platform affordance, not a relationship.

### What's Missing

1. **No supervision relationships** — no way to declare "agent A monitors agent B" or "if B fails, A takes over"
2. **No team/squad/rig concept** — agents are individuals routed by capability; no way to say "these 3 agents form a review team" or "this group shares context"
3. **No hierarchy** — slot is a flat string, not a position in a tree; no delegation chains
4. **No composition** — no concept of "an agent that orchestrates other agents as a unit" (like Gastown's Witness orchestrating its Polecats)
5. **No organizational vocabulary** — no standard terms for how agents relate, what supervisory structures exist, or how authority flows

---

## Motivating Examples

### Gastown Supervision Chain

```
Boot (root watchdog)
  └── Deacon (cross-rig watchdog)
        ├── Witness Alpha (rig monitor)
        │     ├── Polecat-1 (worker)
        │     └── Polecat-2 (worker)
        └── Witness Beta (rig monitor)
              └── Polecat-3 (worker)
```

Boot validates Deacon every 5 minutes. Deacon monitors all Witnesses. Each Witness watches the worker agents in its rig. This hierarchy is structural and operational — it defines monitoring responsibility, escalation paths, and failure recovery. Today it is hardcoded in Gastown's application code.

### DevTown Review Team

A code review requires 3 agents working as a unit: structural reviewer, content reviewer, readability reviewer. The team has collective capabilities (full-stack-review) that no individual member has alone. An orchestrator needs to discover and dispatch the team, not individual reviewers.

### Cross-Application Supervision

A monitoring application deploys oversight agents that supervise worker agents in a separate application. The supervision relationship spans application boundaries — the supervisor needs attestation authority over agents it doesn't co-deploy with.

---

## Cross-Domain Survey

### Multi-Agent Systems Literature

| Framework | Organizational Model | Key Concept |
|-----------|---------------------|-------------|
| **AGR** (Agent-Group-Role) | Agents join groups and play roles within them | Role is contextual to group membership, not global identity |
| **MOISE+** | Structural, functional, and deontic specifications | Separates what agents ARE from what they SHOULD DO from what they MUST DO |
| **Holonic MAS** | Recursive composition — a holon is both a whole and a part | Enables unbounded hierarchy; each level is a self-contained unit |
| **AALAADIN** | Groups as first-class organizational units with internal structure | Groups have their own lifecycle independent of member agents |
| **OperA** | Social model (roles, dependencies) + interaction model (protocols) + normative model (obligations) | Separates organizational structure from operational behavior |

**Key findings:**

1. **Role is contextual** — AGR's central insight: an agent's role depends on which group it's in. An agent can be a "witness" in Rig Alpha and a "backup-witness" in Rig Beta. This argues against using the agent's `slot` field for group-specific roles.

2. **Groups are first-class** — every framework treats organizational units as independent entities with their own properties, not just collections of agents.

3. **Hierarchy is recursive** — holonic MAS demonstrates that fixed-depth hierarchies are unnecessary constraints. Unbounded nesting with cycle detection covers both flat teams and deep supervision chains.

4. **Relationships have semantics** — supervision, delegation, escalation, and backup are fundamentally different relationship types with different operational implications. A generic "relates-to" edge loses critical information.

### Gastown (Concrete Reference Implementation)

Gastown's hardcoded hierarchy validates several design choices:

- **Named structural roles** — Boot, Deacon, Witness, Polecat are positions in a hierarchy, not personality traits
- **Monitoring as supervision** — the Witness doesn't just know about its Polecats; it actively monitors them and takes corrective action
- **Fixed responsibilities per level** — each tier has defined monitoring intervals and failure responses
- **Rig as organizational unit** — a Rig is a self-contained group with a Witness and its Polecats; the rig itself has properties (region, capacity) independent of its members

---

## Design Decisions

### D1: Both Organizational Units and Pairwise Relationships

**Choice:** Both organizational units (teams, squads, rigs) AND pairwise agent relationships are first-class peers in the model.

**Alternatives:**
- Units only (relationships emerge from structure) — simpler but can't express cross-unit relationships like "agent A in Rig-1 backs up agent B in Rig-2"
- Relationships only (groups emerge from graph topology) — more flexible but harder to visualize and harder to assign collective properties

**Rationale:** Gastown needs both. A Rig is a unit with collective properties. A Boot-supervises-Deacon relationship crosses unit boundaries. Forcing one into the other loses information.

### D2: New Module in Eidos

**Choice:** `casehub-eidos-org` as a sibling module family within the eidos repo.

**Alternatives:**
- In eidos-api itself — every consumer gets org types whether they need them or not
- Separate repo — maximum separation but adds cross-repo coordination overhead

**Rationale:** Organizational types reference `AgentDescriptor` (from eidos-api) and `DesiredStateGraph` (from desiredstate-api). A new module in eidos keeps the family together while remaining optional for consumers that don't need org structure.

### D3: Generic Units with Vocabulary

**Choice:** One `OrganizationalUnit` type with a `kind` string grounded in `VocabularyRegistry`.

**Alternatives:**
- Sealed type hierarchy (Team, Squad, Department) — more type-safe but constrains applications
- Marker interface — maximum flexibility but hardest to serialize and diagram

**Rationale:** Follows the established eidos pattern — `slot` is an open string, `disposition` axes use vocabulary terms. Applications define their own organizational kinds (rig, squad, department) via vocabulary, not platform code.

### D4: Supervisory Attestation — Opt-In, Scoped, Evidence-Grounded

**Choice:** Supervisors CAN attest to supervised agents' behavior, but only when explicitly granted attestation authority with scoped dimensions.

**Alternatives:**
- Read-only integration (org structure visible to trust queries but doesn't feed trust) — loses the operational value of supervision
- Automatic attestation for all supervisors — arbitrary and uncontrolled

**Rationale:** Attestation authority is a platform affordance (like `delegation`), not an implicit property of supervision. An `AttestationGrant` on the relationship specifies which `ComplianceDimension` keys the supervisor can attest to, which capabilities the attestation covers, and which `BehavioralSignal` types can be recorded. This feeds the existing `BehavioralSignalStore` → `DefaultCapabilityHealth.probe()` pipeline.

### D5: Declarative Reconciliation via casehub-desiredstate

**Choice:** Organizational structure is a desiredstate domain. YAML is the source of truth; a reconciler diffs POJO snapshots and applies mutations via `OrgRegistry`.

**Alternatives:**
- Command-sourced from diagram editor — editor must understand command semantics; YAML becomes derived
- Independent differ in eidos — duplicates reconciliation logic from desiredstate

**Rationale:** `casehub-desiredstate` already has `GraphDiff.computeMutations()`, `ReconciliationLoop`, `GoalCompiler`, `NodeProvisioner`, and YAML/annotation/TypeScript surfaces. The org model plugs into this infrastructure rather than reinventing it.

**Dependency graph validation:**

```
casehub-platform-api          (universal base)
  ├── casehub-desiredstate-api  (depends on platform-api + ras-api)
  ├── casehub-eidos-api         (depends on platform-api)
  └── casehub-eidos-org-api     (NEW — depends on eidos-api + desiredstate-api)
```

No circularity. `eidos-api` does not depend on `desiredstate-api`, and vice versa. `eidos-org-api` sits at the same tier as `engine-api` — one hop above two peers.

### D6: Unbounded Hierarchy with Cycle Detection

**Choice:** No artificial depth limit. Units can contain sub-units recursively. Cycle detection at registration/reconciliation time.

**Alternatives:**
- Configurable max depth — prevents runaway hierarchies but adds an arbitrary constraint
- Flat + one level — simpler but can't model Gastown's nested structure declaratively

**Rationale:** Applications self-constrain via vocabulary (Gastown defines 4 levels; another app defines 2). Platform validates structural integrity (no cycles) without imposing depth limits.

### D7: Foundation Relationship Kinds + Vocabulary Extension

**Choice:** Platform defines core relationship kinds (`SUPERVISES`, `DELEGATES_TO`, `ESCALATES_TO`, `REPORTS_TO`, `BACKS_UP`) as an enum. Applications extend via `VocabularyRegistry` for domain-specific relationships using `EXTENDED` + `extendedKind`.

**Alternatives:**
- Fully open vocabulary — platform can't reason about relationships structurally
- Closed enum only — applications can't define new relationship types

**Rationale:** Core kinds have platform semantics (supervision triggers attestation, escalation affects routing, backup triggers failover). Vocabulary extensions are advisory — the platform doesn't assign operational semantics to them, but applications can.

### D8: Membership Carries a Role

**Choice:** Each membership has an optional `role` string (vocabulary-grounded), separate from the agent's `slot`.

**Alternatives:**
- Slot is sufficient — can't distinguish an agent's role across different units

**Rationale:** AGR's central insight. An agent's `slot` is their general kind ("monitor"). Their membership `role` is their function within a specific unit ("witness" in Rig Alpha, "backup-witness" in Rig Beta). Same agent, different roles in different contexts.

### D9: Scoped Relationships

**Choice:** Relationships carry optional scope qualifiers (`capabilityName`, `domain`, `custom`).

**Alternatives:**
- Unscoped relationships — forces one-supervisor-per-agent or coarse-grained supervision

**Rationale:** Enables "A supervises B for code-review" while "C supervises B for deployment." Different concerns can have different supervisory structures without requiring separate organizational topologies.

### D10: Units Are Lightweight Descriptors

**Choice:** Units can declare collective capabilities, goals, and constraints — a subset of `AgentDescriptor`.

**Alternatives:**
- Pure containers (no properties) — can't model emergent team capabilities
- Full `AgentDescriptor` on units — blurs the agent/unit distinction

**Rationale:** A review team declares a collective capability "full-stack-review" that no single member has. Team-level goals ("ensure 100% coverage") and constraints ("max 3 concurrent tasks") are properties of the unit, not any individual member.

### D11: Application-Scoped and Cross-App Organization

**Choice:** Both mechanisms from day one. Intra-app org structure lives in `organization/org.yaml` inside a `casehub-application`. Cross-app org structure is a separate `casehub-organization` node type in the deployment graph.

**Alternatives:**
- Application-scoped only — can't model Gastown's cross-app supervision (oversight app monitoring core app)

**Rationale:** The `casehub-organization` deployment node depends on the applications it references, ensuring they're provisioned first. The `@app-id` qualifier on agent references resolves agents across application boundaries.

---

## Core POJO Model

### OrganizationalUnit

```java
public record OrganizationalUnit(
    String unitId,
    String name,
    String kind,                    // vocabulary-grounded
    String kindVocabulary,
    String tenancyId,
    String parentUnitId,            // optional — hierarchy
    List<Membership> members,
    List<AgentCapability> capabilities,  // collective
    List<AgentGoal> goals,
    List<AgentConstraint> constraints
) {}
```

### Membership

```java
public record Membership(
    String agentId,
    String role,                    // optional, vocabulary-grounded
    String roleVocabulary
) {}
```

### AgentRelationship

```java
public record AgentRelationship(
    String sourceAgentId,
    String targetAgentId,
    RelationshipKind kind,
    String extendedKind,            // for EXTENDED kind
    String kindVocabulary,
    RelationshipScope scope,        // optional qualifier
    AttestationGrant attestation,   // optional, opt-in
    String tenancyId
) {}
```

### RelationshipKind

```java
public enum RelationshipKind {
    SUPERVISES,       // authority + monitoring responsibility
    DELEGATES_TO,     // task handoff with retained accountability
    ESCALATES_TO,     // problem escalation path
    REPORTS_TO,       // information flow without authority
    BACKS_UP,         // failover / redundancy
    EXTENDED          // vocabulary-defined (uses extendedKind field)
}
```

### RelationshipScope

```java
public record RelationshipScope(
    String capabilityName,
    String domain,
    String custom
) {}
```

### AttestationGrant

```java
public record AttestationGrant(
    Set<String> dimensions,           // ComplianceDimension keys
    Set<String> capabilityScope,      // empty = all capabilities
    Set<BehavioralSignal> signalTypes // COMPLIANT, VIOLATED
) {}
```

### OrgRegistry SPI

```java
public interface OrgRegistry {
    void registerUnit(OrganizationalUnit unit);
    void removeUnit(String unitId, String tenancyId);
    Optional<OrganizationalUnit> findUnit(String unitId, String tenancyId);
    List<OrganizationalUnit> findUnits(OrgQuery query);
    List<OrganizationalUnit> childUnits(String parentUnitId, String tenancyId);
    List<OrganizationalUnit> ancestorUnits(String unitId, String tenancyId);

    List<OrganizationalUnit> unitsFor(String agentId, String tenancyId);
    List<Membership> membersOf(String unitId, String tenancyId);

    void addRelationship(AgentRelationship relationship);
    void removeRelationship(String sourceAgentId, String targetAgentId,
                            RelationshipKind kind, String tenancyId);
    List<AgentRelationship> relationshipsFrom(String agentId, String tenancyId);
    List<AgentRelationship> relationshipsTo(String agentId, String tenancyId);

    List<AgentRelationship> supervisors(String agentId, String tenancyId);
    List<AgentRelationship> subordinates(String agentId, String tenancyId);
    List<AgentRelationship> escalationPath(String agentId, String tenancyId);
}
```

---

## YAML Representation

### Intra-Application Organization

```yaml
organization:
  units:
    - unitId: rig-alpha
      name: Rig Alpha
      kind: rig
      kindVocabulary: urn:gastown:vocab:org
      tenancyId: gastown
      parent: cluster-1
      members:
        - agentId: witness-alpha
          role: witness
        - agentId: polecat-1
          role: worker
        - agentId: polecat-2
          role: worker
      capabilities:
        - name: full-stack-code-work
      goals:
        - name: rig-throughput
          priority: PRIMARY
      constraints:
        - name: max-concurrent-tasks
          severity: SOFT

  relationships:
    - source: witness-alpha
      target: polecat-1
      kind: SUPERVISES
      attestation:
        dimensions: [LATENCY, ATTESTATION_RATE]
        signalTypes: [COMPLIANT, VIOLATED]
        capabilityScope: [code-analysis, code-generation]

    - source: polecat-1
      target: polecat-2
      kind: BACKS_UP
      scope:
        capability: code-analysis
```

### Cross-Application Organization

```yaml
organization:
  relationships:
    - source: deacon@gastown-oversight
      target: witness-alpha@gastown-core
      kind: SUPERVISES
      scope:
        capability: rig-monitoring
      attestation:
        dimensions: [LATENCY, ATTESTATION_RATE]
        signalTypes: [COMPLIANT, VIOLATED]
```

### Preprocessing Support

The yaml-core preprocessing pipeline applies to organizational YAML — `${var.*}` variables, `forEach` expansion, `when` conditional inclusion:

```yaml
organization:
  units:
    forEach:
      var: rig
      in: [alpha, beta, gamma]
    produce:
      - unitId: rig-${rig}
        kind: rig
        kindVocabulary: urn:gastown:vocab:org
        tenancyId: gastown
        members:
          - agentId: witness-${rig}
            role: witness
```

---

## Desiredstate Integration

### Graph Mapping

| Org concept | Desiredstate concept | NodeType |
|---|---|---|
| OrganizationalUnit | DesiredNode | `org:unit` |
| AgentRelationship | DesiredNode | `org:relationship` |
| Membership | Embedded in unit's NodeSpec | (not a separate node) |
| Unit hierarchy (parentUnitId) | Dependency edge | child → parent |

### Reconciliation Loop

1. Diagram editor (or CI push) saves YAML
2. YAML deserializes to org POJOs via Jackson (`EidosOrgModule`)
3. `OrgGoalCompiler` (implements `GoalCompiler`) translates POJOs → `DesiredStateGraph`
4. `GraphDiff.computeMutations()` computes delta against current graph
5. `OrgNodeProvisioner` provisions/deprovisions via `OrgRegistry`

Memberships are embedded in the unit's NodeSpec because they are structural properties of the unit, not independently lifecycle-managed entities. When a member is added/removed, the unit node's spec changes → `GraphDiff` emits `UpdateNode` → the provisioner reconciles the member list.

### Deployment Integration

Organizational structure participates in the YAML-first deployment model (engine#1017, engine#1019, casehub-ops#83):

```
┌─────────────────────────────────────────────────────────────┐
│  desiredstate runtime                                       │
│  (TransitionPlanner, ReconciliationLoop, FaultPolicy)       │
├─────────────────────────────────────────────────────────────┤
│  casehub-ops/deployment                                     │
│  (topology: load-balancer, HA, service-mesh)                │
├────────────────────────┬────────────────────────────────────┤
│  casehub-application   │  casehub-organization (NEW)        │
│  NodeSpec              │  NodeSpec                          │
│  (app + agents + org)  │  (cross-app relationships)         │
└────────────────────────┴────────────────────────────────────┘
```

**Two deployment mechanisms:**

1. **Intra-app org** — embedded in `casehub-application`. The NodeProvisioner discovers `organization/org.yaml` alongside `agents/descriptors.yaml` and registers both during application provisioning.

2. **Cross-app org** — a separate `casehub-organization` node type in the deployment graph. Depends on the applications it references (`dependsOn: [app-a, app-b]`), ensuring they are provisioned first. The `@app-id` qualifier resolves agent references across application boundaries.

```yaml
# deployment.yaml
nodes:
  - type: casehub-application
    id: gastown-core
    spec:
      source: ./apps/gastown-core

  - type: casehub-application
    id: gastown-oversight
    spec:
      source: ./apps/gastown-oversight

  - type: casehub-organization
    id: gastown-cross-app-org
    dependsOn: [gastown-core, gastown-oversight]
    spec:
      source: ./organization/cross-app.yaml
```

---

## Triangulation — Three Layers

The organizational model is the middle layer in a three-layer architecture:

```
                    deployment.yaml
                    (desiredstate topology)
                         │
          ┌──────────────┼──────────────────┐
          │              │                  │
    casehub-application  │          casehub-organization
    (per-app)            │          (cross-app)
          │              │                  │
    ┌─────┴─────┐   casehub-application     │
    │           │   (per-app)               │
    │           │        │                  │
descriptors  org.yaml  descriptors + org    cross-app.yaml
    │           │        │                  │
    ▼           ▼        ▼                  ▼
 ┌──────────────────────────────────────────────┐
 │              OrgRegistry + AgentRegistry      │
 └──────────────────────────────────────────────┘
```

Each layer has a clear job:

| Layer | Declares | Editable via |
|-------|----------|-------------|
| **Descriptors** | Who each agent IS (identity, capabilities, disposition) | `agents/descriptors.yaml` |
| **Organization** | How agents RELATE (supervision, delegation, backup, teams) | `organization/org.yaml` + diagram editor |
| **Deployment** | Where and how it all RUNS (topology, infrastructure, env config) | `deployment.yaml` + environment overlays |

Each layer is independently editable. Change a descriptor → agent identity updates. Change org structure → relationships reconcile. Change deployment topology → infrastructure converges.

---

## Gastown Application — Full Example

### Project Structure

```
gastown/
├── deployment.yaml
├── environments/
│   ├── dev.yaml
│   └── prod.yaml
├── apps/
│   ├── gastown-core/
│   │   ├── casehub.yaml
│   │   ├── cases/
│   │   │   ├── merge-queue.yaml
│   │   │   └── rig-lifecycle.yaml
│   │   ├── agents/
│   │   │   └── descriptors.yaml
│   │   └── organization/
│   │       └── org.yaml
│   └── gastown-oversight/
│       ├── casehub.yaml
│       ├── agents/
│       │   └── descriptors.yaml
│       └── organization/
│           └── org.yaml
└── organization/
    └── cross-app.yaml
```

### Deployed Organizational Structure

```
Boot (root-watchdog, gastown-oversight)
  └──SUPERVISES──▶ Deacon (cross-rig-watchdog, gastown-oversight)
                      │
            SUPERVISES (cross-app, scoped to rig-monitoring)
                      │
                      ▼
    ┌─── Rig Alpha ──────────────────────┐
    │  Witness Alpha (witness)           │
    │    │ SUPERVISES         │ SUPERVISES│
    │    ▼                    ▼          │
    │  Polecat-1 ◀──BACKS_UP── Polecat-2│
    │  (worker)               (worker)  │
    └────────────────────────────────────┘

  Refinery (orchestrator, standalone)
```

This structure is fully declarative. No hardcoded Java hierarchy. Gastown defines its organizational vocabulary (`urn:gastown:vocab:org` with terms `rig`, `cluster`, `supervision-hierarchy`), its membership roles (`witness`, `worker`, `root-watchdog`, `cross-rig-watchdog`), and its supervision topology — all in YAML.

---

## Module Structure

```
casehub-eidos/
├── org-api/              → casehub-eidos-org-api
│                           POJOs, OrgRegistry SPI, RelationshipKind enum
│                           depends on: eidos-api, desiredstate-api
│
├── org-runtime/          → casehub-eidos-org
│                           CdiOrgRegistry, OrgGoalCompiler, OrgNodeProvisioner,
│                           OrgActualStateAdapter, YAML deserializer, cycle detection
│                           depends on: org-api, desiredstate runtime
│
├── org-memory/           → casehub-eidos-org-memory
│                           InMemoryOrgRegistry @Alternative for tests
│
└── org-vocab/            → casehub-eidos-org-vocab (optional)
                            Well-known org vocabularies
```

Cross-app deployment node in casehub-ops:
- `casehub-ops-org` — `CaseHubOrganizationSpec` NodeSpec + NodeProvisioner

---

## Trust Integration

Supervisory attestation flows through the existing trust pipeline:

```
Supervisor                  BehavioralSignalStore           CapabilityHealth.probe()
    │                              │                              │
    │ record(agentId,              │                              │
    │   capability,                │                              │
    │   VIOLATED,                  │                              │
    │   LATENCY)                   │                              │
    │──────────────────────────────▶│                              │
    │                              │  count(agentId, capability,  │
    │                              │    VIOLATED, LATENCY)        │
    │                              │◀─────────────────────────────│
    │                              │                              │
    │                              │  ≥ threshold?                │
    │                              │  → BehavioralViolation       │
```

Guards against arbitrary attestation:

1. **AttestationGrant required** — no grant on the relationship, no attestation authority
2. **Dimension-scoped** — supervisor can only attest to granted `ComplianceDimension` keys
3. **Capability-scoped** — attestation covers only listed capabilities (empty = all)
4. **Signal-type-scoped** — supervisor can only record granted `BehavioralSignal` types
5. **Store-owned TTL** — signals decay over time per `BehavioralSignalStore` configuration
6. **Threshold-gated** — a single attestation doesn't change health status; it takes accumulated signals exceeding the per-dimension or aggregate threshold

---

## Open Questions

1. **Reactive SPI variants** — should `OrgRegistry` have a `ReactiveOrgRegistry` mirror (like `ReactiveAgentRegistry`)?
2. **JPA persistence** — entity design for units, memberships, and relationships; join tables and indexing strategy
3. **Annotation surface** — `@OrgUnit`, `@Supervises`, `@MemberOf` annotations for the eidos-annotations extension
4. **Org-aware routing** — should `AgentSelector` consider organizational structure when selecting agents? (e.g., prefer agents from the same unit, or respect escalation paths)
5. **Org-aware prompt rendering** — should `SystemPromptRenderer` include organizational context in agent prompts? (e.g., "You are supervised by Deacon. Your teammates are Polecat-1 and Polecat-2.")

---

## References

- [`eidos.md`](eidos.md) — Eidos specification (individual agent identity model)
- [`agent-description-ontology.md`](agent-description-ontology.md) — ontology research (capability, disposition, discovery)
- [`casehub-platform-vocabulary-validation.md`](casehub-platform-vocabulary-validation.md) — vocabulary validation research
- casehubio/engine#1017 — zero-authored-Java deployment epic (desiredstate architecture)
- casehubio/engine#1019 — casehub.yaml project descriptor (NodeSpec integration)
- casehubio/casehub-ops#83 — casehub-application NodeSpec + NodeProvisioner
- PP-20260605-1000ad — delegation is platform-semantic (personality frameworks make no delegation claim)
- PP-20260703-dependency-tier-order — CaseHub dependency tier order
- PP-20260522-platform-api-scope — casehub-platform-api scope rules
- Ferber, Gutknecht, Michel (2004) — AGR (Agent-Group-Role) organizational model
- Hübner, Sichman, Boissier (2002) — MOISE+ organizational model
- Fischer (1999) — Holonic Multi-Agent Systems
