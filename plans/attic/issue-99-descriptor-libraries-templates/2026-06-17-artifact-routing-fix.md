# Artifact Routing Fix — Specs, ARC42STORIES.MD, CLAUDE.md

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the CLAUDE.md routing gap that left ARC42STORIES.MD workspace-only, rescue 2 missing specs, update ARC42STORIES.MD for post-C6 changes, and promote both to the project repo.

**Architecture:** Two repos in play — workspace (`~/claude/public/casehub/eidos`) and project (`~/claude/casehub/eidos`). Specs and ARC42STORIES.MD live in workspace during development and are promoted to project repo at epic close. The routing table in CLAUDE.md omitted ARC42STORIES.MD, so it was never promoted. Blogs go workspace → mdproctor.github.io; all 27 eidos entries are already published (verified).

**Tech Stack:** git (two repos), markdown files, no code changes.

---

## Pre-flight

Key paths used throughout:
- `WORKSPACE=/Users/mdproctor/claude/public/casehub/eidos`
- `PROJECT=/Users/mdproctor/claude/casehub/eidos`

---

## Task 1: Fix CLAUDE.md routing table

**Files:**
- Modify: `~/claude/casehub/eidos/CLAUDE.md` — add ARC42STORIES.MD row to routing table and Artifact Locations table

**Context:** The CLAUDE.md Routing table has entries for adr, specs, blog, plans, design, snapshots, handover — but ARC42STORIES.MD is absent. The file has been accumulating in workspace and never gets promoted.

- [ ] **Step 1: Open CLAUDE.md in project repo**

File: `/Users/mdproctor/claude/casehub/eidos/CLAUDE.md`

Find the routing table:
```
| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | project     | lands in `docs/adr/` — promoted at epic close |
| specs      | project     | lands in `docs/specs/` — promoted at epic close |
```

Add the ARC42STORIES.MD row immediately after the `specs` row:
```
| arc42stories | project   | `ARC42STORIES.MD` at project root — promoted at epic close |
```

- [ ] **Step 2: Update Artifact Locations table**

Find the Artifact Locations table near the top:
```
| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` (workspace staging) |
```

Add a row for ARC42STORIES.MD (after brainstorming row):
```
| update-design / arc42stories | `ARC42STORIES.MD` (workspace staging) |
```

- [ ] **Step 3: Commit to project repo**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs: add ARC42STORIES.MD to routing table — was missing, explaining why file never promoted to project repo"
```

---

## Task 2: Rescue 2 missing specs from workspace branches

**Files:**
- Create: `PROJECT/docs/specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md`
- Create: `PROJECT/docs/specs/2026-05-31-multi-format-eval-validation-design.md`

**Context:** Two specs exist only on closed workspace branches and were never promoted to the project repo. Both are still on their original branches in the workspace repo.

- [ ] **Step 1: Extract spec from issue-016 branch**

```bash
git -C /Users/mdproctor/claude/public/casehub/eidos show \
  issue-016-quality-eval-security:specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md \
  > /Users/mdproctor/claude/casehub/eidos/docs/specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md
```

- [ ] **Step 2: Extract spec from issue-021-020 branch**

```bash
git -C /Users/mdproctor/claude/public/casehub/eidos show \
  issue-021-020-eval-coverage-validation:specs/2026-05-31-multi-format-eval-validation-design.md \
  > /Users/mdproctor/claude/casehub/eidos/docs/specs/2026-05-31-multi-format-eval-validation-design.md
```

- [ ] **Step 3: Verify both files exist and are non-empty**

```bash
wc -l /Users/mdproctor/claude/casehub/eidos/docs/specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md
wc -l /Users/mdproctor/claude/casehub/eidos/docs/specs/2026-05-31-multi-format-eval-validation-design.md
```

Expected: both > 10 lines.

