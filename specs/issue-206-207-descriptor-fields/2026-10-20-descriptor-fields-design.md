# Design: AgentDescriptor field additions

**Issues:** eidos#206, eidos#207
**Branch:** issue-206-207-descriptor-fields

## Summary

Two field additions to existing records — no new modules, no new APIs.

## #206 — AgentDescriptor.description

Add `String description` to `AgentDescriptor` after `name`. Optional
(nullable). Max length follows `MAX_BRIEFING` or a new
`MAX_DESCRIPTION` constant (500 chars — shorter than briefing's 4000).

### Touch points

- `AgentDescriptor.java` — add field to record, compact constructor
  validation, builder
- `AgentDescriptorDeserializer.java` — read `description` from YAML
- `AgentDescriptorValidator.java` — add `MAX_DESCRIPTION` constant
- `EidosAnnotationsProcessor.java` — extract from `@Identity` if
  annotation gains a `description` attribute
- `@Identity` annotation — add `description` attribute (default `""`)
- `AnnotatedAgentConfig.java` — add field
- `EidosAnnotationsRecorder.java` — wire field
- JPA entity (if exists) — add column
- `EidosSystemPromptRenderer` — render description in MARKDOWN/PROSE
  (after name, before capabilities)
- Example YAML descriptors — add descriptions
- `EidosAgentApi` (graphql module) — no change needed, field serializes
  automatically

## #207 — AgentDisposition.mbtiType / enneagramType

Add `String mbtiType` and `String enneagramType` to `AgentDisposition`.
Optional (nullable).

### Touch points

- `AgentDisposition.java` — add fields to record, builder
- `DispositionDeserializer.java` — store parsed values instead of
  discarding (lines 60-61 already read them)
- `@Disposition` annotation — already has these attributes, no change
- `AnnotatedAgentConfig.java` — already carries these, verify wiring
- `EidosAnnotationsRecorder.java` — verify wiring to builder
- JPA entity (if exists) — add columns
- A2A_CARD rendering — include in disposition section
- Example YAML descriptors — add where appropriate

## Testing

- Unit test: construct descriptors with new fields, verify roundtrip
- YAML deserialization test: verify fields parse from YAML
- Annotation processor test: verify fields extracted from annotations
- Renderer test: verify fields appear in rendered output

## References

- api/AgentDescriptor.java — record definition
- api/AgentDisposition.java — record definition
- eidos-core/yaml/DispositionDeserializer.java:60-61 — existing mbtiType/enneagramType read
- eidos-core/yaml/AgentDescriptorDeserializer.java — YAML deserialization
- annotations/runtime/Identity.java — @Identity annotation
- annotations/runtime/Disposition.java — @Disposition annotation (already has mbtiType/enneagramType)
