# Technical Specification: Idea Generation Pipeline

## File Location

`src/aria/pipelines/idea_generation.py`

## Class Hierarchy

```python
class IdeaPipeline(BasePipeline):
    """Generates structured research ideas from keywords via web search."""
```

Shares the `BasePipeline` ABC (`src/aria/pipelines/base.py`) with the Keyword
pipeline, reusing the `build_graph()` / `run()` / `get_name()` contract.

## LangGraph Subgraph

```
read_keywords → generate_search_terms → search_parallel → classify
    → [conditional: retry if >50% rejected] → generate_ideas → deduplicate → write_batch
```

### Node Responsibilities

| Node | Responsibility |
|------|----------------|
| `read_keywords` | Parse keyword file, group keywords by Sub_Section. If missing/empty/unparsable → log error, terminate, no output (Req 4.1, 4.2) |
| `generate_search_terms` | LLM call: keyword → exactly 3 search terms (Req 4.3) |
| `search_parallel` | Fan out 3 concurrent `SearchProvider` calls (Req 4.4) |
| `classify` | LLM call: label each result accepted/rejected by relevance to the topic (Req 4.5) |
| `retry` (conditional edge) | If >50% rejected, regenerate 1 term and re-search; max 1 retry per keyword (Req 4.6) |
| `generate_ideas` | LLM structured-output call: accepted results → exactly 10 `IdeaRecord` conforming to the configured schema (Req 4.8, 4.9) |
| `deduplicate` | Drop ideas whose title already exists in the output file, case-insensitive (Req 4.11) |
| `write_batch` | Append ≤5 records per write to the Sub_Section output file (Req 4.10) |

The zero-accepted guard (Req 4.7) is a conditional edge after `retry`: with no
accepted results, skip `generate_ideas`, log the reason, advance to the next
keyword.

## Pydantic Models

```python
class IdeaRecord(BaseModel):
    """Default schema; field set is configurable at pipeline init (Req 4.9, Req 12)."""
    title: str
    description: str
    resume_worthiness: str   # must name at least one specific technology

class IdeaBatch(BaseModel):
    """A single write unit — max 5 records."""
    sub_section: str
    ideas: list[IdeaRecord] = Field(max_length=5)
```

### Configurable Schema

`IdeaRecord` fields are defined at pipeline init via a schema dict and built with
`create_model()`. Default:

```python
DEFAULT_IDEA_SCHEMA = {
    "title": {"type": "str", "required": True},
    "description": {"type": "str", "required": True},
    "resume_worthiness": {"type": "str", "required": True},
}
```

## State Keys

```python
class IdeaPipelineState(TypedDict):
    current_keyword: str
    sub_section: str
    search_terms: list[str]              # exactly 3, +1 on retry
    search_results: list[dict]
    classifications: list[dict]
    retry_count: int                     # max 1
    generated_ideas: list[IdeaRecord]    # exactly 10 when produced
    deduplicated_ideas: list[IdeaRecord]
```

## Provider Dependencies

| Provider | Used by |
|----------|---------|
| `LLMProvider` (Req 01) | `generate_search_terms`, `classify`, `generate_ideas` |
| `SearchProvider` (Req 02) | `search_parallel`, `retry` |

## Retry Logic

- Trigger condition: `rejected_count / total_count > 0.5`
- Regenerates exactly 1 search term and re-executes the searches
- Maximum 1 retry per keyword; `retry_count` capped at 1 (no recursion)

## Deduplication

- Read existing titles from the target output file at start of `deduplicate`
- Case-insensitive comparison; drop any idea whose title already exists

## Finalization Gates

Reviewer ACCEPT (Req 14.8) is a quality gate only. An idea is finalized — i.e.
written — only when it ALSO passes the structural gates: `deduplicate` (unique
title, Req 4.11) and schema conformance to the configured `IdeaRecord` (Req 12).
A Reviewer-accepted idea that fails either structural gate is not written.

## Batch Writing Rules

- Maximum 5 ideas per append (10 generated → up to 2 writes); file lock per write
