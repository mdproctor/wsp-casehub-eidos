# HANDOFF — eidos

## Last Session

Disposition fixes batch — landed 4 of 6 issues on `issue-239-disposition-fixes-batch`, squashed to 4 commits, published.

| # | Issue | What landed |
|---|-------|-------------|
| #239 | AmpdFacet canonicalization | `AmpdFacets` utility owns 13 canonical AMPD names; `AmpdFacet` and `AmpdFacetShift` canonicalise on construction; `AgentDisposition` rejects duplicate facets |
| #237 | SignalStore int→double | `activationCounts()` returns `Map<String, Double>`; all stores updated; schema column `DOUBLE PRECISION`; decay no longer truncates |
| #241 | SignalStore concurrent safety | JPA atomic upsert with `REQUIRES_NEW`; bulk SQL for decay; in-memory record key + atomic `compute()` for decay; `computeIfPresent` for `clear` |
| #242 | Swap dead zone | Hysteresis margin (`SwapMarginPreference`, default 0.03) on DOMINANT_AUXILIARY_SWAP; property test proves no oscillation |

Code review finding: JPA `recordActivation` catch-and-retry removed (broken — persistence context invalidated after `PersistenceException`). Docs synced for the int→double change.

Blog: "Four Cracks in the Personality Layer" — `docs/blog/2026-11-18-mdp02-four-cracks-in-the-personality-layer.md`

## Immediate Next Step

`work start #235` — registry-backed `DispositionProfileStore` (S/Med). A real implementation backed by `AgentRegistry` that persists disposition profile updates and AMPD facet writes. Hard prerequisite for casehubio/neocortex#1060.

Key requirements from the issue:
- Read-modify-register pattern in one transaction with lost-update guard (pessimistic lock or optimistic version)
- `updateFacets(agentId, tenancyId, Map<String, Integer> absoluteScores)` on the SPI
- Replace `NoOpDispositionProfileStore` as default bean
- `AgentDescriptorUpdated` event fires with previous and current descriptors

After #235: #243 (model-resolving `PromptAssembler` variant, parked — wait for consumer demand).

## Issues Filed This Session

- **#244** — `chore: signal store polish` (XS/Low): `Locale.ROOT` in `AmpdFacets.canonicalise`, native SQL upsert for JPA, `activationCounts` → `activationWeights` rename consideration

## Decisions Made

- D1: `AmpdFacets` utility class (not enum) — facet names have spaces; enum constants would diverge from display names
- D2: `double` throughout signal store (not integer+projection) — simpler, no truncation, `ValenceCounts` rounds in its bridge
- D3: Hysteresis margin compares effective-to-effective (not effective-to-base) — eliminates oscillation by construction; margin 0.03 is tenant-configurable
- D4: JPA retry removed (not fixed) — catch after `PersistenceException` runs against an invalidated persistence context; update-first path covers the common case

## Linked Work

- Master unification issue: casehubio/neocortex#1105 (eidos step done; neocortex and agency steps remain)
- neocortex#1060 — personality evolution design that prompted these fixes
- #244 — signal store polish (filed this session)

## Known Issues

- Fork mirror occasionally fails to follow origin/main on publish — catches up on next run
