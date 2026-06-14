# Req 19 — Per-Research Wiki with Selective Inclusion — Technical Spec

## Architecture

Three pieces: a `WikiEntry` model, a `WikiStore` with swappable SQLite/Markdown
backends, and a `WikiQuery` that ranks by keyword and (optionally) vector similarity.
The Orchestrator (Req 05) reads from the store at run start; pipelines propose entries
subject to the auto-wiki mode.

```
finding (confidence ≥ threshold) → propose WikiEntry
   ├─ auto_wiki = false → await approval → add | reject→discard (unchanged)
   └─ auto_wiki = true  → add without confirmation              (Req 19.3, 19.4)
run start → get_context(topic) → inject ≤ max_context_entries into prompts (Req 19.6)
```

## Models (Req 19.1, 19.2, 19.7)

```python
class WikiEntry(BaseModel):
    id: str; topic: str
    category: Literal["finding", "decision", "source", "rejected", "cross_ref"]
    content: str                                   # ≤ max_entry_size (Req 19.1)
    source: str | None = None
    confidence: float = Field(0.8, ge=0.0, le=1.0)
    tags: list[str] = []; created_at: datetime; session_id: str | None = None

class WikiConfig(BaseModel):
    location: str = "./aria_data/wiki"
    backend: Literal["markdown", "database"] = "markdown"
    auto_wiki: bool = False
    notability_threshold: float = Field(0.8, ge=0.0, le=1.0)
    max_entry_size: int = Field(2000, ge=1)
    max_query_results: int = Field(5, ge=1)
    max_context_entries: int = Field(10, ge=1); max_wiki_size: int = Field(1000, ge=1)
```

`content` is validated against `max_entry_size` at construction; an over-size entry is
rejected before any write (Req 19.1).

## WikiStore (Req 19.1, 19.8, 19.10) — `src/aria/wiki/store.py`

```python
class WikiStore:
    def add_entry(self, entry: WikiEntry) -> str: ...      # Req 19.3/19.4 caller-gated
    def query(self, q: str, top_k: int) -> list[WikiEntry]: ...
    def get_context(self, topic: str, max_entries: int) -> list[WikiEntry]: ...
    def archive_oldest(self, keep: int) -> int: ...        # Req 19.10
```

- **Backends:** SQLite (`wiki_entries` table + **FTS5** for keyword search) or Markdown
  (one `.md` file per entry under `location`).
- **Atomic add (Req 19.8):** writes in a transaction (SQLite) or temp-file + `os.replace`
  (Markdown); on failure raises `WikiWriteError`, the caller returns an error, and the
  Wiki is left unchanged — no partial entry.
- **Archiving (Req 19.10):** after each add, if active count > `max_wiki_size`, the
  oldest entries (by `created_at`) are archived so active size stays within the max.

## Proposal and Inclusion (Req 19.2–19.4)

```python
def maybe_propose(self, finding: Finding) -> WikiEntry | None:
    # propose only when confidence ≥ notability threshold (Req 19.2)
    return self._to_entry(finding) if finding.confidence >= self.cfg.notability_threshold else None

async def include(self, entry: WikiEntry, ui) -> str | None:
    if self.cfg.auto_wiki:                                   # Req 19.4 — no confirmation
        return self.store.add_entry(entry)
    if await ui.confirm_entry(entry):                        # Req 19.3 — explicit approval
        return self.store.add_entry(entry)
    return None                                              # Req 19.3 — discard, unchanged
```

## WikiQuery (Req 19.5, 19.9) — `src/aria/wiki/query.py`

```python
class WikiQuery:
    def search(self, q: str, top_k: int) -> list[WikiEntry]:
        kw = self._keyword_search(q, top_k)                  # SQLite FTS5 / file grep
        if self.vector_store:                                # Req 13 — optional
            return self._merge_rank(kw, self._vector_search(q, top_k))[:top_k]
        return kw[:top_k]                                    # Req 19.5 — ranked, capped
```

- Empty match → returns `[]`; the caller surfaces a **"no matching entries found"**
  indication rather than an error (Req 19.9). Vector search (Req 13) is optional; when
  absent, keyword ranking alone is used.

## Orchestrator Integration (Req 19.6)

```python
# at run start, before dispatching agent tasks
entries = wiki.query.get_context(topic=run.topic, max_entries=cfg.max_context_entries)
prompt_context = render_wiki_context(entries)                # injected into agent prompts
```

Capped at `max_context_entries` (default 10) so known knowledge is reused without
bloating prompts.

## Files & Dependencies

| File                      | Purpose                                          |
|---------------------------|--------------------------------------------------|
| `src/aria/wiki/models.py` | `WikiEntry`, `WikiConfig`                         |
| `src/aria/wiki/store.py`  | `WikiStore` (SQLite FTS5 + Markdown backends)     |
| `src/aria/wiki/query.py`  | `WikiQuery` (keyword + optional vector ranking)   |

- **Req 07 (Configuration)** — all wiki settings and their bounds validation.
- **Req 13 (Data Layer)** — optional Vector_Store for semantic ranking; **Req 05
  (Orchestrator)** — context injection at run start.
