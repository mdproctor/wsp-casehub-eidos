# Design Journal — issue-10-inmemory-registry-tenancyid-null-guard

### 2026-05-26 · §AgentRegistry

`findById(String agentId, String tenancyId)` now enforces `Objects.requireNonNull(tenancyId)`
at the top of all four implementations (InMemoryAgentRegistry, JpaAgentRegistry,
JpaReactiveAgentRegistry — InMemoryReactiveAgentRegistry inherits the guard via delegation).
This aligns the method's contract with `AgentQuery`, which already enforces non-null
`tenancyId` in its constructor, and makes silent empty-return on null an impossible outcome.

Discovery: the Quarkus `@WithSession` CDI interceptor catches synchronous throws and
defers them as `Uni` failures. Tests for `JpaReactiveAgentRegistry` must therefore
use `@RunOnVertxContext` + `UniAsserter.assertFailedWith` — not `assertThatThrownBy`
on the raw method call.

Follow-on issues filed: eidos#11 (agentId null guard), eidos#12 (Javadoc on SPI interface).
