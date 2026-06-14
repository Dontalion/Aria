# Req 13 — Data Layer Architecture — Design

## Purpose

ARIA stores different kinds of data in the storage technology best suited to it.
A relational database tracks the work itself; a cache avoids paying for the same
LLM call or search twice; and two optional stores add semantic and relational
intelligence when a project needs them. The guiding rule is safety with the
primary database: ARIA never lets the LLM touch storage, never recreates a
database behind the user's back, and never loses prior data during an upgrade.

## The Stores

### Required

1. **Primary relational database (SQLite, upgradeable to PostgreSQL)**
   - Holds task state, task dependencies, retry history, run metadata, and
     progress tracking — the system's source of truth.
   - SQLite is the default; PostgreSQL is the upgrade path for larger workloads.

2. **Cache Layer (in-memory, with optional Redis)**
   - Avoids redundant API calls by remembering recent LLM responses and search
     results.
   - In-memory by default; Redis adds persistence across restarts.

### Optional

3. **Vector Store (ChromaDB or similar)**
   - Enables semantic deduplication, finding related ideas across sections, and
     clustering keywords. Two items count as related only when their similarity
     score meets or exceeds a configurable threshold.

4. **Graph Database (Neo4j, or NetworkX for local)**
   - Maps relationships between keywords, ideas, topics, and sources; visualizes
     dependency trees; and finds connection paths between disparate topics.

The config declares which stores are active. Only the relational database and the
cache are required; the vector and graph stores are opt-in extensions.

## Caching Behavior

The cache is keyed precisely so it never returns the wrong answer:

- **LLM responses** are keyed by the **exact prompt text plus the model identifier**.
- **Search results** are keyed by the **exact query string**.

Each entry carries a **configurable TTL** between **60 seconds and 30 days**. The
defaults are **24 hours for search results** and **7 days for LLM responses**.

When ARIA looks something up and finds **no matching entry, or a matching entry
whose age exceeds its TTL**, it treats that as a **cache miss**: it lets the real
API call or search proceed, then stores the fresh result with a new TTL. Expired
entries behave exactly like entries that were never there.

## Primary Database: Initialization and Migration (Key Behavior)

This is the most safety-sensitive part of the data layer.

- **First run, no database yet** — ARIA **creates and initializes the database in
  code** (never via the LLM) before processing any tasks.
- **Existing database, out-of-date schema** — ARIA **takes a backup first**, then
  **applies migrations to that same existing database**, preserving prior data. It
  never deletes and recreates the database to upgrade it.
- **Migration fails** — ARIA **stops before processing any tasks**, leaves the
  existing database **unchanged** so it can be restored from the backup, and reports
  the failure reason.
- **Database present but corrupted or in an unexpected state** — neither a clean
  first run nor a recognized out-of-date schema — ARIA **stops and reports a clear
  "requires manual intervention" error**. It does **not** attempt automatic repair
  or recreation. Protecting existing data outweighs convenience here.

## Optional Stores: Graceful Degradation

If an enabled optional store (vector or graph) is **unavailable during a run**, ARIA
does not stop. It **continues processing tasks without that store's features** and
**reports which store is unavailable**. The run still completes; only the optional
capability is skipped.

## Configuration

```yaml
data:
  primary:
    backend: sqlite            # sqlite | postgresql
    path: ./aria_data/aria.db
  cache:
    backend: memory            # memory | redis
    ttl_search: 86400          # 24h default (60s–30d)
    ttl_llm: 604800            # 7d default (60s–30d)
  vector:
    enabled: false
    similarity_threshold: 0.83
  graph:
    enabled: false
    backend: networkx          # networkx | neo4j
```

## What Users Don't Need to Know

- How migrations are sequenced or how backups are named.
- How cache keys are hashed or how TTL expiry is checked.
- How the vector/graph adapters wrap their underlying libraries.

## Dependencies

- Req 07 (Configuration) — declares which stores are active, the cache TTLs, the
  similarity threshold, and the primary database location.
