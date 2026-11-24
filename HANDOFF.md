# HANDOFF — eidos

## Last Session

Seven issues completed across two domains:

### Docs + Test Hygiene (#258, #259, #260)
- **#258** — ARC42STORIES.MD updated for attestation relevance pipeline (§4, §12 Active Risks, §13 Glossary)
- **#259** — Standardised `@TestTransaction` → `@BeforeEach @Transactional` cleanup in agent state store tests
- **#260** — In-memory signal store tests adopted contract test base classes (−254 lines)

### Cross-Unit Evidence Threshold (#266)
- New `CrossUnitEvidenceAssessor` SPI in `eidos-api` with `EvidenceDiversityLevel` enum and `CrossUnitEvidenceAssessment` record
- `DefaultCrossUnitEvidenceAssessor` in `org-runtime` — reads `org-distance` from attestation snapshots, configurable threshold (default 2)
- `NoOpCrossUnitEvidenceAssessor` `@DefaultBean` via `EidosCoreProducer`
- `OrgDistanceUtil` extracted as shared utility — both `OrgProximityRelevanceFactor` and `IndependenceAwareRelevanceFactor` refactored to use it
- Design spec + decisions promoted to project

### Role Profile Validation (#268, #269, #270)
- **#268** — `RoleDeclaration.roleName` max length (100) + `roleVocabulary` max length (500) via `AgentDescriptorValidator`
- **#269** — `RoleProfile.declaredRoles` max count guard (10) with `AgentValidationException`
- **#270** — JPA persistence: `role_profile TEXT` column in V1 migration, JSON mapping in `AgentDescriptorMapper`

### Code Review Fixes
- Added `@Produces @DefaultBean` for `CrossUnitEvidenceAssessor` in `EidosCoreProducer`
- Added `Objects.requireNonNull` for `level` and `warnings` in `CrossUnitEvidenceAssessment`
- Changed `RoleProfile` max count guard from `IllegalArgumentException` to `AgentValidationException`

### Blog
- "When Confidence Lies" — precision-vs-accuracy distinction, why mass-preserving normalisation makes relevance factors useless for evidence pool quality

### Issues Filed
- **#272** — Missing COMPLIANT TTL expiry test in `InMemoryBehavioralSignalStoreTest` (XS/Low, queued)

## Queue (1 open)

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #272 | add COMPLIANT TTL expiry test | XS | Low |

## Notes for Next Session

- Quarkus augmentation cache can go stale across branches — `mvn clean install -DskipTests` fixes `NoSuchMethodError` on `BriefingCoherenceValidator` startup failures
- `CrossUnitEvidenceAssessor` SPI is ready for engine dispatch integration (engine#1295 or new engine issue)
- #271 (RoleHealth probe) is the next substantial role-profile issue (M/Med) — not yet queued
