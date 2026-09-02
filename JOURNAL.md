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
