# Design Decisions — #271 RoleHealth Probe

## D1: Probe scope — fitness + transition advisory

**Choice:** RoleHealth.probe() returns both role fitness (confidence) AND transition advisory per the descriptor's RoleChangePolicy. Eidos evaluates the policy internally and advises transitions.
**Alternatives:**
- Fitness only — simpler, parallels CapabilityHealth exactly, but pushes policy evaluation to every consumer
- Transition only — skips fitness, loses the per-role health signal that consumers need independently
**Rationale:** RoleChangePolicy is an eidos-owned concept (thresholds, modes, hysteresis). Having every consumer re-implement policy evaluation duplicates logic and risks inconsistency. The probe is the natural place to evaluate policy since it already has all the inputs.
**Trade-offs:** More complex sealed result type. Probe must understand RoleChangePolicy semantics, not just aggregate signals.
**Sources:** `api/RoleChangePolicy.java`, `api/CapabilityHealth.java`, `api/DispositionHealth.java`
**Exploration:** quick
**Status:** captured

## D2: Signal source — dedicated RoleSignalStore SPI

**Choice:** New RoleSignalStore SPI with record/count/learned/clear methods and RoleSignal enum { SUCCESS, STRUGGLE, DECLINE }. Full CDI ladder: NoOp @DefaultBean, InMemory @Alternative, JPA @IfBuildProperty.
**Alternatives:**
- Reuse BehavioralSignalStore — avoids new SPI but overloads qualifier namespace and conflates capability signals with role signals
- External confidence injection — consumers pass confidence to probe, keeps eidos stateless but shifts all responsibility to callers and loses the store's temporal aggregation
**Rationale:** Role fitness is a distinct domain from capability health. The signals have different semantics (SUCCESS/STRUGGLE/DECLINE vs SUCCESS/DECLINE/COMPLIANT/VIOLATED). A dedicated store keeps concerns clean and follows the established pattern exactly.
**Trade-offs:** More code — a new SPI, three CDI beans, JPA table. But the pattern is well-established and mechanical.
**Sources:** `api/BehavioralSignalStore.java`, `api/DispositionSignalStore.java`
**Exploration:** quick
**Status:** captured

## D3: Signal TTL — per-signal TTL model

**Choice:** Per-signal TTL matching BehavioralSignalStore pattern. Each signal type gets its own configurable TTL via @ConfigProperty.
**Alternatives:**
- Cumulative with decay (DispositionSignalStore pattern) — counts accumulate indefinitely, needs active decay management
**Rationale:** Role fitness is temporal — recent failures matter more than old ones. TTL-based expiry matches BehavioralSignalStore and avoids the need for explicit decay management. Natural fit: a DECLINE from 90 days ago shouldn't block an agent from a role today.
**Trade-offs:** Loses long-term role reputation signal. But that's better served by a separate mechanism (e.g. trust scores) than by never-expiring role signals.
**Depends on:** D2 (RoleSignalStore SPI)
**Sources:** `memory/InMemoryBehavioralSignalStore.java`
**Exploration:** quick
**Status:** captured

## D4: Confidence computation — Beta-Binomial with weighted signals

**Choice:** Beta-Binomial conjugate model. Prior: Beta(2,2) (mildly conservative). Signal weights: SUCCESS → α += 1.0, STRUGGLE → β += 0.5, DECLINE → β += 1.0. Confidence = posterior mean = α / (α + β).
**Alternatives:**
- Naive frequency ratio (success/total) — overconfident with sparse data, no uncertainty quantification
- Weighted score with normalisation — more complex, less transparent, no principled uncertainty handling
- Threshold count — loses continuous confidence, incompatible with RoleChangePolicy's threshold model
**Rationale:** Beta-Binomial is the conjugate prior for binary success/failure. It handles sparse data gracefully (2 successes and 0 failures → 0.75, not 1.0), gives posterior mean as point estimate, and the prior ensures sensible behaviour with zero evidence. STRUGGLE at 0.5 negative gives a natural three-level signal without overpenalising agents that completed tasks with difficulty.
**Trade-offs:** Slightly more complex than a frequency ratio, but the Beta-Binomial is well-understood and cheap to compute (just a division).
**Depends on:** D2 (RoleSignalStore SPI), D3 (TTL model)
**Sources:** Beta-Binomial conjugate prior theory, `api/RoleChangePolicy.java` threshold model
**Exploration:** quick
**Status:** captured

## D5: Transition advisory scope — query-scoped

**Choice:** probe(descriptor, roleName) answers only about the queried role. Transition advisory says "demotion advised" or "promotion warranted" but doesn't name a target role. Consumers call probe() for each declared role to find the best fit.
**Alternatives:**
- Cross-role comparison — probe evaluates all declared roles and returns best-fit recommendation. More actionable but couples the probe to multi-role comparison logic and breaks the single-role probe signature
**Rationale:** Single-responsibility. CapabilityHealth doesn't compare across capabilities; it probes one. The same principle applies. Cross-role selection is a consumer concern (engine's dispatch logic), not a health probe concern.
**Trade-offs:** Consumers must call probe() N times for N roles. But N ≤ 10 (max declared roles) and each call is cheap.
**Depends on:** D1 (probe scope)
**Sources:** `api/CapabilityHealth.java` single-capability probe signature
**Exploration:** quick
**Status:** captured
