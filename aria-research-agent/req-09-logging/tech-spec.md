# Technical Specification: Progress Reporting & Logging

## Stack

| Library     | Role                                              |
|-------------|---------------------------------------------------|
| `structlog` | Structured logging (JSON + human-readable renderers) |
| `rich`      | Terminal formatting (colors, severity highlighting)  |
| built-in    | `logging.handlers.RotatingFileHandler`, `os.replace` |

## Module Layout

```
src/aria/logging/
├── __init__.py     # logger factory + structlog/handler configuration
└── progress.py     # ProgressTracker (atomic JSON progress writer)
```

## Logger Setup (`src/aria/logging/__init__.py`)

- Reads `logging.level` and `paths.logs_dir` from the loaded config (Req 5).
- Configures structlog with a shared processor chain feeding two renderers:
  **Console** — `rich` ConsoleRenderer for colored stdout; **File** — JSON renderer
  to `{paths.logs_dir}/aria.log` (Req 9.3).
- Processor chain: add ISO 8601 timestamp → add log level → add component/caller
  name → render. Timestamp + component satisfy Req 9.1.
- Exposes `get_logger(name: str)` returning a bound logger whose `name` becomes the
  originating component (Req 9.1).

### Log Levels (Req 9.2, 9.8)

```python
ALLOWED_LEVELS = {"debug", "info", "warning", "error"}   # exactly four
DEFAULT_LEVEL = "info"
```

- The config loader validates `logging.level` against `ALLOWED_LEVELS` via Pydantic;
  an invalid value is rejected with an error naming the value and the allowed set
  (Req 9.8). Absent level defaults to `info` (Req 9.2).

### File Handler & Rotation (Req 9.3)

```python
from logging.handlers import RotatingFileHandler
handler = RotatingFileHandler(log_path, maxBytes=10 * 1024 * 1024, backupCount=3)
```

- 10 MB cap, at most 3 rotated backups. The log directory is created on first write
  if missing (Req 9.3).

## Logged Events (Req 9.1, 9.4)

| Event                        | Fields beyond timestamp + component   |
|------------------------------|---------------------------------------|
| pipeline start / end         | status, duration                      |
| keyword processing start/end | keyword, step, result summary         |
| errors                       | context, stack trace (debug level)    |
| retries                      | attempt number, backoff, reason       |
| Section completion       | completed count vs total (Req 9.4)   |

## Progress Tracker (`src/aria/logging/progress.py`)

### Models & Class

```python
class SectionProgress(BaseModel):
    name: str; completed: int; total: int
    status: str  # "pending" | "running" | "done" | "error"

class ProgressState(BaseModel):
    sections: list[SectionProgress]
    last_updated: str  # ISO 8601 timestamp
```

`ProgressTracker` creates/overwrites `{paths.logs_dir}/progress.json` on init and
exposes: `register_section(name, total)` (adds a pending Section, `completed=0`);
`update(name, completed)` (increments, sets `running`, writes atomically with a
refreshed `last_updated` — Req 9.5, 9.6); `mark_done(name)` / `mark_error(name)`
(set terminal status — Req 9.5).

### Atomic Write (Req 9.5, 9.7)

```python
def _write(self) -> None:
    tmp = self.path.with_suffix(".json.tmp")
    try:
        tmp.write_text(self.state.model_dump_json(indent=2))
        os.replace(tmp, self.path)        # atomic swap; pollers never see partial
    except OSError as exc:
        logger.warning("progress write failed: %s", exc)   # Req 9.7 — don't crash
```

### Output Example

```json
{
  "sections": [{"name": "keywords", "completed": 7, "total": 10, "status": "running"}],
  "last_updated": "2025-01-15T14:32:01Z"
}
```

## Integration Points

- Pipeline nodes call `get_logger(__name__)` for structured component-tagged logs.
- The orchestrator instantiates one `ProgressTracker` at startup, calls
  `register_section()` per Section, and each node calls `tracker.update()` after
  completing a unit of work (Req 9.6).
- The config loader validates `logging.level` before any pipeline task runs (Req 9.8).

## Dependencies `structlog`, `rich` — declared in `pyproject.toml`.