- [ ] **Step 4: Commit to project repo**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add \
  docs/specs/2026-05-30-quality-eval-prompt-security-reactive-cache-design.md \
  docs/specs/2026-05-31-multi-format-eval-validation-design.md
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs: rescue 2 specs that were branch-only in workspace — quality-eval-security and multi-format-eval-validation"
```

---

## Task 3: Update ARC42STORIES.MD for post-C6 changes

**Files:**
- Modify: `WORKSPACE/ARC42STORIES.MD`

**Context:** The workspace ARC42STORIES.MD is current through Chapter 6 (Knowledge Graph, ~June 4). Since then, significant changes have merged that are not reflected in the layer entries:

**Changes needed:**

### 3A — §1 Description: add briefing + enriched flag + capability signal discrimination note

Current §1 says:
> `AgentDescriptor` — four-layer structured description (identity, slot, capabilities, disposition)

Needs to add `briefing` as an optional freeform field.

Current `SystemPromptRenderer` blurb mentions `A2A_CARD` routing signals but does not say that PROSE/MARKDOWN exclude numeric signals.

The `RenderedPrompt.enriched` flag is not mentioned.

### 3B — §1 Key Design Decisions: update Generative section

The Generative bullet should note:
- Capability rendering is format-discriminated: PROSE/MARKDOWN surface names + `inputTypes`/`outputTypes` only; numeric routing signals (`qualityHint`, `latencyHintP50Ms`, `costHint`, `epistemicDomains`) appear in A2A_CARD only (protocol PP-20260611-228599)
- `RenderedPrompt.enriched` flag skips completeness substring check for LLM-enriched renders
- `AgentDescriptor.briefing` rendered in structural fallback MARKDOWN+PROSE; included in cache key

### 3C — L1 Zero-Dep API: add briefing, enriched, DispositionAxis, VocabularyMetadata, VocabularyRegistrar

The L1 key files list references `AgentDescriptor.java` without mentioning `briefing`. And the vocab SPI types (VocabularyMetadata, VocabularyRegistrar, DispositionAxis) are not listed as L1 key files even though they are in `casehub-eidos-api`.

Add to "What it adds" bullet list:
- `AgentDescriptor.briefing` — optional freeform text field; included in cache key; rendered in MARKDOWN+PROSE structural fallback
- `DispositionAxis` enum: SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE; `jsonKey()` → camelCase JSON key; `description()` → axis description for LLM judge prompts
- `VocabularyMetadata` annotation: `uri` (required), `name`, `version` on vocabulary enum classes
- `VocabularyRegistrar` @FunctionalInterface CDI SPI: `@ApplicationScoped` beans auto-register vocab enums
- `RenderedPrompt.enriched` flag: set by EidosSystemPromptRenderer when Stage 2 enrichment ran; used by eval completeness check to skip substring matching on enriched content

Add to key files:
```
- `api/src/main/java/io/casehub/eidos/api/DispositionAxis.java` — enum: SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE; `jsonKey()` → camelCase JSON key; `description()` → axis description for LLM judge prompts
- `api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java` — annotation: uri (required), name, version on vocabulary enum classes
- `api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java` — interface implemented by vocabulary enum constants; `exactMatch` + `axisExactMatch`
- `api/src/main/java/io/casehub/eidos/api/spi/VocabularyRegistrar.java` — @FunctionalInterface CDI SPI; @ApplicationScoped beans auto-register vocab enums
```

### 3D — L2 CDI Discovery: major update for enum-based vocabulary (eidos#40)

The L2 layer entry uses the OLD string-based vocabulary API. It needs to be rewritten to reflect:
- `Vocabulary` record → removed; `VocabularyRegistrar` @FunctionalInterface replaces it
- `VocabularyTerm` record → replaced by `VocabularyTerm` interface implemented by enum constants
- `CdiVocabularyRegistry` now discovers `Instance<VocabularyRegistrar>`, not `Instance<Vocabulary>`
- Three internal maps: `byUri / byClass / byClassOrdered`
- `register(Class<T>)` reads `@VocabularyMetadata` annotation; typed generic methods alongside string-based methods
- Axis-aware `equivalentValues(fromVocab, value, toVocabClass, axis)` — axis parameter distinguishes DISC `"dominance"` → Conscientiousness differently per disposition axis
- `casehub-eidos-vocab` now provides enum implementations (SVO, Conscientiousness, CasehubSlot, Belbin, DISC, Thomas-Kilmann) via `VocabularyRegistrar` beans, not CDI `Vocabulary` beans

Rewrite key files:
```
- `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java` — @DefaultBean @ApplicationScoped; discovers Instance<VocabularyRegistrar>; three maps: byUri/byClass/byClassOrdered; register(Class<T>) reads @VocabularyMetadata
- `api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java` — SPI: register(Class<T>), isRegistered, resolve, allTerms, equivalentValues (typed + string-based + axis-aware)
- `api/src/main/java/io/casehub/eidos/api/spi/VocabularyRegistrar.java` — @FunctionalInterface; @ApplicationScoped beans auto-register vocab enums
- `vocab/src/main/java/io/casehub/eidos/vocab/SvoTerm.java`, `ConscientiousnessTerm.java`, `CasehubSlotTerm.java`, `BelbinTerm.java`, `DiscTerm.java`, `ThomasKilmannTerm.java` — enum constants implementing VocabularyTerm; each paired with a VocabularyRegistrar CDI bean
```

Key wiring rewrite:
- `@Any Instance<VocabularyRegistrar>` discovers all registrar beans
- `register(Class<T>)` reads `@VocabularyMetadata.uri()` — absent annotation throws `IllegalArgumentException`
- `equivalentValues(String fromVocab, String value, String toVocabClass, DispositionAxis axis)` — axis parameter enables DISC-as-disposition where the same type maps differently per axis

Architecture decision update:
- "Why CDI `Instance<VocabularyRegistrar>` rather than `Instance<Vocabulary>`" — enum class IS the vocabulary; no per-constant repetition; typed cross-vocab methods are impossible on stringly-typed records

### 3E — L5 Rendering Pipeline: capability signal discrimination, enrichment mechanics, briefing, enriched

Add to "What it adds" bullets:
- **Format-discriminated capability signals** (eidos#49, protocol PP-20260611-228599): PROSE/MARKDOWN render capability names + `inputTypes`/`outputTypes` only; numeric signals (`qualityHint`, `latencyHintP50Ms`, `costHint`, `epistemicDomains`) appear in A2A_CARD only
- **Enrichment mechanics rework** (eidos#56): `JsonExtractionUtil.extractJson()` with retry on non-JSON LLM response; selective override (enriched fields merged, not full replacement); `buildEnrichmentPayload()` separates descriptor payload from LLM input
- **`RenderedPrompt.enriched` flag** (eidos#53): set `true` when Stage 2 ran; signals eval layer to skip substring completeness check which produces false negatives on rich LLM-generated content
- **`AgentDescriptor.briefing` rendering** (eidos#57): rendered in structural fallback MARKDOWN+PROSE; `briefingHash` included in cache key alongside `descriptorHash` and `contextHash`
- **`TEMPLATE_HASH` covers all LLM-influencing inputs** (eidos#50, protocol PP-20260613-608684): schema descriptions included in hash — changing a schema description invalidates the cache without bumping a version constant

Add accountability gaps:
- Substring completeness check produces false negatives on enriched content → `RenderedPrompt.enriched` flag; eval layer skips check when `true`
- LLM returns malformed JSON on first try → `JsonExtractionUtil.extractJson()` retries after stripping markdown fences; `MalformedJudgeResponseException` on second failure
- `qualityHint`/`latencyHintP50Ms` appear in PROSE breaking format contract → format-discriminated capability rendering; protocol PP-20260611-228599

### 3F — L6 Eval Framework: briefing in profiles, ProximityJudge improvements, enriched flag

Add to "What it adds":
- **`briefing` in all 8 profiles** (eidos#59): each `AgentProfile` YAML has `briefing:` populated from `vocabularyGap:FULL` entries; provides richer groundtruth for proximity scoring; protocol PP-20260617-bfc66f
- **ProximityJudge descriptor-axis completeness** (eidos#58): checks that each `DispositionAxis` value in the descriptor appears in the rendered output; `jsonKey()` on `DispositionAxis` drives the check; null-safe for absent disposition
- **Eval judge resilience** (eidos#54): `extractJson` stripping before parse; configurable renders-cache path (`casehub.eval.renders-cache.path`); retry on non-JSON LLM response

Add key wiring note:
- `RenderedPrompt.enriched` flag → `ProximityJudge` skips substring completeness check and defers to semantic scoring; prevents false negatives on LLM-enriched content

### 3G — §13 Glossary: add missing terms

Add:
- `DispositionAxis` — enum: SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE; typed axis key for vocabulary equivalence resolution and disposition axis access; `jsonKey()` returns camelCase JSON key; `description()` returns LLM-readable axis description
- `VocabularyMetadata` — annotation on vocabulary enum classes: `uri` (required), `name`, `version`; read by `VocabularyRegistry.register(Class<T>)`
- `VocabularyRegistrar` — @FunctionalInterface CDI SPI; `@ApplicationScoped` beans auto-register vocabulary enum classes; replaces the removed `Vocabulary` record
- `AgentDescriptor.briefing` — optional freeform text field; authored per agent profile; rendered in MARKDOWN+PROSE structural fallback; included in cache key
- `RenderedPrompt.enriched` — boolean flag set by `EidosSystemPromptRenderer` when Stage 2 semantic enrichment ran successfully; used by `ProximityJudge` to skip substring completeness check that produces false negatives on rich LLM-generated content
- `JsonExtractionUtil` — package-private utility in eval; `extractJson(String)` strips markdown code fences and retries extraction; throws `MalformedJudgeResponseException` on second failure

---

- [ ] **Step 1: Update §1 Description — add briefing to AgentDescriptor bullet**

In `WORKSPACE/ARC42STORIES.MD`, locate:
```
- **AgentDescriptor** — four-layer structured description (identity, slot, capabilities, disposition)
```
Replace with:
```
- **AgentDescriptor** — four-layer structured description (identity, slot, capabilities, disposition); optional `briefing` freeform text field for per-agent narrative context
```

- [ ] **Step 2: Update §1 Description — SystemPromptRenderer bullet: add capability discrimination + enriched**

Locate the SystemPromptRenderer bullet. After the existing text, ensure it contains:
- capability rendering is format-discriminated: PROSE/MARKDOWN surface names + `inputTypes`/`outputTypes` only; numeric routing signals appear in A2A_CARD only — protocol PP-20260611-228599
- `RenderedPrompt.enriched` flag set when Stage 2 enrichment ran

- [ ] **Step 3: Update §1 Key Design Decisions — Generative bullet**

Find the Generative bullet in Key Design Decisions. Add after the existing content:
- `RenderedPrompt.enriched` flag: set `true` when LLM enrichment ran; skips substring completeness check in eval
- `AgentDescriptor.briefing`: rendered in MARKDOWN+PROSE structural fallback; included in cache key as `briefingHash`
- Capability rendering is format-discriminated (PP-20260611-228599): numeric signals in A2A_CARD only

- [ ] **Step 4: Update §5 Building Block View — vocab container**

Locate:
```
Container(vocab, "SVO / CasehubSlot / Conscientiousness", "CDI Vocabulary beans", "Well-known starting-point vocabularies with exactMatches cross-references")
```
Replace with:
```
Container(vocab, "SVO / CasehubSlot / Conscientiousness / Belbin / DISC / Thomas-Kilmann", "VocabularyRegistrar CDI beans", "Well-known starting-point vocabularies; enum constants implementing VocabularyTerm with axis-aware exactMatches")
```

- [ ] **Step 5: Update L1 Zero-Dep API — "What it adds" section**

Add bullets for:
- `AgentDescriptor.briefing` optional freeform field
- `DispositionAxis` enum
- `VocabularyMetadata` annotation
- `VocabularyRegistrar` @FunctionalInterface SPI
- `RenderedPrompt.enriched` flag

Add to key files list:
```
- `api/src/main/java/io/casehub/eidos/api/DispositionAxis.java` — enum: SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE; `jsonKey()` → camelCase JSON key; `description()` → axis description for LLM judge prompts
- `api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java` — annotation: uri (required), name, version on vocabulary enum classes
- `api/src/main/java/io/casehub/eidos/api/spi/VocabularyRegistrar.java` — @FunctionalInterface CDI SPI; @ApplicationScoped beans auto-register vocab enums
```

Update the `VocabularyTerm.java` entry from:
```
`api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java` — interface implemented by vocabulary enum constants; `exactMatch` + `axisExactMatch`
```
(if not already present, add it)

- [ ] **Step 6: Rewrite L2 CDI Discovery — key files, key wiring, architectural decisions, pattern to replicate**

L2 currently uses stale `Vocabulary` / `VocabularyTerm record` / `@Any Instance<Vocabulary>` terminology. Replace:

Key files section — replace entirely:
```
- `runtime/src/main/java/io/casehub/eidos/runtime/vocabulary/CdiVocabularyRegistry.java` — `@DefaultBean @ApplicationScoped`; discovers `@Any Instance<VocabularyRegistrar>`; three internal maps: byUri/byClass/byClassOrdered; `register(Class<T>)` reads `@VocabularyMetadata` annotation; axis-aware `equivalentValues`
- `api/src/main/java/io/casehub/eidos/api/VocabularyRegistry.java` — SPI: `register(Class<T>)`, `isRegistered`, `resolve`, `allTerms`, `equivalentValues` (typed generic + string-based + axis-aware with `DispositionAxis`)
- `api/src/main/java/io/casehub/eidos/api/VocabularyTerm.java` — interface implemented by vocabulary enum constants; `exactMatch` cross-references (same vocab); `axisExactMatch` cross-references (axis-discriminated, for DISC-as-disposition)
- `api/src/main/java/io/casehub/eidos/api/VocabularyMetadata.java` — `@Retention(RUNTIME) @Target(TYPE)` annotation; `uri()` required; `name()`, `version()` default to `""`
- `api/src/main/java/io/casehub/eidos/api/spi/VocabularyRegistrar.java` — `@FunctionalInterface`; `void register(VocabularyRegistry registry)`; `@ApplicationScoped` beans call `registry.register(MyVocabEnum.class)` in their single method
- `vocab/src/main/java/io/casehub/eidos/vocab/SvoTerm.java`, `ConscientiousnessTerm.java`, `CasehubSlotTerm.java`, `BelbinTerm.java`, `DiscTerm.java`, `ThomasKilmannTerm.java` — enum constants implementing `VocabularyTerm`; each paired with a `VocabularyRegistrar` CDI bean
```

Key wiring section — replace `@Any Instance<Vocabulary>` with `@Any Instance<VocabularyRegistrar>`:
```
**`@Any Instance<VocabularyRegistrar>`** discovers all registrar beans. `CdiVocabularyRegistry` iterates at `@PostConstruct` and calls `registrar.register(this)` for each; the registrar calls `registry.register(MyVocabEnum.class)`. Do not call `.get()` lazily; the iterator is stateful.

