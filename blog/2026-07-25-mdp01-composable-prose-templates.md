---
layout: post
title: "Composable Prose — Templates for Agent Personalities"
date: 2026-07-25
type: phase-update
entry_type: note
subtype: diary
projects: [eidos]
tags: [templates, spi, render-pipeline]
---

## The Briefing Duplication Problem

Five cartoon characters in the Wacky Manor POC all share the same Hanna-Barbera conventions — expository soliloquy, emotional telegraphing, catchphrase repetition. Each descriptor carried an identical block of prose in its `briefing` field. Change one, forget the others, drift.

The `briefing` field on `AgentDescriptor` was never meant to carry shared genre conventions. It was designed for agent-specific principles that the structured disposition axes can't express. Using it for shared prose creates a copy-paste maintenance problem that scales with the number of agents in a team.

## What Templates Are

A `DescriptorTemplate` is a reusable prose fragment — an `id`, a `name`, optional `parameters`, and `content` with `${variable}` placeholders. A `TemplateRef` on `AgentDescriptor` references a template by ID and supplies parameter values. The renderer resolves refs, substitutes parameters, and composes the result into the system prompt before disposition and briefing.

The interesting design decision was where templates sit conceptually. They're identity, not context — declared at registration time, not per-render. A "Hanna-Barbera cartoon style" template is part of who the agent is, not something that changes between invocations. If a future use case needs situational prose at render time, the same registry machinery can support it — but we're not building that now.

## Three-Layer Validation

The validation architecture ended up being the most carefully designed part. Three layers, each catching different failure modes:

**Layer 1** — compact constructors in the api module. `DescriptorTemplate` and `TemplateRef` validate structurally at construction time: non-blank required fields, max length, character-set filtering against BiDi overrides and control characters. No invalid record can exist.

**Layer 2** — template registration in `CdiTemplateRegistry`. When a template is registered, every `${variable}` placeholder in its content is checked against its declared parameters. Catches typos in template authoring — a `${nemsis}` where the parameter list says `nemesis`.

**Layer 3** — descriptor registration in `DescriptorCollector`. Every template ref on a descriptor must resolve to a registered template, every declared parameter must have a corresponding arg, no extra args allowed. This is where the "references unknown template" error fires.

The design review caught a factual error in the original spec — I'd written that `DescriptorCollector` already validates capability vocabularies, which it doesn't. Capability vocabulary validation runs downstream in `AgentRegistry.register()` via `CapabilityVocabularyValidator`. Template ref validation genuinely belongs in the collector because it should fail fast before any descriptors are registered.

## Single-Pass Substitution

The design review also caught a cross-parameter injection vulnerability in the original `String.replace()` loop approach. If parameter `greeting` has value `"Hello ${name}"` and parameter `name` has value `"Alice"`, a multi-pass loop would expand `greeting` to `"Hello Alice"` — silently modifying a user-supplied value.

The fix is single-pass regex replacement using `Pattern.compile("\\$\\{([^}]+)}")` with `Matcher.replaceAll()`. Each placeholder is matched once and replaced with the corresponding arg value. Values containing `${...}` patterns are never re-scanned.

## The CDI Alternative Gotcha

The most time-consuming bug was invisible. `InMemoryTemplateRegistry` (`@Alternative @Priority(1)`) replaces `CdiTemplateRegistry` in test environments. But `CdiTemplateRegistry` populates itself at `@PostConstruct` by discovering `Instance<TemplateRegistrar>` CDI beans — and the alternative had no such logic. It started empty.

The symptom was `DescriptorCollector` rejecting all template refs with "unknown template" errors, despite the `ClasspathYamlTemplateRegistrar` bean existing on the classpath. The registrar was never invoked because nobody called it — the `@PostConstruct` on `CdiTemplateRegistry` never fires when the alternative replaces it.

CDI `@Alternative` fully replaces the bean. No lifecycle method delegation. The alternative must independently implement the same self-population.

## Disk, Not Database

Templates are developer-authored configuration loaded from `META-INF/eidos/templates.yaml` on the classpath. No database storage for the template content itself — only the refs on `AgentDescriptorEntity` are persisted as a JSON TEXT column. This keeps templates alongside the code that uses them, version-controlled and deployed as part of the application.

The full SPI (`TemplateRegistry`, `TemplateRegistrar`, `CdiTemplateRegistry`, `InMemoryTemplateRegistry`) follows the same pattern as `VocabularyRegistry`. Multiple JARs can each contribute their own `templates.yaml` — a compliance library providing "regulatory communication" templates, a customer service library providing "brand voice" templates, composed at startup.
