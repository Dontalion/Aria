# CLI Interface — Technical Specification

## Stack

| Component       | Library  | Reason                                       |
|-----------------|----------|----------------------------------------------|
| CLI framework   | `typer`  | Type-safe args, auto-generated `--help`      |
| Terminal output | `rich`   | Tables/color, only when a TTY exists         |
| Signals         | `signal` | SIGINT / SIGTERM handling (stdlib)           |

## File Layout

```
src/aria/
├── main.py              # Typer app, entry point
└── cli/
    └── __init__.py      # signal handling, TTY detection, structured output
```

## Entry Point: `src/aria/main.py`

```python
import typer
from aria.config.loader import load_config

app = typer.Typer(name="aria", help="ARIA Research Agent")

def _require_section(cfg, section):              # Req 6.5: unknown section
    if section not in cfg.sections():
        typer.echo(f"Unknown section: {section}", err=True)
        raise typer.Exit(code=1)                 # no state change

@app.command()                                       # Req 6.1, 6.2
def run(section: str = typer.Argument(...), config: str = typer.Option(None)):
    """Run the pipeline for ONE existing section; others unchanged."""
    cfg = load_config(config); _require_section(cfg, section); ...
@app.command(name="run-all")                         # Req 6.1
def run_all(config: str = typer.Option(None)):
    """Run all configured pipelines across all sections."""; ...
@app.command()                                       # Req 6.1, 6.3
def status(config: str = typer.Option(None)):
    """Print completed/total per configured section."""; ...
@app.command()                                       # Req 6.1, 6.4
def reset(section: str = typer.Argument(...), config: str = typer.Option(None)):
    """Clear persisted state for ONE existing section; others unchanged."""
    cfg = load_config(config); _require_section(cfg, section); ...
@app.command()
def chat():
    """Conversational mode for creating skills/workflows (Req 14)."""; ...

if __name__ == "__main__":
    app()
```

## Signal Handling: `src/aria/cli/__init__.py`

No fixed grace window: the flag tells the orchestrator to stop dispatching; the in-progress write completes and the process exits (Req 6.8).

```python
import signal, sys, threading

shutdown_flag = threading.Event()

def _handle_signal(signum, frame):
    shutdown_flag.set()           # stop accepting new work (Req 6.8)
def register_signal_handlers():
    signal.signal(signal.SIGINT, _handle_signal)
    signal.signal(signal.SIGTERM, _handle_signal)
def is_tty() -> bool:
    return sys.stdout.isatty()    # Rich only when TTY (Req 6.6)
```

## Orchestrator Integration

State commits continuously (Req 6.7); between task steps the orchestrator checks the shutdown flag and exits without marking the unfinished task completed (Req 6.8):

```python
from aria.cli import shutdown_flag

for task in pipeline:
    if shutdown_flag.is_set():
        break              # state already persisted; task stays non-completed (Req 6.7, 6.8)
    execute(task)          # commits status before next transition (Req 6.7)
```

## Output Mode Selection (Req 6.6)

```python
from aria.cli import is_tty

if is_tty():
    render_rich_table(rows)       # human-facing tables/colors
else:
    for r in rows:                # structured record: timestamp + level + event
        log.info("status", section=r.name, completed=r.done, total=r.total)
```

## Resume Behavior (Req 6.9)

On startup the orchestrator loads persisted state, skips `completed` tasks, and re-runs any non-completed task — covering graceful (SIGINT/SIGTERM) and abrupt (kill, power loss) interruptions identically.

## pyproject.toml Script Entry & Dependencies

```toml
[project.scripts]
aria = "aria.main:app"
```

- `typer>=0.12` — CLI framework
- `rich>=13.0` — TTY-only formatting
