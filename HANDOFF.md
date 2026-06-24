# eidos Session Handover — 2026-06-24

## Last Session

Cleared 7 issues in one pass: 3 XS (#65 CDI scope, #63 positional constructor, #66 ARC42 stale descriptions), 3 S (#68 briefing newlines, #67 reactive bootstrap, #62 JPA specialization store), 1 M (#61 AgentQuery.taskDomain pre-filter). All pushed to casehubio/eidos main. Key design decisions: `allowedCodePoints` vararg on validator for per-field control char exemptions; `excludedDomains` denormalized from JSON TEXT to `@ElementCollection` join table for clean SQL filtering via `NOT MEMBER OF`.

## Immediate Next Step

Re-run `evaluateRealWorldScenarios` with briefings — the eval baseline command from last session is still valid, and the schema change (V1 modified, V4 added) requires `mvn clean`:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl eval -Peval \
  -Dtest=PromptEvalTest#evaluateRealWorldScenarios \
  -Dcasehub.eval.renders-cache.path=/tmp/eidos-renders-cache.json \
  -f /Users/mdproctor/claude/casehub/eidos/pom.xml
```

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #60 | Optional casehub-eidos-desiredstate bridge module | M | Med | NodeDriftChecker for agents |
| #28 | casehub-engine: Belbin-based agent composition | L | High | Separate repo |

## References

- Blog entry: `blog/2026-06-24-mdp01-seven-fixes-schema-decision.md`
