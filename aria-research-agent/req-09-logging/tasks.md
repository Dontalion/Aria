# Tasks: Progress Reporting & Logging

## Dependencies

- **Req 5 (Configuration)** — needs `logging.level` and `paths.logs_dir` from config

## Implementation Tasks

- [ ] 1. Set up structlog with dual output in `src/aria/logging/__init__.py`
  - Console renderer (rich) + JSON file renderer writing simultaneously (Req 9.3)
  - Processor chain adds ISO 8601 timestamp + component name to every entry (Req 9.1)
  - Expose `get_logger(name)` factory binding the component name
- [ ] 2. Implement log level handling and validation
  - Exactly four levels: debug, info, warning, error; default info (Req 9.2)
  - Reject invalid level via Pydantic, naming the value and allowed set (Req 9.8)
- [ ] 3. Configure rotating file handler and log directory management
  - `RotatingFileHandler` with 10 MB cap and 3 backups (Req 9.3)
  - Auto-create `paths.logs_dir` if missing on first write (Req 9.3)
- [ ] 4. Emit the required logged events
  - Pipeline start/end, keyword processing start/end, errors, retries (Req 9.1)
  - Section completion summary: completed vs total items (Req 9.4)
- [ ] 5. Implement `ProgressTracker` in `src/aria/logging/progress.py`
  - Pydantic models SectionProgress (name/completed/total/status) and ProgressState (Req 9.5)
  - Methods: register_section(), update(), mark_done(), mark_error()
  - Status one of pending | running | done | error, plus last_updated timestamp (Req 9.5)
- [ ] 6. Implement atomic progress writes
  - Write to temp file then `os.replace()` so pollers never read partial state (Req 9.5)
  - Update completed count + refreshed last_updated on each item completion (Req 9.6)
  - On write failure, log a warning and continue without crashing (Req 9.7)
- [ ] 7. Wire integration points
  - Orchestrator instantiates ProgressTracker at startup, registers each Section
  - Pipeline nodes call get_logger(__name__) and tracker.update() per unit of work
- [ ] 8. Write unit tests
  - Logger outputs to both console and file with timestamp + component (Req 9.1, 9.3)
  - Log level filtering and invalid-level rejection (Req 9.2, 9.8)
  - Progress JSON format and per-item update behavior (Req 9.5, 9.6)
  - Atomic write doesn't corrupt under concurrent reads (Req 9.5)
  - Failed progress write logs warning and continues (Req 9.7)

## Acceptance Criteria Coverage

- 11.1 — events logged with ISO 8601 timestamp + component name
- 11.2 — exactly four log levels, default info
- 11.3 — dual output (stdout + file), 10 MB cap, 3 backups, auto-create log dir
- 11.4 — Section completion summary (completed vs total)
- 11.5 — machine-readable JSON progress with per-Section fields + last_updated
- 11.6 — progress updated on each item completion
- 11.7 — failed progress write logs warning, pipeline continues
- 11.8 — invalid log level rejected, naming value and allowed values
