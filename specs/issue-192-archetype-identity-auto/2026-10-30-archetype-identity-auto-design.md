# Archetype Identity Auto-Derivation — Design Spec

**Issues:** casehubio/eidos#192, casehubio/eidos#193
**Branch:** issue-192-archetype-identity-auto
**Date:** 2026-10-30

## Overview

Two linked changes that enable a minimal identity path for agent descriptors:

1. **#192 (remaining):** Wire `ArchetypeCompatibility` to load from `archetypes.yaml` (already landed on main), make the YAML the runtime data source, enforce bidirectional YAML↔Java validation
2. **#193:** Add `domain` field, auto-derive disposition from archetype, auto-compose briefing at render time — so a 6–7 field YAML descriptor produces a fully rendered system prompt

## Prerequisite (landed)

`archetypes.yaml` (1087 lines) in `vocab/src/main/resources/META-INF/eidos/`, `descriptor.schema.json` in `api/src/main/resources/META-INF/eidos/`, and `ArchetypeDataValidationTest` (13 tests) landed on main as `211eb3e`.

---

## Part 1: ArchetypeCompatibility — YAML-backed loading (#192)

### Current state

`ArchetypeCompatibility` (vocab/) has 6 static init methods (`initMbti()`, `initEnneagram()`, etc.) that hardcode ~60 `FAMILY_MAP` entries and an `aff()` helper that populates ~96 `AFFINITY_MAP` entries.

### Change

Replace the 6 init methods and `aff()` calls with a single static initializer that reads `META-INF/eidos/archetypes.yaml` from the classpath via Jackson + YAMLFactory.

```java
private static final Map<String, Map<String, Set<ArchetypeFamily>>> FAMILY_MAP;
private static final Map<ArchetypeTerm, Map<String, Set<String>>> AFFINITY_MAP;

static {
    // Read archetypes.yaml from classpath
    // Parse "compatibility" section → FAMILY_MAP
    // Parse "families.*.subArchetypes.*.affinities" → AFFINITY_MAP
}
```

### Data mapping

- `archetypes.yaml` `compatibility:` section → `FAMILY_MAP` (keyed by framework URI)
- `archetypes.yaml` `families.*.subArchetypes.*.affinities` → `AFFINITY_MAP` (per sub-archetype)

### API

Public API unchanged: `compatibleFamilies(frameworkUri, termValue)` and `subArchetypeAffinity(archetype, uri, value)` remain static with the same signatures. `ArchetypeResolver` and all other callers unaffected.

### Error handling

`ExceptionInInitializerError` if YAML is missing or malformed — fail-fast at class load. Appropriate since the data is bundled in the jar.

### Source of truth split

- Java enums (`ArchetypeTerm`, `ArchetypeFamily`) own structural truth — what archetypes exist
- YAML owns data truth — mappings, weights, compatibility, descriptions, adjectives
- `descriptor.schema.json` enum values validated against YAML

---

## Part 2: Bidirectional validation (#192)

### Current state

`ArchetypeDataValidationTest` validates YAML → Java direction (13 tests).

### Change

Add reverse-direction assertions:

- **Families:** YAML families ⊆ Java enums (existing) + Java enums ⊆ YAML families (new)
- **Sub-archetypes:** YAML sub-archetypes ⊆ Java enums (existing) + Java enums ⊆ YAML sub-archetypes (new)
- **Adjectives, descriptions:** same bidirectional pattern
- **Compatibility:** YAML framework URIs match registered vocabulary URIs

New test validates `descriptor.schema.json` enum arrays match YAML values (schema-YAML sync per D6).

### Effect

Any addition or removal in either Java enums or YAML without updating the other fails the build.

---

## Part 3: `domain` field on AgentDescriptor (#193)

### Change

Add `domain` (String, optional) to `AgentDescriptor` record, positioned between `slot` and `archetype`.

### Semantics

Human-readable domain label (e.g. "software-engineering", "healthcare", "legal"). Distinct from `domainVocabulary` (vocabulary URI for grounding terms). Open string — no validation against any vocabulary.

### Touchpoints

| File | Change |
|------|--------|
| `AgentDescriptor` | Record component + builder method |
| `AgentDescriptorDeserializer` | YAML parsing |
| `AgentDescriptorMapper` | JPA entity mapping |
| `AgentDescriptorEntity` | New field |
| `V1__initial_schema.sql` | New column (nullable, no existing installations) |
| `descriptor.schema.json` | Add field definition |
| `@Identity` annotation | Add `domain()` attribute |
| `AnnotatedAgentConfig` | Add field |
| `EidosAnnotationsRecorder` | Pass through |
| `EidosRenderPipeline` | Render in identity section |
| A2A_CARD format | Include in card |

---

