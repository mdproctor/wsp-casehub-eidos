# Design: DraftHouse Reviewer Descriptors + AgentDescriptorRegistrar + YAML Loader

**Issue:** casehubio/eidos#64  
**Branch:** issue-64-drafthouse-reviewer-descriptors  
**Date:** 2026-06-23

---

## Problem

DraftHouse (casehubio/drafthouse#62) is adding multi-LLM reviewer support: multiple debate agents
with distinct perspectives (structural, content, readability, completeness). Each reviewer is an
`AgentDescriptor` resolved via `AgentRegistry` and rendered via `SystemPromptRenderer`.

Four gaps block this integration:

1. **`MAX_BRIEFING` too restrictive** — `AgentDescriptorValidator.MAX_BRIEFING = 500` is too low
   for agents that need substantive behavioural guidance. A reviewer-class agent describing
   identity, focus areas, and approach requires 800–1500 characters. The compact constructor
   throws `AgentValidationException` at construction time.

2. **No declarative persona registration** — every consumer must write bespoke startup code to call
   `AgentRegistry.register()`. This does not scale beyond a handful of agents.

3. **No YAML-driven persona configuration** — persona definitions are hard-coded in Java. Adding a
   new reviewer type requires a code change, not a config change.

4. **`EidosSystemPromptRenderer` is `@DefaultBean`** — CDI displacement fails. DraftHouse ships a
   `@DefaultBean SimplePromptRenderer` for standalone operation; when eidos is on the classpath,
   both are `@DefaultBean` → `AmbiguousResolutionException`.

---

## Solution Overview

Seven coordinated changes, all in eidos. DraftHouse integration (casehubio/drafthouse#62) is
out of scope — it depends on this work.

| # | Change | Module |
|---|--------|--------|
| 1 | `MAX_BRIEFING` increase (500 → 2000) | `casehub-eidos-api` |
| 2 | `AgentDescriptorRegistrar` SPI (declarative) | `casehub-eidos-api` |
| 3 | `AgentDescriptorBootstrap` auto-discovery | `casehub-eidos-runtime` |
| 4 | `ClasspathYamlDescriptorRegistrar` | `casehub-eidos-runtime` |
| 5 | `EidosSystemPromptRenderer` CDI fix | `casehub-eidos-runtime` |
| 6 | `ThomasKilmannTerm` alias fix | `casehub-eidos-vocab` |
| 7 | `DraftHouseReviewerScenarioTest` + YAML | `casehub-eidos-example-agent-scenarios` |

---

## 1. MAX_BRIEFING Increase

**Location:** `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java`

Change `MAX_BRIEFING` from `500` to `2000`.

This is a general improvement — any agent type that needs multi-sentence identity descriptions
benefits (not a DraftHouse-specific accommodation). The DDL column is `TEXT NULL`
(`V1__initial_schema.sql:21`), the entity uses `columnDefinition = "TEXT"`
(`AgentDescriptorEntity.java:50`) — no schema migration needed, purely a validator constant.

---

## 2. AgentDescriptorRegistrar SPI (Declarative)

**Location:** `api/src/main/java/io/casehub/eidos/api/spi/AgentDescriptorRegistrar.java`

```java
@FunctionalInterface
public interface AgentDescriptorRegistrar {
    List<AgentDescriptor> descriptors();
}
```

**Declarative, not imperative.** Mirrors the actual `VocabularyRegistrar` pattern — the registrar
returns data; the bootstrap handles registration. `VocabularyRegistrar.vocabulary()` returns a
`Class<? extends Enum<?>>`, not `void register(VocabularyRegistry)`. All 6 vocab implementations
are one-liner class returns.

Advantages over the imperative `void register(AgentRegistry)` pattern:
- **Testable** — call `descriptors()`, assert on the list. No mock registry needed.
- **Composable** — the bootstrap sees all descriptors before registration. Can dedup, validate,
  log counts.
- **Pattern-consistent** — registrar returns data, infrastructure handles mutation.

Consumers (DraftHouse, devtown, claudony) either:
- Provide a `META-INF/eidos/descriptors.yaml` on their classpath (loaded by
  `ClasspathYamlDescriptorRegistrar`, which itself is an `AgentDescriptorRegistrar`)
- Or implement `AgentDescriptorRegistrar` directly in Java for dynamic or programmatic cases

---

## 3. AgentDescriptorBootstrap — Registrar Auto-Discovery

**Location:** `runtime/src/main/java/io/casehub/eidos/runtime/registrar/AgentDescriptorBootstrap.java`

```java
@ApplicationScoped
public class AgentDescriptorBootstrap {
    @Inject AgentRegistry registry;
    @Inject @Any Instance<AgentDescriptorRegistrar> registrars;

    void onStartup(@Observes StartupEvent ev) {
        registrars.forEach(r -> r.descriptors().forEach(registry::register));
    }
}
```

**Discovery is in a separate bootstrap bean, not inside any registry implementation.**

`InMemoryAgentRegistry` is `@Alternative @Priority(1)` — it's only active when
`casehub-eidos-memory` is on the classpath. Putting discovery inside it means the JPA production
path gets nothing. `JpaAgentRegistry` has `@Transactional` on `register()`, so
`@PostConstruct` self-invocation would bypass CDI interceptors.

The bootstrap bean in `runtime/` solves both:
- Works with whichever `AgentRegistry` CDI resolves (InMemory or JPA)
- `@Observes StartupEvent` means CDI interceptors (`@Transactional`) are live
- Both registry implementations stay pure storage — no discovery logic

No JPA deferral issue needed. No `InMemoryAgentRegistry` changes needed.

---

## 4. ClasspathYamlDescriptorRegistrar

**Location:** `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`

**Implements `AgentDescriptorRegistrar`** — the declarative SPI. Returns `List<AgentDescriptor>`.

**Behaviour:**
- `@ApplicationScoped` — always active; contributes alongside any other `AgentDescriptorRegistrar`
  beans
- Reads all `META-INF/eidos/descriptors.yaml` files from the classpath via
  `ClassLoader.getResources()` (plural — returns `Enumeration<URL>`, discovers files from all
  JARs on the classpath)
- If no files found → returns `List.of()` (empty, no-op)
- If found → parses each with Jackson YAML dataformat → maps to `AgentDescriptor` via Builder →
  aggregates into the returned list
- The `META-INF/` path convention signals "this file may appear in multiple JARs" and
  `getResources()` is the expected lookup for `META-INF/` resources

**New Maven dependency in `casehub-eidos-runtime/pom.xml`:**
```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
```
This is a new dependency — NOT already on the runtime classpath (verified via
`mvn dependency:tree`). It transitively brings `org.snakeyaml:snakeyaml-engine` regardless of
whether `quarkus-config-yaml` is present in the consumer.

**Internal DTOs** (package-private inner static classes — not part of the public API):

```
DescriptorFile       { List<DescriptorConfig> descriptors }
DescriptorConfig     { agentId, name, slot, tenancyId, version, provider,
                       modelFamily, modelVersion, weightsFingerprint,
                       domainVocabulary, slotVocabulary, dispositionVocabulary,
                       Map<String,String> axisVocabularies,
                       DispositionConfig disposition,
                       List<CapabilityConfig> capabilities,
                       jurisdiction, dataHandlingPolicy, briefing }
DispositionConfig    { socialOrient, ruleFollowing, riskAppetite, autonomy,
                       conflictMode, boolean delegation }
CapabilityConfig     { name, qualityHint, latencyHintP50Ms, costHint,
                       inputTypes, outputTypes, tags,
                       Map<String,Double> epistemicDomains,
                       Set<String> excludedDomains }
```

**Testability:** Package-private `List<AgentDescriptor> loadFrom(InputStream yaml)` is the core
parsing logic. The public `descriptors()` method calls `getResources()` then delegates to
`loadFrom()` for each URL. Unit tests call `loadFrom()` directly with a `ByteArrayInputStream`.

**Error semantics:** Any malformed YAML or invalid descriptor fails the application at startup.
Parsing errors from Jackson and validation errors from the `AgentDescriptor` compact constructor
(`AgentValidationException`) propagate through `@Observes StartupEvent`. This is intentional —
YAML-driven descriptors are validated eagerly, not lazily. Consumers must fix their YAML before
the app starts.

**YAML format — top level:**
```yaml
descriptors:
  - agentId: ...
    name: ...
    slot: ...
    tenancyId: ...
    axisVocabularies:
      CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
      RULE_FOLLOWING: urn:casehub:vocab:conscientiousness
    disposition:
      conflictMode: collaborating
      ruleFollowing: strict
      delegation: false
    capabilities:
      - name: document-review
        tags: [structural]
    briefing: |
      Multi-paragraph briefing text...
```

`axisVocabularies` keys are `DispositionAxis` enum names (e.g. `CONFLICT_MODE`); deserialized via
`DispositionAxis.valueOf()`. Invalid keys throw `IllegalArgumentException` at parse time.

---

## 5. EidosSystemPromptRenderer — CDI Scope Fix

Remove `@DefaultBean` from `EidosSystemPromptRenderer`. Keep plain `@ApplicationScoped`.

**Before:** `@DefaultBean @ApplicationScoped` — any non-`@DefaultBean` displaces it. Two
`@DefaultBean` implementations (eidos + DraftHouse) → `AmbiguousResolutionException`.

**After:** `@ApplicationScoped` (no `@DefaultBean`) — displaces DraftHouse's
`@DefaultBean SimplePromptRenderer` when eidos is on the classpath. When eidos is absent,
DraftHouse's `@DefaultBean` is active.

This follows Pattern B from `alternative-extension-patterns`: base is `@ApplicationScoped`
(active by default), override is `@Alternative @Priority(1)` or `@DefaultBean` (fallback).
Consumers wanting to displace `EidosSystemPromptRenderer` after this change must use
`@Alternative @Priority(N)`.

**Current impact: zero.** Verified: `SystemPromptRenderer` has exactly one implementation across
all casehub repos. No existing consumer displaces it. DraftHouse does not yet depend on
`casehub-eidos-api`.

No `@IfBuildProperty` gating — there is no reactive variant of the renderer.

Note: `CdiVocabularyRegistry` has the same `@DefaultBean` pattern and the same future risk. Not
in scope for this issue; tracked separately.

---

## 6. ThomasKilmannTerm Alias Fix

**Location:** `vocab/src/main/java/io/casehub/eidos/vocab/ThomasKilmannTerm.java`

Add `"collaborative"` as an alias for `COLLABORATING`:

```java
COLLABORATING ("collaborating", "Collaborating",
    "High assertiveness, high cooperativeness; seeks joint problem-solving",
    List.of("cooperative", "collaborative")),
```

**Why:** `"collaborative"` is the natural adjective form people reach for. The issue spec says
`conflictMode=collaborative`; `ConscientiousnessTerm` has a `COLLABORATIVE` term with value
`"collaborative"` — on the wrong axis. Without the alias, a YAML file with
`conflictMode: collaborative` under Thomas-Kilmann would silently fail vocabulary resolution
(no match, no error — just an unresolved disposition value). This is worse than a hard error.

This is a general vocabulary improvement, not DraftHouse-specific.

---

## 7. DraftHouseReviewerScenarioTest + YAML

**Location:** `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DraftHouseReviewerScenarioTest.java`  
**YAML:** `examples/agent-scenarios/src/test/resources/META-INF/eidos/descriptors.yaml`

### YAML Personas

All four: `slot="document-reviewer"`, `tenancyId="drafthouse"`, capability `name="document-review"`.

Vocabulary wiring per-axis via `axisVocabularies`:
- `CONFLICT_MODE` → `urn:casehub:vocab:thomas-kilmann`
- `RULE_FOLLOWING` / `RISK_APPETITE` / `AUTONOMY` → `urn:casehub:vocab:conscientiousness`

| agentId | Name | conflictMode | 2nd axis | tag |
|---------|------|-------------|----------|-----|
| `drafthouse-structural-reviewer` | Structural Reviewer | `collaborating` | `ruleFollowing=strict` | `structural` |
| `drafthouse-content-reviewer` | Content Reviewer | `competing` | `riskAppetite=conservative` | `content` |
| `drafthouse-readability-reviewer` | Readability Reviewer | `accommodating` | `autonomy=directed` | `readability` |
| `drafthouse-completeness-reviewer` | Completeness Reviewer | `collaborating` | `ruleFollowing=strict` | `completeness` |

Note: the issue spec says `riskAppetite=cautious`; "cautious" is an alias for the canonical
Conscientiousness term value `conservative`. The YAML uses the canonical value. Similarly, the
issue says `conflictMode=collaborative`; with the alias fix in Section 6, this resolves to
`COLLABORATING`. The YAML uses the canonical value `collaborating`.

Each descriptor has a 2–3 paragraph briefing defining identity, focus areas, and approach.
Structural and Completeness reviewers have identical disposition axes; their briefings
differentiate them in purpose.

### Test Coverage

`DraftHouseReviewerScenarioTest` is a `@QuarkusTest` with **no `@BeforeEach`** — the YAML
drives registration at startup via `ClasspathYamlDescriptorRegistrar` → `AgentDescriptorBootstrap`.

**Registration tests:**
- All 4 found by slot `"document-reviewer"` + tenancyId `"drafthouse"`
- All 4 found by capability `"document-review"`
- Each found by agentId with complete descriptor (name, slot, disposition, briefing, capability)
- Tenancy isolation: `AgentQuery.all("default")` returns none of the drafthouse agents

**Renderer tests (MARKDOWN structural path — no LLM in examples module):**
- Each agent's rendered output contains: agent name, capability name, at least one
  vocabulary-resolved disposition label (e.g. `"Collaborating (Thomas-Kilmann Conflict Modes)"`),
  and a phrase from the briefing
- Output contains `"## Role"` and `"## How You Operate"` sections (structural MARKDOWN headings)
- Output contains `"## Operating Principles"` section (briefing present)

**ClasspathYamlDescriptorRegistrar unit tests** in `runtime/src/test/java/`:
- Empty YAML (no descriptors) → empty list
- Missing resource (no `META-INF/eidos/descriptors.yaml` on classpath) → empty list, no exception
- Valid single descriptor → list of one, all fields mapped correctly
- Descriptor with all optional fields omitted → registered with nulls for optional fields
- `axisVocabularies` keys deserialised to `DispositionAxis` enum values correctly
- Invalid `axisVocabularies` key → `IllegalArgumentException` (fast-fail)
- Vocabulary resolution: descriptor with `conflictMode: collaborating` and
  `axisVocabularies.CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann` → after registration,
  `VocabularyRegistry.resolve(ThomasKilmannTerm.URI, "collaborating")` returns
  `ThomasKilmannTerm.COLLABORATING`

---

## Out of Scope — Issues to File Before Leaving This Session

| # | What | Why deferred |
|---|------|-------------|
| TBD | `CdiVocabularyRegistry` `@DefaultBean` fix | Same pattern as renderer fix; separate concern |
| TBD | ARC42STORIES line 709 stale `VocabularyRegistrar` description | Says `void register(VocabularyRegistry registry)` — actual source is `Class<...> vocabulary()` |

---

## Protocol Compliance

| Protocol | Compliance |
|----------|-----------|
| `llm-pass-structural-fallback` | `EidosSystemPromptRenderer` unchanged — structural fallback already correct |
| `optional-platform-spi-instance-injection` | `Instance<AgentDescriptorRegistrar>` in `AgentDescriptorBootstrap` — `registrars.forEach()` is a no-op if empty; eidos-owned SPI |
| `eidos-spi-agent-scope-requires-tenancy-id` | All 4 personas set `tenancyId="drafthouse"` |
| `jpa-mapper-positional-constructor-field-completeness` | YAML loader uses `AgentDescriptor.Builder` (config use case — not the JPA mapper path) |
| `eidos-poc-gate-before-jpa-infrastructure` | No JPA entities or Flyway migrations in this issue |
| `render-format-structure-naming` | No new `RenderFormat` values |
| `alternative-extension-patterns` | Section 5 changes `EidosSystemPromptRenderer` from `@DefaultBean` to plain `@ApplicationScoped` — Pattern B: base is `@ApplicationScoped`, fallback is `@DefaultBean` |
| `agent-descriptor-compact-constructor-validation` (PP-20260530-2d6dbd) | YAML loader introduces a new ingestion path; strings validated via `AgentDescriptor` compact constructor. No bypass — all validation rules enforced at construction time |
