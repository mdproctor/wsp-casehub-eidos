# Independence-Aware Attestation Relevance Factor — Design

**Issue:** casehubio/eidos#257
**Branch:** `issue-257-independence-aware-anti-collusion-factor`

## Problem

The existing `OrgProximityRelevanceFactor` is accuracy-biased: same-unit attestors (org-distance = 1) receive weight 1.0 because they share operational context. This widens the anti-gaming gap identified in the ledger ARC42 §Active Risks: a unit of agents whose positive attestations correlate will produce artificially higher trust scores than the same performance attested by cross-unit peers.

The accuracy bias is correct — same-unit attestors *do* have better context. But when a disproportionate share of attestations come from the same unit, the evidence is correlated and the resulting trust score is inflated by repeated measurement of the same signal.

## Solution

A complementary `AttestationRelevanceFactor` that caps the same-unit share of a group's evidence mass. When same-unit attestations exceed a configured threshold, their weights are reduced to bring the share to the cap. Cross-unit and unknown-relationship attestations are unaffected.

This requires a prerequisite change to the ledger's `RelevanceComposer` to call `assessBatch()` per factor instead of `assess()` per attestation, since the independence factor needs to see the full group to compute share ratios.

## Deliverables

### 1. Ledger: RelevanceComposer batch refactor

**Repo:** casehub-ledger (`/Users/mdproctor/claude/casehub/ledger`)
**File:** `ledger-core/src/main/java/io/casehub/ledger/core/trust/RelevanceComposer.java`

Refactor `assess(List<AttestationRelevanceContext>)` from:

```
for each context:
    for each factor:
        product *= factor.assess(context)
    result[context.id] = max(floor, product)
```

To:

```
for each factor:
    batchResult = factor.assessBatch(allContexts)
    accumulate batchResult weights per attestation

for each context:
    result[context.id] = max(floor, accumulatedProduct)
```

**Backward compatibility:** The default `assessBatch()` on `AttestationRelevanceFactor` loops `assess()` per item. Existing factors (OrgProximityRelevanceFactor) work unchanged — they don't override `assessBatch()`, so the default delegates to their `assess()` method.

**New test:** Verify that a factor overriding `assessBatch()` sees the full batch and its results compose correctly with per-attestation factors.

### 2. Eidos: IndependenceAwareRelevanceFactor

**Repo:** casehub-eidos (`/Users/mdproctor/claude/casehub/eidos`)
**Module:** `org-runtime`
**Package:** `io.casehub.eidos.org.runtime.trust`

#### Class: IndependenceAwareRelevanceFactor

`@ApplicationScoped`, implements `AttestationRelevanceFactor`.

**Configuration:**

| Property | Type | Default | Validation |
|----------|------|---------|------------|
| `casehub.eidos.trust.independence.max-same-unit-share` | `double` | `0.6` | Must be in (0.0, 1.0] |

#### Algorithm

`assessBatch(List<AttestationRelevanceContext>)`:

1. **Partition** attestations by reading `org-distance` from `ctx.snapshot()`:
   - **Same-unit:** org-distance = 1
   - **Cross-unit:** org-distance > 1
   - **Unknown:** no snapshot, missing key, non-numeric, or distance ≤ 0

2. **Compute share:** `sameUnitFraction = sameCount / (sameCount + crossCount)`. Unknown attestations are excluded from the denominator — they have unknown organizational relationship and cannot be classified.

3. **Apply cap:**
   - If `crossCount == 0` → all NEUTRAL. No cross-unit voices exist to protect.
   - If `sameUnitFraction <= maxShare` → all NEUTRAL. Within cap.
   - If over-represented → same-unit attestations get weight:
     ```
     w = maxShare × crossCount / ((1 - maxShare) × sameCount)
     ```
     Cross-unit and unknown attestations get NEUTRAL (1.0).

4. **Reason string:** e.g. `"independence: same-unit 8/10 (80%) > cap 60%, weight: 0.375"`

**`assess(AttestationRelevanceContext)`:** Returns `RelevanceAssessment.NEUTRAL`. The per-item method satisfies the interface contract but the factor's semantics are batch-only.

**`name()`:** Returns `"independence"`.

#### Why org-distance = 1 defines "same-unit"

The org-distance algorithm is LCA-based (Lowest Common Ancestor). Distance 1 means two distinct agents are members of the same `OrganizationalUnit`. This is a crisp boundary:

- Distance 1: shared operational context — same team, same work, same observations. Evidence is correlated.
- Distance 2+: structurally separate units with different observation vantage points, even if they share a parent division.
- Distance 0: self-attestation (enricher writes it but proximity factor returns NEUTRAL).
- Distance -1: unknown/uncomputable (enricher skips writing to snapshot).

