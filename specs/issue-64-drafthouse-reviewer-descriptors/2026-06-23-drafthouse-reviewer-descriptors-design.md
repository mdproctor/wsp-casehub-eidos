# Design: DraftHouse Reviewer Descriptors + AgentDescriptorRegistrar + YAML Loader

**Issue:** casehubio/eidos#64  
**Branch:** issue-64-drafthouse-reviewer-descriptors  
**Date:** 2026-06-23

---

## Problem

DraftHouse (casehubio/drafthouse#62) is adding multi-LLM reviewer support: multiple debate agents
with distinct perspectives (structural, content, readability, completeness). Each reviewer is an
`AgentDescriptor` resolved via `AgentRegistry` and rendered via `SystemPromptRenderer`.

Three gaps block this integration:

1. **No declarative persona registration** — every consumer must write bespoke startup code to call
   `AgentRegistry.register()`. This does not scale beyond a handful of agents.

2. **No YAML-driven persona configuration** — persona definitions are hard-coded in Java. Adding a
   new reviewer type requires a code change, not a config change.

3. **`EidosSystemPromptRenderer` is `@DefaultBean`** — CDI displacement fails. DraftHouse ships a
   `@DefaultBean SimplePromptRenderer` for standalone operation; when eidos is on the classpath,
   both are `@DefaultBean` → `AmbiguousResolutionException`.

---

## Solution Overview

Five coordinated changes, all in eidos. DraftHouse integration (casehubio/drafthouse#62) is
out of scope — it depends on this work.

| # | Change | Module |
|---|--------|--------|
| 1 | `AgentDescriptorRegistrar` SPI | `casehub-eidos-api` |
| 2 | `InMemoryAgentRegistry` auto-discovery | `casehub-eidos-memory` |
| 3 | `ClasspathYamlDescriptorRegistrar` | `casehub-eidos-runtime` |
| 4 | `EidosSystemPromptRenderer` CDI fix | `casehub-eidos-runtime` |
| 5 | `DraftHouseReviewerScenarioTest` + YAML | `casehub-eidos-example-agent-scenarios` |

---

## 1. AgentDescriptorRegistrar SPI

**Location:** `api/src/main/java/io/casehub/eidos/api/spi/AgentDescriptorRegistrar.java`

```java
@FunctionalInterface
public interface AgentDescriptorRegistrar {
    void register(AgentRegistry registry);
}
```

Mirrors `VocabularyRegistrar` exactly. Any `@ApplicationScoped` CDI bean implementing this
interface is auto-discovered by the registry at startup.

Consumers (DraftHouse, devtown, claudony) either:
- Provide an `eidos-descriptors.yaml` on their classpath (loaded by `ClasspathYamlDescriptorRegistrar`)
- Or implement `AgentDescriptorRegistrar` directly in Java for dynamic or programmatic cases

---

## 2. InMemoryAgentRegistry — Registrar Auto-Discovery

**Location:** `persistence-memory/src/main/java/io/casehub/eidos/memory/InMemoryAgentRegistry.java`

Add:
```java
@Inject @Any Instance<AgentDescriptorRegistrar> registrars;

@PostConstruct
void init() {
    registrars.forEach(r -> r.register(this));
}
```

`register()` on `InMemoryAgentRegistry` is a pure `ConcurrentHashMap.put()` — no transactional
concerns. `@PostConstruct` + `Instance<T>` is safe and follows the `CdiVocabularyRegistry`
pattern exactly.

**JpaAgentRegistry wiring is deferred.** Calling `@Transactional register()` from `@PostConstruct`
in the same bean bypasses CDI interceptors (self-invocation). Correct wiring requires a separate
`@ApplicationScoped @IfBuildProperty @Observes StartupEvent` bootstrap bean. Tracked as a
follow-up issue to be filed before leaving this session.

---

## 3. ClasspathYamlDescriptorRegistrar

**Location:** `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`

**Behaviour:**
- `@ApplicationScoped` — always active; contributes alongside any other `AgentDescriptorRegistrar` beans
- Reads `eidos-descriptors.yaml` from the classpath root via `ClassLoader.getResourceAsStream()`
- If the file is absent → no-op (returns immediately)
- If present → parses with Jackson YAML dataformat → maps to `AgentDescriptor` via Builder → calls `registry.register()` for each

**New Maven dependency in `casehub-eidos-runtime/pom.xml`:**
```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
</dependency>
```
SnakeYAML is already on the classpath via Quarkus config-yaml; this adds only the Jackson bridge.

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

**Testability:** Package-private `void loadFrom(InputStream yaml, AgentRegistry registry)` is
the core parsing logic. The public `register()` method calls `getResourceAsStream()` then
delegates to `loadFrom()`. Unit tests call `loadFrom()` directly with a `ByteArrayInputStream`.

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
`DispositionAxis.valueOf()`.

---

## 4. EidosSystemPromptRenderer — CDI Scope Fix

Remove `@DefaultBean` from `EidosSystemPromptRenderer`. Keep plain `@ApplicationScoped`.

**Before:** `@DefaultBean @ApplicationScoped` — loses to any non-default bean; fails when
DraftHouse also ships a `@DefaultBean SimplePromptRenderer` (both are defaults → ambiguous).

**After:** `@ApplicationScoped` — displaces DraftHouse's `@DefaultBean SimplePromptRenderer`
when eidos is on the classpath. When eidos is absent, DraftHouse's `@DefaultBean` is active.

No `@IfBuildProperty` gating — there is no reactive variant of the renderer.

Note: `CdiVocabularyRegistry` has the same `@DefaultBean` pattern and the same future risk. Not
in scope for this issue; tracked separately.

---

## 5. DraftHouseReviewerScenarioTest + YAML

**Location:** `examples/agent-scenarios/src/test/java/io/casehub/eidos/examples/DraftHouseReviewerScenarioTest.java`  
**YAML:** `examples/agent-scenarios/src/test/resources/eidos-descriptors.yaml`

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
Conscientiousness term value `conservative`. The YAML uses the canonical value.

Each descriptor has a 2–3 paragraph briefing defining identity, focus areas, and approach.
Structural and Completeness reviewers have identical disposition axes; their briefings
differentiate them in purpose.

### Test Coverage

`DraftHouseReviewerScenarioTest` is a `@QuarkusTest` with **no `@BeforeEach`** — the YAML
drives registration at startup via `ClasspathYamlDescriptorRegistrar`.

**Registration tests:**
- All 4 found by slot `"document-reviewer"` + tenancyId `"drafthouse"`
- All 4 found by capability `"document-review"`
- Each found by agentId with complete descriptor (name, slot, disposition, briefing, capability)
- Tenancy isolation: `AgentQuery.all("default")` returns none of the drafthouse agents

**Renderer tests (MARKDOWN structural path — no LLM in examples module):**
- Each agent's rendered output contains: agent name, capability name, at least one vocabulary-resolved
  disposition label (e.g. `"Collaborating (Thomas-Kilmann Conflict Modes)"`), and a phrase from
  the briefing
- Output contains `"## Role"` and `"## How You Operate"` sections (structural MARKDOWN headings)
- Output contains `"## Operating Principles"` section (briefing present)

**ClasspathYamlDescriptorRegistrar unit tests** in `runtime/src/test/java/`:
- Empty YAML (no descriptors) → zero registrations
- Missing file (null InputStream) → no-op, no exception
- Valid single descriptor → registered correctly
- Descriptor with all optional fields omitted → registered with nulls for optional fields
- `axisVocabularies` keys deserialised to `DispositionAxis` enum values correctly
- Invalid `axisVocabularies` key → `IllegalArgumentException` (fast-fail)

---

## Out of Scope — Issues to File Before Leaving This Session

| # | What | Why deferred |
|---|------|-------------|
| TBD | `JpaAgentRegistry` registrar bootstrap | `@PostConstruct` + `@Transactional` self-invocation problem; needs `@Observes StartupEvent` bootstrap bean |
| TBD | `CdiVocabularyRegistry` `@DefaultBean` fix | Same pattern as renderer fix; separate concern |

---

## Protocol Compliance

| Protocol | Compliance |
|----------|-----------|
| `llm-pass-structural-fallback` | `EidosSystemPromptRenderer` remains unchanged — structural fallback already correct |
| `optional-platform-spi-instance-injection` | `Instance<AgentDescriptorRegistrar>` in `InMemoryAgentRegistry` — `registrars.forEach()` is a no-op if empty; no `isUnsatisfied()` guard needed (eidos-owned SPI, no competing `@DefaultBean`) |
| `eidos-spi-agent-scope-requires-tenancy-id` | All 4 personas set `tenancyId="drafthouse"` |
| `jpa-mapper-positional-constructor-field-completeness` | YAML loader uses `AgentDescriptor.Builder` (test/config use case — not the JPA mapper path) |
| `eidos-poc-gate-before-jpa-infrastructure` | No JPA entities or Flyway migrations in this issue |
| `render-format-structure-naming` | No new `RenderFormat` values |
