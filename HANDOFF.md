# eidos Session Handover — 2026-07-26

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Closed eidos#99 — descriptor templates: full SPI (DescriptorTemplate, TemplateRef, TemplateRegistry, TemplateRegistrar), CdiTemplateRegistry with @PostConstruct discovery, ClasspathYamlTemplateRegistrar, InMemoryTemplateRegistry, three-layer validation, single-pass regex substitution, render pipeline integration, JPA persistence (V7), end-to-end scenario tests. Design-reviewed (10 rounds, 19 issues, 15 verified). Garden entry GE-20260725-a4aa6c (CDI @Alternative gotcha).
- Created engine#670 epic grouping five engine-side eidos adoption issues (#632, #638, #639, #645, #647).
- Slot 35 created for eidos#100 (goals/aims/constraints) — parallel work.

## Immediate Next Step

#96 should be closed (covered by #97). After that, pick from epic #82 examples: #78 (learned specialization lifecycle) or #80 (full probe pipeline). Run `/work` to start.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #78 | Example: learned specialization lifecycle (SUCCESS/DECLINE signals) | M | Med | Epic #82 |
| #80 | Example: full probe pipeline (all five decision steps) | M | Med | Epic #82 |
| #81 | Example: cost-aware multi-agent routing | M | Med | Epic #82 |
| #92 | Engine observation logic for delegation/escalation compliance | M | High | |
| #93 | track: Engine adoption of behavioral contracts (engine#647) | M | Low | Tracking — blocked by engine#647 |
| #94 | Reactive tests for examples | S | Low | Blocked by #78, #80, #81 |
| #96 | Eval: isolate A2A_CARD declared-description fallback path | S | Med | Should be closed — covered by #97 |
| #101 | Goal-based querying, routing, advanced goal use cases | M | Med | Follow-on from #100 |
| #102 | AgentCapability name uniqueness not enforced | S | Low | |
| #103 | ConstraintSeverity: hard vs soft constraints | S | Med | |

## References

- Design review: `~/adr/casehub-eidos/descriptor-templates-20260725-203147/`
- Blog: `blog/2026-07-25-mdp01-composable-prose-templates.md`
- Spec: `specs/issue-99-descriptor-libraries-templates/2026-07-14-descriptor-templates-design.md`
- Garden: `GE-20260725-a4aa6c` — CDI @Alternative @PostConstruct gotcha
