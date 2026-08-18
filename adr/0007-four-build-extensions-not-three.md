# ADR-0007: Four Build Extensions — Eidos Owns Its Annotation Processing

**Date:** 2026-08-18
**Status:** Accepted
**Context:** casehubio/eidos#139, casehubio/blocks#115

## Decision

The eidos annotation module (`casehub-eidos-annotations`) has its own Quarkus build extension (`EidosAnnotationsProcessor`) rather than relying on the blocks build extension to process eidos annotations.

This means four build extensions in the annotation-driven model (LC4j, Engine, Eidos, Blocks), not the three shown in the blocks#115 design spec.

## Context

The blocks#115 spec (§Build Extension Architecture) envisioned three build extensions with the blocks extension processing eidos annotations (`@Identity`, `@Disposition`, etc.) as part of cross-module composition. This creates a layering problem: any app that wants `@Identity` on an agent class would need `casehub-blocks` on the classpath — even if it doesn't use debate, voting, HTN, or any blocks pattern.

## Alternatives Considered

1. **Blocks processes eidos annotations** (spec's original design) — rejected: forces blocks dependency for identity-only apps
2. **Annotations in eidos-api, processing in existing eidos-deployment** — rejected: forces annotation processing on all eidos consumers, not opt-in
3. **Annotations in eidos-api, no build extension** — rejected: no build-time validation, no synthetic bean generation

## Coordination

`EidosAnnotationsProcessor` produces `EidosAnnotationProcessedBuildItem` listing processed classes. When both eidos-annotations and blocks are on the classpath, the blocks build extension consumes this build item and skips `AgentDescriptor` generation for already-processed classes. When eidos-annotations is absent, blocks generates descriptors itself (backward compatible).

## Consequences

- Apps using only eidos identity annotations need `casehub-eidos-annotations`, not `casehub-blocks`
- Two new Maven modules per repo that adopts this pattern (runtime + deployment)
- `@Discoverable` lives in `casehub-eidos-api` (pure marker, no enum dependencies) so both extensions can see it without circular dependencies
