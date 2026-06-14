# Technical Specification: State Persistence & Resumability

## Stack

- **ORM**: SQLAlchemy 2.x (async mode)
- **SQLite driver**: aiosqlite (default backend)
- **PostgreSQL driver**: asyncpg (optional backend)
- **Migrations**: Alembic (async engine support)

## Database Models

### TaskStatus Enum

```python
class TaskStatus(str, Enum):
    pending = "pending"
    in_progress = "in_progress"
    completed = "completed"
    failed = "failed"
    retry_queued = "retry_queued"
    blocked_by_dependency = "blocked_by_dependency"
```

### Task Table (Req 6.2)

| Column         | Type                | Notes                                       |
|----------------|---------------------|---------------------------------------------|
| id             | UUID (PK)           | Auto-generated                              |
| type           | String              | e.g. "search", "synthesize"                 |
| section        | String              | Research section this task belongs to       |
| subsection     | String (nullable)   | Optional subsection                         |
| status         | TaskStatus enum     | Default: pending                            |
| dependencies   | JSON                | List of task UUIDs that must complete first |
| attempt_count  | Integer             | Default: 0; cumulative across retry cycles  |
| last_error     | Text (nullable)     | Error message from last failure             |
| created_at     | DateTime            | Auto-set on creation                        |
| updated_at     | DateTime            | Auto-updated on any change                  |
| completed_at   | DateTime (nullable) | Set only when status → completed (Req 6.4)  |

## StateStore Class

```python
class StateStore:
    async def initialize(self) -> None: ...           # create schema, run migrations
    async def load_on_startup(self) -> None: ...       # skip completed, in_progress → pending (Req 6.3)
    async def create_task(self, type, section, ...) -> Task: ...
    async def get_pending / get_failed / get_by_section / get_progress(...): ...  # Req 6.9
    async def get_ready_tasks(self) -> list[Task]: ... # pending + deps completed (Req 6.8)
    async def mark_in_progress / mark_completed / mark_failed(...): ...  # commit before next (Req 6.10, 6.4)
    async def enqueue_retry(self, task_id) -> None: ... # retry_queued, attempt_count += 1 (Req 6.7)
```

## Continuous Persistence (Req 6.10) — Key Behavior

Every status transition (pending / in_progress / completed / failed /
retry_queued) is committed **before** the orchestrator initiates the next one.
Each `mark_*` / `enqueue_*` method updates the row and `await session.commit()`s
before returning. There is **no grace window** and no in-memory batching, so an
interruption at any instant — graceful signal OR abrupt loss (power, network,
kill, Ctrl+C) — leaves the last committed state intact and resumable.

## Startup & Resume (Req 6.3, 6.6)

`load_on_startup()` runs once at boot: load all tasks, leave `completed` ones
untouched (skipped), and run `UPDATE tasks SET status='pending' WHERE
status='in_progress'` so any task mid-flight at interruption is re-executed.

## Completion, Retry & Dependencies

- **Completion (Req 6.4, 6.5):** `mark_completed()` runs only after output is
  written to the target file AND confirmed present; sets `completed_at`. A
  Sub_Section is completed when all its tasks are completed.
- **Retry queue (Req 6.7):** `enqueue_retry()` sets `retry_queued` and increments
  `attempt_count` by 1, up to the configured max (default 5); then stays `failed`
  with `last_error` set.
- **Ready tasks (Req 6.8):** `get_ready_tasks()` returns pending tasks whose
  dependency UUIDs are all `completed`; others stay deferred.
- **Unsatisfiable deps (Req 6.11):** detect cycles (DFS / topological sort) and
  references to non-existent task IDs; set such tasks to `failed` with
  `last_error="unsatisfiable dependency"` rather than deferring forever.

## Progress Query (Req 6.9)

```python
# get_progress() returns:
{"total": int, "completed": int, "failed": int,
 "pending": int, "retry_queued": int,
 "pct": float}  # 0–100, completed / total * 100
```

## SQLite Configuration & Migrations

- WAL enabled for concurrent reads during writes, via `PRAGMA journal_mode=WAL` on the engine `connect` event.
- Alembic (async) auto-generates migrations from model changes and runs pending migrations on startup without data loss.

## File Layout

```
src/aria/state/
├── __init__.py
├── database.py      # Engine, session factory, WAL setup, backend switch
├── models.py        # SQLAlchemy Task model, TaskStatus enum
├── store.py         # StateStore class implementation
└── migrations/      # Alembic migrations (env.py, versions/)
```

## Key Dependencies

- `sqlalchemy[asyncio] >= 2.0`, `aiosqlite` (default), `asyncpg` (optional), `alembic`; internal: Config (Req 07) DB path/URL, Data Layer (Req 13) relational store
