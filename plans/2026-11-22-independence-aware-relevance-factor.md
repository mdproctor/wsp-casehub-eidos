# Independence-Aware Attestation Relevance Factor — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #257 — independence-aware attestation relevance factor
**Issue group:** #257

**Goal:** Cap same-unit attestation share in trust scoring to prevent correlated evidence from inflating scores.

**Architecture:** Two-repo change. First, refactor ledger's `RelevanceComposer` to call `assessBatch()` per factor (backward-compatible — default method loops `assess()`). Then add `IndependenceAwareRelevanceFactor` in eidos `org-runtime` that overrides `assessBatch()` to compute group-level share ratios and downweight same-unit attestations when over-represented.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ

## Global Constraints

- Java 21 source (Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build with `mvn` not `./mvnw`
- Ledger repo: `/Users/mdproctor/claude/casehub/ledger`
- Eidos repo: `/Users/mdproctor/claude/casehub/eidos`
- All commits reference issue: `Refs casehubio/eidos#257`
- `RelevanceAssessment.weight()` must be in `[0.0, 1.0]`
- `RelevanceAssessment.NEUTRAL` has weight `1.0`
- Existing ledger tests use JUnit assertions; eidos tests use AssertJ

---

## Batch 1: Ledger — RelevanceComposer batch refactor

### Task 1: Refactor RelevanceComposer to call assessBatch() per factor

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/ledger/ledger-core/src/main/java/io/casehub/ledger/core/trust/RelevanceComposer.java:33-52`
- Test: `/Users/mdproctor/claude/casehub/ledger/ledger-core/src/test/java/io/casehub/ledger/core/trust/RelevanceComposerTest.java`

**Interfaces:**
- Consumes: `AttestationRelevanceFactor.assessBatch(List<AttestationRelevanceContext>)` → `Map<UUID, RelevanceAssessment>`
- Produces: No new interfaces — `RelevanceComposer.assess()` signature unchanged

- [ ] **Step 1: Write failing test for batch factor composition**

Add to `RelevanceComposerTest.java` after the existing `fixedFactor` helper (line 133):

```java
@Test
void batchFactor_composesWithPerItemFactor() {
    UUID id1 = UUID.randomUUID();
    UUID id2 = UUID.randomUUID();

    // Per-item factor: gives 0.8 to all
    AttestationRelevanceFactor perItem = fixedFactor(0.8, "per-item");

    // Batch factor: gives 0.5 to id1, NEUTRAL to id2
    AttestationRelevanceFactor batchFactor = new AttestationRelevanceFactor() {
        @Override
        public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
            return RelevanceAssessment.NEUTRAL;
        }

        @Override
        public Map<UUID, RelevanceAssessment> assessBatch(List<AttestationRelevanceContext> ctxs) {
            Map<UUID, RelevanceAssessment> result = new java.util.LinkedHashMap<>();
            for (AttestationRelevanceContext ctx : ctxs) {
                if (ctx.attestationId().equals(id1)) {
                    result.put(ctx.attestationId(), new RelevanceAssessment(0.5, "capped"));
                } else {
                    result.put(ctx.attestationId(), RelevanceAssessment.NEUTRAL);
                }
            }
            return result;
        }

        @Override
        public String name() {
            return "batch";
        }
    };

    var composer = new RelevanceComposer(List.of(perItem, batchFactor), 0.3);
    var result = composer.assess(List.of(ctx(id1), ctx(id2)));

    // id1: 0.8 * 0.5 = 0.4
    assertEquals(0.4, result.get(id1), 1e-9);
    // id2: 0.8 * 1.0 = 0.8
    assertEquals(0.8, result.get(id2), 1e-9);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=RelevanceComposerTest#batchFactor_composesWithPerItemFactor -f /Users/mdproctor/claude/casehub/ledger/pom.xml`

Expected: FAIL — the current composer calls `assess()` per item, so the batch factor's `assessBatch()` override is never called. Both IDs get `0.8 * 1.0 = 0.8` instead of id1 getting `0.8 * 0.5 = 0.4`.

- [ ] **Step 3: Refactor RelevanceComposer.assess()**

Replace the body of `assess()` method (lines 33-52) with:

```java
public Map<UUID, Double> assess(List<AttestationRelevanceContext> contexts) {
    if (factors.isEmpty()) {
        Map<UUID, Double> result = new LinkedHashMap<>();
        for (AttestationRelevanceContext ctx : contexts) {
            result.put(ctx.attestationId(), 1.0);
        }
        return result;
    }

    Map<UUID, Double> products = new LinkedHashMap<>();
    for (AttestationRelevanceContext ctx : contexts) {
        products.put(ctx.attestationId(), 1.0);
    }

    for (AttestationRelevanceFactor factor : factors) {
        Map<UUID, RelevanceAssessment> batch = factor.assessBatch(contexts);
        for (var entry : batch.entrySet()) {
            products.merge(entry.getKey(), entry.getValue().weight(),
                    (existing, weight) -> existing * weight);
        }
    }

    Map<UUID, Double> result = new LinkedHashMap<>();
    for (var entry : products.entrySet()) {
        result.put(entry.getKey(), Math.max(minWeight, entry.getValue()));
    }
    return result;
}
```

- [ ] **Step 4: Run all RelevanceComposer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=RelevanceComposerTest -f /Users/mdproctor/claude/casehub/ledger/pom.xml`

Expected: ALL PASS — existing tests use per-item factors whose default `assessBatch()` loops `assess()`, so behavior is unchanged. New test passes because the composer now calls `assessBatch()`.

- [ ] **Step 5: Run full ledger-core test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -f /Users/mdproctor/claude/casehub/ledger/pom.xml`

Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add ledger-core/src/main/java/io/casehub/ledger/core/trust/RelevanceComposer.java ledger-core/src/test/java/io/casehub/ledger/core/trust/RelevanceComposerTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#257): refactor RelevanceComposer to call assessBatch() per factor

Calls assessBatch() on each factor instead of assess() per attestation.
The default assessBatch() loops assess(), so existing per-item factors
are unchanged. Factors that override assessBatch() now see the full
batch for group-level computation.

Refs casehubio/eidos#257

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Eidos — IndependenceAwareRelevanceFactor

### Task 2: IndependenceAwareRelevanceFactor with full test suite

**Files:**
- Create: `/Users/mdproctor/claude/casehub/eidos/org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java`
- Test: `/Users/mdproctor/claude/casehub/eidos/org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactorTest.java`

**Interfaces:**
- Consumes: `AttestationRelevanceFactor` SPI, `AttestationRelevanceContext.snapshot()`, `RelevanceAssessment`, `OrgProximityRelevanceFactor.SNAPSHOT_KEY`
- Produces: `IndependenceAwareRelevanceFactor` CDI bean (discovered by ledger's `RelevanceComposer` via `Instance<AttestationRelevanceFactor>`)

- [ ] **Step 1: Write the full test class**

Create `/Users/mdproctor/claude/casehub/eidos/org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactorTest.java`:

```java
package io.casehub.eidos.org.runtime.trust;

import io.casehub.ledger.api.spi.AttestationRelevanceContext;
import io.casehub.ledger.api.spi.RelevanceAssessment;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.within;

class IndependenceAwareRelevanceFactorTest {

    private static final double DEFAULT_CAP = 0.6;

    private IndependenceAwareRelevanceFactor factor(double maxSameUnitShare) {
        return new IndependenceAwareRelevanceFactor(maxSameUnitShare);
    }

    private AttestationRelevanceContext ctx(UUID id, Map<String, Object> snapshot) {
        return new AttestationRelevanceContext(
                id, "attestor-" + id.toString().substring(0, 4), ActorType.AGENT, "reviewer",
                "scored-1", "tenant-1", "code-review", null,
                Instant.now(), snapshot);
    }

    private AttestationRelevanceContext sameUnitCtx(UUID id) {
        return ctx(id, Map.of("org-distance", 1));
    }

    private AttestationRelevanceContext crossUnitCtx(UUID id, int distance) {
        return ctx(id, Map.of("org-distance", distance));
    }

    private AttestationRelevanceContext unknownCtx(UUID id) {
        return ctx(id, null);
    }

    @Test
    void name_returns_independence() {
        assertThat(factor(DEFAULT_CAP).name()).isEqualTo("independence");
    }

    @Test
    void assess_per_item_returns_neutral() {
        var result = factor(DEFAULT_CAP).assess(sameUnitCtx(UUID.randomUUID()));
        assertThat(result).isEqualTo(RelevanceAssessment.NEUTRAL);
    }

    @Test
    void under_cap_all_neutral() {
        var ids = uuids(10);
        var contexts = new java.util.ArrayList<AttestationRelevanceContext>();
        for (int i = 0; i < 3; i++) contexts.add(sameUnitCtx(ids.get(i)));
        for (int i = 3; i < 10; i++) contexts.add(crossUnitCtx(ids.get(i), 2));

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        for (UUID id : ids) {
            assertThat(result.get(id)).isEqualTo(RelevanceAssessment.NEUTRAL);
        }
    }

    @Test
    void over_cap_same_unit_downweighted() {
        var ids = uuids(10);
        var contexts = new java.util.ArrayList<AttestationRelevanceContext>();
        for (int i = 0; i < 8; i++) contexts.add(sameUnitCtx(ids.get(i)));
        for (int i = 8; i < 10; i++) contexts.add(crossUnitCtx(ids.get(i), 3));

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        // w = 0.6 * 2 / (0.4 * 8) = 0.375
        for (int i = 0; i < 8; i++) {
            assertThat(result.get(ids.get(i)).weight()).isCloseTo(0.375, within(1e-9));
        }
        for (int i = 8; i < 10; i++) {
            assertThat(result.get(ids.get(i))).isEqualTo(RelevanceAssessment.NEUTRAL);
        }
    }

    @Test
    void exactly_at_cap_all_neutral() {
        var ids = uuids(10);
        var contexts = new java.util.ArrayList<AttestationRelevanceContext>();
        for (int i = 0; i < 6; i++) contexts.add(sameUnitCtx(ids.get(i)));
        for (int i = 6; i < 10; i++) contexts.add(crossUnitCtx(ids.get(i), 2));

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        for (UUID id : ids) {
            assertThat(result.get(id)).isEqualTo(RelevanceAssessment.NEUTRAL);
        }
    }

    @Test
    void all_same_unit_all_neutral() {
        var ids = uuids(5);
        var contexts = ids.stream().map(this::sameUnitCtx).toList();

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        for (UUID id : ids) {
            assertThat(result.get(id)).isEqualTo(RelevanceAssessment.NEUTRAL);
        }
    }

    @Test
    void all_cross_unit_all_neutral() {
        var ids = uuids(5);
        var contexts = ids.stream().map(id -> crossUnitCtx(id, 3)).toList();

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        for (UUID id : ids) {
            assertThat(result.get(id)).isEqualTo(RelevanceAssessment.NEUTRAL);
        }
    }

    @Test
    void unknowns_excluded_from_share_and_get_neutral() {
        var ids = uuids(12);
        var contexts = new java.util.ArrayList<AttestationRelevanceContext>();
        // 8 same-unit, 2 cross-unit, 2 unknown
        for (int i = 0; i < 8; i++) contexts.add(sameUnitCtx(ids.get(i)));
        for (int i = 8; i < 10; i++) contexts.add(crossUnitCtx(ids.get(i), 2));
        for (int i = 10; i < 12; i++) contexts.add(unknownCtx(ids.get(i)));

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        // Share computed on 8+2=10 known, not 12 total
        // w = 0.6 * 2 / (0.4 * 8) = 0.375
        for (int i = 0; i < 8; i++) {
            assertThat(result.get(ids.get(i)).weight()).isCloseTo(0.375, within(1e-9));
        }
        for (int i = 8; i < 10; i++) {
            assertThat(result.get(ids.get(i))).isEqualTo(RelevanceAssessment.NEUTRAL);
        }
        for (int i = 10; i < 12; i++) {
            assertThat(result.get(ids.get(i))).isEqualTo(RelevanceAssessment.NEUTRAL);
        }
    }

    @Test
    void empty_batch_returns_empty() {
        var result = factor(DEFAULT_CAP).assessBatch(List.of());
        assertThat(result).isEmpty();
    }

    @Test
    void single_attestation_neutral() {
        UUID id = UUID.randomUUID();
        var result = factor(DEFAULT_CAP).assessBatch(List.of(sameUnitCtx(id)));
        assertThat(result.get(id)).isEqualTo(RelevanceAssessment.NEUTRAL);
    }

    @Test
    void extreme_imbalance_weight_correct() {
        var ids = uuids(10);
        var contexts = new java.util.ArrayList<AttestationRelevanceContext>();
        for (int i = 0; i < 9; i++) contexts.add(sameUnitCtx(ids.get(i)));
        contexts.add(crossUnitCtx(ids.get(9), 2));

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        // w = 0.6 * 1 / (0.4 * 9) = 0.1666...
        double expected = 0.6 / (0.4 * 9);
        for (int i = 0; i < 9; i++) {
            assertThat(result.get(ids.get(i)).weight()).isCloseTo(expected, within(1e-9));
        }
        assertThat(result.get(ids.get(9))).isEqualTo(RelevanceAssessment.NEUTRAL);
    }

    @Test
    void reason_includes_share_and_weight() {
        var ids = uuids(10);
        var contexts = new java.util.ArrayList<AttestationRelevanceContext>();
        for (int i = 0; i < 8; i++) contexts.add(sameUnitCtx(ids.get(i)));
        for (int i = 8; i < 10; i++) contexts.add(crossUnitCtx(ids.get(i), 2));

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        assertThat(result.get(ids.get(0)).reason()).contains("same-unit 8/10");
        assertThat(result.get(ids.get(0)).reason()).contains("cap 60%");
    }

    @Test
    void cap_zero_rejected() {
        assertThatThrownBy(() -> factor(0.0))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void cap_above_one_rejected() {
        assertThatThrownBy(() -> factor(1.1))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void cap_one_accepted_disables_factor() {
        var ids = uuids(10);
        var contexts = new java.util.ArrayList<AttestationRelevanceContext>();
        for (int i = 0; i < 9; i++) contexts.add(sameUnitCtx(ids.get(i)));
        contexts.add(crossUnitCtx(ids.get(9), 2));

        var result = factor(1.0).assessBatch(contexts);

        for (UUID id : ids) {
            assertThat(result.get(id)).isEqualTo(RelevanceAssessment.NEUTRAL);
        }
    }

    @Test
    void missing_key_treated_as_unknown() {
        var ids = uuids(10);
        var contexts = new java.util.ArrayList<AttestationRelevanceContext>();
        for (int i = 0; i < 8; i++) contexts.add(sameUnitCtx(ids.get(i)));
        contexts.add(ctx(ids.get(8), Map.of("other-key", 42)));
        contexts.add(crossUnitCtx(ids.get(9), 2));

        var result = factor(DEFAULT_CAP).assessBatch(contexts);

        // Share: 8 same / (8 same + 1 cross) = 88.9% > 60%
        // w = 0.6 * 1 / (0.4 * 8) = 0.1875
        double expected = 0.6 / (0.4 * 8);
        for (int i = 0; i < 8; i++) {
            assertThat(result.get(ids.get(i)).weight()).isCloseTo(expected, within(1e-9));
        }
        assertThat(result.get(ids.get(8))).isEqualTo(RelevanceAssessment.NEUTRAL);
        assertThat(result.get(ids.get(9))).isEqualTo(RelevanceAssessment.NEUTRAL);
    }

    @Test
    void distance_zero_treated_as_unknown() {
        UUID sameUnit = UUID.randomUUID();
        UUID selfAttest = UUID.randomUUID();
        UUID cross = UUID.randomUUID();

        var result = factor(DEFAULT_CAP).assessBatch(List.of(
                sameUnitCtx(sameUnit),
                ctx(selfAttest, Map.of("org-distance", 0)),
                crossUnitCtx(cross, 2)));

        // 1 same, 1 cross, 1 unknown (distance 0) → share = 1/2 = 50% ≤ 60%
        assertThat(result.get(sameUnit)).isEqualTo(RelevanceAssessment.NEUTRAL);
        assertThat(result.get(selfAttest)).isEqualTo(RelevanceAssessment.NEUTRAL);
        assertThat(result.get(cross)).isEqualTo(RelevanceAssessment.NEUTRAL);
    }

    @Test
    void negative_distance_treated_as_unknown() {
        UUID sameUnit = UUID.randomUUID();
        UUID negative = UUID.randomUUID();

        var result = factor(DEFAULT_CAP).assessBatch(List.of(
                sameUnitCtx(sameUnit),
                ctx(negative, Map.of("org-distance", -1))));

        // 1 same, 0 cross, 1 unknown → crossCount == 0 → all NEUTRAL
        assertThat(result.get(sameUnit)).isEqualTo(RelevanceAssessment.NEUTRAL);
        assertThat(result.get(negative)).isEqualTo(RelevanceAssessment.NEUTRAL);
    }

    private List<UUID> uuids(int count) {
        return java.util.stream.IntStream.range(0, count)
                .mapToObj(i -> UUID.randomUUID())
                .toList();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=IndependenceAwareRelevanceFactorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: FAIL — `IndependenceAwareRelevanceFactor` class does not exist yet.

- [ ] **Step 3: Implement IndependenceAwareRelevanceFactor**

Create `/Users/mdproctor/claude/casehub/eidos/org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java`:

```java
package io.casehub.eidos.org.runtime.trust;

import io.casehub.ledger.api.spi.AttestationRelevanceContext;
import io.casehub.ledger.api.spi.AttestationRelevanceFactor;
import io.casehub.ledger.api.spi.RelevanceAssessment;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class IndependenceAwareRelevanceFactor implements AttestationRelevanceFactor {

    private final double maxSameUnitShare;

    public IndependenceAwareRelevanceFactor(
            @ConfigProperty(name = "casehub.eidos.trust.independence.max-same-unit-share",
                    defaultValue = "0.6")
            double maxSameUnitShare) {
        if (maxSameUnitShare <= 0.0 || maxSameUnitShare > 1.0) {
            throw new IllegalArgumentException(
                    "maxSameUnitShare must be in (0.0, 1.0], got: " + maxSameUnitShare);
        }
        this.maxSameUnitShare = maxSameUnitShare;
    }

    @Override
    public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
        return RelevanceAssessment.NEUTRAL;
    }

    @Override
    public Map<UUID, RelevanceAssessment> assessBatch(List<AttestationRelevanceContext> ctxs) {
        Map<UUID, RelevanceAssessment> result = new LinkedHashMap<>();
        if (ctxs.isEmpty()) {
            return result;
        }

        List<UUID> sameUnit = new ArrayList<>();
        List<UUID> crossUnit = new ArrayList<>();
        List<UUID> unknown = new ArrayList<>();

        for (AttestationRelevanceContext ctx : ctxs) {
            int distance = extractDistance(ctx);
            if (distance == 1) {
                sameUnit.add(ctx.attestationId());
            } else if (distance > 1) {
                crossUnit.add(ctx.attestationId());
            } else {
                unknown.add(ctx.attestationId());
            }
        }

        int sameCount = sameUnit.size();
        int crossCount = crossUnit.size();

        if (crossCount == 0 || sameCount == 0) {
            for (AttestationRelevanceContext ctx : ctxs) {
                result.put(ctx.attestationId(), RelevanceAssessment.NEUTRAL);
            }
            return result;
        }

        double fraction = (double) sameCount / (sameCount + crossCount);

        if (fraction <= maxSameUnitShare) {
            for (AttestationRelevanceContext ctx : ctxs) {
                result.put(ctx.attestationId(), RelevanceAssessment.NEUTRAL);
            }
            return result;
        }

        double weight = maxSameUnitShare * crossCount
                / ((1.0 - maxSameUnitShare) * sameCount);
        int knownCount = sameCount + crossCount;
        long capPercent = Math.round(maxSameUnitShare * 100);
        long sharePercent = Math.round(fraction * 100);
        String reason = "independence: same-unit " + sameCount + "/" + knownCount
                + " (" + sharePercent + "%) > cap " + capPercent + "%, weight: " + weight;

        RelevanceAssessment capped = new RelevanceAssessment(weight, reason);

        for (UUID id : sameUnit) {
            result.put(id, capped);
        }
        for (UUID id : crossUnit) {
            result.put(id, RelevanceAssessment.NEUTRAL);
        }
        for (UUID id : unknown) {
            result.put(id, RelevanceAssessment.NEUTRAL);
        }

        return result;
    }

    @Override
    public String name() {
        return "independence";
    }

    private static int extractDistance(AttestationRelevanceContext ctx) {
        var snapshot = ctx.snapshot();
        if (snapshot == null || !snapshot.containsKey(OrgProximityRelevanceFactor.SNAPSHOT_KEY)) {
            return -1;
        }
        Object raw = snapshot.get(OrgProximityRelevanceFactor.SNAPSHOT_KEY);
        if (!(raw instanceof Number)) {
            return -1;
        }
        return ((Number) raw).intValue();
    }
}
```

- [ ] **Step 4: Run all IndependenceAwareRelevanceFactor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -Dtest=IndependenceAwareRelevanceFactorTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: ALL PASS

- [ ] **Step 5: Run full org-runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl org-runtime -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add org-runtime/src/main/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactor.java org-runtime/src/test/java/io/casehub/eidos/org/runtime/trust/IndependenceAwareRelevanceFactorTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#257): add IndependenceAwareRelevanceFactor

Caps same-unit attestation share at a configurable threshold (default
60%). When same-unit attestors are over-represented in a trust
computation batch, their relevance weight is reduced to bring the
share to the cap. Cross-unit and unknown attestors are unaffected.

Uses assessBatch() override to see the full group and compute share
ratios. The per-item assess() returns NEUTRAL — batch semantics only.

Refs casehubio/eidos#257

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-11-21-independence-aware-relevance-factor-design.md] — design spec this plan implements
- [RelevanceComposer.java] — ledger-core factor composition (refactored in Task 1)
- [RelevanceComposerTest.java] — existing ledger-core tests (extended in Task 1)
- [OrgProximityRelevanceFactor.java] — existing per-attestation proximity factor (pattern reference)
- [OrgProximityRelevanceFactorTest.java] — existing eidos test (pattern reference)
- [AttestationRelevanceFactor.java] — SPI with assessBatch() default method
- [GitHub casehubio/eidos#257] — focal issue
