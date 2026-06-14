# Search Provider Abstraction — Technical Specification

## Data Model

```python
from pydantic import BaseModel, Field

class SearchResult(BaseModel):
    title: str = Field(min_length=1)   # non-empty (Req 2.1)
    url: str
    snippet: str
```

## Abstract Base Class

```python
# src/aria/providers/base.py
from abc import ABC, abstractmethod

class SearchProvider(ABC):
    @abstractmethod
    async def search(self, query: str, max_results: int = 5) -> list[SearchResult]:
        """Return 0..max_results normalized results, each with a non-empty title."""  # Req 2.1
        ...
```

## Concrete Implementations

| Class (`file`) | Library | API key | Mapping | Req |
|----------------|---------|---------|---------|-----|
| `DuckDuckGoProvider` (`duckduckgo.py`) | `duckduckgo-search` (`AsyncDDGS`) | none | `AsyncDDGS.text()` → `SearchResult` | 2.2 |
| `TavilyProvider` (`tavily.py`) | `tavily-python` | required | `TavilyClient.search()` → `SearchResult` | 2.3 |
| `BraveSearchProvider` (`brave.py`) | `httpx` → `GET api.search.brave.com/res/v1/web/search` | required | JSON `web.results[]` → `SearchResult` | 2.4 |

## Configuration Model (ref: req-07-config)

Pydantic validates the provider and required credentials at load time (Req 2.7–2.9):

```python
from pydantic import BaseModel, Field, model_validator
from typing import Literal

class SearchConfig(BaseModel):
    provider: Literal["duckduckgo", "tavily", "brave"]   # Req 2.7, 2.8
    api_key: str | None = None
    max_results: int = Field(5, ge=1)

    @model_validator(mode="after")
    def _require_key(self) -> "SearchConfig":
        if self.provider in ("tavily", "brave") and not self.api_key:
            raise ValueError(f"{self.provider} requires an api_key credential")  # Req 2.8, 2.9
        return self
```

An unsupported provider (caught by the `Literal`) or a missing required key raises
a `ValidationError`; the loader logs the offending field and exits before any
search (Req 2.9).

## Factory

```python
# src/aria/providers/search/__init__.py
from aria.config.schema import SearchConfig

def create_search_provider(config: SearchConfig) -> SearchProvider:
    match config.provider:
        case "duckduckgo": return DuckDuckGoProvider()
        case "tavily":     return TavilyProvider(api_key=config.api_key)
        case "brave":      return BraveSearchProvider(api_key=config.api_key)
        case _:            raise ValueError(f"Unknown search provider: {config.provider}")
```

## Retry Logic

Each provider wraps its network call with one delayed retry (Req 2.5, 2.6):

```python
import asyncio

async def _search_with_retry(self, query: str, max_results: int) -> list[SearchResult]:
    try:
        return await self._execute(query, max_results)
    except Exception:
        await asyncio.sleep(3)
        try:
            return await self._execute(query, max_results)  # success returns its results
        except Exception:
            return []                                        # empty 0-result set
```

One retry, 3-second delay, results on retry success, empty list on final failure.

## File Structure

```
src/aria/providers/search/
├── __init__.py       # Exports SearchProvider, SearchResult, create_search_provider
├── base.py           # ABC + SearchResult model
├── duckduckgo.py     # DuckDuckGoProvider
├── tavily.py         # TavilyProvider
└── brave.py          # BraveSearchProvider
```

## Dependencies

- `duckduckgo-search>=7.0` — DuckDuckGo provider
- `tavily-python>=0.5` — Tavily provider
- `httpx>=0.27` — Brave provider
- `pydantic>=2.0` — config + result validation
