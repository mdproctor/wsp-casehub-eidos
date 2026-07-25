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
) {
    public DescriptorTemplate {
        AgentDescriptorValidator.validateRequired("template.id", id, MAX_TEMPLATE_ID);
        AgentDescriptorValidator.validateRequired("template.name", name, MAX_TEMPLATE_NAME);
        AgentDescriptorValidator.validateRequired("template.content", content, MAX_TEMPLATE_CONTENT, 0x000A);
        parameters = parameters != null ? List.copyOf(parameters) : List.of();
        AgentDescriptorValidator.validateItems("template.parameters", parameters, MAX_PARAMETER_NAME);
    }
}
```

Validation constants added to `AgentDescriptorValidator`:

| Constant | Value | Rationale |
|---|---|---|
| `MAX_TEMPLATE_ID` | 100 | Same as `MAX_SLOT`, `MAX_CAPABILITY_NAME` — short identifier |
| `MAX_TEMPLATE_NAME` | 200 | Same as `MAX_NAME` — human-readable label |
| `MAX_TEMPLATE_CONTENT` | 4000 | Larger than `MAX_BRIEFING` (2000) because templates are shared genre/style guides covering conventions for multiple agents; per-template bound, not aggregate |
| `MAX_PARAMETER_NAME` | 100 | Same as `MAX_CAPABILITY_NAME` — short identifier |

Content is validated against the same `isBanned()` character-set rules as all other prompt-facing strings (BiDi overrides, control characters, zero-width joiners). Newlines (`0x000A`) are allowed since template content is multi-line prose.

### `TemplateRef`

A descriptor's reference to a template, with optional parameter values:

```java
public record TemplateRef(
    String templateId,            // which template to resolve
    Map<String, String> args      // parameter values (empty for static templates)
) {
    static final int MAX_TEMPLATE_ARG_VALUE = 1000;

    public TemplateRef {
        AgentDescriptorValidator.validateRequired("templateRef.templateId", templateId, AgentDescriptorValidator.MAX_TEMPLATE_ID);
        args = args != null ? Map.copyOf(args) : Map.of();
        AgentDescriptorValidator.validateMapKeys("templateRef.args", args.keySet(), AgentDescriptorValidator.MAX_PARAMETER_NAME);
        AgentDescriptorValidator.validateItems("templateRef.args.values", args.values(), MAX_TEMPLATE_ARG_VALUE);
    }
}
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

Follows the `AgentDescriptorRegistrar` return-data pattern: the registrar returns data, and the caller (`CdiTemplateRegistry`) iterates and registers each template. Both existing registrar SPIs (`AgentDescriptorRegistrar.descriptors()`, `VocabularyRegistrar.vocabulary()`) follow this same pattern.

### `AgentDescriptor` Changes

New field: `List<TemplateRef> templates` — nullable, ordered. Builder gets `.templates(List<TemplateRef>)`.

---

## Validation

### Three validation layers

**Layer 1 — Compact constructors (api module, construction time)**

`DescriptorTemplate` and `TemplateRef` validate structural field correctness at construction time via `AgentDescriptorValidator`: non-blank required fields, max length, character-set filtering (`isBanned()`). No invalid record can be constructed — consistent with the platform quality goal (ARC42STORIES §1).

**Layer 2 — Template registration (`CdiTemplateRegistry`, runtime)**

- Duplicate ID detection (hard error)
- `${variable}` placeholders in `content` must all match declared `parameters` (catches typos in template authoring)

**Layer 3 — Descriptor registration (`DescriptorCollector`, runtime)**

Template ref validation requires a live `TemplateRegistry`, so it belongs in `DescriptorCollector` (runtime), not `AgentDescriptorValidator` (api, no registry access). This is a new validation responsibility for `DescriptorCollector` — it currently validates only duplicate agent IDs. Capability vocabulary validation follows a different pattern: it runs in `AgentRegistry.register()` implementations via `CapabilityVocabularyValidator`. Template ref validation belongs in the collector rather than the registry because it is a cross-cutting concern that should fail fast before any descriptors are registered, regardless of registry implementation.

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

### YAML → Java mapping for template refs

The descriptor YAML uses `ref:` for readability; the Java `TemplateRef` record uses `templateId`. `ClasspathYamlDescriptorRegistrar` uses Jackson for deserialization — the YAML config class (`TemplateRefConfig`) maps `ref` → `TemplateRef.templateId` during construction. `args` maps directly.

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

