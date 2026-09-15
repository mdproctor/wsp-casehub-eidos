---
layout: post
title: "The First Untyped Hole"
date: 2026-09-15
entry_type: note
subtype: diary
projects: [casehubio/eidos]
tags: [eidos, agent-descriptor, extension-data, type-safety, immutability]
---

`AgentDescriptor` is a record where every field earns its type. Strings have length bounds. Lists get `List.copyOf()`. Maps have typed keys. The compact constructor rejects anything that doesn't pass validation before the record exists. That's the design philosophy: no invalid record can be constructed.

Then I added `Map<String, Object>`.

The motivating case was wacky-manor — a game application that needed social configuration (drives, norms, beliefs) embedded alongside the standard agent identity. The workaround was a separate YAML file with a custom loader, which split character definition across two files. extensionData puts it back in one place: the descriptor YAML carries the agent's identity and the application's config in a single document.

The type choice was deliberate. `Map<String, String>` would force applications to double-serialize nested structures. `JsonNode` would leak Jackson into the api module, which is currently pure Java and I want to keep it that way. A raw JSON `String` just pushes type-unsafety to every consumer. `Map<String, Object>` is the honest choice — extension data IS untyped by design. The descriptor's job is to carry it safely, not to understand it.

"Safely" is where it got interesting. Claude ran a decision review against the design and caught something I'd glossed over: the `Map.copyOf()` analogy was false. I'd cited `axisVocabularies` as precedent — but that map has `DispositionAxis` keys and `String` values, both immutable. `Map<String, Object>` from YAML deserialization contains mutable `LinkedHashMap` and `ArrayList` instances. `Map.copyOf()` on that produces an unmodifiable top-level map with freely mutable nested values. A caller who does `((List<?>) descriptor.extensionData().get("drives")).clear()` corrupts the descriptor for everyone sharing that instance.

The fix was a recursive deep-copy utility — about 40 lines of pure Java that walks the tree and copies every `Map` and `List`, passing through immutable types (String, Number, Boolean, null). Zero dependencies, consistent with every other field's immutability guarantee. We also hit the `Map.copyOf()` null-rejection gotcha during implementation — YAML values can be null, and `Map.copyOf()` throws NPE on null values. `Collections.unmodifiableMap()` handles both concerns.

The decision review also pushed back on splitting validation across layers — my original plan was to validate key count in the api module and byte size in JPA. That contradicts the "no invalid record" invariant; every other field is fully validated at construction time. We replaced it with a recursive size estimation that walks the map tree and counts characters — conservative, but catches abuse without needing Jackson in the api module.

extensionData is never rendered in any prompt format. It doesn't participate in the A2A structural hash, doesn't appear in cache keys, doesn't show up in MARKDOWN or PROSE output. It's application-level configuration that travels with the descriptor but stays invisible to the rendering pipeline. The protocol guard is documented in the spec — if someone later decides to render it, they'll hit the note about PP-20260613-608684 before they create a cache coherence bug.

The annotation path (`@ExtensionData` with `@ExtensionEntry`) handles flat key-value pairs. Nested structures stay in YAML. That's a natural limitation of Java annotation syntax, not a gap — applications that start with annotations and later need nested config migrate to YAML, which is progressive disclosure rather than a trap.

What's worth noting is the tension this creates. AgentDescriptor's identity as a fully-typed, fully-validated record now has one deliberately opaque field. The compact constructor still rejects invalid records — extensionData is deep-copied and size-limited — but the type system can't help consumers who cast to the wrong type at runtime. That's the trade-off for not forcing every application to maintain a separate config file. Whether it holds up depends on how applications use it. If wacky-manor's `SocialConfig` is the template — a well-known shape read once and cast to a typed record on the application side — the opaque field is a thin bridge between two typed worlds. If applications start treating it as a dumping ground, the convention-only namespacing becomes load-bearing in a way it wasn't designed for.
