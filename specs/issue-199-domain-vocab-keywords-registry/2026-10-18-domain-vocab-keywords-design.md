# Domain Vocabulary Keywords in VocabularyRegistry

**Issue:** casehubio/eidos#199
**Date:** 2026-10-18
**Status:** Approved

## Problem & Scope

`epistemicDomains` keys on `AgentCapability` are freeform strings ("java", "analysis", "security") with no vocabulary grounding. Three consumers need semantic understanding of these domains:

1. **Assertion code** — needs associated keywords for text matching (currently hardcoded regex patterns per domain in casehub-ops FleetLlmExerciser)
2. **LLM consumers** — eval judges, routing, prompt rendering need domain scope, hierarchy, and concepts
3. **Agent routing** — OWLS-MX matching should work across epistemic domains (a `software-engineering` agent should Plugin-match a `java` task)

### In scope

- Add `keywords()` to VocabularyTerm
- Extend VocabularyRegistry for data-driven (non-enum) vocabularies
- Create an epistemic domain vocabulary in YAML with JSON Schema
- Provide a `DomainKeywords` utility for scope-inclusive keyword lookup
- Refactor CdiVocabularyRegistry to URI-keyed primary storage
- Introduce `VocabularyInfo` record to replace annotation return type on `vocabularyMetadata()`

### Out of scope

- Renderer changes (A2A_CARD, MARKDOWN/PROSE)
- Validation of `epistemicDomains` keys against vocabulary
- Cross-vocabulary specialization for data-driven terms

## API Changes (eidos-api)

### VocabularyTerm — new default method

```java
default List<String> keywords() { return List.of(); }
```

### VocabularyTerm — contract update

Javadoc changes from "Implemented by enum constants" to permit non-enum implementations. All existing default methods (`exactMatch`, `axisExactMatch`, `opposite`, `defaultProfile`, `responseStyleGuidance`, `antiPatternWarning`, etc.) return safe empty defaults — data-driven terms inherit these unchanged.

### DataDrivenTerm — new record

```java
public record DataDrivenTerm(
    String value, String label, String description,
    List<String> aliases, List<String> keywords,
    List<VocabularyTerm> resolvedParents
) implements VocabularyTerm {
    @Override public List<VocabularyTerm> specializes() { return resolvedParents; }
}
```

Compact constructor validates: value required, non-blank. All lists defensively copied to immutable.

### DataDrivenVocabulary — new record

```java
public record DataDrivenVocabulary(
    String uri, String name, String version, String description,
    List<DataDrivenTerm> terms
) {}
```

### DataDrivenVocabularyRegistrar — new SPI

```java
@FunctionalInterface
public interface DataDrivenVocabularyRegistrar {
    List<DataDrivenVocabulary> vocabularies();
}
```

CDI-discovered alongside existing `VocabularyRegistrar` for enum vocabs. Follows the established SPI pattern: `AgentDescriptorRegistrar` returns `List<AgentDescriptor>`, `TemplateRegistrar` returns `List<DescriptorTemplate>`.

### VocabularyInfo — new record

```java
public record VocabularyInfo(String uri, String name, String version, String description) {}
```

`VocabularyRegistry.vocabularyMetadata(String uri)` return type changes from `Optional<VocabularyMetadata>` (annotation) to `Optional<VocabularyInfo>` (record). For enum vocabs, populated from the `@VocabularyMetadata` annotation at registration time. For data-driven vocabs, populated from YAML metadata. The `@VocabularyMetadata` annotation is unchanged — it remains the declaration mechanism for enum vocab classes.

Breaking change sites: ~14 total (8 callers, ~6 stub/test implementations). All updates are mechanical — method signatures on the record match the annotation accessor names.

### VocabularyRegistry — new registration method

```java
void register(String uri, String name, String version, String description,
              List<? extends VocabularyTerm> terms);
```

No new `keywords()` method on the registry. Consumers compose: `resolve(vocabUri, value).map(VocabularyTerm::keywords).orElse(List.of())`.

### DomainKeywords — new utility

```java
public final class DomainKeywords {
    public static List<String> expanded(VocabularyRegistry registry,
                                         String vocabUri, String value) { ... }
}
```

Returns the term's own keywords plus all ancestor keywords. The assertion consumer's primary entry point. Composes `resolve()` + `VocabularyTerm.keywords()` + `ancestors()`.

## CdiVocabularyRegistry Refactor

### Internal storage migration

**Current** (class-keyed primary):

| Map | Type | Scope |
|-----|------|-------|
| `byUri` | `Map<String, Class<? extends Enum<?>>>` | uri → enum class |
| `byClass` | `Map<Class<?>, Map<String, VocabularyTerm>>` | class → (value/alias → term) |
| `byClassOrdered` | `Map<Class<?>, List<? extends VocabularyTerm>>` | class → terms in order |

