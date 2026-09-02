# HANDOFF — eidos

## Last Session

Audited eidos DSL, annotations, and YAML systems for composition gaps, parity issues, and yaml-core adoption opportunities. Ran three parallel IntelliJ-driven audits. Created epic #153 with 15 child issues across four categories.

Completed #154 — layered org-annotations on eidos-annotations: deleted duplicate NameDerivation, extracted shared AnnotationProcessorUtils, wired coordinated processing via EidosAnnotationProcessedBuildItem, added orphan warnings, switched org recorder to createWith() pattern. Full test suite green.

## What's Next

| Issue | Title | Scale | Complexity |
|-------|-------|-------|------------|
| #155 | Align org annotation container naming — Supervises.List to top-level | XS | Low |
| #156 | Add orphan warnings to org-annotations processor | S | Low |
| #157 | Org annotation field parity (tenancyId, capabilities, goals, constraints, attestation, scope) | L | Med |
| #158 | Per-descriptor tenancyId in annotations | S | Low |
| #159 | AgentGoal.attributes — surface or remove | S | Low |
| #160 | YAML goal/constraint defaults | XS | Low |
| #161 | Expand parity tests | M | Low |
| #162 | Adopt yaml-core module system | L | High |
| #163 | Add preprocessing to org YAML | M | Med |
| #164-#166 | ForEachAdapter refs, IterationValueExpander, DeferredPrefixHandler | XS-S | Low-Med |
| #167 | Annotation composition — @AgentProfile | L | High |
| #168 | Agent team DSL | M | Med |

All edits via IntelliJ MCP per user preference. #155 is the natural next — XS rename of `Supervises.List` to top-level `Supervisions`.

## Key Decisions

- **Quarkus extension chain**: can't exclude transitive deployment deps — Quarkus extension validation requires the full deployment counterpart chain. Org-annotations tests need H2 + datasource config because transitive eidos-deployment brings JPA entities.
- **Org-annotations layers on eidos-annotations**: org-annotations-runtime depends on eidos-annotations; deployment depends on eidos-annotations-deployment. Shared AnnotationProcessorUtils lives in eidos-annotations-deployment.
- **Inconsistency alignment**: eidos-annotations is the more mature implementation in all cases — org-annotations aligns to it.

## References

- `JOURNAL.md` — design journal with session notes and decision log
- `.plan` — work queue with 15 issues, position 0/15, #154 done
- Epic #153 — full scope checklist on GitHub
