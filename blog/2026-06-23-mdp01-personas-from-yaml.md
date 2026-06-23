---
layout: post
title: "Personas from YAML"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-eidos]
tags: [spi, cdi, yaml, drafthouse]
---

DraftHouse needs four reviewer agents — structural, content, readability, completeness — each with a distinct personality grounded in Thomas-Kilmann conflict modes and Conscientiousness disposition axes. The issue asked for hard-coded descriptors and a renderer validation test. I pushed the scope wider: if DraftHouse needs four personas now, devtown and claudony will need their own sets soon. Hard-coding each one in Java doesn't scale.

The right pattern was already sitting in the codebase. `VocabularyRegistrar` is a `@FunctionalInterface` that returns data — the registry handles storage. Agent descriptors needed the same thing: a declarative `AgentDescriptorRegistrar` where implementations return `List<AgentDescriptor>` and the infrastructure wires it up. The parallel is exact, down to the CDI discovery mechanism.

## Where the Bootstrap Lives

Claude and I spent the most design time on where auto-discovery should happen. The first spec put `Instance<AgentDescriptorRegistrar>` inside `InMemoryAgentRegistry` — mirroring how `CdiVocabularyRegistry` discovers vocabularies in its own `@PostConstruct`. The spec review caught why this breaks: `InMemoryAgentRegistry` is `@Alternative @Priority(1)`. It only activates when `casehub-eidos-memory` is on the classpath. The JPA production path gets nothing.

The vocabulary registry gets away with self-discovery because there's exactly one implementation and it's `@DefaultBean`. Agent registries have two implementations with different transactional semantics. A separate `AgentDescriptorBootstrap` bean in the runtime module, using `@Observes StartupEvent`, works with whichever `AgentRegistry` CDI resolves — and `@Transactional` interceptors are live, which `@PostConstruct` self-invocation wouldn't guarantee.

The bootstrap also needed `@IfBuildProperty` gating. In reactive-only mode without the memory module, no `AgentRegistry` bean exists at all — an ungated bootstrap would fail at build time with `UnsatisfiedResolutionException`.

## YAML, Not Java

The second design push was making persona definitions configuration rather than code. `ClasspathYamlDescriptorRegistrar` reads `META-INF/eidos/descriptors.yaml` from every JAR on the classpath via `ClassLoader.getResources()`. Adding a new reviewer persona is now a YAML change, not a Java change.

One constraint surfaced late: `AgentDescriptorValidator.isBanned()` bans all C0 control characters, including newlines. YAML literal block scalars (`|`) produce newlines. Multi-paragraph briefings — the whole reason `MAX_BRIEFING` was increased to 2000 — need `>-` (folded scalar) instead. The error message says `contains banned character U+000A` with no hint about YAML scalar styles. That one went straight to the garden.

## The CDI Displacement Fix

`EidosSystemPromptRenderer` was `@DefaultBean`. DraftHouse plans to ship a `@DefaultBean SimplePromptRenderer` as its standalone fallback. Two `@DefaultBean` implementations on the same type → `AmbiguousResolutionException`. The fix: drop `@DefaultBean` from eidos's renderer, making it plain `@ApplicationScoped`. When eidos is on the classpath, it wins. When absent, DraftHouse's fallback activates. Pattern B from the CDI displacement protocol.

The same `@DefaultBean` pattern exists on `CdiVocabularyRegistry` — same future risk, separate issue.

## What This Unlocks

DraftHouse can now add `casehub-eidos` as a dependency and get four vocabulary-grounded reviewer personas without writing any registration code. The YAML registrar means adding the fifth reviewer — or the twentieth — is a config change. The `AgentDescriptorRegistrar` SPI means devtown and claudony get the same mechanism when they're ready.
