# CaseHub Eidos — Design Document

## AgentRegistry

`AgentRegistry` (blocking) and `ReactiveAgentRegistry` (reactive, build-gated) provide the SPI for registering and discovering `AgentDescriptor` instances. All `findById(agentId, tenancyId)` implementations enforce `Objects.requireNonNull` on both `agentId` and `tenancyId`. Without explicit guards, failure modes differ by backend: `ConcurrentHashMap` throws an unhelpful messageless NPE; JPA silently returns empty. The null contract is documented via `@throws NullPointerException` on both SPI interfaces and on `AgentQuery`'s compact constructor. All null messages use the bare field name convention (`"agentId"`, `"tenancyId"`).

Implementations:
- `InMemoryAgentRegistry` / `InMemoryReactiveAgentRegistry` — in-memory, `casehub-eidos-memory`, `@Alternative @Priority(1)`
- `JpaAgentRegistry` — default, `@ApplicationScoped`, active when `casehub.eidos.reactive.enabled` is false or absent
- `JpaReactiveAgentRegistry` — build-gated via `@IfBuildProperty(name="casehub.eidos.reactive.enabled", stringValue="true")`
