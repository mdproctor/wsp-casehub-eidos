# eidos Session Handover - 2026-08-04

**Previous handover:** git show HEAD~1:HANDOFF.md

## What Changed This Session

- Closed #136 — ran minimal briefing experiment via Vertex AI (4 runs: Sonnet/Sonnet ×2, Haiku/Haiku, Haiku/Sonnet). Framework contributes +0.06–0.10 TAA independently. Results in `research/2026-08-04-briefing-experiment-results.md`.
- Closed #126 — FunctionActivationJudge Ni/Ne calibration. Enriched judge prompt with contrastive discriminators, rewrote Ni scenarios to force convergent answers, added confidence field. Ni accuracy went from 0% to 72%.
- Closed #94 — ReactiveAgentStateStore parity tests. Closed as obsolete (reactive tier deleted in parent#384).
- Built VertexChatModel — direct Vertex AI rawPredict integration via java.net.http.HttpClient. No new dependencies. CDI producer activated by build property.
- Built split-model eval — agent and judge can use different models via `casehub.eval.vertex.judge-model`. Validated: framework signal survives a stricter judge.
- Fixed CDI bean discovery for casehub-platform-agent-claude via `quarkus.index-dependency`.
- Landed as 414f562 on main.

## Immediate Next Step

Blog entry "The Framework Earns Its Keep" written and published. No immediate blocking work. Pick from backlog.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #78 | Example: learned specialization lifecycle | M | Med | Epic #82 |
| #80 | Example: full probe pipeline | M | Med | Epic #82 |
| #81 | Example: cost-aware multi-agent routing | M | Med | Epic #82 |

## References

- Research: research/2026-08-04-briefing-experiment-results.md
- Blog: blog/2026-08-04-mdp01-the-framework-earns-its-keep.md
- Downstream: casehubio/engine#860 (GoalFailureRecorder per-goal discrimination)