**`register(Class<T>)` reads `@VocabularyMetadata.uri()`** — absent annotation throws `IllegalArgumentException`. The enum class IS the vocabulary; per-constant URI methods are architecturally impossible.

**`equivalentValues` axis-aware overload** — `equivalentValues(fromVocab, value, toVocabClass, DispositionAxis axis)` resolves DISC `"dominance"` to different Conscientiousness values depending on which disposition axis is being queried. Without the axis parameter, a single DISC type maps ambiguously to multiple Conscientiousness values.
```

Architectural decisions — update:
```
**Why `VocabularyRegistrar` @FunctionalInterface rather than `Vocabulary` record CDI bean:** The enum class IS the vocabulary — all term values are compile-time constants, not runtime instances. A `Vocabulary` record required a CDI bean that existed solely to return a list of `VocabularyTerm` records that mirrored the enum constants. `VocabularyRegistrar` eliminates the mirror: domain apps just write `registry.register(MyVocab.class)` and the enum constants themselves are the terms. See ADR-0004.
```

Pattern to replicate — update step 1 to use VocabularyRegistrar pattern:
```
1. Define a `@VocabularyMetadata`-annotated enum implementing `VocabularyTerm`. Each constant provides `exactMatch()` and optionally `axisExactMatch(DispositionAxis)`.
2. Create a `@ApplicationScoped VocabularyRegistrar` bean that calls `registry.register(MyVocab.class)`.
3. `CdiVocabularyRegistry` discovers all `VocabularyRegistrar` beans and registers them at startup.
4. Consumers call `registry.equivalentValues(fromUri, value, ToVocab.class, axis)` for typed cross-vocab resolution, or the string-based overload for runtime descriptor lookups.
```

- [ ] **Step 7: Update L5 Rendering Pipeline — add post-C6 changes**

In "What it adds" section, add bullets for:
- Format-discriminated capability signals (eidos#49, PP-20260611-228599)
- Enrichment mechanics rework (eidos#56): `JsonExtractionUtil.extractJson()` with retry
- `RenderedPrompt.enriched` flag (eidos#53)
- `AgentDescriptor.briefing` rendering (eidos#57)
- `TEMPLATE_HASH` covers all LLM-influencing inputs (eidos#50, PP-20260613-608684)

In accountability gaps table, add rows:
```
| Substring completeness check gives false negatives on enriched content | A2A card with rich LLM-generated descriptions fails substring check | `RenderedPrompt.enriched` flag; eval skips check when true |
| LLM returns malformed JSON on first try | Judge call fails, losing an eval result | `JsonExtractionUtil.extractJson()` strips markdown fences and retries |
| `qualityHint`/`latencyHintP50Ms` appear in PROSE breaking format contract | PROSE prompt bloated with routing metadata irrelevant to LLM runtime | Format-discriminated rendering; protocol PP-20260611-228599 |
```

Update `AgentDescriptor.briefing` in cache key note under key wiring.

- [ ] **Step 8: Update L6 Eval Framework — add post-C6 changes**

In "What it adds" section, add bullets for:
- Briefing in all 8 profiles (eidos#59): `vocabularyGap:FULL` entries from ProximityJudge; protocol PP-20260617-bfc66f
- ProximityJudge descriptor-axis completeness (eidos#58): per-axis check using `DispositionAxis.jsonKey()`; null-safe for absent disposition
- Eval judge resilience (eidos#54): `extractJson` stripping, configurable renders-cache path

Add key wiring note:
```
**`RenderedPrompt.enriched` skips substring completeness check.** Enriched renders contain LLM-generated narrative that doesn't contain exact capability names — substring matching produces false negatives. When `enriched = true`, `ProximityJudge` skips the completeness substring check and relies on semantic proximity scoring instead.
```

- [ ] **Step 9: Update §13 Glossary — add missing terms**

At the end of the glossary table, add rows for:
- `DispositionAxis` 
- `VocabularyMetadata`
- `VocabularyRegistrar`
- `AgentDescriptor.briefing`
- `RenderedPrompt.enriched`
- `JsonExtractionUtil`

- [ ] **Step 10: Commit updated ARC42STORIES.MD to workspace main**

```bash
git -C /Users/mdproctor/claude/public/casehub/eidos add ARC42STORIES.MD
git -C /Users/mdproctor/claude/public/casehub/eidos commit -m "docs: reconstitute ARC42STORIES.MD — L2 vocab enum redesign, L5 capability discrimination/enrichment/briefing, L6 eval improvements; missing terms in glossary"
```

---

## Task 4: Promote ARC42STORIES.MD to project repo

**Files:**
- Create: `PROJECT/ARC42STORIES.MD`

- [ ] **Step 1: Copy file to project repo**

```bash
cp /Users/mdproctor/claude/public/casehub/eidos/ARC42STORIES.MD \
   /Users/mdproctor/claude/casehub/eidos/ARC42STORIES.MD
