# Decisions — #179 Model Selection Schema Integration

## D1: Replace flat fields with ModelQuery model

**Choice:** Replace `modelTier` (String) and `modelCapabilities` (Set<String>) on `AgentCapability` with a single `ModelQuery model` field from `platform-api`. YAML supports the union type: `model: "reasoning-heavy"` (string shorthand) or `model: {tier: FAST, capabilities: [reasoning]}` (inline constraints).
**Alternatives:**
- Add ModelQuery alongside existing fields with coexistence validation — adds complexity, old fields redundant
- Keep flat fields, add toModelQuery() conversion — misses richer constraint dimensions and union type ergonomics
**Rationale:** Pre-release with no deployed instances — no backward compat needed. ModelQuery is a strict superset covering tier, capabilities, plus vendor, family, locality, cost, context window, output size, and vendor preference. Single field eliminates duplication.
**Trade-offs:** Breaking change to AgentCapability record — all existing callers (tests, examples, YAML profiles, annotations) must update. Acceptable given pre-release status.
**Sources:** AgentCapability.java (existing modelTier + modelCapabilities), ModelQuery.java (platform-api), model-selection.schema.json (agent-config-core), issue-172 spec (rationale for per-capability model requirements)
**Exploration:** quick
**Status:** captured

## D2: eidos-api depends on platform-api for ModelQuery

**Choice:** eidos-api takes a compile dependency on `casehub-platform-api` to use `ModelQuery` directly. No eidos-owned thin type or conversion layer.
**Alternatives:**
- eidos-owned sealed interface (ModelSelection.Ref / ModelSelection.Constraints) with runtime conversion to ModelQuery — duplicates all fields, adds converter, no real benefit
**Rationale:** platform-api is pure Java (no framework pull-in). The dependency direction is correct (eidos → platform-api). Eidos runtime already depends on platform-api in practice. Avoids type duplication and the conversion layer that #172 D1 deliberately avoided.
**Trade-offs:** eidos-api loses its zero-dependency status. Any consumer of eidos-api now transitively pulls in platform-api. Acceptable because platform-api is lightweight and the coupling is architecturally sound.
**Sources:** eidos api/pom.xml (current zero-dep), platform-api pom.xml (pure Java), issue-172 decisions D1 (boundary principle)
**Exploration:** quick
**Depends on:** D1 (using ModelQuery requires the dependency)
**Status:** captured
