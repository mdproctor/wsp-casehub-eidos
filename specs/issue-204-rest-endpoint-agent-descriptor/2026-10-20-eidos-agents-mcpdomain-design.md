# Design: @McpDomain for AgentDescriptor — eidos/agents

**Issue:** eidos#204
**Branch:** issue-204-rest-endpoint-agent-descriptor

## Summary

Expose `AgentRegistry` data via the platform's `@McpDomain` unified API
framework. A single `@McpDomain("eidos/agents")` class generates REST,
GraphQL, and MCP tool endpoints from one definition — no hand-written
endpoints.

## Module

New Maven module `graphql/` with artifactId `casehub-eidos-graphql`.
Follows the ecosystem convention (engine, ledger, work, connectors all
use `graphql/` for their @McpDomain modules). Consuming apps opt-in by
adding the dependency.

### Module structure

```
graphql/
  pom.xml                          (casehub-eidos-graphql)
  src/main/java/io/casehub/eidos/graphql/
    EidosAgentApi.java             (@McpDomain class)
```

### Dependencies

- `casehub-eidos-api` — AgentRegistry, AgentDescriptor, AgentQuery
- `casehub-eidos` (runtime) — CdiVocabularyRegistry and runtime beans
- `casehub-platform-api` — @McpDomain, @PlatformQuery, CurrentPrincipal
- `quarkus-arc` — CDI
- `jandex-maven-plugin` — index for GraphQLModelScanner discovery

### Build configuration

`maven-compiler-plugin` with `casehub-platform-graphql-generator` as
annotation processor path. Generates:
- JAX-RS resource class (REST endpoints)
- GraphQL resolver class (GraphQL queries)

MCP tool registration happens at runtime via `GraphQLModelScanner`.

```xml
<annotationProcessorPaths>
    <path>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-graphql-generator</artifactId>
        <version>${casehub.version}</version>
    </path>
    <path>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-eidos-api</artifactId>
        <version>${project.version}</version>
    </path>
</annotationProcessorPaths>
<compilerArgs>
    <arg>-AdomainFilter=eidos/agents</arg>
</compilerArgs>
```

## @McpDomain class: EidosAgentApi

```java
@McpDomain(value = "eidos/agents", app = "eidos",
    basePath = "/api/eidos/agents",
    summary = "Agent identity registry — rich personality, capability, and disposition data")
@ApplicationScoped
public class EidosAgentApi {

    private final AgentRegistry registry;
    private final CurrentPrincipal currentPrincipal;

    @Inject
    public EidosAgentApi(AgentRegistry registry, CurrentPrincipal currentPrincipal) {
        this.registry = registry;
        this.currentPrincipal = currentPrincipal;
    }
}
```

## Operations

Two operations. `findAgents` subsumes listing (no params = list all).
Both use `@RestPath("")` for clean REST paths — the generator would
otherwise kebab-case the method names into path segments.

### 1. findAgents

```java
@PlatformQuery("Find agents matching optional criteria — no params lists all")
@RestPath("")
public List<AgentDescriptor> findAgents(
    @QueryParam("slot") String slot,
    @QueryParam("capability") String capabilityName,
    @QueryParam("goal") String goalName,
    @QueryParam("domain") String taskDomain,
    @QueryParam("maxDepth") Integer maxDepth)
```

- Builds `AgentQuery` from provided parameters + `currentPrincipal.tenancyId()`
- All parameters are optional — no params = list all
- Calls `registry.find(query)`
- Maps `AgentMatch::descriptor`, filters PRIVATE visibility
- Returns `List<AgentDescriptor>` (match metadata not exposed — D4)

**Generated endpoints:**
- REST: `GET /api/eidos/agents?slot=X&capability=Y&goal=Z&domain=D&maxDepth=N`
- GraphQL: `query { findAgents(slot: "X", capability: "Y") { ... } }`
- MCP: `casehub_action(domain="eidos/agents", operation="findAgents", params={capability: "code-review"})`

