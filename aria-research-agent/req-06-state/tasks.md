# Tasks: State Persistence & Resumability

## Dependencies

- **Req 07** — Config (database path/URL and `max_attempts` come from the config system)
- **Req 13** — Data Layer (relational store engine/session foundation this layer builds on)

## Task List

- [ ] 1. Create SQLAlchemy async models and `TaskStatus` enum
  - File: `src/aria/state/models.py`
  - Define `TaskStatus` with `pending`, `in_progress`, `completed`, `failed`, `retry_queued`, `blocked_by_dependency`
  - Define `Task` with fields: id, type, section, subsection, status, dependencies (JSON), attempt_count, last_error, created_at, updated_at, completed_at
  - _Requirements: 6.1, 6.2_

- [ ] 2. Implement database engine setup with WAL and backend switch
  - File: `src/aria/state/database.py`
  - Build async engine + session factory; default to SQLite (aiosqlite), allow PostgreSQL (asyncpg) via config
  - Enable WAL via `PRAGMA journal_mode=WAL` on the SQLite connect event
  - _Requirements: 6.1_

- [ ] 3. Implement `StateStore` core with continuous-commit transitions
  - File: `src/aria/state/store.py`
  - Methods: `create_task`, `mark_in_progress`, `mark_completed`, `mark_failed`
  - Each status transition commits to the database BEFORE the orchestrator initiates the next transition (no grace window, no batching)
  - _Requirements: 6.2, 6.10_

- [ ] 4. Implement startup load / resume logic
  - `load_on_startup()`: load persisted state, skip `completed` tasks, and reset any `in_progress` task to `pending` so it re-executes
  - Ensure an interruption leaving a task `in_progress` results in re-execution on the next run, never a skip
  - _Requirements: 6.3, 6.6_

- [ ] 5. Implement verified completion
  - `mark_completed()` sets `status=completed` and records `completed_at` only after the task output is written to its target file AND confirmed present
  - Mark a Sub_Section completed when all of its tasks are completed
  - _Requirements: 6.4, 6.5_

- [ ] 6. Implement retry queue logic
  - `enqueue_retry()` sets `status=retry_queued` and increments `attempt_count` by 1, up to the configured maximum attempt limit (default 5)
  - On reaching the limit, keep the task `failed` with `last_error` populated
  - _Requirements: 6.7_

- [ ] 7. Implement dependency checking (`get_ready_tasks`)
  - Return pending tasks whose dependency tasks are all `completed`; defer the rest
  - _Requirements: 6.8_

- [ ] 8. Implement unsatisfiable-dependency detection
  - Detect dependency cycles and references to non-existent tasks
  - Set such tasks to `failed` with `last_error` indicating an unsatisfiable dependency, rather than deferring indefinitely
  - _Requirements: 6.11_

- [ ] 9. Implement state queries
  - `get_pending`, `get_failed`, `get_by_section`, and `get_progress`
  - `get_progress()` returns total, completed, failed, pending, retry-queued counts and a percentage from 0 to 100
  - _Requirements: 6.9_

- [ ] 10. Add Alembic migration setup
  - Configure async Alembic env; generate the initial migration from models
  - Run pending migrations on application startup without data loss
  - _Requirements: 6.1_

- [ ] 11. Write unit tests
  - Cover each status transition committing before the next; startup reset of `in_progress` → `pending` with `completed` skipped
  - Cover verified completion (output present), retry queue increment up to max attempts, ready-task dependency gating, and unsatisfiable-dependency (cycle / missing task) → failed
  - Cover progress reporting counts and 0–100 percentage
  - _Requirements: 6.2, 6.3, 6.4, 6.5, 6.7, 6.8, 6.9, 6.10, 6.11_