```

- [ ] **Step 2: Verify it landed**

```bash
wc -l /Users/mdproctor/claude/casehub/eidos/ARC42STORIES.MD
```

Expected: > 1200 lines.

- [ ] **Step 3: Commit to project repo**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/eidos commit -m "docs: promote ARC42STORIES.MD to project repo — Foundation tier complete through C6, updated for post-C6 changes"
```

---

## Task 5: Verify blog publish status

**No action needed.** All 27 eidos workspace blog entries (2026-05-23 through 2026-06-17) are confirmed present in `mdproctor.github.io/_notes/`. The most recent (`2026-06-17-mdp01-judging-the-judge.md`) was published in the `2380120 chore: publish blog entries from issue-53-allcasescomplete-fix` commit.

Document this as ✓ complete.

---

## Task 6: Push all project repo commits

- [ ] **Step 1: Push project repo changes**

```bash
git -C /Users/mdproctor/claude/casehub/eidos log --oneline origin/main..HEAD
git -C /Users/mdproctor/claude/casehub/eidos push origin main
```

Expected: 3-4 new commits (CLAUDE.md fix, 2 specs, ARC42STORIES.MD).

- [ ] **Step 2: Push workspace repo changes**

```bash
git -C /Users/mdproctor/claude/public/casehub/eidos log --oneline origin/main..HEAD
git -C /Users/mdproctor/claude/public/casehub/eidos push origin main
```

---

## Self-Review

**Spec coverage:**
- CLAUDE.md routing fix ✓ (Task 1)
- 2 missing specs rescued ✓ (Task 2)
- ARC42STORIES.MD reconstitution ✓ (Task 3 — L2, L5, L6, §1, §13)
- ARC42STORIES.MD promoted to project repo ✓ (Task 4)
- Blog publish verified ✓ (Task 5)
- All commits pushed ✓ (Task 6)

**Sections NOT needing update:**
- L3 JPA Registry — no changes since C6
- L4 Capability Health — no changes since C6
- L7 Knowledge Graph — no changes since C6
- §6 Runtime View scenarios — still accurate
- §7 Deployment View — still accurate
- §8 Governing protocols — PP-20260611-228599 should be referenced but is captured in L5
- §9.2 Chapter Index — all chapters still ✅ Complete
- §11 Quality Requirements — still accurate
- §12 Risks — `BlockingToReactiveGraphBridge` worker-thread-per-call risk still open; no new risks to add

**Changes since last published ARC42STORIES.MD commit that are NOT in scope:**
- §9.3 Chapter entries for post-C6 chapters: these are architectural phases, not post-C6 chapters. The 6 chapters are complete and don't need new entries — the post-C6 work is evolution within existing layers.
