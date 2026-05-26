---
layout: post
title: "Interceptors Eat Exceptions"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [tdd, null-safety, quarkus, reactive, cdi, interceptor]
---

eidos#10: `findById(String agentId, String tenancyId)` had no null guard on `tenancyId`. The blocking impls silently returned `Optional.empty()` — `"default".equals(null)` is false, not an exception. The JPA version passed null to JPQL, which also returns empty silently. Both are wrong. The fix is `Objects.requireNonNull(tenancyId, "tenancyId")` at the top of all four implementations — consistent with how `AgentQuery` already enforces non-null in its constructor.

The interesting part was the reactive case. Writing the test for `JpaReactiveAgentRegistry`:

```java
assertThatThrownBy(() -> registry.findById("any", null))
    .isInstanceOf(NullPointerException.class);
```

This fails — not because the null guard is missing, but because `@WithSession` is a CDI interceptor that catches all exceptions from the method body and re-emits them as `Uni` failures. The synchronous `Objects.requireNonNull` throw never propagates out of the proxy. The test sees no throwable and fails with "Expecting code to raise a throwable."

The fix is to test via the Uni channel:

```java
@Test
@RunOnVertxContext
void findById_with_null_tenancyId_throws(UniAsserter asserter) {
    asserter.assertFailedWith(() -> registry.findById("any", null), NullPointerException.class);
}
```

The in-memory reactive registry delegates synchronously — `delegate.findById(agentId, tenancyId)` is evaluated before `Uni.createFrom().item(...)` wraps it — so the NPE propagates normally there. The two reactive implementations have different failure propagation semantics depending on whether there's an interceptor in the call path.

Code review (code-reviewer subagent) caught the gap: the initial diff only fixed three of four implementations. `JpaReactiveAgentRegistry` was missed entirely — a CRITICAL finding that blocked the commit. Caught and fixed before merge.

Filed as follow-on: eidos#11 (agentId null guard — symmetric with tenancyId), eidos#12 (Javadoc on SPI interface declaring the null contract).
