# eidos Session Handover — 2026-06-23

## Last Session

Implemented eidos#64: declarative agent persona registration. `AgentDescriptorRegistrar` SPI (returns `List<AgentDescriptor>`), `AgentDescriptorBootstrap` (`@Observes StartupEvent`, `@IfBuildProperty` gated, duplicate `(agentId, tenancyId)` detection), `ClasspathYamlDescriptorRegistrar` (reads `META-INF/eidos/descriptors.yaml` from all JARs via `getResources()`). `EidosSystemPromptRenderer` changed from `@DefaultBean` to plain `@ApplicationScoped` (Pattern B). MAX_BRIEFING 500→2000. "collaborative" alias on `ThomasKilmannTerm.COLLABORATING`. 4 DraftHouse reviewer YAML personas with TK + Conscientiousness vocabulary wiring. Branch squashed to 1 commit, pushed to casehubio/eidos main.

## Immediate Next Step

Re-run `evaluateRealWorldScenarios` with briefings — eidos#52 zombie fix is in main; briefings are in all 8 eval profiles. Compare proximity scores against baseline:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Left

- **eidos#61** — `AgentQuery.taskDomain` pre-filter at registry level · M · Med
- **eidos#62** — JPA-backed `CapabilitySpecializationStore` for production persistence · S · Low
- **eidos#63** — `AgentDescriptorMapper.toCapability()` must revert to positional constructor · XS · Low
- **eidos#65** — `CdiVocabularyRegistry` `@DefaultBean` → `@ApplicationScoped` · XS · Low
- **eidos#66** — ARC42STORIES stale `VocabularyRegistrar` descriptions (lines 709, 714, 724) · XS · Low
- **eidos#67** — Reactive-mode `AgentDescriptorBootstrap` · S · Med
- **eidos#68** — Allow newlines in `AgentDescriptor.briefing` field · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #65 | CdiVocabularyRegistry `@DefaultBean` fix | XS | Low | Same pattern as renderer fix |
| #63 | Revert AgentDescriptorMapper to positional constructor | XS | Low | Protocol violation |
| #66 | ARC42STORIES stale VocabularyRegistrar descriptions | XS | Low | Lines 709, 714, 724 |
| #68 | Allow newlines in briefing field | S | Med | Validator design — per-field isBanned |
| #67 | Reactive-mode bootstrap | S | Med | Needs ReactiveAgentRegistry.register() |
| #28 | casehub-engine: Belbin-based agent composition | L | High | Separate repo |

## References

- Blog entry: `blog/2026-06-23-mdp01-personas-from-yaml.md`
- Spec: `docs/specs/issue-64-drafthouse-reviewer-descriptors/2026-06-23-drafthouse-reviewer-descriptors-design.md`
- Garden: GE-20260623-3ecb0f — isBanned() bans newlines in briefing; YAML `|` fails, use `>-`
- Follow-up issues: eidos#65, #66, #67, #68
