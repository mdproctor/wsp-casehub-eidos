# Descriptor Field Additions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #206 — AgentDescriptor: add top-level description field
**Issue group:** #206, #207

**Goal:** Add `description` field to AgentDescriptor and promote `mbtiType`/`enneagramType` to AgentDisposition.

**Architecture:** Pure field additions to existing Java records. Touch points: record definitions, builders, YAML deserializers, annotation processor, renderer. No new modules or APIs.

**Tech Stack:** Java 21, Quarkus 3.32.2

## Global Constraints

- `MAX_DESCRIPTION = 500` already exists in `AgentDescriptorValidator`
- `AnnotatedAgentConfig` already has `mbtiType`/`enneagramType` fields
- Both fields are optional (nullable Strings)
- Follow existing field validation patterns (`validateOptional`)

---

## Batch 1: Field additions + tests

### Task 1: Add description to AgentDescriptor (#206)

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java`
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/yaml/AgentDescriptorDeserializer.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/Identity.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/AnnotatedAgentConfig.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java`

**Interfaces:**
- Produces: `AgentDescriptor.description()` → `String` (nullable)

- [ ] **Step 1: Add `description` field to AgentDescriptor record**

Add `String description` after `name` in the record parameters. Update the compact constructor to validate it. Update the builder and `toBuilder()`.

In `AgentDescriptor.java`:
- Record: add `String description,` after `String name,`
- Compact constructor: add `AgentDescriptorValidator.validateOptional("description", description, AgentDescriptorValidator.MAX_DESCRIPTION);` after the `validate(agentId, name, slot, tenancyId)` call
- Builder: add `private String description;` to the field declarations, add setter method, add to `build()` call
- `toBuilder()`: add `.description(this.description)` after `.name(this.name)`

Use `ide_replace_text_in_file` for each change in the record.

- [ ] **Step 2: Update AgentDescriptorDeserializer**

In `AgentDescriptorDeserializer.java`, add after the `builder.name(...)` line:
```java
ifString(root, "description", builder::description);
```

- [ ] **Step 3: Update @Identity annotation**

In `Identity.java`, add after `String name() default "";`:
```java
String description() default "";
```

- [ ] **Step 4: Update AnnotatedAgentConfig**

In `AnnotatedAgentConfig.java`, add after `public String name;`:
```java
public String description;
```

- [ ] **Step 5: Update EidosAnnotationsRecorder**

In `EidosAnnotationsRecorder.java`, add description wiring after the identity cluster line (`.agentId(config.agentId).name(config.name)...`):
```java
if (notEmpty(config.description)) {builder.description(config.description);}
```

- [ ] **Step 6: Update EidosAnnotationsProcessor**

In `EidosAnnotationsProcessor.java` (deployment), find where `@Identity` attributes are extracted to `AnnotatedAgentConfig`. Add extraction for `description` following the same pattern as `name`.

- [ ] **Step 7: Run full build to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all existing tests pass with the new nullable field defaulting to null.

- [ ] **Step 8: Commit**

```bash
git add api/ eidos-core/ annotations/
git commit -m "feat: add description field to AgentDescriptor

Optional String field (max 500 chars) for short UI-facing agent
descriptions. Positioned in the identity cluster after name.
Supported in YAML, annotations, and builder.

Refs #206"
```

### Task 2: Promote mbtiType/enneagramType to AgentDisposition (#207)

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDisposition.java`
- Modify: `eidos-core/src/main/java/io/casehub/eidos/core/yaml/DispositionDeserializer.java`
- Modify: `annotations/runtime/src/main/java/io/casehub/eidos/annotations/runtime/EidosAnnotationsRecorder.java`

**Interfaces:**
- Produces: `AgentDisposition.mbtiType()` → `String` (nullable), `AgentDisposition.enneagramType()` → `String` (nullable)

- [ ] **Step 1: Add fields to AgentDisposition record**

In `AgentDisposition.java`:
- Record: add `String mbtiType, String enneagramType` after `styleProfile`
- Builder: add `private String mbtiType;` and `private String enneagramType;` fields, add setter methods, add to `build()` call

No compact constructor validation needed — nullable Strings.

- [ ] **Step 2: Store values in DispositionDeserializer**

In `DispositionDeserializer.java`, after the `PersonalityTypeDeriver.derive(...)` call (line 63-65) and before `return builder.build()`, add:
```java
builder.mbtiType(mbtiType.isEmpty() ? null : mbtiType);
builder.enneagramType(enneagramType.isEmpty() ? null : enneagramType);
```

These lines read from the existing `mbtiType` and `enneagramType` local variables (lines 60-61) which are already parsed from YAML but currently discarded.

- [ ] **Step 3: Store values in EidosAnnotationsRecorder**

In `EidosAnnotationsRecorder.java`, before the `builder.disposition(db.build())` call, add:
```java
if (notEmpty(config.mbtiType)) {db.mbtiType(config.mbtiType);}
if (notEmpty(config.enneagramType)) {db.enneagramType(config.enneagramType);}
```

`AnnotatedAgentConfig` already has `mbtiType` and `enneagramType` fields — they just need to be wired to the builder.

- [ ] **Step 4: Run full build to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — existing tests pass, DispositionDeserializerTest exercises mbtiType/enneagramType parsing.

- [ ] **Step 5: Commit**

```bash
git add api/ eidos-core/ annotations/
git commit -m "feat: promote mbtiType/enneagramType to AgentDisposition

Previously read from YAML for derivation but discarded. Now stored
as optional String fields on AgentDisposition, making them available
to the REST/GraphQL/MCP API and blocks-ui.

Refs #207"
```

## References

- [2026-10-20-descriptor-fields-design.md] — design spec
- [api/AgentDescriptor.java] — record (27→28 fields)
- [api/AgentDisposition.java] — record (8→10 fields)
- [api/AgentDescriptorValidator.java:25] — MAX_DESCRIPTION already defined
- [eidos-core/yaml/AgentDescriptorDeserializer.java] — YAML deserialization
- [eidos-core/yaml/DispositionDeserializer.java:60-61] — existing mbtiType/enneagramType read
- [annotations/runtime/Identity.java] — @Identity annotation
- [annotations/runtime/AnnotatedAgentConfig.java:29-30] — mbtiType/enneagramType already present
- [annotations/runtime/EidosAnnotationsRecorder.java] — annotation→descriptor wiring
- [GitHub #206] — description field
- [GitHub #207] — mbtiType/enneagramType promotion
