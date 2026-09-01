# Design Journal — issue-150-org-model

## 2026-09-01 — Foundation + surfaces

Built the agent organizational model from research through implementation.

**Design:** Three-layer architecture — descriptors (who), organization (how they relate), deployment (where they run). Built on casehub-desiredstate for declarative reconciliation. 11 design decisions captured in research doc. Key choices: both units and pairwise relationships as peers, generic vocabulary-grounded unit kinds, opt-in scoped attestation grants, unbounded hierarchy with cycle detection.

**Implemented:**
- `org-api` — 8 POJO types (OrganizationalUnit, Membership, AgentRelationship, RelationshipKind, RelationshipScope, AttestationGrant, OrgQuery, OrgRegistry SPI, OrgRegistrar SPI) + OrgStructure compositional DSL builder
- `org-memory` — InMemoryOrgRegistry with cycle detection
- `org-runtime` — OrgGoalCompiler + NodeSpec types (desiredstate integration), EidosOrgModule + ClasspathYamlOrgRegistrar (YAML surface)
- `examples/org-scenarios` — 4 domain examples (Gastown, Review Team, Customer Support, Clinical Triage) with capability matrix, both DSL and YAML

**Deferred:** Annotation surface (@OrgUnit, @Supervises — Quarkus build extension + recorder, M/High complexity).

**Key insight from session:** No JPA persistence needed — org structure is declared in YAML (the durable store), loaded at boot via registrar, and held in-memory. Desiredstate handles reconciliation state. This differs from AgentDescriptor which has JPA because descriptors can be created programmatically at runtime.
