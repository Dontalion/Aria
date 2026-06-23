# LLM Provider Abstraction — Technical Specification

## Abstract Base Class

```python
# src/aria/providers/base.py
from abc import ABC, abstractmethod

class LLMProvider(ABC):
    """Base interface all LLM providers must implement."""
    @abstractmethod
    async def generate(self, prompt: str, **kwargs) -> str:
        """Send prompt string to LLM, return completion text string."""  # Req 1.1
        ...
    async def healthcheck(self) -> bool:
        """Verify provider connectivity. Override per provider."""
        return True

class LLMProviderError(Exception):
    """Raised after all retries are exhausted; carries provider name + attempt count."""
```

## Concrete Implementations

| Class | File | SDK/Method | Req |
|-------|------|------------|-----|
| `OpenRouterProvider` | `src/aria/providers/llm/openrouter.py` | `openai` SDK, `base_url="https://openrouter.ai/api/v1"` | 1.2 |
| `OllamaProvider` | `src/aria/providers/llm/ollama.py` | `ollama` async client at configured `base_url` | 1.3 |
| `CLIToolProvider` | `src/aria/providers/llm/cli_tool.py` | `asyncio.create_subprocess_exec` (kiro-cli/codex/claude-code/gemini-cli) | 1.4 |
| `DirectAPIProvider` | `src/aria/providers/llm/direct_api.py` | native `anthropic` / `openai` SDK | 1.5 |

## Factory Pattern

```python
# src/aria/providers/llm/__init__.py
from aria.providers.base import LLMProvider
from aria.config.schema import LLMConfig  # Defined in req-5

def create_llm_provider(config: LLMConfig) -> LLMProvider:
    """Instantiate the correct provider from validated config."""
    match config.provider:
        case "openrouter":
            from .openrouter import OpenRouterProvider
            return OpenRouterProvider(config)
        case "ollama":
            from .ollama import OllamaProvider
            return OllamaProvider(config)
        case "cli_tool":
            from .cli_tool import CLIToolProvider
            return CLIToolProvider(config)
        case "direct_api":
            from .direct_api import DirectAPIProvider
            return DirectAPIProvider(config)
        case _:
            raise ValueError(f"Unknown LLM provider: {config.provider}")
```

## Configuration Model (ref: req-5-config)

Pydantic enforces every bound at load time (Req 1.8–1.10):

```python
from pydantic import BaseModel, Field
from typing import Literal

class LLMConfig(BaseModel):
    provider: Literal["openrouter", "ollama", "cli_tool", "direct_api"]  # Req 1.9
    model: str | None = None
    api_key_env: str | None = None
    base_url: str | None = None
    tool: str | None = None                                  # CLI binary name
    args: list[str] = []                                     # CLI arguments
    sdk: Literal["anthropic", "openai"] | None = None
    temperature: float = Field(0.3, ge=0.0, le=2.0)          # Req 1.9
    max_tokens: int | None = Field(None, ge=1)               # Req 1.9
    request_timeout: int = Field(60, ge=1, le=600)           # Req 1.6, 1.9
    max_retries: int = Field(3, ge=0, le=10)                 # Req 1.9
```

An unsupported provider (caught by the `Literal`) or an out-of-bounds parameter raises a `ValidationError`; the loader converts it into a field-level error and exits before any pipeline task starts (Req 1.10).

## Retry Logic

A shared `tenacity`-based decorator wraps every provider's `generate()`:

- A connection error or a timeout exceeding `config.request_timeout` (default 60s, range 1–600s) triggers a retry (Req 1.6).
- Exponential backoff 1s → 2s → 4s, up to `config.max_retries` (default 3) (Req 1.6).
- On final failure: raise `LLMProviderError` naming the provider and attempt count; the Orchestrator marks the task failed — **no fallback** to another provider (Req 1.7).

## File Structure

```
src/aria/providers/
├── base.py              # LLMProvider ABC, LLMProviderError
└── llm/
    ├── __init__.py      # create_llm_provider factory
    ├── openrouter.py    # OpenRouterProvider
    ├── ollama.py        # OllamaProvider
    ├── cli_tool.py      # CLIToolProvider
    └── direct_api.py    # DirectAPIProvider
```

## Dependencies

- `openai>=1.0` — OpenRouter + direct OpenAI provider
- `ollama>=0.3` — Ollama async client
- `anthropic>=0.39` — direct Anthropic provider
- `tenacity>=9.0` — exponential-backoff retry policy
- `pydantic>=2.0` — config validation
