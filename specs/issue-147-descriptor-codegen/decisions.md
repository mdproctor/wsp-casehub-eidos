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

## D4: Generate AnnotatedAgentConfig from record via reflection

**Choice:** Generate AnnotatedAgentConfig (the Quarkus recorder boundary POJO) from AgentDescriptor via reflection. A test validates the generated config matches the record.
**Alternatives:**
- Generate + commit as source — same source layout but auto-generated, staleness test catches drift
- Redesign recorder pattern — investigate Quarkus record support; higher risk, depends on Quarkus internals
**Rationale:** AnnotatedAgentConfig can't be eliminated (Quarkus recorders require POJOs). Generating it ensures parity gaps are structurally impossible.
**Trade-offs:** Generated code at build time adds a build step. Recorder (EidosAnnotationsRecorder) must also be updated when fields change — but the generated config gives it the right fields to wire.
**Sources:** AnnotatedAgentConfig.java, EidosAnnotationsRecorder.java in annotations/runtime
**Exploration:** quick
**Status:** captured

## D5: Convenience field resolution in Jackson Module

**Choice:** Convenience fields (mbtiType, enneagramType) resolved in the EidosDescriptorModule (Jackson Module) which receives VocabularyRegistry via CDI injection. Custom deserializer delegates to it.
**Alternatives:**
- Static methods on the record — adds VocabularyRegistry dependency to api/ (currently pure Java)
- Builder extension methods — makes builder stateful (needs VocabularyRegistry parameter)
**Rationale:** Keeps api/ as pure Java with no framework dependencies. Follows engine's pattern (CaseDefinitionModule handles all marshalling customisation). Vocabulary resolution is a runtime concern, not a domain model concern.
**Trade-offs:** Convenience fields are invisible in the record's API — only available via YAML. Builder users must do vocabulary resolution manually.
**Sources:** CaseDefinitionModule in engine, ClasspathYamlDescriptorRegistrar.toDescriptor() lines 70-99 for current mbtiType/enneagramType logic
**Exploration:** quick
**Status:** captured

## D6: Publish JSON Schema as classpath resource

**Choice:** Generated JSON Schema packaged as META-INF/eidos/descriptor-schema.json in the runtime artifact. Any consumer can validate descriptors.yaml against it.
**Alternatives:**
- Test-only validation — schema not published, external tools can't consume
- Separate schema artifact — more modules to manage
**Rationale:** Personality generation tools (quarkmind#283) and IDE plugins need the schema. Classpath resource is zero-cost for consumers already depending on eidos.
**Trade-offs:** Schema generation must run at build time (not just test time). Adds victools as a build dependency of eidos runtime.
**Sources:** engine schema/src/main/resources/schema/CaseDefinition.yaml precedent
**Exploration:** quick
**Depends on:** D3 (shared generator produces the schema)
**Status:** captured
