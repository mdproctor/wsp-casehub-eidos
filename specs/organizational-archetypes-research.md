# Organizational Archetypes — Research and Terminology Mapping

## Purpose

Map eidos's organizational model (`casehub-eidos-org`) to established
organizational theory — both classical management science (Mintzberg) and
multi-agent systems research (MOISE+, AGR, Horling & Lesser). Identify
which archetypes the existing examples cover, which are missing, and
whether the model needs extension.

---

## Frameworks Surveyed

### Mintzberg's Organizational Configurations (1979, 1989)

Henry Mintzberg identified five (later seven) structural configurations
based on the dominant coordination mechanism:

| Configuration | Coordination mechanism | Key part | Structure |
|---|---|---|---|
| **Simple Structure** | Direct supervision | Strategic apex | Flat, centralized, one leader |
| **Machine Bureaucracy** | Standardization of work | Technostructure | Deep hierarchy, formal procedures |
| **Professional Bureaucracy** | Standardization of skills | Operating core | Flat authority, high specialization |
| **Divisionalized Form** | Standardization of outputs | Middle line | Semi-autonomous divisions under HQ |
| **Adhocracy** | Mutual adjustment | Support staff | Project-based, temporary, organic |
| **Missionary** | Standardization of norms | Ideology | Belief-driven coordination |
| **Political** | None dominant | — | Power and conflict |

Source: Mintzberg, H. (1979). *The Structuring of Organizations*. Prentice-Hall.
Extended in Mintzberg, H. (1989). *Mintzberg on Management*. Free Press.

### Horling & Lesser's MAS Organizational Paradigms (2004)

The canonical multi-agent systems taxonomy. Ten paradigms describing how
agents are organized:

