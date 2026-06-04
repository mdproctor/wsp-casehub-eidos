---
layout: post
title: "The timestamp that was always wrong"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [graph, api, cdi, ieee754]
---

`AgentOutcome` has had an `observed_at` column in its table since the knowledge graph shipped. What it didn't have was any way to put a meaningful value there. The entity was calling `Instant.now()` at persistence time — so every outcome was recorded as "when it hit the database", which could be anywhere from milliseconds to minutes after the agent actually produced the result. And `toRecord()` dropped the field entirely on the way back out, so callers never even saw it.

The fix looks small — add `Instant observedAt` as the fourth field of `AgentOutcome`, wire it through `from()` and `toRecord()`, update the five test call sites. The real complexity was in defining what `observedAt` actually means.

For live recording it's straightforward: `Instant.now()` at the call site, captured before entering any async chain. For backfill — which is still a stub, still waiting on the casehub-ledger integration — the right value will be `AttestationRef.attestedAt()`, the closest proxy we have for when a historical task actually completed. Those timestamps can differ from the actual task completion time by an arbitrary amount, which the spec now acknowledges directly rather than glossing over.

The compact constructor ended up doing more validation than I expected. `observedAt` is non-null, obviously. But `confidence` has a `CHECK (confidence BETWEEN 0 AND 1)` in the migration that was never enforced at the Java layer. I added the range guard — and then code review caught that it was still silently wrong. `Double.NaN < 0.0` is `false` in Java. `Double.NaN > 1.0` is also `false`. IEEE 754 comparison with NaN always returns false, so a bare `< 0.0 || > 1.0` guard accepts NaN without throwing. `Double.POSITIVE_INFINITY` would have been caught correctly; NaN wouldn't. The fix is one token: `Double.isNaN(confidence) ||` prefixed to the condition.

I'd have caught that eventually. It would have taken a very specific test case. The code review found it before the branch closed.

`ReactiveAgentGraphQuery` was simpler — the interface only had two of the four blocking methods. Adding `historyByCapability` and `attestationsFor` to the interface and the bridge was mechanical, except for one CDI detail I'd forgotten. The bridge is field-injected, so Weld needs the no-arg constructor to instantiate the bean before running `@Inject`. If you add any explicit constructor — even a package-private one for testing — the compiler's generated no-arg disappears. The application fails at startup, and nothing in the compilation tells you why. Both constructors have to be there: `public BlockingToReactiveGraphBridge() {}` for CDI, and the package-private one for plain Mockito tests.

Two issues closed, two garden entries written (the NaN gotcha, the CDI no-arg loss). The backfill wiring is still a stub — that comes when the engine integration lands.
