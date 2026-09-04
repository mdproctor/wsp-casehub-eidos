# HANDOFF — eidos

## Last Session

Continued epic #153 — DSL, annotations & YAML composition audit. Closed 7 of 15 child issues across two sessions. This session completed the quick wins: container naming (#155), goal attributes surfaced (#159), YAML defaults (#160), expanded parity tests (#161). Closed #156 (already done) and #158 (tenancyId is infrastructure, not identity — won't fix).

## What's Next

| Issue | Title | Scale | Complexity |
|-------|-------|-------|------------|
| #157 | Org annotation field parity (capabilities, goals, constraints, attestation, scope) | L | Med |
| #162 | Adopt yaml-core module system for descriptor YAML | L | High |
| #163 | Add preprocessing to org YAML | M | Med |
| #164 | ForEachAdapter getReferences/withReferences | S | Med |
| #165 | Adopt IterationValueExpander | S | Low |
| #166 | Evaluate DeferredPrefixHandler | XS | Low |
| #167 | Annotation composition — @AgentProfile | L | High |
| #168 | Agent team DSL | M | Med |

Recommended next: #157 (org annotation field parity) — it uses the layering infrastructure from #154 and the parity tests from #161 will immediately catch gaps.

## Key Decisions

- **tenancyId is infrastructure, not identity** (#158 closed). The annotation path's config-driven approach is correct. YAML per-descriptor tenancyId is an escape hatch, not the primary mechanism.
- **AgentGoal.attributes is live** — has JPA, comparator, and test support. Was just missing from declaration paths. Now surfaced in both YAML and annotations.
- **Quarkus extension chain**: can't exclude transitive deployment deps. Org-annotations tests need H2 + datasource config.

## References

- `JOURNAL.md` — design journal with session notes and decisions
- `.plan` — work queue, 7/15 done
- Epic #153 — full scope checklist on GitHub
