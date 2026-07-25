# Descriptor Templates — Reusable Prose for Agent Personalities

**Issue:** casehubio/eidos#99
**Date:** 2026-07-14
**Status:** Draft

## Problem

Behavioral instructions live in a single `briefing` text field per `AgentDescriptor`. When multiple agents share genre-level conventions (e.g. "Hanna-Barbera cartoon character style") or role-level conventions (e.g. "cartoon villain monologue patterns"), the instructions must be copy-pasted into every briefing. This creates duplication and drift.

## Design

### Naming

The reusable prose fragments are called **templates**. A template with no parameters is a static template; a template with `${variable}` placeholders is a parameterized template. Both are the same type — `DescriptorTemplate`.

The name "resource" was rejected to avoid collision with the existing `Resource` record (`uri`/`label`/`type` for agent-accessible external resources in `AgentPromptContext`).

### Scope

Templates are **identity** — part of who the agent is, declared at registration time on `AgentDescriptor`. They are not per-render or per-invocation. If a future use case requires situational prose composition at render time, the same `TemplateRegistry` machinery can support `AgentPromptContext.withTemplates()` without redesign.

### Storage

Templates are developer-authored configuration artifacts loaded from the classpath. No database storage. The registry is disk-backed via YAML files in JARs.

---

## API Types (`casehub-eidos-api`)

### `DescriptorTemplate`

```java
public record DescriptorTemplate(
    String id,                    // unique identifier, e.g. "hanna-barbera-cartoon-style"
    String name,                  // human-readable label
    List<String> parameters,      // declared parameter names (empty for static templates)
    String content                // prose with ${variable} placeholders
) {}
```

### `TemplateRef`

A descriptor's reference to a template, with optional parameter values:

```java
public record TemplateRef(
    String templateId,            // which template to resolve
    Map<String, String> args      // parameter values (empty for static templates)
) {}
```

### `TemplateRegistry` (SPI)

```java
public interface TemplateRegistry {
    void register(DescriptorTemplate template);
    Optional<DescriptorTemplate> resolve(String id);
    List<DescriptorTemplate> all();
}
```

### `TemplateRegistrar` (CDI SPI, in `api/spi/`)

```java
@FunctionalInterface
public interface TemplateRegistrar {
    List<DescriptorTemplate> templates();
}
```

### `AgentDescriptor` Changes

New field: `List<TemplateRef> templates` — nullable, ordered. Builder gets `.templates(List<TemplateRef>)`.

---

## Validation

### At template registration time (`CdiTemplateRegistry`)

- `id` required, non-blank
- `content` required, non-blank
- `parameters` entries non-blank (no empty parameter names)
- Duplicate ID detection (hard error)
- `${variable}` placeholders in `content` must all match declared `parameters` (catches typos in template authoring)

### At descriptor registration time (`DescriptorCollector`)

`DescriptorCollector` already validates capability vocabularies against `VocabularyRegistry`. Template ref validation follows the same pattern — it requires a live `TemplateRegistry`, so it belongs in `DescriptorCollector` (runtime), not `AgentDescriptorValidator` (api, no registry access).

- Every `TemplateRef.templateId` must resolve to a registered template
- Every declared parameter in the template must have a corresponding arg in the ref
- No extra args that don't match declared parameters

All validation failures are hard errors — fail-fast at startup.

---

## YAML Loading

### Template file: `META-INF/eidos/templates.yaml`

Single file per JAR, mirrors the descriptor pattern. Multiple JARs can each contribute their own file.

```yaml
templates:
  - id: hanna-barbera-cartoon-style
    name: Hanna-Barbera Cartoon Character Style
    content: |
      You are a character in a Hanna-Barbera cartoon. You follow these
      conventions:

      EXPOSITORY SOLILOQUY: You frequently narrate your situation aloud —
      describing what is happening to you, how you feel about it, and what
      you intend to do next.

      EMOTIONAL TELEGRAPHING: State your emotions explicitly and with
      exaggeration. "Why, I'm simply DELIGHTED!" not "That's nice."

  - id: cartoon-villain
    name: Cartoon Villain Archetype
    parameters: [catchphrase, nemesis, scheme_style]
    content: |
      As a villain, your catchphrase is "${catchphrase}". Your nemesis is
      ${nemesis}. You monologue your plans in a ${scheme_style} manner.
      You explain your scheme step by step, especially when you believe
      you are about to succeed.
```

### Descriptor references in `META-INF/eidos/descriptors.yaml`

