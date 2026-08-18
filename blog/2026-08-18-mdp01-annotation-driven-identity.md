---
layout: post
title: "Annotation-driven agent identity"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [eidos]
tags: [annotations, quarkus-extension, build-time, agent-identity]
---

CaseHub's agent identity system is rich — `AgentDescriptor` alone has 21 fields, and a fully-specified agent with disposition, goals, constraints, and capabilities spans 40+ lines of builder code. That's the right level of detail for the platform, but the entry cost for a developer who just wants to say "this is a cautious legal analyst" is too high.

The annotation-driven model we built today collapses that to what it should be:

```java
@Identity(slot = "legal-analyst", jurisdiction = "EU",
          briefing = "Senior legal analyst specialising in regulatory compliance")
@Disposition(socialOrient = "collaborative", ruleFollowing = "strict",
             riskAppetite = "cautious")
@Discoverable(capabilities = {"document-analysis", "risk-assessment"})
public interface LegalAnalyst {}
```

Eight lines. The build extension generates the `AgentDescriptorRegistrar` bean, validates disposition terms against the vocabulary (when vocab modules are on the classpath), and plugs into the existing `DescriptorCollector` pipeline. At runtime, the agent is indistinguishable from one defined via the builder or YAML.

**The architectural decision that mattered most:** where the build extension lives. The blocks#115 design spec envisioned three build extensions (LC4j, Engine, Blocks) with the blocks extension processing eidos annotations. That creates a layering problem — an app that just wants `@Identity` would need all of `casehub-blocks` on the classpath. We went with four extensions instead: each repo processes its own annotations, blocks adds cross-cutting governance composition on top. The coordination protocol is clean — `EidosAnnotationProcessedBuildItem` tells the blocks extension which classes eidos already handled.

**The Quarkus build→runtime boundary was the implementation surprise.** The natural pattern — capture annotation values at build time, construct domain records, pass them to a `Supplier` lambda — doesn't work. Quarkus needs to encode the supplier into Arc bytecode, and Java records aren't serializable. The fix is the recorder pattern: a `@Recorder` class accepts simple types (strings, enums, arrays) at build time, and the method body that constructs domain objects runs at runtime. Three layers of indirection for what feels like it should be a direct call, but the Quarkus extension model makes no promises about what survives augmentation.

`@Discoverable` landed in `casehub-eidos-api` rather than the annotations module. It's a pure marker annotation — `String[] capabilities()`, no enum dependencies. Any module that already depends on eidos-api can use it without pulling in the annotations extension. This follows the blocks#115 design principle that annotations live in the module that owns the concept, not the one that processes them.

Along the way we fixed a pre-existing bug in `DescriptorCollector.deriveDispositionAxes()` — the method was rebuilding `AgentDescriptor` field by field and silently dropping `styleProfile` and `styleVocabulary`. Switched to `descriptor.toBuilder()`, which preserves all fields including future additions. The kind of bug that doesn't surface until someone actually uses the omitted field, which is exactly what annotations would have done.

This module is the first of six annotation modules across the CaseHub platform. Engine, work, blocks, desiredstate, and ledger each get their own `*-annotations` extension following the same pattern — opt-in, recorder-based, with a coordination build item for cross-extension composition.
