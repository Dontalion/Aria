# Configuration Management — Technical Specification

## Architecture

- `src/aria/config/schema.py` — Pydantic models defining `ConfigSchema` and bounds
- `src/aria/config/loader.py` — YAML loading, environment variable resolution, validation

Credentials (API keys) are **not** part of the schema. They are resolved from the
environment by the components that need them (LLM client, search client), not by
the config loader.

Dynamic runtime values such as the research topic are **not** part of `ConfigSchema`.
They are passed as arguments through the call stack.

## Schema (`schema.py`)

Pydantic enforces every allowed set and numeric bound at load time (Req 7.2, 7.7):

```python
from pydantic import BaseModel, Field
from typing import Literal
from pathlib import Path

class LLMConfig(BaseModel):
    provider: Literal["openrouter", "ollama", "cli_tool", "direct_api"]  # Req 7.2
    model: str | None = None
    base_url: str | None = None

class SearchConfig(BaseModel):
    provider: Literal["duckduckgo", "tavily", "brave"] = "duckduckgo"   # Req 7.2
    max_results: int = Field(5, ge=1, le=20)

class ExecutionConfig(BaseModel):
    max_parallel: int = Field(3, ge=1, le=10)      # Req 7.2: integer 1–10, default 3
    retry_limit: int = Field(2, ge=0, le=5)        # Req 7.2: integer 0–5

class PathsConfig(BaseModel):
    input_file: Path = Path("./keywords.md")
    output_dir: Path = Path("./output")

class ConfigSchema(BaseModel):
    llm: LLMConfig                                  # required section
    search: SearchConfig = SearchConfig()
    execution: ExecutionConfig = ExecutionConfig()
    paths: PathsConfig = PathsConfig()
```

`ConfigSchema` contains **no credentials and no runtime inputs**. API keys are
read from the environment by the clients that use them. The research topic and
other per-run values travel as function arguments.

A value outside an allowed `Literal` set, or outside a `Field` bound, raises a
`ValidationError`. The loader converts it into a field-level message and exits
non-zero before any research starts (Req 7.7).


## Loader (`loader.py`)

```python
import os
from pathlib import Path
from typing import Any
import yaml
from pydantic import ValidationError
from aria.config.schema import ConfigSchema

DEFAULT_PATH = Path("./config.yaml")

class ConfigError(Exception):
    """Field-level config failure; message names the offending field/var."""

def load_config(path: Path | None = None) -> ConfigSchema:
    resolved = path or DEFAULT_PATH
    if not resolved.exists():
        return ConfigSchema.model_validate({            # Req 7.6: defaults, no termination
            "llm": {"provider": "openrouter"}
        })
    raw = yaml.safe_load(resolved.read_text()) or {}
    try:
        return ConfigSchema.model_validate(raw)         # Req 7.3 / 7.5 / 7.7
    except ValidationError as e:
        err = e.errors()[0]
        field = ".".join(str(p) for p in err["loc"])
        raise ConfigError(f"{field}: {err['msg']}")
```

The loader does **not** perform `${ENV_VAR}` substitution. Environment variables
are never embedded in `config.yaml`; secrets belong in the environment only.

## Startup Sequence

```
resolve path (--config flag → default config.yaml)
   → file missing? → build ConfigSchema from documented defaults (Req 7.6)
   → file present? → parse YAML
       → Pydantic validate (missing required / out-of-bounds → ConfigError) (Req 7.3, 7.7)
   → return validated ConfigSchema
   → instantiate provider clients
       → each client resolves its env var (unset → ConfigError) (Req 7.8)
→ accept runtime inputs (research topic, …) from CLI / caller
→ begin research
```

Any `ConfigError` is printed to stderr and the process exits non-zero **before**
any research starts (Req 7.3, 7.7, 7.8).

## Dependencies

- `pydantic>=2.0` — schema validation and bound enforcement
- `pyyaml>=6.0` — YAML parsing