1. `CdiVocabularyRegistry` populates at `@PostConstruct`
2. `AgentDescriptorBootstrap` fires (`@Observes StartupEvent`), injects `AgentRegistry` and `Instance<AgentDescriptorRegistrar>`

New flow:

1. `CdiVocabularyRegistry` populates at `@PostConstruct`
2. `CdiTemplateRegistry` populates at `@PostConstruct` (discovers `Instance<TemplateRegistrar>`, registers all templates)
3. `AgentDescriptorBootstrap` fires, with `TemplateRegistry` injected — CDI ensures `CdiTemplateRegistry` is fully constructed and populated before injection
4. `DescriptorCollector.collectAndValidate()` validates template refs against the injected `TemplateRegistry`

`CdiTemplateRegistry` follows the `CdiVocabularyRegistry` self-population pattern: `@PostConstruct` discovers CDI `TemplateRegistrar` beans and registers their templates. The bootstrap declares `@Inject TemplateRegistry` to trigger CDI ordering — it passes the registry to `DescriptorCollector` for ref validation. Note: `AgentDescriptorBootstrap` does NOT currently inject `VocabularyRegistry` — vocabulary validation happens downstream in `AgentRegistry.register()` via `CapabilityVocabularyValidator`.

### `InMemoryTemplateRegistry`

In `casehub-eidos-memory`. `@Alternative @Priority(1)`. For tests — programmatic `register()` without YAML files. Same pattern as `InMemoryAgentRegistry`.

---

## Render Pipeline Integration

### Variable substitution

Single-pass regex replacement using `Pattern.compile("\\$\\{([^}]+)\\}")` with `Matcher.replaceAll(matchResult -> ...)`. Each `${variable}` in the template content is matched once and replaced with the corresponding arg value from the `TemplateRef`. Unmatched placeholders are left as-is (caught earlier by Layer 3 validation).

Single-pass guarantees no cross-parameter injection: if an arg value itself contains `${...}` patterns, those are never expanded because they are inserted as replacement text, not re-scanned as template content. No expression engine, no escaping, no recursion.

### Template resolution

New method on `EidosRenderPipeline` resolves and concatenates all template refs for a descriptor, substituting parameters.

### Assembly order (MARKDOWN)

1. Header (name, agent ID, model, provider)
2. Role (slot)
3. Capabilities
4. **Templates** (resolved prose) — **new**
5. Disposition / briefing
6. Data handling
7. Goal
8. Resources
9. Situational context

Templates are placed immediately before disposition/briefing — both are behavioural prose and adjacency lets the LLM process all personality/behavioural instructions as one coherent block. Capabilities (a technical abilities list) sits before both.

### PROSE

Same ordering. Templates rendered as paragraphs before disposition/briefing.

### A2A_CARD

Templates do not appear in the A2A card JSON. They are behavioural prose, not machine-readable metadata. Same principle as briefing.

### LLM enrichment

`EidosRenderPipeline` receives an injected `TemplateRegistry` (new constructor parameter alongside `VocabularyRegistry` and `ObjectMapper`). Template resolution (ref → content → parameter substitution) happens inside `buildDescriptorPayload()`: for each `TemplateRef` on the descriptor, the pipeline resolves the template via `TemplateRegistry.resolve()`, substitutes parameters, and concatenates the results into a `templates` string field on the JSON payload.

`buildEnrichmentPayload()` must include the resolved template content so the LLM enricher sees genre/style conventions when writing disposition narratives. Without it, an agent with a "Hanna-Barbera cartoon style" template and a formal disposition gets a straight-faced enriched narrative that ignores the cartoon conventions. Addition: `copyIfPresent(payload, descriptorNode, "templates")`.

**`PROMPT_TEMPLATE` update:** The enrichment prompt must tell the LLM enricher about the new field. Two changes to `PROMPT_TEMPLATE` in `EidosRenderPipeline`:

1. Add to the "payload may contain" list:
   ```
   - templates: shared behavioral conventions — genre/style guides that frame
     the agent's personality (when present)
   ```

2. Extend the `dispositionNarrative` field instruction:
   ```
   Weave briefing principles and template conventions naturally when present
   — do not quote verbatim.
   ```
   (Previously: "Weave briefing principles naturally when present")

