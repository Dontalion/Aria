# Tasks — Req 11: Data Layer Architecture

## Dependencies

- Req 5 (Configuration) — declares active stores, cache TTLs, similarity
  threshold, and primary database backend/location

## Tasks

- [ ] 1. Add the `data` config section
  - Primary backend (sqlite/postgresql) + path, cache backend + TTLs, vector
    `enabled`/threshold, graph `enabled`/backend
  - Validate cache TTLs within 60s–30d; defaults 24h search / 7d LLM
  - _Requirements: 13.5, 13.6_

- [ ] 2. Implement primary database init-in-code
  - Create `src/aria/state/database.py`; create and initialize the schema in code
    (never via the LLM) on first run, before processing tasks
  - _Requirements: 13.1, 13.7_

- [ ] 3. Implement database state inspection
  - Classify the database as NEW, CURRENT, OUTDATED, or CORRUPTED
  - _Requirements: 13.7, 13.8, 13.11_

- [ ] 4. Implement backup-then-migrate for out-of-date schema
  - Take a backup of the existing database first, then apply Alembic migrations to
    that same database, preserving prior data (no delete-recreate)
  - _Requirements: 13.8_

- [ ] 5. Implement migration-failure handling
  - On migration failure stop before any task, leave the existing database
    unchanged for restore, and report the failure reason
  - _Requirements: 13.10_

- [ ] 6. Implement corrupted/unexpected-state handling
  - When neither a clean first run nor a known out-of-date schema, stop and emit a
    clear "requires manual intervention" error; never self-repair or recreate
  - _Requirements: 13.11_

- [ ] 7. Implement the Cache Layer with exact keys and TTL
  - Create `src/aria/state/cache.py`; key LLM by exact prompt+model, search by
    exact query; return an entry only when age ≤ TTL
  - _Requirements: 13.2, 13.6_

- [ ] 8. Implement miss/expired semantics
  - Treat a missing or expired entry as a miss: let the call proceed and store the
    fresh result with a new TTL
  - _Requirements: 13.9_

- [ ] 9. Add Redis cache backend
  - Optional Redis adapter using native key TTL, selectable via config
  - _Requirements: 13.2_

- [ ] 10. Implement the optional Vector Store
  - Create `src/aria/data/vector.py` (ChromaDB); treat items as related only when
    similarity ≥ configurable threshold; support dedup, related-ideas, clustering
  - _Requirements: 13.3_

- [ ] 11. Implement the optional Graph Store
  - Create `src/aria/data/graph.py` (NetworkX local, Neo4j adapter); relationships,
    dependency trees, connection paths
  - _Requirements: 13.4_

- [ ] 12. Implement optional-store graceful degradation
  - When an enabled optional store is unavailable at runtime, continue processing
    tasks without its features and report which store is unavailable
  - _Requirements: 13.12_

- [ ] 13. Write unit tests
  - Cache: hit within TTL, miss when absent, miss when expired, store-fresh-on-miss,
    exact prompt+model vs exact query keys, TTL bounds 60s/30d
  - Database: NEW→create, OUTDATED→backup+migrate preserves data, migration failure
    leaves db unchanged, CORRUPTED→manual-intervention error, no recreate
  - _Requirements: 13.2, 13.6, 13.7, 13.8, 13.9, 13.10, 13.11_

- [ ] 14. Write integration tests for optional stores
  - Vector relatedness honors the threshold; enabled-but-unavailable store lets the
    run continue and reports the unavailable store
  - _Requirements: 13.3, 13.4, 13.12_
