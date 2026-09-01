---
layout: post
title: "Org Annotations — Closing the Loop"
date: 2026-09-01
entry_type: note
subtype: diary
projects: [casehubio/eidos]
tags: [eidos, org-model, annotations, quarkus-extension]
series: issue-150-org-model
---

# Org Annotations — Closing the Loop

The org model landed last session — units, memberships, relationships, cycle
detection, a compositional DSL, YAML surface, and desiredstate integration.
What it didn't have was annotation-driven declaration, the same zero-code
path that `@Identity` and `@Disposition` give agent descriptors.

That's what this session adds: `@OrgUnit`, `@OrgMembers`, and `@Supervises`.

```java
@OrgUnit(kind = "rig")
@OrgMembers({
    @OrgMemberDef(agentId = "witness-1", role = "witness"),
    @OrgMemberDef(agentId = "polecat-1", role = "worker")
})
@Supervises(source = "witness-1", target = "polecat-1")
@Supervises(source = "witness-1", target = "polecat-2")
public interface RigAlpha {}
```

The unit ID derives from the class name (kebab-case, same as `@Identity`).
`@Supervises` is `@Repeatable` for the common case. `@OrgRelationships` with
`@OrgRelationshipDef` covers everything else — `BACKS_UP`, `ESCALATES_TO`,
`DELEGATES_TO`, and `EXTENDED`.

The build extension follows the same Quarkus pattern: Jandex scan at build
time, config extraction into a recordable data class, synthetic
`OrgRegistrar` CDI beans produced via a `@Recorder`. Nine deployment tests
cover the full surface — explicit IDs, auto-derived IDs, members, supervision,
general relationships, hierarchy via `parentUnit`, and tenancy.

## The Extension Chain Problem

I expected this to be mechanical — copy the eidos-annotations pattern, swap
types, done. One thing bit: the Quarkus extension descriptor plugin enforces
that if a deployment module depends on another extension's runtime artifact,
the corresponding runtime module must also declare that dependency. The
processor needed `NameDerivation` from `casehub-eidos-annotations`, and
pulling that in would have dragged the entire eidos extension chain into
org-annotations.

The fix was simpler than the problem: duplicate the 42-line utility. Two
copies of a pure function beats one tangled dependency graph.

## The Missing Bootstrap

Building the annotations surfaced a gap in the previous session's work.
The org model had `OrgRegistrar` as its declarative SPI and
`ClasspathYamlOrgRegistrar` implementing it — but nothing consumed the
registrars. No startup observer collected `OrgRegistrar` beans and fed
their definitions into `OrgRegistry`. The equivalent of
`AgentDescriptorBootstrap` simply didn't exist for org.

`OrgBootstrap` is 15 lines. It observes `StartupEvent`, iterates
`Instance<OrgRegistrar>`, and registers units and relationships. Without
it, neither YAML nor annotation-declared orgs would actually register
at runtime. The kind of gap you only find when you wire the consumer.
