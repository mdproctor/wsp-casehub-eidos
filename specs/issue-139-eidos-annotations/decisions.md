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
