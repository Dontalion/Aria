# Technical Specification: Keyword Enrichment Pipeline

## File Location

`src/aria/pipelines/keyword_enrichment.py`

## Class Hierarchy

```
BasePipeline (ABC)        # src/aria/pipelines/base.py
└── KeywordPipeline(BasePipeline)
```

`KeywordPipeline` inherits from `BasePipeline` and implements the keyword
enrichment flow as a LangGraph subgraph. The base class supplies
`build_graph()`, `run()`, and `get_name()` contracts shared with other
pipelines.

## LangGraph Subgraph

```
read_keywords → generate_search_terms → search_parallel → classify_results
    → [conditional: retry if >50% rejected] → generate_keywords → write_batch
```

### Node Responsibilities

| Node | Responsibility |
|------|----------------|
| `read_keywords` | Parse source file; collect keywords under the target Section. If absent/empty → log error, terminate Section, no output (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`), keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)) |
| `generate_search_terms` | LLM call: produce exactly 3 distinct `SearchTerm` per keyword (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)) |
| `search_parallel` | Fan out 3 concurrent `SearchProvider` calls; await all 3 (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)) |
| `classify_results` | LLM call: label each result `accepted`/`rejected` vs research topic, with reason (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)) |
| `retry` (conditional edge) | If `rejected/total > 0.5`, add 1 replacement term, re-search + re-classify; max 1 retry (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)) |
| `generate_keywords` | LLM call: produce exactly 10 new keywords from ≥1 accepted result (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)) |
| `write_batch` | Append complete batches of ≤20 keywords via `fs_append` (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`), keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)) |

The zero-accepted guard (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)) is a conditional edge after `retry`: if no
accepted results remain, skip `generate_keywords`/`write_batch` for that keyword,
log the reason, advance to the next keyword.

## Pydantic Models

```python
class KeywordInput(BaseModel):
    keyword: str
    section: str

class SearchTerm(BaseModel):
    term: str
    source_keyword: str

class ClassifiedResult(BaseModel):
    url: str
    title: str
    snippet: str
    classification: Literal["accepted", "rejected"]
    reason: str

class GeneratedKeyword(BaseModel):
    keyword: str            # distinct from source + prior generated (keyword-enrichment workflow (see `workflows/keyword-enrichment.md`))
    source_keyword: str
    section: str
```

## State Keys

```python
class KeywordPipelineState(TypedDict):
    section: str
    current_keyword: str
    search_terms: list[SearchTerm]        # exactly 3, +1 on retry
    classified: list[ClassifiedResult]
    retry_count: int                      # max 1
    generated_keywords: list[GeneratedKeyword]   # exactly 10 when produced
```

## Provider Dependencies

| Provider | Used by |
|----------|---------|
| `LLMProvider` (Req 1) | `generate_search_terms`, `classify_results`, `generate_keywords` |
| `SearchProvider` (Req 2) | `search_parallel`, `retry` |

## Retry Logic

- Trigger condition: `rejected_count / total_count > 0.5`
- Generates exactly 1 replacement search term
- Runs one additional search + classify cycle for that term only
- Maximum 1 retry per source keyword (no recursion); `retry_count` capped at 1

## Batch Writing Rules

- Maximum 20 keywords per `fs_append` call (10 generated → 1 write)
- Output file is separate from the read-only source file
- Each batch maintains valid Section markdown structure
- Atomic at the batch level — partial batches are never written
- Distinctness enforced before write: new keyword ≠ any source keyword and ≠ any
  keyword already generated within the same Section