## Part 4: Auto-derivation — archetype → disposition (#193)

### Where

New step in `DescriptorCollector.collectAndValidate()`, before the existing `deriveDispositionAxes()` call.

### Trigger

`archetype` is set AND `disposition` is null (or has no explicit axis values) AND no `mbtiType`/`enneagramType` already provided.

### Chain

1. Resolve archetype string to `ArchetypeTerm` enum constant (case-insensitive match on `value()`)
2. Look up MBTI affinities from `ArchetypeCompatibility` AFFINITY_MAP
3. Take the first listed MBTI type (ordered by relevance in YAML)
4. Pass to `PersonalityTypeDeriver.derive()` → produces `dispositionProfile` (Jungian function weights)
5. Existing `deriveDispositionAxes()` kicks in — projects profile weights onto 5 axes

### Conflict resolution

- Explicit `disposition` always wins
- Explicit `mbtiType`/`enneagramType` always wins
- Archetype derivation is the fallback when nothing else is specified

### Edge cases

- Archetype doesn't resolve to an `ArchetypeTerm`: skip derivation silently (may be from a domain vocabulary)
- Archetype has no MBTI affinities in YAML: skip derivation silently
- Archetype resolves but MBTI affinity list is empty: skip derivation silently

### Result

A YAML descriptor with just `archetype: sage` gets a fully derived 5-axis disposition.

---

## Part 5: Auto-briefing — structural composition (#193)

### Where

`EidosRenderPipeline`, at the briefing hook (lines 754–758).

### Trigger

`descriptor.briefing()` is null AND `archetype` is present AND `slot` is present.

### Composition

Structural assembly from available fields — no LLM needed:

```
## Operating Principles
You are a {slot} in the {domain} domain, embodying the {archetype} archetype.
{adjectives sentence if archetypeAdjectives present}
{archetype description from ArchetypeTerm if resolvable}
```

Example for `{slot: "reviewer", domain: "software-engineering", archetype: "sage", archetypeAdjectives: ["cautious", "methodical"]}`:

```
## Operating Principles
You are a reviewer in the software-engineering domain, embodying the sage archetype.
Your approach is cautious and methodical.
Seeks truth through analysis, research, and reflection.
```

### Format discrimination

- MARKDOWN/PROSE: structural composition as above
- A2A_CARD: no auto-briefing — cards carry structured data, not prose

### Conflict resolution

Explicit `briefing` always wins. Auto-briefing only fires when briefing is null.

### Optional fields

- `domain` absent → omit domain clause
- `archetypeAdjectives` empty → omit adjectives sentence
- Archetype doesn't resolve to `ArchetypeTerm` → omit description line
- Minimum viable: just archetype + slot

---

## Acceptance criteria

### #192

- [ ] `ArchetypeCompatibility` loads FAMILY_MAP and AFFINITY_MAP from `archetypes.yaml` at class load
- [ ] Hardcoded init methods removed
- [ ] Bidirectional validation: YAML↔Java enum consistency enforced in both directions
- [ ] `descriptor.schema.json` enum values validated against YAML values
- [ ] All existing tests pass — no API change

### #193

- [ ] `domain` field on `AgentDescriptor` (optional String) — YAML, JPA, annotations, schema, renderer
- [ ] Auto-disposition: archetype → first MBTI affinity → PersonalityTypeDeriver → 5-axis
- [ ] Auto-briefing: structural composition from archetype + adjectives + domain + slot at render time
- [ ] Explicit disposition/briefing always wins over derived
- [ ] YAML descriptor with 6–7 fields produces a fully rendered system prompt
- [ ] Unit tests for derivation and composition paths

## References

- `vocab/src/main/java/io/casehub/eidos/vocab/ArchetypeCompatibility.java` — current hardcoded data
- `vocab/src/main/resources/META-INF/eidos/archetypes.yaml` — consolidated YAML data file
- `api/src/main/resources/META-INF/eidos/descriptor.schema.json` — JSON Schema for descriptor validation
- `vocab/src/test/java/io/casehub/eidos/vocab/ArchetypeDataValidationTest.java` — existing validation tests
- `api/src/main/java/io/casehub/eidos/api/PersonalityTypeDeriver.java` — MBTI → disposition derivation
- `eidos-core/src/main/java/io/casehub/eidos/core/registrar/DescriptorCollector.java` — registration-time derivation
- `eidos-core/src/main/java/io/casehub/eidos/core/renderer/EidosRenderPipeline.java` — render pipeline
- `renderer/capability-metadata-rendering.md` — precedent for format-discriminated rendering
- `eval/eval-profile-briefing-from-vocabulary-gaps.md` — prior art for briefing composition
- casehubio/eidos#192 — consolidate archetype data into shared YAML
- casehubio/eidos#193 — minimal demo identity path
