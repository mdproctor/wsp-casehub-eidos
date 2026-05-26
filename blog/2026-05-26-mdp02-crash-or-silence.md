---
layout: post
title: "Crash or silence: symmetric nulls in a four-backend registry"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
---

The `tenancyId` null guard from last time left `agentId` exposed. `findById`
takes two required parameters — I only protected one of them.

## Crash or silence: what null means without a guard

Without `Objects.requireNonNull`, a null `agentId` behaves differently depending
on which registry backend is active. `InMemoryAgentRegistry` uses a `ConcurrentHashMap`,
which throws `NullPointerException` immediately — but with no message, which makes
the stack trace roughly as helpful as the error itself. `JpaAgentRegistry` passes
null to the JPQL parameter and silently returns empty. No exception, no complaint.

Two backends, two completely different failure modes, neither diagnosable.

The reactive JPA registry was the trickier one to test. The `@WithSession` CDI
interceptor wraps synchronous exceptions into failed Unis rather than letting them
propagate directly — I covered this last time. `assertThatNullPointerException().isThrownBy(...)`
doesn't work here; the NPE surfaces as a Uni failure instead.

`UniAsserter` has an obvious form: `assertFailedWith(supplier, NullPointerException.class)`.
That checks the type but not the message — which matters when two different failure
modes both produce NPEs. Claude decompiled `quarkus-test-vertx-3.32.2.jar` to check
whether a better overload existed. There was:

```java
asserter.assertFailedWith(
    () -> registry.findById(null, "default"),
    t -> assertThat(t).isInstanceOf(NullPointerException.class)
                      .hasMessageContaining("agentId"));
```

The `Consumer<Throwable>` overload isn't in the Quarkus documentation — the type-class
form is the one every example shows. We found it only by looking at the interface directly.

## Javadoc as the authoritative surface

The SPI interfaces — `AgentRegistry`, `ReactiveAgentRegistry` — had no `@throws`
documentation on `findById`. Once the guards exist, the interface is where callers
learn about them:

```java
/**
 * @throws NullPointerException if agentId or tenancyId is null
 */
Optional<AgentDescriptor> findById(String agentId, String tenancyId);
```

The interface is the contract; the Javadoc is what IDEs surface when a caller
autocompletes `findById`. That's the right place for it — not in a README, not
in a comment inside an implementation.

One small housekeeping item: `AgentQuery`'s compact constructor had been using
`"tenancyId must not be null"` as its message while everything else used the bare
field name. I normalized it. Inconsistency in null messages makes a stack trace
harder to read at 2am.
