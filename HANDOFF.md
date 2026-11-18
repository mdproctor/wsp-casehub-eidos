# HANDOFF — eidos

## Last Session

Closed #235 — `LivedDispositionSource` SPI and `findEffective()` on `AgentRegistry`. The original issue asked for a registry-backed `DispositionProfileStore` with persistence. Design review revealed that neocortex already has a full composition layer (`LivedComposition`, `PersonalityStateService`, three-channel facet model), so a registry-backed write store would cause double-composition and lose the richer neocortex state. The fix was a read-only query SPI instead.

| What | Detail |
|------|--------|
| `LivedDispositionSource` | `@FunctionalInterface` in api/ — `resolveEffective(agentId, tenancyId, authored)` returns composed `AgentDisposition`. Authored disposition passed as parameter to avoid callback chain. |
| `findEffective()` | Default method on `AgentRegistry` delegates to `findById()`. CDI impls (`InMemoryAgentRegistry`, `JpaAgentRegistry`) compose via `Instance<LivedDispositionSource>`. |
| `findById()` | Unchanged — returns authored descriptor only. `EidosDescriptorViewSource` continues getting clean values. |
| `NoOpLivedDispositionSource` | `@DefaultBean` via `EidosCoreProducer` — returns authored unchanged. |
| Tests | 4 new in `AgentRegistrySpiTest`: default delegation, empty result, composition with lived source, no-op identity semantics |

Blog: "The Store That Shouldn't Store" — `docs/blog/2026-11-18-mdp03-the-store-that-shouldnt-store.md`

## Immediate Next Step

`work start #243` — model-resolving `PromptAssembler` variant. Parked — wait for consumer demand.

Queue after #243:
- #246 — extend `EidosCognitiveProfileBridge` to compose AMPD facets in `findEffective()` (S/Med)
- #247 — clarify `DispositionProfileStore` role after `LivedDispositionSource` SPI (XS/Low)

## Issues Filed This Session

- **#246** — `feat: extend EidosCognitiveProfileBridge to compose AMPD facets in findEffective()` (S/Med): bridge composes disposition terms but not AMPD facets; facet composition only happens inside `LivedComposition.compose()`. Deferred from #235.
- **#247** — `chore: clarify DispositionProfileStore role after LivedDispositionSource SPI` (XS/Low): with `LivedDispositionSource` as the read path, `DispositionProfileStore`'s write-only SPI is redundant with neocortex's internal persistence. Options: deprecate, rename to listener, or document as-is.

## Decisions Made

- D1: Read-only SPI over registry-backed store — neocortex already handles authored/evolved separation via `LivedComposition.fingerprint()` baseline detection and `PersonalityStateService` persistence. A parallel eidos store would fight the existing composition.
- D2: Authored disposition as parameter — avoids circular dependency (`LivedDispositionSource` → `AgentRegistry` → `LivedDispositionSource`). Implementation is a pure function.
- D3: `findById()` stays authored-only — prevents double-composition when neocortex's `EidosDescriptorViewSource` feeds `EffectiveCognitiveDefaults`.
- D4: No reactive mirror — `PersonalityStateService` is cache-first (ConcurrentHashMap); synchronous lookup is effectively non-blocking.
- D5: AMPD facet composition deferred (#246) — existing bridge doesn't compose facets; three-channel model (homeOffset/displacement/promoted) needs careful projection.

## Linked Work

- casehubio/neocortex#1060 — personality evolution design; this SPI is the hard prerequisite
- #246 — AMPD facet composition follow-up
- #247 — `DispositionProfileStore` role clarification
- #244 — signal store polish (from prior session)

## Known Issues

- Fork mirror occasionally fails to follow origin/main on publish — catches up on next run
