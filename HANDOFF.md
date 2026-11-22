# HANDOFF — eidos

## Last Session

**#257** — Independence-aware anti-collusion factor. Two-repo change:

1. **Ledger** (`/Users/mdproctor/claude/casehub/ledger`, branch `issue-239-independence-relevance-factor`): Refactored `RelevanceComposer` to call `assessBatch()` per factor instead of `assess()` per attestation. Backward-compatible — the default `assessBatch()` loops `assess()`. 11 tests pass (9 existing + 2 new: batch factor composition, partial batch map).

2. **Eidos** (branch `issue-257-independence-aware-anti-collusion-factor`): New `IndependenceAwareRelevanceFactor` in `org-runtime/trust/`. Caps same-unit attestation share at a configurable threshold (default 60%) using `assessBatch()` override. Formula: `w = cap × crossCount / ((1-cap) × sameCount)`. 19 tests pass. NaN guard added after code review.

**Blog:** "The Unit That Vouches Too Much" — written, promoted to project.
**Specs:** Design spec + decisions promoted to project.
**Squash:** eidos branch squashed 3→2 commits, review recorded.
**Ledger not yet pushed** — 2 commits on `issue-239-independence-relevance-factor` need landing via ledger's work-end.

## Immediate Next Step

#258 — ARC42STORIES attestation relevance update. Scale S, Complexity Low. Same domain — update ARC42STORIES.MD to reflect the new independence-aware factor.

## Queue (3 open)

| # | Title | Scale | Complexity | Blocked by |
|---|-------|-------|------------|------------|
| #258 | ARC42STORIES attestation relevance update | S | Low | — |
| #259 | standardise contract test cleanup | XS | Low | — |
| #260 | adopt contract tests for in-memory stores | XS | Low | — |

## References

- Design spec: `docs/specs/issue-257-independence-aware-anti-collusion-factor/`
- Blog: `docs/blog/2026-11-22-mdp01-the-unit-that-vouches-too-much.md`
- Ledger change: `ledger-core/src/main/java/io/casehub/ledger/core/trust/RelevanceComposer.java`
- Eidos factor: `org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java`
