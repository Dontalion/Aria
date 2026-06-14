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
| `read_keywords` | Parse source file; collect keywords under the target Sub_Section. If absent/empty → log error, terminate Sub_Section, no output (Req 3.1, 3.2) |
| `generate_search_terms` | LLM call: produce exactly 3 distinct `SearchTerm` per keyword (Req 3.3) |
| `search_parallel` | Fan out 3 concurrent `SearchProvider` calls; await all 3 (Req 3.4) |
| `classify_results` | LLM call: label each result `accepted`/`rejected` vs research topic, with reason (Req 3.5) |
| `retry` (conditional edge) | If `rejected/total > 0.5`, add 1 replacement term, re-search + re-classify; max 1 retry (Req 3.6) |
| `generate_keywords` | LLM call: produce exactly 10 new keywords from ≥1 accepted result (Req 3.8) |
| `write_batch` | Append complete batches of ≤20 keywords via `fs_append` (Req 3.9, 3.10) |

The zero-accepted guard (Req 3.7) is a conditional edge after `retry`: if no
accepted results remain, skip `generate_keywords`/`write_batch` for that keyword,
log the reason, advance to the next keyword.

## Pydantic Models

```python
class KeywordInput(BaseModel):
    keyword: str
    sub_section: str

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
    keyword: str            # distinct from source + prior generated (Req 3.8)
    source_keyword: str
    sub_section: str
```

## State Keys

```python
class KeywordPipelineState(TypedDict):
    sub_section: str
    current_keyword: str
    search_terms: list[SearchTerm]        # exactly 3, +1 on retry
    classified: list[ClassifiedResult]
    retry_count: int                      # max 1
    generated_keywords: list[GeneratedKeyword]   # exactly 10 when produced
```

## Provider Dependencies

| Provider | Used by |
|----------|---------|
| `LLMProvider` (Req 01) | `generate_search_terms`, `classify_results`, `generate_keywords` |
| `SearchProvider` (Req 02) | `search_parallel`, `retry` |

## Retry Logic

- Trigger condition: `rejected_count / total_count > 0.5`
- Generates exactly 1 replacement search term
- Runs one additional search + classify cycle for that term only
- Maximum 1 retry per source keyword (no recursion); `retry_count` capped at 1

## Batch Writing Rules

- Maximum 20 keywords per `fs_append` call (10 generated → 1 write)
- Output file is separate from the read-only source file
- Each batch maintains valid Sub_Section markdown structure
- Atomic at the batch level — partial batches are never written
- Distinctness enforced before write: new keyword ≠ any source keyword and ≠ any
  keyword already generated within the same Sub_Section
