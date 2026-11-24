# Design Decisions — #271 RoleHealth Probe

## D1: Probe scope — fitness only

**Choice:** RoleHealth.probe() returns role fitness (confidence score and status) for the queried role. Policy evaluation (RoleChangePolicy thresholds, hysteresis, hold-cycles) is handled by a separate RoleTransitionAdvisor SPI.
**Alternatives:**
- Fitness + transition advisory bundled — single call returns both fitness and transition recommendation. Simpler consumer API but bundles read-only fitness computation with stateful hold-cycle tracking; makes the probe a read-write operation
- Transition only — skips fitness, loses the per-role health signal that consumers need independently
**Rationale:** `minHoldCycles` requires persistent state — a counter per (agentId, roleName) tracking consecutive below-threshold probe cycles. This makes transition advisory inherently stateful, while fitness computation is stateless (reads task outcomes, computes Wilson score). Bundling both forces the probe to have side effects. Separating them follows the established DispositionHealth/DispositionEvolution pattern: probe reads state and returns status; a separate SPI evaluates policy and manages mutation.
**Trade-offs:** Consumers who want transition advice must call two SPIs. But this makes statefulness explicit and testable — the probe is always safe to call repeatedly without side effects.
**Sources:** `api/RoleChangePolicy.java`, `api/CapabilityHealth.java`, `api/DispositionHealth.java`, `api/DispositionEvolution.java`
**Exploration:** quick
**Status:** revised — separated fitness from transition advisory (from R1-01, R1-09)

## D2: Signal source — knowledge graph task outcomes

