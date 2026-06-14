# Design: State Persistence & Resumability

## Purpose

Let ARIA resume research from exactly where it left off after any kind of stop —
a crash, a power or network loss, a process kill, or a deliberate Ctrl+C. No
finished work is lost and no expensive API call is repeated. Every task's
progress is written to a database the moment it changes, so the database always
reflects the most recent truth about what has been done.

## How It Works

### Database-Backed State

All task state lives in a database. SQLite is the default for single-machine use
(zero setup, just a file), and PostgreSQL is available for production via a
config switch — same behavior, different backend. The database is the single
source of truth for what is done, what is in progress, and what still needs to
happen.

### What's Tracked

Each research task is stored with:

- **Status**: pending, in_progress, completed, failed, retry_queued, or
  blocked_by_dependency
- **Type and section/subsection**: what kind of work it is and where it belongs
- **Dependencies**: which other tasks must finish before this one can start
- **Attempt count**: how many times the task has been tried
- **Last error**: the most recent failure reason, kept for debugging
- **Timestamps**: when it was created, last updated, and completed

### Continuous Persistence — Key Behavior

Every time a task changes status — pending, in_progress, completed, failed, or
retry_queued — that change is committed to the database **before** the
orchestrator starts the next change. There is **no grace window** and no batching
of updates. Because the latest state is always already saved, an interruption at
any instant is safe. It does not matter whether the stop was graceful (a
shutdown signal) or abrupt (power cut, network drop, kill): the next run simply
continues from the last saved state.

### Starting Up and Resuming

When ARIA starts, it loads the saved state and:

- **Skips** any task already marked completed — that work is done.
- **Resets** any task left in_progress back to pending, so it runs again. A task
  that was mid-flight when the process stopped cannot be trusted as finished, so
  it is re-executed rather than skipped.

### When a Task Counts as Done

A task is only marked completed when its output has been written to the target
file **and** confirmed present, at which point the completion time is recorded.
Writing alone is not enough — the output must be verified to exist. When every
task in a sub-section is completed, the sub-section itself is marked completed.

### Retry Queue

Failed tasks are never quietly abandoned. A failed task is re-queued with status
retry_queued and its attempt count increased by one, up to the configured
maximum number of attempts. Once that limit is reached the task stays failed
with its last error recorded, so the cause is visible for debugging.

### Task Dependencies

A task waits until all of its listed dependencies are completed. This keeps work
in the right order — for example, "gather sources" must finish before
"synthesize findings." When a task can never be satisfied — its dependencies
form a cycle, or it points at a task that does not exist — ARIA does not wait
forever. It marks that task failed and records an unsatisfiable-dependency note
in its last error, so the problem surfaces immediately instead of stalling the
run.

### Queryable State

At any moment the system can answer:

- Which tasks are pending?
- Which tasks have failed, and why?
- What tasks belong to a given section?
- What is the overall progress — total, completed, failed, pending, and
  retry-queued counts, plus a percentage from 0 to 100?

### Setup and Upgrades

- On first run, the database and tables are created automatically.
- When the data shape changes, migrations run gracefully without losing data.
- No manual setup is needed — just start the system.

## Database Choice

SQLite is the default: file-based, zero configuration, ideal for a single
machine. PostgreSQL is supported for multi-machine or production use through a
single config change. The rest of ARIA talks to the same interface either way,
so switching backends never touches pipeline code.

## What Users Don't Need to Know

- How status changes are committed under the hood
- How the database tables and migrations are structured
- How resume logic decides which tasks to skip or re-run

ARIA keeps the bookkeeping invisible. The user just restarts after any
interruption and trusts that finished work stays finished.