### 2. getAgent

```java
@PlatformQuery("Get a specific agent descriptor by ID")
@RestPath("")
public AgentDescriptor getAgent(@PathParam("agentId") String agentId)
```

- Calls `registry.findById(agentId, currentPrincipal.tenancyId())`
- Returns the descriptor with PRIVATE goals/constraints filtered
- Returns null (404 via generated resource) when not found

**Generated endpoints:**
- REST: `GET /api/eidos/agents/{agentId}`
- GraphQL: `query { getAgent(agentId: "...") { ... } }`
- MCP: `casehub_action(domain="eidos/agents", operation="getAgent", params={agentId: "..."})`

## Visibility filtering

`AgentDescriptor` goals and constraints have `Visibility.PUBLIC/PRIVATE`.
PRIVATE items appear only in the owning agent's system prompt — they must
not leak via the API.

Before returning any descriptor, rebuild it with filtered lists:

```java
private AgentDescriptor filterPrivate(AgentDescriptor d) {
    return d.toBuilder()
        .goals(d.publicGoals())
        .constraints(d.publicConstraints())
        .build();
}
```

This uses the existing `publicGoals()` and `publicConstraints()` methods
on `AgentDescriptor` which filter by `Visibility.PUBLIC`.

## Serialization

`AgentDescriptor` is a Java record. Jackson handles records natively via
`quarkus-jackson`. Nested types (`AgentDisposition`, `AgentCapability`,
`AgentGoal`, `AgentConstraint`, `AgentVoiceProfile`, `ProviderConfig`,
`TemplateRef`) are all records or simple types.

Key serialization considerations:
- `DispositionAxis` enum as map key in `axisVocabularies` — serializes
  as enum name by default
- `ModelQuery` from `casehub-platform-api` — already Jackson-serializable
  in platform
- `Map<String, Object> extensionData` — passthrough, Jackson handles
- `GoalLifecycleState`, `GoalHorizon` enums — serialize as names

No custom serializer or DTO needed.

## Error handling

- `getAgent` with unknown ID: returns null → generated resource produces
  404. If the generator doesn't handle null→404, add `@RestStatus(404)`
  or return `Optional<AgentDescriptor>` (check generator behavior).
- Invalid query combinations: `AgentQuery` constructor validates
  (`tenancyId` required, `maxDepth >= 1`) — invalid state shouldn't
  occur since tenancyId comes from principal and maxDepth is optional.

## Testing

- Unit test `EidosAgentApi` with mocked `AgentRegistry` and
  `CurrentPrincipal` — verify visibility filtering, query construction,
  null handling
- Integration test with `@QuarkusTest` — register descriptors with
  PRIVATE goals, call the API, assert PRIVATE items are absent
- REST integration test via `RestAssured` — verify generated endpoints
  respond correctly (200 for list, 404 for missing agent)

## Scope exclusions

- No pagination — descriptor counts are small (tens, not thousands)
- No mutations — registration stays programmatic (YAML/CDI)
- No match metadata (MatchDegree) — internal dispatch concern
- No ModelEnricher — can be added later for richer MCP discovery
- `#205` (AgentDescriptorYamlRegistrar) is a separate branch

## References

- eidos#204 — issue
- eidos#205 — related: YAML bootstrap registrar (separate branch)
- `api/AgentRegistry.java` — registry SPI
- `api/AgentDescriptor.java` — 27-field record with builder
- `api/AgentQuery.java` — query criteria with factory methods
- `api/Visibility.java` — PUBLIC/PRIVATE enum
- `platform-api/mcp/McpDomain.java` — unified API annotation
- `platform-api/mcp/PlatformQuery.java` — read operation annotation
- `platform-api/identity/CurrentPrincipal.java:61` — tenancyId() method
- `connectors/graphql/ConnectorOperationsImpl.java` — reference @McpDomain implementation
- `ledger/docs/protocols/casehub/mcp-domain-hierarchical-naming.md` — naming protocol
