# Design Journal — issue-153-dsl-annotations-yaml-audit

## 2026-09-02 — Audit and epic creation, #154 layering complete

### Session scope
Comprehensive audit of eidos DSL, annotations, and YAML systems. Three parallel
audit forks investigated: yaml-core module system usage, annotation composition
patterns, and YAML/annotation parity.

### Key findings
1. **yaml-core module system completely unused** — eidos uses forEach/variables/CSV
   but not modules (YamlModule, YamlImport, ModuleExpander, typed params, outputs)
2. **Org YAML has zero preprocessing** — raw Jackson, no variables/forEach/when
3. **No annotation composition** — no @Inherited, no meta-annotations, no profiles
4. **Two annotation extensions are independent peers** — should layer (org on eidos)
5. **Parity gaps** — AgentGoal.attributes unreachable, YAML defaults missing,
   org annotations missing 6 fields, tenancyId behavioral divergence

### Epic #153 created
15 child issues across 4 categories: annotation layering (5), parity (3),
yaml-core adoption (5), annotation composition & DSL (2).

### #154 — Layer org-annotations on eidos-annotations (DONE)
- Deleted duplicate NameDerivation; org-annotations now imports from eidos-annotations
- Extracted AnnotationProcessorUtils with shared stringValue()/enumValue()
- Org processor consumes EidosAnnotationProcessedBuildItem (Optional parameter)
- Added orphan warnings for @OrgMembers/@Supervises/@OrgRelationships without @OrgUnit
- Switched org recorder from Supplier to createWith() (BeanCreator pattern)
- Added H2 + datasource test config (required by transitive eidos-deployment JPA entities)

### Decision: Quarkus extension dependency chain
Adding eidos-annotations-deployment as a dependency to org-annotations-deployment
transitively brings in eidos-deployment, which brings JPA entities and Hibernate.
Tried excluding — Quarkus extension validation rejects missing deployment counterparts.
Resolution: keep full chain, configure H2 datasource in org tests.

## 2026-09-03 — Quick wins batch: #155, #156, #159, #160, #161; #158 closed

### #155 — Container naming (DONE)
Renamed `Supervises.List` → top-level `Supervisions` annotation. Aligns with
eidos-annotations convention (AgentCapabilities, AgentTemplates).

### #156 — Orphan warnings (CLOSED)
Already implemented in #154's `warnOrphanAnnotations()` method.

### #158 — Per-descriptor tenancyId (CLOSED — won't fix)
Design discussion: tenancyId is infrastructure plumbing for data partitioning,
not agent identity. Application developers shouldn't think about it. The annotation
path's current behavior (tenancyId from config property) is correct. Adding it to
`@Identity` would leak deployment infrastructure into identity definitions.

### #159 — AgentGoal.attributes (DONE)
Field was NOT dead — has JPA persistence, comparator, and 5 tests. Gap was that
neither YAML nor annotations could set it. Surfaced in both paths:
- YAML: `AgentDescriptorDeserializer` now parses `attributes` map
- Annotations: new `@GoalAttribute` annotation, `@AgentGoalDef.attributes()` field

### #160 — YAML goal/constraint defaults (DONE)
YAML deserializer now defaults to PRIMARY/PUBLIC for goals and PUBLIC/HARD for
constraints when fields are omitted — matching annotation defaults.

### #161 — Expand parity tests (DONE)
Added capability, template, and reverse checks to `AnnotationParityTest`.
New `OrgAnnotationParityTest` covers @OrgUnit, @OrgMemberDef, @OrgRelationshipDef
with forward and reverse checks. Infrastructure fields excluded from reverse checks.