```yaml
descriptors:
  - agentId: hooded-claw
    name: The Hooded Claw
    templates:
      - ref: hanna-barbera-cartoon-style
      - ref: cartoon-villain
        args:
          catchphrase: "Nyah-ha-ha-HA!"
          nemesis: Penelope Pitstop
          scheme_style: grandiose and theatrical
    briefing: >-
      You are The Hooded Claw, Penelope Pitstop's secret guardian
      who moonlights as her nemesis...
```

---

## Runtime

### `CdiTemplateRegistry`

`@ApplicationScoped`. Discovers `Instance<TemplateRegistrar>` CDI beans at startup. Stores templates in `Map<String, DescriptorTemplate>` by ID. Duplicate ID detection at registration.

### `ClasspathYamlTemplateRegistrar`

`@ApplicationScoped`, implements `TemplateRegistrar`. Scans classpath for `META-INF/eidos/templates.yaml` using `Thread.currentThread().getContextClassLoader().getResources()`. Same pattern as `ClasspathYamlDescriptorRegistrar`.

### Bootstrap ordering

Templates must be available before descriptor validation. Current flow:

1. `CdiVocabularyRegistry` populates
2. `AgentDescriptorBootstrap` fires (`@Observes StartupEvent`)

New flow:

1. `CdiVocabularyRegistry` populates
2. `CdiTemplateRegistry` populates (eager injection in bootstrap)
3. `AgentDescriptorBootstrap` validates template refs against `TemplateRegistry`

Eager injection — the bootstrap already injects `VocabularyRegistry` this way.

### `InMemoryTemplateRegistry`

In `casehub-eidos-memory`. `@Alternative @Priority(1)`. For tests — programmatic `register()` without YAML files. Same pattern as `InMemoryAgentRegistry`.

---

## Render Pipeline Integration

### Variable substitution

Simple `String.replace()` loop over declared parameters. `${variable}` → literal value. No expression engine, no escaping, no recursion.

### Template resolution

New method on `EidosRenderPipeline` resolves and concatenates all template refs for a descriptor, substituting parameters.

### Assembly order (MARKDOWN)

1. Header (name, agent ID, model, provider)
2. Role (slot)
3. **Templates** (resolved prose) — **new**
4. Capabilities
5. Disposition / briefing
6. Data handling
7. Goal
8. Resources
9. Situational context

### PROSE

Same ordering. Templates rendered as paragraphs before capabilities.

### A2A_CARD

Templates do not appear in the A2A card JSON. They are behavioural prose, not machine-readable metadata. Same principle as briefing.

### LLM enrichment

Resolved template content is included in `buildDescriptorPayload()` so the LLM sees it during semantic enrichment. Added as a `templates` string field on the JSON payload (already resolved and substituted).

### Cache key

Resolved template content is included in the descriptor hash for `RenderedPromptCache`. Template content changes → cache invalidation. Consistent with the template-hash-input-coverage protocol.

---

## Module Placement

| Module | Changes |
|---|---|
| `casehub-eidos-api` | `DescriptorTemplate`, `TemplateRef`, `TemplateRegistry` (SPI), `TemplateRegistrar` (CDI SPI) |
| `casehub-eidos` (runtime) | `CdiTemplateRegistry`, `ClasspathYamlTemplateRegistrar`, validation in `DescriptorCollector`, render pipeline integration |
| `casehub-eidos-memory` | `InMemoryTemplateRegistry` |
| `casehub-eidos-deployment` | `EidosProcessor` update if build-time registration needed |

### Existing files modified

- `AgentDescriptor` — new `templates` field + builder
- `AgentDescriptorValidator` — template ref validation
- `ClasspathYamlDescriptorRegistrar` — parse `templates:` refs from descriptor YAML
- `EidosRenderPipeline` — template resolution, assembly, cache key, enrichment payload
- `AgentDescriptorEntity` / `AgentDescriptorMapper` — persist template refs as JSON column
- `AgentDescriptorComparator` — include templates in drift detection

### No changes to

- `casehub-eidos-vocab`
- `casehub-eidos-eval`
- No JPA entities for templates, no Flyway migrations

---

## Use Cases

- **Entertainment:** "Hanna-Barbera cartoon style" shared across five character agents
- **Compliance:** "Regulatory communication standards" shared across 20 compliance agents
- **Healthcare:** "Clinical communication" parameterized per specialty (ICU vs general ward)
- **Customer service:** "Brand voice" shared across all support agents
- **Legal:** "Contract review conventions" shared across reviewer agents
