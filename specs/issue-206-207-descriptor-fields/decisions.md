# Decisions — eidos#206 + eidos#207

## D1: description field position on AgentDescriptor

**Choice:** After `name` in the identity cluster: agentId, name, description, version
**Alternatives:**
- Near briefing — groups text fields but conflates UI metadata with prompt content
**Rationale:** description is identity metadata for UI cards; briefing is LLM prompt content. Different purposes, different positions.
**Trade-offs:** None — field ordering on Java records is cosmetic for JSON serialization.
**Sources:** api/AgentDescriptor.java
**Exploration:** quick
**Status:** captured

## D2: promote mbtiType/enneagramType to AgentDisposition

**Choice:** Add `mbtiType` and `enneagramType` as stored String fields on AgentDisposition
**Alternatives:**
- Keep derivation-only — blocks-ui removes these fields; eval profiles keep them separately
**Rationale:** DispositionDeserializer already reads them from YAML; @Disposition annotation already declares them. They're used for derivation AND are part of agent identity (personality-workbench uses them). Storing them eliminates the fabrication in blocks-ui.
**Trade-offs:** None — the data flow already exists, we're just retaining instead of discarding.
**Sources:** eidos-core/yaml/DispositionDeserializer.java:60-61, annotations/runtime/Disposition.java
**Exploration:** quick
**Status:** captured