**Choice:** Extend AgentTask with an optional `role` field recording which role the agent was serving when dispatched. Add role-scoped query methods to AgentGraphQuery (e.g. `outcomesByRole(agentId, roleName, tenancyId)`). The existing knowledge graph provides role fitness evidence through task outcomes filtered by role.
**Alternatives:**
- Dedicated RoleSignalStore SPI — new SPI with record/count/learned/clear and RoleSignal enum. Structurally identical to BehavioralSignalStore, creating a third signal store with its own CDI ladder (NoOp + InMemory + JPA = 5+ new types)
- Reuse BehavioralSignalStore — avoids new SPI but overloads the capabilityName parameter (documented as "must be the agent's declared capability name") with role names, creating semantic conflation
- DetectedRole / stigmergy — emergent role detection from observed behavior patterns (as referenced in issue #271). A valid future signal source that could feed into the knowledge graph as additional evidence, but no implementation exists in eidos today
- External confidence injection — consumers pass confidence to probe, keeps eidos stateless but shifts all responsibility to callers
**Rationale:** The knowledge graph already records per-agent, per-capability task outcomes with attestation chains. Adding a `role` field to AgentTask (one field, one column) enables role-scoped outcome queries without new SPIs, CDI ladders, or signal stores. Task outcomes are objective (agent succeeded or failed at a task while serving in a role) rather than subjective (someone decided to record SUCCESS/STRUGGLE/DECLINE). The existing Wilson lower-bound ranking infrastructure is reused directly.
**Trade-offs:** Task outcomes are coarse-grained — an agent can succeed at a task while performing poorly in their role-specific responsibilities. Future refinement may add explicit role-performance signals as a supplementary evidence source. The `role` field on AgentTask couples the knowledge graph schema to role concepts.
**Sources:** `api/AgentGraphStore.java`, `api/AgentGraphQuery.java`, `api/AgentTask.java`, `graph/JpaAgentGraphQuery.java`
**Exploration:** quick
**Status:** revised — knowledge graph replaces dedicated signal store (from R1-03, R1-08)

## D3: Temporal relevance — query-time window

**Choice:** Confidence computation considers task outcomes within a configurable temporal window (default: 90 days). Knowledge graph records are permanent; temporal filtering is at query time via a `since` parameter on the role-scoped outcome query.
**Alternatives:**
- Per-signal TTL (BehavioralSignalStore pattern) — individual records expire. Not applicable since the knowledge graph records are permanent audit-trail entries
- Cumulative with decay (DispositionSignalStore pattern) — counts accumulate with multiplicative decay. Unnecessary complexity when a query-time window achieves the same temporal relevance
- No temporal filtering — all history counts equally. Unfair to agents who improved over time; ancient failures permanently drag down confidence
**Rationale:** Role fitness is temporal — recent performance matters more than history from months ago. A query-time window is simpler than individual record TTL (which would require deleting knowledge graph entries, conflicting with the audit-trail purpose). The knowledge graph's `startedAt` timestamp enables efficient temporal filtering.
**Trade-offs:** Loses gradual decay — an outcome at day 89 counts as much as one at day 1, then drops off entirely at day 91. Acceptable for the initial design; a weighted recency model can be introduced later if needed.
**Depends on:** D2 (knowledge graph as evidence source)
**Sources:** `api/AgentTask.java` (startedAt field), `graph/JpaAgentGraphQuery.java`
**Exploration:** quick
**Status:** revised — query-time window replaces per-signal TTL (from R1-08)

## D4: Confidence computation — Wilson lower-bound

**Choice:** Wilson 95% lower-bound confidence score, consistent with `AgentGraphQuery.topAgentsByOutcome()`. Quality per outcome = confidence × multiplier (SUCCEEDED=1.0, PARTIALLY=0.5, FAILED=0.0). Zero evidence → declared fit by default (parallels CapabilityHealth's Ready status for declared capabilities).
**Alternatives:**
- Posterior mean (Beta-Binomial) — optimistic estimator; gives high scores quickly with sparse data. Contradicts the platform's established Wilson convention which penalizes small samples
- Naive frequency ratio (success/total) — overconfident with sparse data, no uncertainty quantification
- Threshold count — loses continuous confidence, incompatible with RoleChangePolicy's threshold model
**Rationale:** The platform's quality goal states: "Evidence-grounded ranking — Wilson lower-bound confidence scores penalise small samples" (ARC42STORIES.MD §1). Using Wilson for role fitness maintains a consistent statistical philosophy: an agent needs sufficient positive evidence to earn a high score, regardless of whether the question is "how capable?" or "how fit for this role?" For zero-evidence agents with declared roles, the probe returns a "declared fit" status (analogous to CapabilityHealth.Ready) rather than a numeric score, avoiding the semantic inconsistency where declared capabilities are Ready but declared roles start at 0.5 or 0.0.
**Trade-offs:** Wilson is conservative — agents need more evidence to reach high confidence. This is appropriate for safety-critical role transitions where false positives (premature promotion) are more costly than false negatives (delayed promotion).
**Depends on:** D2 (knowledge graph as evidence source), D3 (temporal window)
**Sources:** `graph/JpaAgentGraphQuery.java` (wilsonScore static method), ARC42STORIES.MD §1 Quality Goals, `api/RoleChangePolicy.java` threshold model
**Exploration:** quick
**Status:** revised — Wilson lower-bound replaces posterior mean (from R1-05, R1-10)

## D5: Fitness probe scope — query-scoped, stateless

**Choice:** RoleHealth.probe(descriptor, roleName) answers only about the queried role. Returns fitness status (confidence score and semantic status) but NOT transition advisory. The probe is stateless — it reads from the knowledge graph but writes nothing.
**Alternatives:**
- Cross-role comparison — probe evaluates all declared roles and returns best-fit recommendation. More actionable but couples the probe to multi-role comparison logic and breaks the single-role probe signature
- Bundled transition advisory — probe evaluates RoleChangePolicy and returns transition recommendations. Requires hold-cycle state, making the probe stateful (addressed in D1 and D8)
**Rationale:** Single-responsibility. CapabilityHealth doesn't compare across capabilities; it probes one. The same principle applies. The probe is purely read-only, like CapabilityHealth.probe(). Transition advisory (which requires hold-cycle state for `minHoldCycles` evaluation) is handled by the separate RoleTransitionAdvisor SPI (D8).
**Trade-offs:** Consumers must call probe() N times for N roles. But N ≤ 10 (max declared roles per RoleProfile validation) and each call is cheap (a single query with Wilson computation).
**Depends on:** D1 (probe scope), D8 (separation of probe and advisor)
**Sources:** `api/CapabilityHealth.java` single-capability probe signature, `api/RoleProfile.java` (MAX_ROLE_DECLARATIONS limit)
**Exploration:** quick
**Status:** revised — clarified stateless, fitness-only scope (from R1-01, R1-05)

## D6: Domain-scoping — available via taskDomain, not required on probe signature

**Choice:** RoleHealth.probe(descriptor, roleName) does not take a domain parameter. Domain-specific role fitness is available by filtering the knowledge graph query by taskDomain (AgentTask already has this field). The probe computes aggregate confidence across all domains by default.
**Alternatives:**
- Mandatory domain parameter (like ProbeContext.taskDomain on CapabilityHealth) — domain-specific role fitness per probe call. More precise but adds complexity; role fitness is often domain-agnostic ("is this agent a good coordinator?") unlike capability fitness ("can this agent review Rust code?")
- Optional domain parameter with aggregate fallback — probe takes Optional<String> domain; computes domain-specific confidence if present, aggregate otherwise
**Rationale:** Role fitness is typically less domain-bound than capability fitness. A coordinator's effectiveness depends more on coordination skills than on the subject domain. CapabilityHealth needs domain-scoping because epistemic domain confidence varies dramatically (a Java expert reviewing Rust code). Role fitness varies less by domain. When domain-specific role fitness IS needed, the knowledge graph's taskDomain field provides it at the query layer without changing the probe signature.
**Trade-offs:** Loses fine-grained domain discrimination at the probe API level. The optional-domain-parameter alternative (Alternative 2) can be introduced as a backward-compatible enhancement if empirical evidence shows domain-specific role fitness is needed.
**Sources:** `api/CapabilityHealth.java` ProbeContext, `api/AgentTask.java` taskDomain field
**Exploration:** new (implicit decision made explicit from R1-06)
**Status:** captured

## D7: Signal recording protocol — engine extends existing recordTask flow

**Choice:** Role fitness evidence comes from the existing `AgentGraphStore.recordTask()` flow in casehub-engine. The engine already dispatches tasks to agents and records outcomes; it also assigns roles. Adding the role to the recordTask call is the recording mechanism — no new recording protocol is needed.
**Alternatives:**
- Explicit signal recording by consumers — consumers call a RoleSignalStore.record() method after observing role behavior. Requires a recording protocol (when to record, what to record, who decides) that doesn't exist
- Stigmergy / DetectedRole — emergent role detection from observed behavior patterns. A valid future signal source referenced in issue #271, but no implementation exists in eidos. Could augment knowledge graph evidence if implemented in casehub-engine
- Self-assessment by agents — agents record their own role performance signals. Subject to bias; not suitable for trust-critical routing decisions
**Rationale:** The recording question ("who decides to record what, and when?") is answered by the existing knowledge graph flow: the engine records tasks at dispatch time and outcomes at completion time. This is objective (task succeeded or failed), auditable (attestation chains), and already in production. No new protocol or decision-making is required.
**Trade-offs:** Recording granularity is task-level, not behavior-level. An agent's role-specific behaviors within a task (communication quality, decision timing, delegation effectiveness) are not captured. Future work could add behavioral observation as a supplementary evidence source.
**Depends on:** D2 (knowledge graph as evidence source)
**Sources:** `api/AgentGraphStore.java`, casehub-engine task dispatch flow, issue #271 reference to DetectedRole / stigmergy
**Exploration:** new (implicit decision made explicit from R1-07)
**Status:** captured

## D8: Separation of probe and transition advisory — two SPIs

**Choice:** Two separate SPIs: RoleHealth (fitness probe, stateless) and RoleTransitionAdvisor (policy evaluation, stateful). RoleHealth.probe() computes confidence and returns status. RoleTransitionAdvisor.evaluate() reads RoleChangePolicy, tracks hold-cycles, and returns transition advice (promote/demote/hold).
**Alternatives:**
- Bundled in one SPI — single probe() call returns both fitness and transition advice. Simpler API but conflates stateless computation with stateful tracking; makes the probe a read-write operation
- Transition logic in consumers — each consumer evaluates RoleChangePolicy independently. Duplicates logic and risks inconsistency (D1's original concern). Policy evaluation belongs in eidos
**Rationale:** `minHoldCycles` requires a persistent counter per (agentId, roleName) pair that tracks consecutive below-threshold probe cycles. This counter must be incremented when confidence is below demotionConfidence and reset when it rises above. This is inherently stateful — the advisor writes state on each evaluation. Keeping this in a separate SPI makes the statefulness explicit, testable, and replaceable. The pattern follows DispositionHealth (probe, stateless) / DispositionEvolution (evaluate, stateful).
**Trade-offs:** Two SPIs instead of one. Consumers wanting transition advice call probe() then evaluate(). But this makes the state boundary explicit: the probe is always safe to call without side effects; the advisor is the only component that tracks state.
**Depends on:** D1 (probe scope), D5 (fitness probe statelessness)
**Sources:** `api/DispositionHealth.java`, `api/DispositionEvolution.java`, `api/RoleChangePolicy.java` (minHoldCycles field)
**Exploration:** new (implicit decision made explicit from R1-01, R1-09)
**Status:** captured
