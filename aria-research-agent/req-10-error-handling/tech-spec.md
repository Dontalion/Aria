# Technical Specification: Error Handling, Retry, and Task Recovery

## Stack

| Library    | Role                                                       |
|------------|------------------------------------------------------------|
| `tenacity` | Retry decorators with configurable exponential backoff     |
| `pydantic` | `RetryPolicy` and `RunSummary` models                      |
| built-in   | Integration with ARIA `StateStore` (Req 06) for tracking   |

## Module Layout

```
src/aria/retry/
├── __init__.py     # exports RetryPolicy, with_retry, RunSummary, RetryQueue
└── policy.py       # RetryPolicy + RunSummary models, with_retry, queue logic
```

## Models

### RetryPolicy (Pydantic)

```python
class RetryPolicy(BaseModel):
    max_attempts: int = 3            # immediate retries per cycle (Req 10.1)
    backoff_multiplier: float = 2.0  # exponential factor (Req 10.1)
    initial_delay: float = 1.0       # first retry delay in seconds (Req 10.1)
    max_total_attempts: int = 5      # across immediate + queue cycles (Req 10.5)
    halt_on_failure: bool = True     # pause pipeline on escalation (Req 10.9)
```

### RunSummary (Pydantic)

```python
class RunSummary(BaseModel):         # Req 10.7
    total: int = 0
    successful: int = 0
    failed: int = 0
    retry_queued: int = 0
    blocked_by_dependency: int = 0
```

## Retry Decorator (Req 10.1, 10.2)

```python
@with_retry(policy: RetryPolicy)
```

- **LLM calls** — `tenacity.retry` with `wait_exponential` seeded by
  `policy.initial_delay` and `policy.backoff_multiplier`, stopped by
  `stop_after_attempt(policy.max_attempts)`. Delays: 1s → 2s → 4s.
- **Search calls** — a fixed policy: exactly one retry after a 3-second delay
  (`wait_fixed(3)`, `stop_after_attempt(2)`); on second failure the search is marked
  failed and the task proceeds to the retry queue (Req 10.2).
- On final failure of a cycle, the decorator calls
  `state_store.mark_failed(task_id, error, attempt)` and enqueues the task.

## StateStore Integration (Req 06)

```python
state_store.mark_failed(task_id, error: str, attempt: int)
state_store.mark_retry_queued(task_id)        # status -> retry_queued (Req 10.3)
state_store.increment_attempts(task_id)       # persist cumulative count (Req 10.6)
state_store.mark_escalated(task_id)           # ERROR + pause (Req 10.9)
state_store.get_dependents(task_id)           # downstream lookup (Req 10.10)
state_store.mark_blocked(dependent_id)        # blocked_by_dependency (Req 10.10)
```

## Retry Queue & Cumulative Attempts (Req 10.3–10.6)

- A task that exhausts immediate retries is set to `retry_queued` and added to the
  queue — never discarded (Req 10.3).
- The queue is processed only **after all other pending tasks in the current batch
  complete** — never mid-batch (Req 10.4).
- Each re-attempt grants a **fresh** set of immediate retries with the backoff delay
  reset to `initial_delay`, while a single **cumulative attempt count** spanning
  immediate retries and every queue cycle is preserved and persisted on each attempt
  (Req 10.4, 10.6).
- A task with downstream dependents is re-attempted until success or until the
  cumulative count reaches `max_total_attempts` (Req 10.5).

## Escalation (Req 10.9, 10.10)

When `cumulative_attempts >= max_total_attempts` without success:

1. Log the failure at **ERROR** level with task id and last error.
2. **Halt dispatch** of new tasks to pause the pipeline (gated by `halt_on_failure`).
3. **Retain all persisted state** so the run can be resumed or the task skipped via CLI.
4. For each `state_store.get_dependents(task_id)`, set status `blocked_by_dependency`.

## End-of-Run Summary (Req 10.7)

After a run, aggregate StateStore counts into a `RunSummary` and emit it: total,
successful, failed, retry_queued, blocked_by_dependency.

## Configuration (from Req 07)

```toml
[retry]
max_attempts = 3
backoff_multiplier = 2.0
initial_delay = 1.0
max_total_attempts = 5
halt_on_failure = true
```

## Dependencies

`tenacity`, `pydantic` — declared in `pyproject.toml`.
