# Design Journal — issue-011-registry-agentid-guard

### 2026-05-26 · §AgentRegistry

`findById(agentId, tenancyId)` now enforces `Objects.requireNonNull` on `agentId`
symmetrically with the existing `tenancyId` guard across all four implementations
(InMemoryAgentRegistry, JpaAgentRegistry, JpaReactiveAgentRegistry,
InMemoryReactiveAgentRegistry). Previously, a null `agentId` produced
implementation-specific failure: ConcurrentHashMap threw an unhelpful NPE,
while JPA silently returned empty. The contract is now identical and diagnosable
regardless of which backend is active. `@throws NullPointerException` is
documented on both SPI interfaces and on `AgentQuery`'s compact constructor.
`AgentQuery`'s null message was normalized from verbose form to the bare field
name `"tenancyId"` for consistency with the rest of the codebase.
