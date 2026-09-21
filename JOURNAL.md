# Design Journal — issue-180-archetype-vocabulary

## 2026-09-21 — Session 1: Design exploration and vocabulary foundation

### Problem identified

The eidos vocabulary system has 85 personality terms across 11 frameworks (Belbin, MBTI, DISC, Big Five, Enneagram, Thomas-Kilmann, Jungian, SDI, SVO), each with rich descriptions. All collapse to 17 coarse canonical terms (5 axes: ConscientiousnessTerm + ThomasKilmannTerm) before reaching the renderer or avatar system. A Belbin Plant and a DISC Dominance produce near-identical prompts despite describing meaningfully different archetypes. Only the Jungian path has a rich rendering section (Cognitive Style with function descriptions, orientation, anti-patterns); all other frameworks get only the canonical axis decomposition.

### Design direction

Rather than just fixing the renderer to surface source vocabulary descriptions (the initial ask), the session evolved toward a higher-level abstraction: **archetypes**. The Hartwell & Chen model (Archetypes in Branding, 2012) provides 48 sub-archetypes in 12 families, grouped by 4 motivation quadrants. Each archetype is a human/LLM-readable personality label that sits above the existing personality frameworks.

The key insight: personality frameworks are complementary (MBTI measures cognition, DISC measures behavior, Enneagram measures motivation), and their combinations can be mapped to archetypes via set intersection. Each framework value maps to a set of compatible archetype families; intersecting all specified values converges on 1-2 archetypes.

### What was built

- `ArchetypeFamily` enum — 12 families with motivation quadrants
- `ArchetypeTerm` enum — 48 sub-archetypes with valid/invalid adjectives
- `ArchetypeCompatibility` — mapping data for MBTI, Enneagram, DISC, Belbin, Big Five, SDI to archetype families, plus sub-archetype affinity data
- `ArchetypeResolver` — set-intersection algorithm (Converged/Narrowed/Conflict result types)
- Compatibility matrix spec — raw mapping data document
- Avatar generator contract — faceted selection + visual generation specification for blocks-ui

### What remains (Batches 3-5)

- AgentDescriptor integration (archetype field, YAML support, auto-derivation)
- Renderer changes (archetype section in MARKDOWN/PROSE/A2A_CARD, source vocabulary description fix)
- Integration tests and consumer guide documentation
