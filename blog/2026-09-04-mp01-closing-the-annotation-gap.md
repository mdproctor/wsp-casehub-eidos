---
layout: post
title: "Closing the Annotation Gap"
date: 2026-09-04
entry_type: note
subtype: diary
projects: [casehubio/eidos]
tags: [org-annotations, parity, annotations, quarkus]
series: issue-153-dsl-annotations-yaml-audit
---

# Closing the Annotation Gap

The eidos org-annotations module had a real parity problem. YAML-driven org structures could express capabilities, goals, constraints, attestation grants, and structured relationship scope. The annotation path couldn't. If you chose annotations for your org structure, you lost half the model.

The fix was to reuse what eidos-annotations already had. `@AgentGoalDef`, `@AgentConstraintDef`, and `@AgentCapabilityDef` — the same annotations that declare agent-level identity — now nest directly inside `@OrgUnit` as array members. No duplication, no org-specific copies. The layering infrastructure from an earlier commit made this possible: shared `AnnotationProcessorUtils` and `EidosAnnotationProcessedBuildItem` give both processors a common foundation.

The interesting design question was whether `@AgentCapabilityDef` — which has `@Target(ElementType.TYPE)` — could be used as a nested annotation value. Java doesn't enforce `@Target` for annotations appearing as element values in other annotations; it only governs where the annotation can be applied as a decorator. So `@OrgUnit(capabilities = @AgentCapabilityDef(name = "code-review"))` compiles and works without changing the capability annotation at all.

Relationship scope had a different kind of gap. `@OrgRelationshipDef` already had `scope()` as a bare string mapping to `RelationshipScope.capabilityName`. But `RelationshipScope` is a three-field record — `capabilityName`, `domain`, and `custom`. We added `scopeDomain()` and `scopeCondition()` to both `@OrgRelationshipDef` and `@Supervises`, with the recorder constructing the full `RelationshipScope` from the three fields.

Attestation grants needed a new annotation — `@AttestationGrantDef` with `dimensions`, `capabilityScope`, and `signalTypes`. The Java annotation syntax requires `AttestationGrantDef[]` as the member type (annotations can't default to `null`), even though `AttestationGrant` is semantically singular on a relationship.

The parity test now catches drift automatically. `ORG_UNIT_INFRA_FIELDS` shrank from five entries to two — only `tenancyId` and `members` remain infrastructure-only. `tenancyId` stays infrastructure because of a deliberate decision from earlier in the epic: it's derived from config, not declared per-unit. The Principal identity model in platform reinforces this — `principalId` is the canonical identity key, and tenancy is orthogonal.
