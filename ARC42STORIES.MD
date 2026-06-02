# casehub-eidos — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** After casehub-ledger (runtime depends on ledger); casehub-eidos-api depends on nothing. Publishes before casehub-engine and claudony.
**Consumed by:** casehub-engine (optional eidos-api dep), claudony, application-tier apps that need agent identity
**Depends on:** `casehub-eidos-api` → none. `casehub-eidos` runtime → casehub-ledger.

---

## §1 Introduction and Goals

### Description

`casehub-eidos` provides agent identity management — descriptor schemas, a discovery registry, a vocabulary system for agent capability classification, and system prompt generation from agent metadata.

Core capabilities: `AgentDescriptor` (typed agent identity — id, displayName, capabilities, trust dimensions, persona), `AgentRegistry` (discovery registry — find agents by capability, by trust threshold, by availability), vocabulary system (canonical string constants for agent capability classification — aligned with the typed vocabulary classes in application-tier domain models), system prompt generation (constructs SYSTEM prompts from AgentDescriptor metadata for Claude sessions via claudony).

Module structure: `api` (AgentDescriptor types, AgentRegistry SPI — zero casehubio deps), `runtime` (JPA-backed AgentRegistry implementation, system prompt generator — depends on casehub-ledger for trust score read access).

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume this module's foundation primitives |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |

### Quality Goals

| Goal | Scenario |
|---|---|
| Zero-dep API | casehub-eidos-api depends on nothing — any consumer can reference agent identity types without pulling the full eidos stack |
| Capability vocabulary alignment | Eidos vocabulary constants align with application-tier domain typed classes — prevents classification drift across the ecosystem |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:casehub-eidos:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

---

## §3 Context and Scope

🔲 C4Context diagram — add in dedicated session.

**Boundary rules:** casehub-eidos owns only the concerns described in §1. Domain logic (business rules, application-specific workflows) belongs in application-tier repos (devtown, aml, clinical, life, drafthouse, quarkmind).

See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

---

## §4 Solution Strategy

🔲 Core architectural patterns and module structure — complete in dedicated session from DESIGN.md and repo architecture.

---

## §5 Building Block View

🔲 C4Container diagram and module structure — complete in dedicated session.

---

## §6 Runtime View

🔲 Key runtime scenarios — complete in dedicated session.

---

## §7 Deployment View

Published to GitHub Packages. CI: GitHub Actions. No GraalVM native target (foundation modules).

---

## §8 Crosscutting Concepts

References to governing protocols:

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |

🔲 Anti-patterns inline (Symptom → Cause → Fix) — complete in dedicated session. Must be present per profile rules; do not defer to AGENTIC-HARNESS-GUIDE.md reference alone.

---

## §9 Journeys and Chapters

🔲 **Bootstrap — full Journeys and Chapters pending dedicated session.**

Foundation modules use Chapters to track delivery milestones of the foundation capability itself (transport types, SPI increments, new adapters) rather than integrating external foundation modules.

Read `../parent/docs/arc42stories-spec.md §4 Solution Strategy` — "Foundation: Chapters = delivery milestones of the foundation capability itself."

The full §9 entry requires:
- Reading DESIGN.md (or equivalent) for the delivery arc
- Mapping milestones to Chapters
- Writing §9.4 Layer/Chapter entries with all 18 required subsections

---

## §10 Architectural Decisions

Per the Arc42Stories spec: only decisions not captured inline elsewhere. Should be sparse.

🔲 Cross-cutting decisions — complete in dedicated session.

---

## §11 Quality Requirements

See §1 Quality Goals for the primary scenarios.

🔲 Full quality requirements table — complete in dedicated session.

---

## §12 Risks and Technical Debt

🔲 Active risks and technical debt — complete in dedicated session.

---

## §13 Glossary

| Term | Definition |
|---|---|
| `AgentDescriptor` | Typed agent identity: id, displayName, capabilities (strings), trust dimensions, persona metadata. |
| `AgentRegistry` | SPI for agent discovery — find agents by capability tag, by minimum trust score, by availability. |
| `Vocabulary system` | Canonical string constants for agent capability classification — the shared vocabulary consumed by trust scoring, routing, and agent dispatch. |

---

*Bootstrap document — generated 2026-06-02. §3–§9 require a dedicated session to complete from repository source material.*
