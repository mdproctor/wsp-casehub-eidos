# Design Decisions — casehub-eidos-annotations (#139)

## D1: Module architecture — annotations in eidos, not blocks

**Choice:** New `casehub-eidos-annotations` Quarkus extension (runtime + deployment). Annotations in the runtime module, build extension in deployment. No langchain4j-agentic dependency.
**Alternatives:**
- Annotations in `casehub-eidos-api`, processing in blocks build extension — layering problem: apps need blocks for identity annotations
- Annotations in `casehub-eidos-api`, processing in existing `casehub-eidos-deployment` — forces annotation processing on all eidos users
**Rationale:** Each repo processes its own annotations. Eidos annotations reference only eidos-api types (no LC4j dependency). Blocks adds cross-cutting governance composition separately. Opt-in model follows Quarkus conventions (like Panache).
**Trade-offs:** Two new Maven modules (runtime + deployment) for 6 annotation classes. Worth it for clean layering and opt-in.
**Sources:** blocks#115 design spec (Design Principle 2, Build Extension Architecture), eidos-api source, Quarkus extension model
**Exploration:** deep-analysis
**Status:** captured

## D2: agentId, name, tenancyId mapping

**Choice:** Add `id()` and `name()` fields to `@Identity` with empty defaults (derive from kebab-cased / spaced class name). `tenancyId` from MicroProfile config (`casehub.eidos.default-tenancy-id`) only — no annotation field.
**Alternatives:**
- Derive from class name only (no annotation fields) — too restrictive, no override for the 20% case
- All three on annotation including tenancyId — tenancyId varies per deployment, not per agent definition
**Rationale:** Progressive disclosure: simple annotations just work, power users override. Identity-level fields go on the annotation; operational fields come from configuration.
**Trade-offs:** Class-name derivation may produce unexpected IDs for unusual class names; documented convention mitigates.
**Sources:** AgentDescriptor.java (required fields), AgentDescriptorBootstrap.java (registration flow), spec @Identity definition
**Exploration:** quick
**Status:** captured

## D3: styleProfile and styleVocabulary coverage

**Choice:** Add `String[] styleProfile() default {}` to `@Disposition` and `String styleVocabulary() default ""` to `@Identity`. Same pattern as dispositionProfile/dispositionVocabulary.
**Alternatives:**
- Omit — builder/YAML only for style — leaves annotation surface incomplete for a shipped API field
- Separate @StyleProfile annotation — over-annotates for a single field
**Rationale:** Annotation surface should be complete relative to the API. styleProfile and dispositionProfile are parallel concepts with the same mapping pattern (String[] → List<DispositionValue>).
**Trade-offs:** None significant — trivial addition.
**Sources:** AgentDisposition.java:13 (styleProfile field), AgentDescriptor.java:11 (styleVocabulary field)
**Exploration:** quick
**Status:** captured

## D4: Vocabulary validation timing

**Choice:** Hybrid — build-time validation when vocabulary enums are Jandex-indexed on the classpath, runtime fallback via existing DescriptorCollector otherwise.
**Alternatives:**
- Runtime only — simpler but misses compile-time error detection for typos
- Full build-time only — requires vocab always on classpath, too restrictive
**Rationale:** Quarkus philosophy is fail-fast at build time. Jandex already indexes @VocabularyMetadata-annotated enums. Checking disposition term strings against enum constants is straightforward. Catches typos like "collaboartive" at compile time when vocab is available.
**Trade-offs:** Build extension is slightly more complex (term-existence check against Jandex-indexed enums). Graceful fallback when vocab isn't available means errors can still slip to runtime.
**Sources:** VocabularyMetadata.java, VocabularyTerm.java, CdiVocabularyRegistry.java, Quarkus Jandex build API
**Exploration:** quick
**Status:** captured

## D5: Bean generation strategy

**Choice:** Build extension generates synthetic CDI beans implementing `AgentDescriptorRegistrar`. Existing `DescriptorCollector.collectAndValidate()` handles validation, disposition axis derivation, and duplicate detection.
**Alternatives:**
- Generate AgentDescriptor beans directly — bypasses DescriptorCollector, would need to replicate validation
- Quarkus recorder pattern — more Quarkus-idiomatic but requires new recorder class for no benefit
**Rationale:** Plugs into the existing registration pipeline. Annotation-defined and builder/YAML-defined descriptors flow through the same path. No validation duplication.
**Trade-offs:** None — this is the natural integration point.
**Sources:** AgentDescriptorBootstrap.java, DescriptorCollector.java, AgentDescriptorRegistrar SPI
**Exploration:** quick
**Status:** captured

## D6: @Discoverable in scope

**Choice:** Include `@Discoverable(capabilities, tags)` in #139 scope. @DiscoverFrom and @Route remain blocks concerns.
**Alternatives:**
- Defer to separate issue — creates unnecessary backlog for minimal incremental work
**Rationale:** @Discoverable is an eidos concept (agent declares it CAN be discovered). The build extension already scans for @Identity — adding capability extraction from @Discoverable is trivial. @Discoverable creates AgentCapability instances with name only; full capability metadata uses the builder.
**Trade-offs:** @Discoverable capabilities are name-only (no description, epistemicDomains, etc.). Progressive disclosure: simple via annotation, rich via builder.
**Sources:** blocks#115 spec §Layer 2 Dynamic composition annotations, AgentCapability.java
**Exploration:** quick
**Status:** captured