| Paradigm | Description | Coordination |
|---|---|---|
| **Hierarchy** | Tree-structured authority, top-down control | Command |
| **Holarchy** | Recursive nested wholes (Koestler's holons) | Each unit is both whole and part |
| **Team** | Cooperative group, shared goal, mutual awareness | Joint intention |
| **Coalition** | Goal-directed temporary alliance, dissolves when done | Negotiation |
| **Congregation** | Long-lived grouping of similar agents, no single goal | Affinity |
| **Society** | Open system, heterogeneous agents, social laws | Norms |
| **Federation** | Groups represented by delegates who negotiate | Delegation |
| **Market** | Competitive, auction/bidding-based | Price signals |
| **Matrix** | Dual reporting lines (functional + project) | Dual authority |
| **Pipeline** | Sequential handoff, output→input chain | Process flow |

Source: Horling, B. & Lesser, V. (2004). A survey of multi-agent
organizational paradigms. *The Knowledge Engineering Review*, 19(4), 281–316.

### MOISE+ (2002)

Three-dimensional organizational model for multi-agent systems:

| Dimension | Covers | Eidos equivalent |
|---|---|---|
| **Structural** | Roles, groups, links (authority, acquaintance, communication) | `OrgUnit`, `Membership`, `AgentRelationship`, `RelationshipKind` |
| **Functional** | Goal decomposition, plans, missions | `AgentGoal` (with `capabilities` mapping), `AgentCapability`, goal decomposition via `AgentGoal.capabilities` |
| **Deontic** | Obligations, permissions, role-mission bindings | `AgentConstraint` (with `ConstraintSeverity`), `AttestationGrant`, `Visibility` |

Source: Hübner, J.F., Sichman, J.S. & Boissier, O. (2002). A model for
the structural, functional, and deontic specification of organizations
in multiagent systems. *AAMAS 2002*, pp. 118–128.

### AGR — Agent/Group/Role (2004)

Minimal organizational model. Three primitives only:

| Concept | Definition | Eidos equivalent |
|---|---|---|
| **Agent** | Autonomous entity | `AgentDescriptor.agentId` |
| **Group** | Set of agents sharing a context | `OrganizationalUnit` |
| **Role** | Function an agent plays within a group | `Membership.role` (+ `roleVocabulary`) |

Source: Ferber, J., Gutknecht, O. & Michel, F. (2004). From agents to
organizations: an organizational view of multi-agent systems. *AOSE 2003*,
LNCS 2935, pp. 214–230.

---

## Eidos Model Assessment — AGR vs MOISE+

**Eidos is structurally AGR, functionally and deontically MOISE+.**

The structural layer maps directly to AGR's three primitives:

| AGR | Eidos |
|---|---|
| Agent | `AgentDescriptor` (identity, disposition, capabilities) |
| Group | `OrganizationalUnit` (members, kind, parent, capabilities, goals, constraints) |
| Role | `Membership.role` (with `roleVocabulary` for vocabulary-grounded roles) |

But eidos goes beyond AGR in two directions:

**Functional dimension (MOISE+ alignment):**
- `AgentGoal` with `GoalPriority` and `capabilities` mapping — goals linked to the capabilities that serve them
- `AgentCapability` with `epistemicDomains`, `excludedDomains`, `qualityHint` — rich capability metadata
- Unit-level goals and capabilities on `OrganizationalUnit` — group goals distinct from member goals

**Deontic dimension (MOISE+ alignment):**
- `AgentConstraint` with `ConstraintSeverity` (HARD/SOFT) and `Visibility` — obligations and prohibitions
- `AttestationGrant` on relationships — who may attest what about whom (dimensions, signal types)
- `BehavioralSignalStore` — accumulated evidence of compliance/violation

**Beyond MOISE+:**
- `RelationshipKind` enum (SUPERVISES, DELEGATES_TO, ESCALATES_TO, REPORTS_TO, BACKS_UP, EXTENDED) — typed inter-agent links richer than MOISE+'s authority/acquaintance/communication triple
- `RelationshipScope` (capabilityName, domain, custom) — relationships scoped to specific capabilities or domains
- `DispositionHealth` and `CapabilityHealth` — runtime probing of agent state, not just structural declaration
- Vocabulary grounding via `VocabularyRegistry` — roles, slots, and dispositions grounded in registered vocabulary terms

**Does the model need extension?** No. The current model can express all
ten Horling & Lesser paradigms and all five Mintzberg configurations
without structural changes. The expressiveness comes from the combination
of typed relationships, scoped capabilities, unit nesting, and the
EXTENDED relationship kind (which allows arbitrary named relationship
types via `extendedKind`). See the mapping table below.

---

## Archetype Mapping — Full Cross-Reference

### Eidos terminology (canonical names)

| Eidos concept | Formal origin | Purpose |
|---|---|---|
| `OrganizationalUnit` | AGR Group | Container for agents with shared context |
| `Membership` | AGR Role binding | Agent's function within a unit |
| `AgentRelationship` | MOISE+ structural link | Typed directed link between agents |
| `RelationshipKind` | Extended MOISE+ link types | Authority pattern classification |
| `RelationshipScope` | Novel (eidos) | Capability/domain scoping of relationships |
| `AttestationGrant` | Deontic (MOISE+ inspired) | Trust delegation — who may attest what |
| `AgentGoal` | MOISE+ functional | Standing objective with priority |
| `AgentConstraint` | MOISE+ deontic | Behavioral guardrail with severity |
| `AgentCapability` | MOISE+ functional (extended) | Declared competency with quality metadata |

### Archetype → framework → eidos mapping

| Archetype | Mintzberg | Horling & Lesser | Eidos modelling | Example |
|---|---|---|---|---|
| **Supervision Hierarchy** | Machine Bureaucracy | Hierarchy | `SUPERVISES` chain, scoped authority, deep unit nesting | Gastown |
| **Professional Team** | Professional Bureaucracy | Team | Flat `OrgUnit`, peer `Membership`, collective capabilities on unit | Review Team |
| **Tiered Escalation** | Machine Bureaucracy (variant) | Hierarchy + escalation | `ESCALATES_TO` chain, domain-scoped via `RelationshipScope` | Customer Support |
| **Divisional / Holarchic** | Divisionalized Form | Holarchy | Nested `OrgUnit` with `parentUnitId`, cross-unit `DELEGATES_TO` and `REPORTS_TO` | Clinical Triage |
| **Orchestrator-Worker** | Adhocracy | Federation | Coordinator unit with `DELEGATES_TO` to specialist agents, scoped by capability | *proposed* |
| **Pipeline** | — | Pipeline (process-based) | Sequential `DELEGATES_TO` chain, each agent's output capability matches next agent's input | *proposed* |
| **Advisory Board** | — | Coalition | Multiple advisors in a unit, `REPORTS_TO` a judge/decision-maker, `EXTENDED("advises")` | *proposed* |
| **Simple / Flat** | Simple Structure | — | Single `OrgUnit`, one leader with `SUPERVISES` to all members, no middle management | — |
| **Market** | — | Market | Agents in a `Congregation`-style unit, `EXTENDED("bids-on")` relationships, no fixed authority | — |
| **Matrix** | — | Matrix | Agent in multiple `OrgUnit`s simultaneously (functional + project), dual `REPORTS_TO` | — |

### How eidos expresses each paradigm (no model changes needed)

**Hierarchy** — `SUPERVISES` relationships in a tree. `RelationshipScope`
limits authority to specific capabilities. `AttestationGrant` controls
what a supervisor can attest about subordinates. Already demonstrated
in Gastown (4-level chain).

**Holarchy** — Nested `OrganizationalUnit` via `parentUnitId`. Each unit
is simultaneously a whole (has its own members, goals, capabilities) and
a part (child of a parent unit). Cross-unit delegation via `DELEGATES_TO`.
Already demonstrated in Clinical Triage (Hospital → ER / Radiology).

**Team** — Flat `OrganizationalUnit` with no internal hierarchy. Members
have roles but no supervision relationships between them. Unit-level
capabilities emerge from the combination. Already demonstrated in Review
Team.

**Coalition** — Temporary `OrganizationalUnit` (lifecycle managed by the
application). Members join for a specific goal, unit is dissolved when
the goal is achieved. `AgentGoal` on the unit declares the shared objective.
No built-in lifecycle — the application creates and removes the unit.

**Federation** — Multiple `OrganizationalUnit`s, each represented by a
delegate agent. Delegates have `DELEGATES_TO` relationships between them.
The coordinator dispatches to delegates who dispatch internally. This is
the Orchestrator-Worker pattern.

**Congregation** — Long-lived `OrganizationalUnit` grouping similar agents.
No shared goal — agents are grouped by affinity (same capabilities, same
domain). `OrgQuery.byKind()` finds units by type.

**Society** — Multiple `OrganizationalUnit`s within a tenancy, governed by
shared `AgentConstraint`s. No single authority — constraints and norms
are the coordination mechanism. `Visibility.PUBLIC` constraints are visible
to all participants.

**Market** — Agents in a unit with `EXTENDED("bids-on")` or
`EXTENDED("offers")` relationships. No fixed authority — coordination
via competitive dynamics. The `EXTENDED` relationship kind with custom
`extendedKind` string handles arbitrary relationship semantics.

**Matrix** — An agent appears as a `Membership` in multiple `OrgUnit`s
simultaneously (e.g. a functional team and a project team). Dual
`REPORTS_TO` relationships to different managers. Already supported —
`Membership` is per-unit, and an agent can be a member of multiple units.

**Pipeline** — Sequential `DELEGATES_TO` chain. Agent A delegates to B,
B delegates to C. Each agent's output capability matches the next agent's
expected input. `RelationshipScope` scopes each delegation to the
specific capability being handed off.

---

## Normative Framework Alignment

Eidos org does not exist in isolation. It is the structural layer within
CaseHub's 5-layer normative accountability framework. Each layer is
owned by a different component:

### The 5-layer normative accountability framework

| Layer | Concern | Owner | What it knows |
|---|---|---|---|
| **L1 — Illocutionary** | What was said (speech act type) | Qhorus | QUERY, COMMAND, RESPONSE, DECLINE, HANDOFF, DONE, FAILURE, PROPOSE, JUDGMENT, STATUS, EVENT |
| **L2 — Commitment** | What was obligated (lifecycle) | Qhorus | 7-state obligation lifecycle, `causedByEntryId` chains |
| **L3 — Protocol** | Is the interaction following the expected pattern? | Choreography | DCR constraints, conformance checking, progressive formalism |
| **L4 — Temporal** | When obligations become stale | Qhorus | Deadline enforcement, obligation expiry |
| **L5 — Enforcement** | React to commitment/protocol outcomes | Engine | Drools-based evaluation, remediation |

Source: Choreography brief; engine ADR-0006.

### Qhorus speech acts → Searle's illocutionary categories

| Searle category | Qhorus types | Direction of fit | Deontic effect |
|---|---|---|---|
| **Directive** | QUERY, COMMAND, JUDGMENT | World-to-word | Creates obligation on receiver |
| **Commissive** | PROPOSE | Word-to-world | Sender binds to conditional action |
| **Assertive** | RESPONSE, STATUS, DONE, FAILURE | Word-to-world | Reports state, may discharge obligation |
| **Declarative** | HANDOFF | Both | Constitutively changes participation |
| **Expressive** | DECLINE | Neither | Refuses obligation with reason |
| **Observer** | EVENT | — | Telemetry only, not agent-visible |

Source: Qhorus `MessageType.java`; Searle, J.R. (1969). *Speech Acts*.

### Where eidos org sits in the framework

Eidos org provides the **structural preconditions** that the normative
framework operates on:

| Normative concern | Eidos org provides |
|---|---|
| **Who may speak to whom** | `AgentRelationship` + `RelationshipKind` — typed directed links define authority patterns |
| **What a participant may do** | `AgentCapability` on descriptors, `OrganizationalUnit.capabilities` on units |
| **What a participant must do** | `AgentGoal` (standing objectives), `AgentConstraint` (behavioral guardrails) |
| **Who may attest about whom** | `AttestationGrant` on relationships — dimensions, capability scope, signal types |
| **What roles exist in an interaction** | `Membership.role` (with vocabulary grounding) — maps to choreography roles |
| **How authority flows** | `SUPERVISES`, `DELEGATES_TO`, `ESCALATES_TO`, `REPORTS_TO` — the authority topology |
| **How trust propagates** | Discovery lineage via relationship chains (ADR-0006: trust derives from lineage, not assertion) |

### MOISE+ dimension mapping — extended for normative integration

| MOISE+ dimension | Eidos org concept | Normative framework connection |
|---|---|---|
| **Structural** — roles, groups, links | `OrgUnit`, `Membership`, `AgentRelationship`, `RelationshipKind` | L3 (Protocol): defines which roles interact in choreography |
| **Functional** — goals, plans, missions | `AgentGoal` (with `capabilities` mapping), `AgentCapability` | L1 (Illocutionary): capabilities determine what COMMAND/QUERY types an agent can receive |
| **Deontic** — obligations, permissions | `AgentConstraint` (HARD/SOFT severity), `AttestationGrant`, `Visibility` | L2 (Commitment): constraints bind to commitment lifecycle; attestation governs who may record COMPLIANT/VIOLATED signals |

MOISE+ does not have temporal or enforcement dimensions. CaseHub extends
the model with L4 (Qhorus deadline enforcement) and L5 (engine Drools
evaluation). The deontic dimension in eidos is richer than MOISE+'s
permission/obligation pair — it includes severity discrimination
(HARD vs SOFT), visibility (PUBLIC vs PRIVATE), and behavioral signal
accumulation (`BehavioralSignalStore`).

### Normative acts in organizational operations (ADR-0006)

Worker registration is itself a normative act — a declarative speech act
that constitutively creates a new participant. Trust derives from the
discovery chain, not from the worker's self-assertion:

| Discovery path | Deontic standing | Eidos modelling |
|---|---|---|
| Statically declared | Highest (baked in by system owner) | `AgentDescriptorRegistrar` in code or YAML |
| Provisioned by trusted provisioner | Inherits from provisioner's trust | `DELEGATES_TO` relationship from provisioner |
| Introduced by existing participant | Derived from introducer's chain | `EXTENDED("introduces")` relationship |
| Self-announced | Lowest (no voucher) | No relationship — agent registers itself |

The `causedByEntryId` chain in the normative ledger makes the discovery
path permanently traversable — who introduced whom, and who introduced
the introducer.

### Implications for archetype examples

Each archetype example should demonstrate not just the structural topology
but also the normative properties it enables:

| Archetype | Key normative property |
|---|---|
| **Hierarchy** | Authority chain determines COMMAND/DECLINE flow; supervisor attestation |
| **Professional Team** | Peer QUERY/RESPONSE; no COMMAND authority between members |
| **Tiered Escalation** | ESCALATES_TO chain defines HANDOFF path; each tier's DECLINE triggers next tier |
| **Divisional** | Cross-unit DELEGATES_TO carries delegation semantics; attestation scoped to delegated capability |
| **Orchestrator-Worker** | Coordinator issues COMMAND; workers DONE/FAILURE/DECLINE; coordinator aggregates |
| **Pipeline** | Sequential HANDOFF; each stage's DONE is next stage's trigger |
| **Advisory Board** | Advisors issue independent JUDGMENT; decision-maker RESPONSE synthesizes |

---

## Assessment Summary

The eidos org model is sufficient for all surveyed archetypes. No
structural extension is needed. The key enablers:

1. **`EXTENDED` relationship kind** — any relationship type the standard
   enum doesn't cover can be expressed as `EXTENDED("advises")`,
   `EXTENDED("mentors")`, `EXTENDED("bids-on")`. This is the escape
   hatch that makes the model open.

2. **Multi-unit membership** — an agent can belong to multiple units
   simultaneously, enabling matrix structures without model changes.

3. **Unit nesting** — `parentUnitId` creates recursive composition
   (holarchy) without a separate concept.

4. **Scoped relationships** — `RelationshipScope` means the same
   relationship kind can mean different things in different capability
   domains.

5. **Unit-level goals and capabilities** — groups have their own goals
   and capabilities independent of their members, enabling team-level
   objectives.

The model's expressiveness comes from composing these five mechanisms,
not from having a dedicated type for each paradigm. This is deliberate —
organizational structures are combinatorial, not categorical. A real
deployment might combine hierarchy (for supervision), federation (for
cross-team coordination), and coalition (for ad-hoc task forces) within
a single tenancy.

---

## What's Missing From Examples, Not From the Model

The current 6 examples cover 4 of the 10 Horling & Lesser paradigms.
Three high-value additions would bring coverage to 7:

| Priority | Archetype | Formal names | Why it matters |
|---|---|---|---|
| 1 | **Orchestrator-Worker** | Adhocracy (Mintzberg) / Federation (H&L) | Dominant production pattern in 2026 multi-agent systems |
| 2 | **Pipeline** | Pipeline (H&L) | Common in content generation, data processing, CI flows |
| 3 | **Advisory Board** | Coalition (H&L) | Architecture review, security audit, medical second-opinion |

The remaining 3 paradigms (Congregation, Society, Market) are less
common in current LLM agent deployments and can be deferred.

---

## References

- Mintzberg, H. (1979). *The Structuring of Organizations*. Prentice-Hall.
- Mintzberg, H. (1989). *Mintzberg on Management*. Free Press.
- Horling, B. & Lesser, V. (2004). A survey of multi-agent organizational
  paradigms. *The Knowledge Engineering Review*, 19(4), 281–316.
- Hübner, J.F., Sichman, J.S. & Boissier, O. (2002). A model for the
  structural, functional, and deontic specification of organizations in
  multiagent systems. *AAMAS 2002*, pp. 118–128.
- Ferber, J., Gutknecht, O. & Michel, F. (2004). From agents to
  organizations: an organizational view of multi-agent systems. *AOSE 2003*,
  LNCS 2935, pp. 214–230.
- Koestler, A. (1967). *The Ghost in the Machine*. Hutchinson. (holarchy concept)
- Searle, J.R. (1969). *Speech Acts: An Essay in the Philosophy of Language*. Cambridge University Press.
- Searle, J.R. (1979). *Expression and Meaning: Studies in the Theory of Speech Acts*. Cambridge University Press.
- Singh, M.P. (1999). An ontology for commitments in multiagent systems.
  *Artificial Intelligence and Law*, 7(1), 97–113.
- Hilpinen, R. (ed.) (1981). *New Studies in Deontic Logic*. Reidel. (deontic logic foundations)
- Munindar P. Singh & Michael N. Huhns (2005). *Service-Oriented Computing:
  Semantics, Processes, Agents*. Wiley. (commitment-based coordination)
