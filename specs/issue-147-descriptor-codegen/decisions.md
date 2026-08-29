# Decisions — #147 Descriptor Codegen

## D1: Builders in scope for generation

**Choice:** Generate builders (AgentDescriptor.Builder, AgentDisposition.Builder, AgentCapability.Builder) from the records
**Alternatives:**
- Keep builders hand-written — convenience overloads (varargs, String shortcuts) would need manual extension points
- Generate base, hand-write extensions — added complexity of inheritance for little gain
**Rationale:** Builders are mechanical derivatives — one setter per field, build() calls constructor. Generation eliminates drift.
**Trade-offs:** Convenience overloads (e.g., `socialOrient(String)` vs `socialOrient(DispositionValue...)`) must be expressible by the generator.
**Sources:** AgentDescriptor.Builder, AgentDisposition.Builder, AgentCapability.Builder in eidos api/
**Exploration:** quick
**Status:** captured

## D2: Eliminate DescriptorConfig — follow engine's Jackson Module pattern

**Choice:** Eliminate DescriptorConfig intermediate classes. Deserialize YAML directly into AgentDescriptor via a Jackson Module with custom deserializers and mixins.
**Alternatives:**
- Keep DescriptorConfig but generate it — simpler deserializer but adds a generated intermediate layer
- Keep DescriptorConfig hand-written — status quo, doesn't solve the root sync problem
**Rationale:** Engine already uses this pattern (CaseDefinitionModule + CaseDefinitionMixin + CaseDefinitionDeserializer). Eliminating the intermediate means one fewer derivative to maintain. Convenience fields (mbtiType, enneagramType) handled by the custom deserializer.
**Trade-offs:** Custom deserializer is more complex than flat POJO mapping. Type adaptations (String → List<DispositionValue>) move into deserializer logic.
**Sources:** io.casehub.api.model.converter.CaseDefinitionModule, CaseDefinitionMixin, CaseDefinitionDeserializer, WorkerMarshaller in engine
**Exploration:** deep-analysis
**Status:** captured

## D3: Shared schema generator in casehub-parent

**Choice:** Extract a shared `casehub-schema-generator` module in casehub-parent. All repos inherit it and add only their domain-specific custom modules. Write an indexed protocol so all repos follow the pattern.
**Alternatives:**
- Eidos-own generator/ module — duplicates victools base setup, doesn't benefit other repos
- In existing deployment/ module — conflates build-time extension with schema tooling
**Rationale:** Both engine and eidos need victools-based schema generation with the same base config (SchemaVersion.DRAFT_2020_12, JacksonModule, JakartaValidationModule). Extracting to platform avoids duplication and ensures all repos produce schemas the same way. Protocol ensures future repos adopt it.
**Trade-offs:** Platform-level change — requires casehub-parent release. Engine's existing generator/ module needs migration.
**Sources:** io.casehub.generator.CaseHubSchemaGenerator in engine, casehub-dependency-tier-order protocol
**Exploration:** quick
**Status:** captured
