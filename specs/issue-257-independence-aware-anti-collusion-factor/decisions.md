## D1: Cross-repo strategy

**Choice:** Both changes in this session — update RelevanceComposer in ledger AND build the eidos factor
**Alternatives:**
- Composer change first — file a ledger issue, build eidos factor after. Cleaner separation but delays delivery.
- Stateful workaround — accumulate state across per-attestation assess() calls. Fragile, computation pass boundaries are implicit.
**Rationale:** The composer change is backward-compatible (~10 lines), the SPI already has the assessBatch() default method, and both repos are locally available. No reason to serialize.
**Trade-offs:** Cross-repo session, two commits to track
**Sources:** RelevanceComposer.java (ledger-core), AttestationRelevanceFactor.java (ledger-api)
**Exploration:** quick
**Status:** captured

## D2: Same-unit definition

**Choice:** org-distance = 1 in the attestation's relevance snapshot (attestor is in the same OrgUnit as the scored agent)
**Alternatives:**
- Configurable threshold (distance ≤ N) — more flexible but blurs the concept and adds config surface
- Weighted by distance — near attestors count fractionally. Most nuanced but hardest to reason about.
**Rationale:** First-principles investigation of the LCA-based org-distance algorithm. Distance 1 is a crisp, well-defined concept: members of the same OrganizationalUnit. This is where operational context overlap is maximal and evidence correlation is highest. Distance 2+ attestors are in structurally separate units with different observation vantage points.
**Trade-offs:** Doesn't capture sibling-unit correlation (distance 2). Acceptable because the concern is specifically about intra-unit amplification.
**Sources:** DefaultOrgAwareDispatch.java:80-124 (LCA algorithm), OrgDistanceAttestationEnricher.java (writes snapshot), DefaultOrgAwareDispatchTest.java:191-301
**Exploration:** quick
**Status:** captured

## D3: Cap mechanics

**Choice:** Precise share cap — compute exact weight to hit target share
**Alternatives:**
- Smooth decay curve — continuous penalty increasing with over-representation. Softer but resulting share depends on distribution, harder to reason about.
**Rationale:** Simple closed-form formula: `w = cap × cross_count / ((1-cap) × same_count)`. Transparent semantics — the configured cap is the actual resulting share. The existing RelevanceComposer floor (0.5) prevents extreme composed weights.
**Trade-offs:** Can produce low per-attestation weights in extreme imbalance (e.g., 9/10 same-unit → w=0.167), but the floor bounds the composed product.
**Sources:** RelevanceComposer.java (floor mechanism), LedgerConfig.RelevanceConfig (minWeight default 0.5)
**Exploration:** quick
**Status:** captured

## D4: All-same-unit edge case

**Choice:** Return NEUTRAL for all attestations when 100% are same-unit (zero cross-unit)
**Alternatives:**
- Apply a fixed penalty (e.g., 0.8) — signals reduced independence even without cross-unit comparison
**Rationale:** The cap's purpose is to prevent same-unit voices from drowning out cross-unit voices. When there are no cross-unit voices, there is nothing to protect. Penalizing would reduce evidence mass with no redistribution target.
**Trade-offs:** A unit could self-reinforce without any penalty if no cross-unit attestors ever attest. Acceptable — the concern is relative over-representation, not absolute homogeneity.
**Sources:** Issue #257 description ("cap same-unit share")
**Exploration:** quick
**Status:** captured

## D5: RelevanceComposer refactor

**Choice:** Replace per-attestation loop with per-factor assessBatch() call
**Alternatives:**
- Try batch, fall back to per-item — over-defensive, the default method can't fail independently of assess()
**Rationale:** The default assessBatch() already loops assess() per item. Existing factors work unchanged. One code path, no fallback needed.
**Trade-offs:** None — strictly better than the current implementation
**Sources:** AttestationRelevanceFactor.java (default assessBatch), RelevanceComposer.java
**Exploration:** quick
**Status:** captured
