# Decisions — #229 OrganizationalUnit Model Enrichment

## D1: Obligations and permissions placement

**Choice:** Add `List<String> obligations` and `List<String> permissions` directly on `RoleSpec`
**Alternatives:**
- Extend existing `RolePolicy` — conflates capability-based matching with normative identifiers; different consumers, different concerns
- Separate `NormativeRolePolicy` record — unnecessary indirection; norm identifiers are just string lists, no wrapper needed
- On `GroupPolicy` as spec envisioned (`Map<String, RolePolicy> roles`) — splits role definition across two records; implementation already chose RoleSpec as the single authority for role definitions
**Rationale:** RoleSpec defines everything about a role: identity, cardinality, incompatibilities, capability requirements. Adding normative expectations here keeps the definition complete and follows eidos's pattern (AgentDescriptor has goals/constraints as direct fields). The existing `RolePolicy(capabilityName, PolicyKind, expectations)` serves a different concern (capability-based matching) and remains unchanged.
**Trade-offs:** Norm identifiers on RoleSpec are not validated against agency-core's normative framework at construction time — they're opaque strings until enforcement. This is consistent with eidos's open-string philosophy (slot, topology, etc.).
**Sources:** spec §2.2 (role obligations/permissions), `RoleSpec.java`, `RolePolicy.java`, `PolicyKind.java`, `AgentDescriptor.java` (pattern precedent)
**Exploration:** quick
**Status:** captured

## D2: Dissolution policy shape

**Choice:** Add `Duration dissolutionDrainTimeout` to `GroupPolicy` (default 30 minutes). No strategy enum.
**Alternatives:**
- Full `DissolutionPolicy` record with strategy enum (force-reassign, escalate, force-dissolve) — spec says strategy is CBR-informed (runtime decides), not declarative
- No addition (timeout hardcoded) — spec explicitly says "configurable via GroupPolicy"
**Rationale:** The only declarative dissolution parameter is the drain timeout. The spec explicitly states the specific action after timeout is "CBR-informed" — that's a runtime concern for SocietyLifecycleManager, not a model concern. If more dissolution configuration emerges later, add fields then.
**Trade-offs:** No declarative control over post-timeout strategy. Acceptable because the spec delegates this to runtime CBR.
**Sources:** spec §4 (dissolution, PENDING_DRAIN timeout), `GroupPolicy.java`
**Exploration:** quick
**Status:** captured

## D3: Topology transition governance

**Choice:** `TopologyFamily` enum in org-api + `TopologyFamilyClassifier` SPI for runtime resolution. No fields added to GroupPolicy.
**Alternatives:**
- `Set<String> allowedTopologyTransitions` on GroupPolicy — adds model weight for a classification concern; doesn't compose (must declare per-group); is policy, not classification
- TopologyFamily enum + static utility only — hardcodes mapping, custom topologies have no way to declare family
- Vocabulary-driven only (`TopologyFamilyAware` marker interface) — right long-term but no topology vocabulary enum exists in eidos-vocab yet; over-engineers the initial delivery
**Rationale:** Topology family is a classification, not a policy — it shouldn't live on GroupPolicy. An SPI gives the runtime what it needs (resolve topology string → family) while allowing future vocabulary-driven implementations. The default implementation maps well-known values: swarm/mesh → DISTRIBUTED, hive → CENTRALISED, many → INDEPENDENT. Unknown topologies return `Optional.empty()` → treated as cross-family → require approval (safe default).
**Trade-offs:** Custom topologies require a custom `TopologyFamilyClassifier` implementation to get auto-approval for within-family transitions. Without one, they always require approval — safe but potentially inconvenient.
**Sources:** spec §2.1 (topology transitions, selfDetermination thresholds), `OrganizationalUnit.java`, `AuthorityModel.java`
**Exploration:** deep-analysis
**Status:** captured
