# eidos Session Handover — 2026-06-19

## Last Session

Implemented eidos#55: capability sub-specialization metadata. `AgentCapability` now has `excludedDomains: Set<String>` for declared negative specialization and a `Builder` inner class. `CapabilitySpecializationStore` SPI accumulates learned DECLINE-based exclusions with store-owned 30-day TTL. `CapabilityStatus` gained an `Excluded(domain, ExclusionSource, declineCount)` sealed variant. `DefaultCapabilityHealth.probe()` checks declared then learned exclusions (using `Instance<PreferenceProvider>` for per-tenancy threshold). `InMemoryCapabilitySpecializationStore @Alternative @Priority(1)` in persistence-memory. A2A_CARD renderer includes `excludedDomains` in capability JSON. Schema, entity, and mapper updated. Branch squashed 14→7 commits, pushed to casehubio/eidos main.

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
- **eidos#63** — `AgentDescriptorMapper.toCapability()` must revert to positional constructor (protocol violation found in sweep) · XS · Low
- **parent#281** — PLATFORM.md capability ownership sync (CapabilitySpecializationStore) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Re-run `evaluateRealWorldScenarios` with briefings | XS | Low | Zombie fix now in main; compare vs pre-briefing baseline |
| — | `evaluateWithIndependentJudge` (Qwen) after | XS | Low | Check if briefing lifts FACTUAL_FIDELITY |
| #63 | Revert AgentDescriptorMapper to positional constructor | XS | Low | Protocol violation found in sweep; immediate fix |
| #28 | casehub-engine: Belbin-based agent composition | L | High | casehubio/eidos repo |

## References

- Blog entry: `blog/2026-06-19-mdp01-teaching-an-agent-what-it-cannot-do.md`
- Spec: `docs/specs/2026-06-18-capability-specialization-design.md` (eidos project repo)
- Renders cache: `/tmp/eidos-renders-cache.json` (survives `mvn clean`)
- Protocols added: PP-20260619-409a36 (optional-platform-spi-instance-injection), PP-20260619-eee0fd (store-owned-ttl-vs-spi-ttl)
- Garden revision: GE-20260601-fcf0d9 — added Instance<T> isUnsatisfied() alternative