### Cache key

Resolved template content is included in the descriptor hash for `RenderedPromptCache`. Template content changes → cache invalidation via `descriptorHash`.

**Naming clarification:** `TEMPLATE_HASH` in the `EidosRenderPipeline` cache key formula is derived from `PROMPT_TEMPLATE` + `A2A_PROMPT_TEMPLATE` + schema descriptions. Descriptor template content enters the cache key through `descriptorHash` (because `buildDescriptorPayload()` includes the resolved `templates` field). These are two unrelated uses of the word "template."

**`TEMPLATE_HASH` does change** as a consequence of the `PROMPT_TEMPLATE` update (adding the `templates` field description and enrichment instruction). This correctly invalidates all cached renders — the enrichment contract has changed, so previously cached enrichments must be regenerated with template-aware instructions.

---

## Module Placement

| Module | Changes |
|---|---|
| `casehub-eidos-api` | `DescriptorTemplate`, `TemplateRef`, `TemplateRegistry` (SPI), `TemplateRegistrar` (CDI SPI) |
| `casehub-eidos` (runtime) | `CdiTemplateRegistry`, `ClasspathYamlTemplateRegistrar`, validation in `DescriptorCollector`, render pipeline integration |
| `casehub-eidos-memory` | `InMemoryTemplateRegistry` |
| `casehub-eidos-deployment` | `EidosProcessor` update if build-time registration needed |

### Existing files modified

- `AgentDescriptor` — new `templates` field (`List<TemplateRef>`, nullable) + builder method
- `AgentDescriptorValidator` — new validation constants (`MAX_TEMPLATE_ID`, `MAX_TEMPLATE_NAME`, `MAX_TEMPLATE_CONTENT`, `MAX_PARAMETER_NAME`); new `validateRequired(String, String, int, int...)` overload (mirrors existing `validateOptional` varargs overload, delegates to `validateField`); structural field validation for `DescriptorTemplate` and `TemplateRef` compact constructors (character-set, length). Does NOT perform template ref resolution — that requires a live `TemplateRegistry` and belongs in `DescriptorCollector`.
- `DescriptorCollector` — template ref validation against `TemplateRegistry` (ref resolution, parameter completeness). New parameter: `TemplateRegistry`. Signature changes from `collectAndValidate(Iterable<AgentDescriptorRegistrar>)` to `collectAndValidate(Iterable<AgentDescriptorRegistrar>, TemplateRegistry)`.
- `AgentDescriptorBootstrap` — inject `TemplateRegistry`, pass to `DescriptorCollector`
- `ClasspathYamlDescriptorRegistrar` — parse `templates:` refs from descriptor YAML
- `EidosRenderPipeline` — inject `TemplateRegistry`, template resolution in `buildDescriptorPayload()`, `copyIfPresent` in `buildEnrichmentPayload()`, assembly order, cache key via `descriptorHash`
- `AgentDescriptorEntity` / `AgentDescriptorMapper` — persist template refs as JSON TEXT column
- `AgentDescriptorComparator` — include templates in drift detection; increment `COMPARED_FIELD_COUNT` to match new component count
- `AgentDescriptorComparatorTest` — structural sync test (`comparatorCoversAllDescriptorComponents`) automatically catches incomplete comparator updates when `AgentDescriptor.class.getRecordComponents().length` changes

### Flyway migration

`V7__descriptor_templates.sql` — adds nullable `templates` TEXT column to `agent_descriptor` table. Nullable because existing descriptors don't have templates. Stores template refs as JSON (same pattern as `axis_vocabularies` TEXT column).

```sql
ALTER TABLE agent_descriptor ADD COLUMN templates TEXT NULL;
```

### No changes to

- `casehub-eidos-vocab`
- `casehub-eidos-eval`
- No JPA entities for templates (templates are classpath-loaded, not DB-stored; only the refs on `AgentDescriptorEntity` are persisted)

---

## Use Cases

- **Entertainment:** "Hanna-Barbera cartoon style" shared across five character agents
- **Compliance:** "Regulatory communication standards" shared across 20 compliance agents
- **Healthcare:** "Clinical communication" parameterized per specialty (ICU vs general ward)
- **Customer service:** "Brand voice" shared across all support agents
- **Legal:** "Contract review conventions" shared across reviewer agents
