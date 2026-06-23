# Tasks: Error Handling, Retry, and Task Recovery

## Dependencies

- **Req 4 (State Persistence)** — needs StateStore for mark_failed(), mark_retry_queued(), increment_attempts(), get_dependents(), mark_blocked()
- **Req 5 (Configuration)** — retry settings loaded from config (max_attempts, backoff_multiplier, initial_delay, max_total_attempts, halt_on_failure)

## Implementation Tasks

- [ ] 1. Create `src/aria/retry/__init__.py` with module exports
  - Export RetryPolicy, with_retry, RunSummary, retry queue logic
- [ ] 2. Create `RetryPolicy` Pydantic model in `src/aria/retry/policy.py`
  - Fields and defaults: max_attempts=3, backoff_multiplier=2.0, initial_delay=1.0, max_total_attempts=5, halt_on_failure=true (Req 8.8)
- [ ] 3. Implement `@with_retry(policy)` decorator using `tenacity`
  - LLM path: `wait_exponential` from initial_delay × multiplier, `stop_after_attempt(max_attempts)` (Req 8.1)
  - Search path: exactly one retry after a 3s delay, then mark failed (Req 8.2)
  - On cycle exhaustion, call StateStore.mark_failed() and enqueue
- [ ] 4. Integrate with StateStore failure tracking
  - mark_failed(task_id, error, attempt), mark_retry_queued(task_id) (Req 8.3)
  - increment_attempts(task_id) to persist cumulative count (Req 8.6)
  - mark_escalated(task_id), mark_blocked(dependent_id)
- [ ] 5. Implement retry queue processing
  - Set exhausted tasks to retry_queued, never discard (Req 8.3)
  - Process the queue only between batches, not mid-batch (Req 8.4)
  - Fresh immediate retries with backoff reset, cumulative count preserved (Req 8.4)
- [ ] 6. Implement cumulative attempt accounting
  - Single counter spanning immediate + queue cycles, persisted each attempt (Req 8.5, 8.6)
  - Re-attempt tasks with dependents until success or max_total_attempts (Req 8.5)
- [ ] 7. Implement dependency-aware escalation
  - At max_total_attempts: log ERROR, halt new dispatch to pause pipeline, retain state (Req 8.9)
  - Set each downstream dependent to blocked_by_dependency (Req 8.10)
- [ ] 8. Create `RunSummary` model and end-of-run report generator
  - total / successful / failed / retry_queued / blocked_by_dependency (Req 8.7)
- [ ] 9. Write unit tests for retry logic
  - Test exponential backoff timing and search single-retry behavior (Req 8.1, 8.2)
  - Test retry queue processing order between batches (Req 8.4)
  - Test cumulative count preservation across cycles (Req 8.6)
  - Test escalation triggers + dependent blocking (Req 8.9, 8.10)
  - Test RunSummary accuracy after mixed success/failure runs (Req 8.7)

## Acceptance Criteria Coverage

- 10.1 — LLM exponential backoff (1s × multiplier, up to immediate retry limit)
- 10.2 — search retried exactly once after 3s, then marked failed
- 10.3 — exhausted task set to retry_queued, never discarded
- 10.4 — retry queue processed after batch, backoff reset, cumulative count kept
- 10.5 / 10.6 — cumulative attempts persisted; dependents retried to max_total_attempts
- 10.7 — end-of-run summary with all five counts
- 10.8 — retry behavior fully configurable
- 10.9 — escalation logs ERROR, pauses pipeline, retains state
- 10.10 — downstream dependents set to blocked_by_dependency