Multi-unit membership: the enricher writes the minimum distance across all unit pairings. An agent in both "eng" and "sales" has distance 1 to any "eng" member. This correctly counts them as same-unit for anti-collusion purposes.

#### Cap formula derivation

Target: same-unit attestations contribute at most `cap` fraction of total weighted evidence mass.

Given `S` same-unit attestations (each weight `w`) and `C` cross-unit attestations (each weight 1.0):

```
S × w / (S × w + C × 1.0) = cap
S × w = cap × (S × w + C)
S × w × (1 - cap) = cap × C
w = cap × C / ((1 - cap) × S)
```

Example: 8 same-unit, 2 cross-unit, cap 60%:
- `w = 0.6 × 2 / (0.4 × 8) = 0.375`
- Same-unit mass: 8 × 0.375 = 3.0
- Cross-unit mass: 2 × 1.0 = 2.0
- Same-unit share: 3.0 / 5.0 = 60% ✓

The RelevanceComposer floor (default 0.5) bounds the composed product of all factors. In extreme imbalance (e.g., 9/10 same-unit → w = 0.167), the independence factor's weight composes with the proximity factor's weight (1.0 for distance 1), giving 0.167 — floored to 0.5. This prevents the cap from being too aggressive.

#### Interaction with existing factors

Factors compose multiplicatively in the RelevanceComposer. The independence factor and proximity factor operate on different concerns:

- **Proximity factor:** accuracy weighting — same-unit attestors have better context, so their individual attestations are more relevant.
- **Independence factor:** diversity weighting — when too many attestations come from the same unit, their collective contribution is capped to prevent correlated evidence from dominating.

After both factors, the `TrustScoreComputer` applies mass-preserving normalisation: above-mean relevance weights get normalised > 1.0. This means the independence factor's downweighting of same-unit attestations correctly redistributes evidence mass toward cross-unit attestors.

#### Interaction with qhorus CollusionAwareCredibilityPolicy

Qhorus implements a separate anti-collusion mechanism at the **credibility** level: `CollusionAwareCredibilityPolicy` flags mutual-endorsement pairs via `CredibilityFlag.COLLUSION_SUSPECT`. This operates on attestor identity patterns (pair detection), not organizational structure.

The independence factor operates at the **relevance** level on organizational structure. The two mechanisms are complementary:
- Credibility: "is this attestor suspicious?" (individual)
- Independence: "is this group of attestors over-concentrated?" (collective)

Both weights feed into `AttestationWeights` and compose in the trust computation pipeline.

### Testing

**IndependenceAwareRelevanceFactorTest** — unit tests, no CDI:

| Test | Assertion |
|------|-----------|
| Under cap (3/10 same-unit, cap 60%) | All NEUTRAL |
| Over cap (8/10, cap 60%) | Same-unit w=0.375, cross-unit NEUTRAL |
| Exactly at cap (6/10, cap 60%) | All NEUTRAL |
| 100% same-unit | All NEUTRAL |
| 100% cross-unit | All NEUTRAL |
| Mixed with unknowns | Unknowns excluded from share, get NEUTRAL |
| Empty batch | Empty map |
| Single attestation | NEUTRAL |
| Extreme imbalance (9/10) | Correct weight computation |
| `assess()` per-item | Returns NEUTRAL |
| Constructor: cap 0.0 rejected | IllegalArgumentException |
| Constructor: cap > 1.0 rejected | IllegalArgumentException |
| Constructor: cap 1.0 accepted | Valid (effectively disables the factor) |

**RelevanceComposerTest** addition — in ledger:

| Test | Assertion |
|------|-----------|
| Batch factor composes with per-item factor | assessBatch() override sees full batch, weights multiply with per-item factor results |

## References

- `OrgProximityRelevanceFactor.java` — existing per-attestation proximity factor
- `OrgDistanceAttestationEnricher.java` — writes `org-distance` to snapshot at attestation time
- `DefaultOrgAwareDispatch.java:80-124` — LCA-based org-distance algorithm
- `RelevanceComposer.java` — ledger-core factor composition with floor
- `TrustScoreComputer.java` — mass-preserving normalisation
- `AttestationRelevanceFactor.java` — SPI with `assessBatch()` default method
- Ledger ARC42 §Active Risks — anti-gaming gap: social gaming remains open
- Blog: "Whose Evidence Matters" (2026-11-19) — independence-aware factor proposal
- Qhorus `CollusionAwareCredibilityPolicy` — complementary credibility-level mechanism
