# HANDOFF — eidos

## Last Session

Closed 4 issues across the disposition/personality evolution SPI boundary:

| # | Issue | What landed |
|---|-------|-------------|
| #235 | LivedDispositionSource SPI | Closed as superseded by #245 |
| #245 | LivedDispositionSource SPI (canonical) | `LivedDispositionSource` `@FunctionalInterface` in api/; `findEffective()` on `AgentRegistry`; `NoOpLivedDispositionSource` `@DefaultBean`. Read-only query SPI — neocortex implements, eidos provides the bridge. `findById()` stays authored-only. |
| #243 | Model-resolving PromptAssembler | Closed as not planned — parked by design. Model resolution is a dispatch concern; extract if consumers demonstrate boilerplate. |
| #247 | DispositionProfileStore → DispositionProfileListener | Pure rename: interface, method (`update` → `onProfileEvolved`), NoOp, producer. `@FunctionalInterface` removed. Javadoc clarifies notification-not-persistence semantics with `@see LivedDispositionSource`. |

Transferred: #246 (AMPD facet composition) → casehubio/neocortex#1135. The scope was entirely neocortex code.

Blog: "The Store That Shouldn't Store" — `docs/blog/2026-11-18-mdp03-the-store-that-shouldnt-store.md` (revised to include the rename coda)

## Queue

Drained. No active eidos issues.

## Decisions Made

- D1: Read-only SPI over registry-backed store — neocortex already handles authored/evolved separation via `LivedComposition`
- D2: Authored disposition as parameter to `resolveEffective()` — avoids circular dependency
- D3: `findById()` stays authored-only — prevents double-composition in neocortex
- D4: No reactive mirror — `PersonalityStateService` is cache-first
- D5: AMPD facet composition deferred to neocortex (transferred as #1135)
- D6: `DispositionProfileStore` renamed to `DispositionProfileListener` — aligns name with actual semantics (notification, not persistence)
- D7: #243 (model-resolving assembler) closed as not planned — model resolution is dispatch, not assembly

## Cross-Repo Impact

The `DispositionProfileStore` → `DispositionProfileListener` rename was verified clean: zero references in neocortex, zero in engine. No cross-repo breakage.

## Linked Work

- casehubio/neocortex#1060 — personality evolution design; `LivedDispositionSource` is the hard prerequisite
- casehubio/neocortex#1135 — AMPD facet composition (transferred from eidos#246)
- #244 — signal store polish (from earlier session, still open)

## Known Issues

- Fork mirror occasionally fails to follow origin/main on publish — catches up on next run
