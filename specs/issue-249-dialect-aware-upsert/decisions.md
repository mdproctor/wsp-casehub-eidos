## D1: Dialect-aware upsert approach

**Choice:** Runtime dialect detection — same JPA store class detects Hibernate dialect at `@PostConstruct`, selects the appropriate SQL string for upsert operations.
**Alternatives:**
- `@IfBuildProperty` split — 4 new classes (2 per store × 2 dialects) for a one-line SQL difference; over-engineers a simple dialect toggle
- Hibernate `@SQLInsert` — tight coupling to Hibernate internals, doesn't match existing native-query pattern
**Rationale:** The difference is purely the SQL string constant, not behaviour. Runtime dialect detection via `em.getEntityManagerFactory().getProperties()` is standard Quarkus/Hibernate. No new config properties, no class proliferation.
**Trade-offs:** Cannot use a dialect-specific store in a build-time-only CDI context (not relevant — these stores are runtime beans).
**Sources:** `JpaDispositionSignalStore.java:24-38` (current MERGE), `JpaBehavioralSignalStore.java:40-54` (current find-then-persist), issue #249 analysis
**Exploration:** quick
**Status:** captured
