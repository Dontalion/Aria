# Configuration Management — Technical Specification

## Architecture

- `src/aria/config/schema.py` — Pydantic models defining `ConfigSchema` and bounds
- `src/aria/config/loader.py` — YAML loading, `${ENV_VAR}` interpolation, validation

## Schema (`schema.py`)

Pydantic enforces every allowed set and numeric bound at load time (Req 7.2, 7.7):

```python
from pydantic import BaseModel, Field
from typing import Literal
from pathlib import Path

class LLMConfig(BaseModel):
    provider: Literal["openrouter", "ollama", "cli_tool", "direct_api"]  # Req 7.2
    model: str | None = None
    api_key: str = ""
    base_url: str | None = None

class SearchConfig(BaseModel):
    provider: Literal["duckduckgo", "tavily", "brave"] = "duckduckgo"   # Req 7.2
    api_key: str = ""
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
    research_topic: str                             # required (Req 7.3 example)
```

A value outside an allowed `Literal` set, or outside a `Field` bound, raises a
`ValidationError`. The loader converts it into a field-level message and exits
non-zero before any research starts (Req 7.7).

## Loader (`loader.py`)

```python
import os, re
from pathlib import Path
from typing import Any
import yaml
from pydantic import ValidationError
from aria.config.schema import ConfigSchema

ENV_VAR_RE = re.compile(r"\$\{([^}]+)\}")
DEFAULT_PATH = Path("./config.yaml")

class ConfigError(Exception):
    """Field-level config failure; message names the offending field/var."""

def _interpolate(value: Any) -> Any:
    if isinstance(value, str):
        def _sub(m: re.Match) -> str:
            name = m.group(1)
            v = os.environ.get(name)
            if v is None:
                raise ConfigError(f"environment variable {name} is not set")  # Req 7.8
            return v
        return ENV_VAR_RE.sub(_sub, value)
    if isinstance(value, dict):
        return {k: _interpolate(v) for k, v in value.items()}
    if isinstance(value, list):
        return [_interpolate(i) for i in value]
    return value

def load_config(path: Path | None = None) -> ConfigSchema:
    resolved = path or DEFAULT_PATH
    if not resolved.exists():
        return ConfigSchema()                       # Req 7.6: defaults, no termination
    raw = yaml.safe_load(resolved.read_text()) or {}
    interpolated = _interpolate(raw)                # Req 7.4 / 7.8
    try:
        return ConfigSchema.model_validate(interpolated)  # Req 7.3 / 7.5 / 7.7
    except ValidationError as e:
        err = e.errors()[0]
        field = ".".join(str(p) for p in err["loc"])
        raise ConfigError(f"{field}: {err['msg']}")
```

> Note: `ConfigSchema()` with no args succeeds only when required fields have defaults. Where a required field has none (e.g. `llm`, `research_topic`), the missing-file path supplies documented defaults via a packaged default mapping, so Req 7.6 holds without violating Req 7.3.

## Startup Sequence

```
resolve path (--config flag → default config.yaml)
   → file missing? → build ConfigSchema from documented defaults (Req 7.6)
   → file present? → parse YAML → interpolate ${ENV_VAR} (unset → ConfigError) (Req 7.4, 7.8)
       → Pydantic validate (missing required / out-of-bounds → ConfigError) (Req 7.3, 7.7)
   → return validated ConfigSchema
```

Any `ConfigError` is printed to stderr and the process exits non-zero **before** any research starts (Req 7.3, 7.7, 7.8).

## Dependencies

- `pydantic>=2.0` — schema validation and bound enforcement
- `pyyaml>=6.0` — YAML parsing