**New** (URI-keyed primary, class-keyed secondary):

| Map | Type | Scope |
|-----|------|-------|
| `termsByUri` | `Map<String, Map<String, VocabularyTerm>>` | uri → (value/alias → term) — ALL vocabs |
| `orderedByUri` | `Map<String, List<? extends VocabularyTerm>>` | uri → terms in order — ALL vocabs |
| `metadataByUri` | `Map<String, VocabularyInfo>` | uri → metadata — ALL vocabs |
| `byUri` | `Map<String, Class<? extends Enum<?>>>` | uri → enum class — enum only, retained |
| `byClass` | `Map<Class<?>, Map<String, VocabularyTerm>>` | class → lookup — enum only, secondary |
| `byClassOrdered` | `Map<Class<?>, List<? extends VocabularyTerm>>` | class → ordered — enum only, secondary |

### Read-path migration

All string-based methods (`resolve(uri, value)`, `allTerms(uri)`, `vocabularyMetadata(uri)`, `equivalentValues(fromUri, ...)`, `subsumes(...)`, `match(...)`, `ancestors(...)`, `descendants(...)`) switch to URI-keyed maps. Typed methods (`resolve(Class, value)`, `equivalentValues(S, Class)`) continue using class-keyed maps.

### Hierarchy builder changes

`buildAllHierarchyIndexes()` parameter type changes from `Map<String, Class<? extends Enum<?>>>` to a representation covering both vocab types. The `((Enum<?>) parent).getDeclaringClass()` cast replaced by a `termToVocabUri` map populated during term collection from both enum constants and data-driven term lists. The `vocab.getEnumConstants()` iteration unified to iterate over `orderedByUri` which contains both types.

### Init sequence

```java
@PostConstruct
void init() {
    // Phase 1a: enum registrars (existing)
    for (VocabularyRegistrar r : registrars) {
        registerTerms(r.vocabulary());
    }
    // Phase 1b: data-driven registrars (new)
    for (DataDrivenVocabularyRegistrar r : dataDrivenRegistrars) {
        for (DataDrivenVocabulary v : r.vocabularies()) {
            registerDataDriven(v);
        }
    }
    // Phase 2: build hierarchy (unchanged logic, adapted iteration)
    buildAllHierarchyIndexes();
}
```

Enum registrars populate `byUri`, `byClass`, `byClassOrdered`, AND the new `termsByUri`, `orderedByUri`, `metadataByUri`. Data-driven registrars populate only the URI-keyed maps. Hierarchy builder uses URI-keyed maps exclusively.

## YAML Vocabulary Format & Schema

### ClasspathYamlVocabularyRegistrar

In eidos-core. Reads `META-INF/eidos/vocabularies/*.yaml`. Implements `DataDrivenVocabularyRegistrar`. Scans classpath at construction time (same pattern as `ClasspathYamlDescriptorRegistrar`).

Specializes references resolved within the vocabulary at load time: string value → `DataDrivenTerm` instance. Unresolvable references throw at startup. Cross-vocabulary specialization is not supported for data-driven terms.

### YAML format

```yaml
# yaml-language-server: $schema=https://casehub.io/eidos/vocabulary.schema.json
uri: "urn:casehub:vocab:epistemic-domain"
name: Epistemic Domain Taxonomy
version: "1.0"
description: Knowledge areas for epistemic domain qualification

terms:
  - value: software-engineering
    label: Software Engineering
    description: Design, development, testing, and maintenance of software systems
    keywords:
      - code
      - software
      - debug
      - implement
      - refactor
      - compile
      - test
      - deploy
      - architecture
      - design pattern

  - value: java
    label: Java
    description: JVM-based programming language and ecosystem
    specializes: [software-engineering]
    keywords:
      - jvm
      - bytecode
      - classpath
      - maven
      - gradle
      - spring
      - quarkus
      - servlet
      - annotation processing
      - garbage collection
```

### JSON Schema (vocabulary.schema.json)

Shipped in `META-INF/eidos/` alongside `descriptor.schema.json`. Validates:

- **Root:** `uri` (required string), `name` (string), `version` (string), `description` (string), `terms` (required array)
- **Term:** `value` (required string), `label` (string), `description` (string), `aliases` (string array), `keywords` (string array), `specializes` (string array — references to other term values in same vocabulary)

### Epistemic domain vocabulary (~20 terms)

Shipped in eidos-vocab as `META-INF/eidos/vocabularies/epistemic-domains.yaml`:

**Top-level (no parent):**
- `software-engineering` — code, software, debug, implement, refactor, compile, test, deploy, architecture, design pattern
- `analysis` — investigation, examination, assessment, pattern recognition, diagnostic, root cause, evidence, hypothesis
- `security` — vulnerability, threat, exploit, defense, compliance, audit, penetration, encryption
- `data-engineering` — pipeline, ETL, schema, warehouse, ingestion, transformation, streaming, batch
- `clinical-research` — clinical, trial, patient, diagnosis, treatment, evidence-based, protocol, outcome
- `product-management` — product, roadmap, stakeholder, user research, prioritization, metrics, backlog
- `finance` — financial, trading, portfolio, risk, valuation, compliance, accounting, audit
- `legal` — contract, compliance, regulation, liability, intellectual property, jurisdiction, statute

**Under software-engineering:**
- `java` — jvm, bytecode, classpath, maven, gradle, spring, quarkus, servlet, annotation processing, garbage collection
- `python` — pip, virtualenv, pandas, numpy, django, flask, cpython, asyncio, type hints
- `rust` — cargo, ownership, borrowing, lifetime, unsafe, crate, trait, pattern matching
- `go` — goroutine, channel, module, defer, interface, concurrency, garbage collection
- `typescript` — types, generics, declaration, webpack, eslint, node, interface, union type
- `distributed-systems` — consensus, partition, replication, consistency, availability, latency, fault tolerance
- `machine-learning` — model, training, inference, dataset, feature, neural network, gradient, embedding
- `devops` — ci/cd, container, kubernetes, deployment, monitoring, infrastructure, pipeline, observability

**Under analysis:**
- `static-analysis` — ast, linter, rule, code smell, metric, complexity, dead code
- `data-analysis` — statistics, visualization, correlation, regression, hypothesis, trend, outlier

**Under security:**
- `application-security` — owasp, injection, xss, authentication, authorization, csrf, session
- `network-security` — firewall, intrusion, encryption, certificate, protocol, tls, vpn

Each term carries 8-15 keywords — full natural-language concepts, not regex fragments. Consumers build their own matching from these.

## What Stays Unchanged

- **Existing enum vocabularies** — Belbin, DISC, Conscientiousness, Jungian, MBTI, Sarc7, SVO, CasehubCapability, CasehubSlot, ModelTier, BigFive, Enneagram, SDI, Archetype: untouched. Their registrars continue working via `Instance<VocabularyRegistrar>`.
- **Enum-based `register(Class<T>)`** — backward compatible, still works.
- **Typed `resolve(Class, value)`** — unchanged, uses class-keyed secondary index.
- **`epistemicDomains` keys** — remain freeform strings. No validation against vocabulary. WARN log when a descriptor's domain key has no matching vocabulary registration.
- **Renderer** — no changes. A2A_CARD and MARKDOWN/PROSE output unchanged.
- **`@VocabularyMetadata` annotation** — unchanged as the declaration mechanism for enum vocab classes. Only the registry query method's return type changes (annotation → `VocabularyInfo` record).
- **Hierarchy builder semantics** — same global DAG, same cross-vocabulary injection, same cycle detection. Only the iteration and term-to-URI lookup mechanism changes.

## Module Placement

| Component | Module | Rationale |
|-----------|--------|-----------|
| `VocabularyTerm.keywords()` | eidos-api | Interface method |
| `DataDrivenTerm` | eidos-api | Tier 1 data record |
| `DataDrivenVocabulary` | eidos-api | Tier 1 data record |
| `DataDrivenVocabularyRegistrar` | eidos-api | SPI interface |
| `VocabularyInfo` | eidos-api | Tier 1 data record |
| `DomainKeywords` | eidos-api | Static utility, Tier 1 |
| `vocabulary.schema.json` | eidos-api | Alongside descriptor.schema.json |
| `ClasspathYamlVocabularyRegistrar` | eidos-core | Parallels other YAML registrars |
| CdiVocabularyRegistry refactor | runtime | Internal storage change |
| `epistemic-domains.yaml` | eidos-vocab | Data file alongside enum vocabs |

## References

- VocabularyTerm.java — existing interface, target for keywords() addition
- VocabularyRegistry.java — existing SPI, target for register() addition and vocabularyMetadata() return type change
- CdiVocabularyRegistry.java — existing implementation, target for URI-keyed refactor
- ClasspathYamlDescriptorRegistrar.java — pattern for YAML registrar
- ClasspathYamlTemplateRegistrar.java — pattern for YAML registrar
- descriptor.schema.json — pattern for JSON Schema
- CasehubCapabilityTerm.java — existing hierarchical vocabulary (pattern for specializes())
- eval profiles (sw-engineer-bold.yaml, data-engineer-*.yaml, security-analyst-*.yaml, clinical-researcher.yaml, product-manager.yaml) — existing epistemicDomains usage
- [SKOS Simple Knowledge Organization System Reference](https://www.w3.org/TR/skos-reference/) — broader/narrower hierarchy model
- [SKOS Primer](https://www.w3.org/TR/skos-primer/) — minimal ontological commitment principle
- casehubio/eidos#199 — issue
- casehubio/eidos#198 — epic
