---
layout: post
title: "Bridging Identity to HTTP"
date: 2026-10-29
entry_type: article
subtype: diary
projects: [casehubio/eidos]
tags: [eidos, spi, http, agent-identity, platform-integration]
series: issue-208-agent-resolver-spi
---

# Bridging Identity to HTTP

Eidos agents have always been JVM-resident. You inject `AgentRegistry`, query by capability or slot, render a system prompt, and pass it to the LLM. All of that happens inside the same process. If you're outside the JVM — a deployment test harness, an external integration, a browser-based agent catalog — you get nothing.

Platform changed this by adding `POST /api/agent/chat` with two modes: explicit (you bring your own system prompt) and resolved (you provide an agentId, the server does the rest). The resolved path needs someone to turn an agentId into a rendered system prompt. That's eidos's job.

## The SPI

Platform defines `AgentResolver` — two methods:

```java
public interface AgentResolver {
    Optional<ResolvedAgent> resolve(String agentId);
    List<AgentDescriptor> discover();
}
```

The implementation is `EidosAgentResolver`. The resolve path does three things: finds the agent, renders its identity into a system prompt, and maps the result to the platform's wire type.

```java
@Override
public Optional<ResolvedAgent> resolve(String agentId) {
    return registry.findById(agentId, currentPrincipal.tenancyId())
            .map(this::toResolvedAgent);
}

private ResolvedAgent toResolvedAgent(AgentDescriptor descriptor) {
    var prompt = renderer.render(descriptor,
            AgentPromptContext.forFormat(RenderFormat.MARKDOWN));
    return new ResolvedAgent(
            descriptor.agentId(),
            prompt.content(),
            resolveModel(descriptor),
            null,
            buildMetadata(descriptor));
}
```

An HTTP client sends `{"agentId": "legal-reviewer"}`. The endpoint calls `resolve("legal-reviewer")`. Eidos looks up the descriptor, renders the full system prompt — role, capabilities, disposition, goals, constraints, templates — and returns it wrapped in a `ResolvedAgent`. The caller never sees the 29-field descriptor. It gets a system prompt and enough metadata to route the request.

`discover()` is the other half: enumerate all agents the current tenant can see. It queries `AgentQuery.all(tenancyId)` and maps each match to platform's thin `AgentDescriptor` — four fields where eidos has twenty-nine. The HTTP surface doesn't need disposition axes or template refs. It needs agentId, a display name, a description, and whatever metadata matters for routing.

## Design decisions

**Tenancy is ambient.** The SPI only takes `agentId` — no tenancyId parameter. Tenancy comes from `CurrentPrincipal`, which is `@RequestScoped` and derived from the authenticated security context. The resolver is inherently multi-tenant. An HTTP client authenticated against tenant A sees tenant A's agents. No query parameter, no header, no trust-the-caller. The security context is the source of truth.

**`@DefaultBean` yields to consumers.** Eidos provides the default implementation. If a consumer wants custom resolution logic — maybe they filter agents by deployment environment, or they render prompts differently — they provide their own `AgentResolver` as an `@Alternative @Priority(1)` and the eidos default steps aside. Same pattern as `SimpleAgentSelector`.

**Model resolution is pragmatic.** `ResolvedAgent` carries a nullable `model` field. Eidos walks the agent's capabilities and takes the first `modelRef` it finds. Most agents have a primary capability that defines their model needs — a code reviewer wants Opus, a summariser wants Haiku. If no capability declares a model preference, the field is null and the platform picks a default at dispatch time. There is no attempt to merge or reconcile multiple capabilities' model preferences. First one wins. If that turns out to be wrong, the fix is explicit ordering on the descriptor, not heuristics in the resolver.

**Metadata carries what the HTTP layer needs to route.** Slot, domainVocabulary, and archetype flow through as a `Map<String, String>`. The HTTP endpoint can use these for request routing, logging, or display without depending on eidos types. The metadata map is the decompression boundary between a rich domain model and a thin wire protocol.

## Two AgentDescriptors

Both eidos and platform define a record called `AgentDescriptor`. Eidos has the full identity model — twenty-nine fields covering everything from jurisdiction to disposition axes to archetype adjectives. Platform has four fields: agentId, displayName, description, metadata. The implementation uses fully qualified names for platform's version where both appear in the same scope. This is a genuine design tension — two legitimate concepts with the same name — but the right resolution is keeping them separate and mapping at the boundary. The rich model is internal. The thin model is the API surface.

## What this opens up

HTTP clients can now invoke agents by ID. Send an agentId, get a fully rendered system prompt back, pass it to the LLM. No CDI, no classpath dependency on eidos, no knowledge of disposition profiles or vocabulary registries. The rendered prompt captures all of that.

This also means deployment testing can exercise real agent identities over HTTP. The functional exerciser in casehub-ops can POST to a running instance, resolve a real agent, and verify the prompt renders correctly — without mocking the registry or faking the identity model. External integrations get the same capability: structured agent access without JVM coupling.

The `discover()` method is the more interesting long-term piece. An HTTP client can enumerate all agents available to the current tenant. That's the foundation for agent catalogs, selection UIs, and any system that needs to present available capabilities to a user before invocation. The agents are already registered. Now they're visible.
