# Req 11 — Data Layer Architecture — Technical Spec

## Stack

| Concern    | Choice                                   | Required? |
|------------|------------------------------------------|-----------|
| Relational | SQLAlchemy (async) + Alembic migrations  | yes       |
| Cache      | in-memory dict + optional `redis` client | yes       |
| Vector     | ChromaDB                                 | optional  |
| Graph      | NetworkX (local) / Neo4j driver          | optional  |

```
src/aria/state/database.py    # init-in-code, migration + backup, integrity check
src/aria/state/cache.py       # CacheLayer (memory + redis), exact-key + TTL
src/aria/data/vector.py       # VectorStore (similarity threshold)
src/aria/data/graph.py        # GraphStore (networkx / neo4j)
```

## Primary Database (Req 11.1, 11.7, 11.8, 11.10, 11.11)

```python
def ensure_database(cfg) -> None:
    state = inspect_database(cfg)                # NEW | CURRENT | OUTDATED | CORRUPTED
    if state is DBState.NEW:                      # Req 11.7 — create in code (never LLM)
        create_schema_in_code(cfg)
    elif state is DBState.OUTDATED:               # Req 11.8 — backup FIRST, migrate same db
        backup = backup_database(cfg)
        try:
            run_migrations(cfg)                   # preserve data, no delete-recreate
        except MigrationError as e:               # Req 11.10 — leave db unchanged, report
            raise FatalDataError(f"migration failed: {e}; db unchanged, backup at {backup}")
    elif state is DBState.CORRUPTED:              # Req 11.11 — stop, no self-repair/recreate
        raise FatalDataError("database in unexpected state — requires manual intervention")
    # CURRENT → proceed
```

`inspect_database`: absent → NEW; known prior revision → OUTDATED; head revision →
CURRENT; unreadable / unknown revision / integrity failure → CORRUPTED. `FatalDataError`
halts before any task dispatch.

## Cache Layer (Req 11.2, 11.6, 11.9)
```python
@dataclass
class CacheEntry:
    value: Any; created_at: float; ttl: int      # 60 .. 2_592_000 (30d)

class CacheLayer:
    @staticmethod
    def llm_key(prompt, model):                   # exact prompt + model id
        return "llm:" + sha256(f"{model}\x00{prompt}".encode()).hexdigest()
    @staticmethod
    def search_key(query):                        # exact query string
        return "search:" + sha256(query.encode()).hexdigest()

    def get(self, key):
        e = self._store.get(key)
        if e is None or self._age(e) > e.ttl:     # Req 11.9 — miss OR expired → miss
            return None
        return e.value

    def set(self, key, value, ttl):
        assert 60 <= ttl <= 2_592_000             # Req 11.6 bounds (default 24h search/7d LLM)
        self._store[key] = CacheEntry(value, now(), ttl)
```

A miss or expired entry returns `None`; the caller proceeds with the real call and
stores the fresh result with a new TTL (Req 11.9). Redis backend uses native key TTL.

## Optional Stores (Req 11.3, 11.4, 11.5, 11.12)

```python
class VectorStore:                                # ChromaDB
    def related(self, text, top_k=5):
        return [r for r in self._query(text, top_k) if r.score >= self.threshold]  # Req 11.3
    def is_duplicate(self, text):                 # dedup, related-ideas, clustering
        return any(self.related(text, top_k=1))

class GraphStore:                                 # NetworkX local / Neo4j
    def add_edge(self, src, dst, relation): ...   # relationships, dep trees
    def path(self, src, dst): ...                 # connection paths

def acquire_optional(store, name):
    if not store.enabled:
        return None
    try:
        return store.connect()
    except StoreUnavailable as e:                 # Req 11.12 — do NOT stop the run
        logger.error("optional store %s unavailable: %s; continuing", name, e)
        return None
```

The orchestrator checks `enabled` and the acquired handle before using any optional
store; unavailability degrades to skipped features, reported but non-fatal.

## Validation Summary

| Behavior                   | Rule                                          | Criteria  |
|----------------------------|-----------------------------------------------|-----------|
| Init in code               | create schema in code on first run            | 13.7      |
| Out-of-date schema         | backup first, migrate same db, preserve data  | 13.8      |
| Migration failure          | stop, leave db unchanged, report              | 13.10     |
| Corrupted/unexpected state | stop, "manual intervention", no recreate      | 13.11     |
| Cache key + TTL            | exact prompt+model / exact query; 60s–30d     | 13.2,13.6 |
| Miss or expired            | proceed + store fresh                         | 13.9      |
| Optional store unavailable | continue, report which                        | 13.12     |

## Dependencies

- Req 5 (Configuration) — active stores, cache TTLs, similarity threshold, and
  primary database backend/location.
