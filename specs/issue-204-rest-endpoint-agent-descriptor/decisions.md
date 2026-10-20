# Decisions — eidos#204 REST endpoint for AgentDescriptor

## D1: Module placement

**Choice:** New `graphql/` module → `casehub-eidos-graphql`
**Alternatives:**
- Existing runtime module — every consumer gets endpoints automatically, no opt-in control
- New module with different name (domain/, mcp/) — more accurate but breaks ecosystem convention
**Rationale:** 4 of 5 CaseHub repos (engine, ledger, work, connectors) use `graphql/` for their @McpDomain modules. AML is the sole outlier with `api/`. Consistency outweighs naming precision.
**Trade-offs:** Name doesn't describe what the module does (REST + GraphQL + MCP generation), but it's the established convention.
**Sources:** engine/graphql/pom.xml, ledger/graphql/pom.xml, work/graphql/pom.xml, connectors/graphql/pom.xml, aml/api/pom.xml
**Exploration:** quick
**Status:** captured

## D2: @McpDomain hierarchical name

**Choice:** `@McpDomain("eidos/agents")`
**Alternatives:**
- `"eidos"` — flat, single domain; harder to extend with sub-domains later
- `"agents"` — top-level without eidos prefix; less scoped, conflicts risk
**Rationale:** Hierarchical naming follows the platform protocol (mcp-domain-hierarchical-naming.md). Leaves room for `eidos/vocabularies`, `eidos/org`, `eidos/templates` later. MCP progressive discovery navigates the tree.
**Trade-offs:** Slightly longer domain path in MCP tool names.
**Sources:** ledger/docs/protocols/casehub/mcp-domain-hierarchical-naming.md, engine @McpDomain values (engine/cases, engine/control)
**Exploration:** quick
**Status:** captured

## D3: Operations — list, get, find

**Choice:** Three @PlatformQuery operations: `listAgents`, `getAgent`, `findAgents`
**Alternatives:**
- List + Get only — matches issue scope exactly but leaves query functionality to a future issue
- Full CRUD with mutations — register/deregister via API; over-scoped for read-only UI consumption
**Rationale:** The full AgentQuery surface (slot, capability, goal, domain, maxDepth) is already implemented in AgentRegistry. Exposing it via findAgents with optional query params is low effort and immediately useful for MCP agent consumers.
**Trade-offs:** More surface area to maintain. Mutations excluded — registration stays programmatic (YAML/CDI).
**Sources:** api/AgentQuery.java, api/AgentRegistry.java
**Exploration:** quick
**Status:** captured

## D4: API shape — flat query parameters

**Choice:** Single `findAgents` with optional @QueryParam filters (slot, capability, goal, domain, maxDepth). Returns `List<AgentDescriptor>`, not `List<AgentMatch>`.
**Alternatives:**
- Separate list and find with match metadata — `findAgents` returns `List<AgentMatch>` carrying `resolvedCapability` with `MatchDegree`. Two overlapping list operations.
**Rationale:** Match metadata (MatchDegree, resolved capability) is an internal dispatch concern for engine, not needed by UI or MCP consumers. If match data becomes needed, add a separate `matchAgents` operation later.
**Trade-offs:** Capability match quality information is not exposed. Engine should use AgentRegistry directly, not this API.
**Sources:** api/AgentMatch.java, api/MatchDegree.java, api/AgentQuery.java
**Exploration:** quick
**Status:** captured

## D5: Projection — raw record with visibility filter

**Choice:** Return `AgentDescriptor` directly. Filter PRIVATE goals and constraints before serialization. No projection DTO.
**Alternatives:**
- Projected DTO with only UI-relevant fields — cleaner for blocks-ui but duplicates structure and requires manual mapping
- Raw record with no filtering — simplest but leaks PRIVATE goals/constraints
**Rationale:** All 27 fields are potentially useful — vocabulary URIs for MCP agents, providerConfigs for admin tooling, extensionData for domain apps. Jackson handles records natively. PRIVATE visibility filtering is a security requirement (PRIVATE items should only appear in owning agent's prompt).
**Trade-offs:** Internal fields (weightsFingerprint, vocabulary URIs) are visible to all callers. Acceptable given this is a platform-internal API, not public internet.
**Sources:** api/AgentDescriptor.java:169-175 (publicGoals/publicConstraints methods), api/Visibility.java
**Exploration:** quick
**Status:** captured

## D6: Tenancy resolution — CurrentPrincipal

**Choice:** Extract tenancyId from `@ContextParam CurrentPrincipal`. Callers don't pass tenancyId explicitly.
**Alternatives:**
- Explicit @QueryParam tenancyId — flexible for cross-tenant admin queries but bypasses auth scoping
- Both principal default with param override — complex, security risk without proper authorization checks
**Rationale:** Matches platform convention (connectors inject CurrentPrincipal). Tenancy is an auth concern, not a query parameter. Cross-tenant queries are an admin/superuser feature that can be added later with proper authorization.
**Trade-offs:** No cross-tenant queries without extending the API. Fine for the UI and MCP consumer use cases.
**Sources:** connectors/graphql/ConnectorOperationsImpl.java (@ContextParam usage), platform-api CurrentPrincipal
**Exploration:** quick
**Status:** captured
