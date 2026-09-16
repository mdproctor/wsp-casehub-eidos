# Decisions — #179 Model Selection Schema Integration

## D1: Replace flat fields with ModelQuery model

**Choice:** Replace `modelTier` (String) and `modelCapabilities` (Set<String>) on `AgentCapability` with two mutually exclusive fields: `String modelRef` (string shorthand — alias, tier ref, model ID) and `ModelQuery model` (inline constraint query from `platform-api`). YAML supports the union type via a single `model:` key: `model: "reasoning-heavy"` (→ `modelRef`) or `model: {tier: FAST, capabilities: [reasoning]}` (→ `model`). Mirrors `AgentSessionConfig` on the platform side.
**Alternatives:**
- Add ModelQuery alongside existing fields with coexistence validation — adds complexity, old fields redundant
- Keep flat fields, add toModelQuery() conversion — misses richer constraint dimensions and union type ergonomics
**Rationale:** Pre-release with no deployed instances — no backward compat needed. ModelQuery is a strict superset covering tier, capabilities, plus vendor, family, locality, cost, context window, output size, and vendor preference. Single field eliminates duplication.
**Trade-offs:** Breaking change to AgentCapability record — all existing callers (tests, examples, YAML profiles, annotations) must update. Acceptable given pre-release status. ModelQuery carries operational fields (maxCostTier, locality, preferVendor) that #172 D5/D6 explicitly excluded from agent identity. This is a deliberate evolution: the full constraint surface is more valuable than prevention-by-absence. Eidos validation can reject or ignore inappropriate fields if needed, but the type signature does not prevent them.
**Sources:** AgentCapability.java (existing modelTier + modelCapabilities), ModelQuery.java (platform-api), agent-config-core/src/main/resources/schema/model-selection.schema.json (casehubio/platform), issue-172 spec (rationale for per-capability model requirements)
**Review findings:** R1-01 (boundary reversal), R1-02 (operational field leakage) — acknowledged and consciously accepted. The #172 boundary was correct at the time; platform#335 and #342 changed the landscape by providing schema, router alias resolution, and direct ModelQuery dispatch. The integration value outweighs the boundary purity.
**Exploration:** quick
**Status:** captured

## D2: eidos-api depends on platform-api for ModelQuery

**Choice:** eidos-api takes a compile dependency on `casehub-platform-api` to use `ModelQuery` directly. No eidos-owned thin type or conversion layer.
**Alternatives:**
- eidos-owned sealed interface (ModelSelection.Ref / ModelSelection.Constraints) with runtime conversion to ModelQuery — duplicates all fields, adds converter, no real benefit
**Rationale:** platform-api is pure Java (no framework pull-in). The dependency direction is correct (eidos → platform-api). Eidos runtime already depends on platform-api in practice. Avoids type duplication and the conversion layer that #172 D1 deliberately avoided.
**Trade-offs:** eidos-api loses its zero-dependency status (ARC42STORIES §1 quality goal). Any consumer of eidos-api now transitively pulls in platform-api. Acceptable because platform-api is lightweight (pure Java, no framework) and the coupling direction is architecturally correct (eidos → platform-api, never reverse).
**Sources:** eidos api/pom.xml (current zero-dep), platform-api pom.xml (pure Java), issue-172 decisions D1 (boundary principle), ARC42STORIES §1 (zero-dep quality goal)
**Review findings:** R1-03 (zero-dep quality goal) — acknowledged and consciously accepted. The quality goal served its purpose during eidos's initial development. The platform-api dependency is the minimal coupling needed for type-safe model selection; platform-api itself has no transitive dependencies beyond Jackson.
**Exploration:** quick
**Depends on:** D1 (using ModelQuery requires the dependency)
**Status:** captured

## D3: Org integration — no new org fields

**Choice:** No new model selection fields on `Membership` or `OrganizationalUnit`. Org units already carry `List<AgentCapability>` which inherits the `ModelQuery model` field from D1. Role-level model requirements are expressed through capability declarations on the org unit.
**Alternatives:**
- ModelQuery on Membership (per-agent-per-role override) — conflates agent identity with infrastructure policy
- ModelQuery on OrganizationalUnit (unit-level default) — model requirements are per-capability, not per-unit; a unit doing code review (FLAGSHIP) and lint checking (FAST) needs different tiers per capability, not a unit-level default
**Rationale:** Model selection is a capability-level concern (D2 from #172). Org units compose capabilities — the model requirements flow through that composition naturally.
**Trade-offs:** No way to express "all agents in this unit default to FAST" without declaring it on each capability. Acceptable because that use case is rare and explicit is better than implicit for routing-critical metadata.
**Sources:** OrganizationalUnit.java (capabilities field), Membership.java, issue-172 decisions D2 (per-capability rationale)
**Exploration:** quick
**Depends on:** D1 (ModelQuery on AgentCapability)
**Status:** captured

## D4: Annotation parity — string shorthand + structured attributes

**Choice:** `@AgentCapabilityDef` keeps `modelTier` and `modelCapabilities` as annotation attributes for the structured form, adds `model` for the string shorthand. The recorder builds a `ModelQuery` from whichever form is set. Validation rejects setting both `model` and any structured model attribute simultaneously.
**Alternatives:**
- Only string `model` attribute (tier refs, aliases, model IDs all as strings) — simpler annotation surface but loses build-time vocabulary validation of modelTier against ModelTierTerm
**Rationale:** Annotations can't express union types. Structured attributes preserve the build-time hybrid validation from #172 (compile-time check of modelTier against ModelTierTerm when casehub-eidos-vocab is on the classpath). The recorder layer does the conversion to ModelQuery — annotation attributes are ergonomic input, not the storage type.
**Trade-offs:** Annotation retains `modelTier` and `modelCapabilities` attributes even though the `AgentCapability` record no longer has those fields — slight conceptual mismatch. Acceptable because annotations are an input surface with different constraints than records.
**Sources:** EidosAnnotationsProcessor.java (existing hybrid validation), @AgentCapabilityDef.java (existing annotation), AnnotatedAgentConfig.java (build→runtime transfer)
**Exploration:** quick
**Depends on:** D1 (ModelQuery replaces flat fields on record, but annotation keeps them as input)
**Status:** captured
