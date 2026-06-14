# Req 19 — Per-Research Wiki with Selective Inclusion — Tasks

## Dependencies

- Req 07 (Configuration) — wiki settings (location, backend, auto-wiki, thresholds, limits)
- Req 13 (Data Layer) — optional Vector_Store for semantic relevance ranking
- Req 05 (Orchestrator) — context injection of relevant entries at run start

## Tasks

- [ ] 19.1 Define `WikiEntry` and `WikiConfig` Pydantic models in `src/aria/wiki/models.py` with categories (findings, decisions, sources, rejected approaches, cross-references) _(Req 19.1)_
- [ ] 19.2 Enforce per-entry size cap (default 2000 chars): reject over-size content before any write _(Req 19.1)_
- [ ] 19.3 Implement `WikiStore` SQLite backend with `wiki_entries` table and FTS5 virtual table _(Req 19.1, 19.5)_
- [ ] 19.4 Implement `WikiStore` Markdown backend (one file per entry under the configured location) _(Req 19.1, 19.7)_
- [ ] 19.5 Implement notability-based proposal: propose an entry only when finding confidence >= notability threshold (default 0.8) _(Req 19.2)_
- [ ] 19.6 Implement selective inclusion with auto-wiki disabled: add only after explicit user approval; on rejection discard and leave wiki unchanged _(Req 19.3)_
- [ ] 19.7 Implement auto-wiki enabled path: add proposed entries without confirmation _(Req 19.4)_
- [ ] 19.8 Implement atomic `add_entry` write: on failure return an error and leave existing wiki unchanged (no partial write) _(Req 19.8)_
- [ ] 19.9 Implement `WikiQuery.search()` keyword ranking, capped at max query results (default 5) _(Req 19.5)_
- [ ] 19.10 Implement optional vector search via Req 13 Vector_Store and merge/rank with keyword results _(Req 19.5)_
- [ ] 19.11 Handle empty-match query: return an empty set with a "no matching entries found" indication _(Req 19.9)_
- [ ] 19.12 Implement `get_context()` and wire Orchestrator run-start injection capped at max context entries (default 10) _(Req 19.6)_
- [ ] 19.13 Implement archiving: when active entries reach max wiki size, archive the oldest so active size stays within the maximum _(Req 19.10)_
- [ ] 19.14 Add the `wiki` config section with all settings and bounds validation _(Req 19.7)_
- [ ] 19.15 Write unit tests for store backends, size cap, proposal threshold, inclusion modes, empty-match, and archiving _(Req 19.1–19.5, 19.8–19.10)_
- [ ] 19.16 Write integration test for the full wiki lifecycle: propose → approve/auto-add → query → context injection → archive _(Req 19.2–19.6, 19.10)_
