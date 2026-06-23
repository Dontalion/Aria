# Design: Error Handling, Retry, and Task Recovery

## Purpose

Research runs span many tasks across unreliable services — LLM APIs time out,
search backends rate-limit, networks drop. ARIA treats failure as expected, not
exceptional. A failed task is never silently abandoned. It is retried, re-queued,
or escalated with a clear log so the operator always knows what happened and why.

## Core Principle

No silent failures. Every task reaches one of three honest outcomes:

- **Completes successfully** after one or more attempts, or
- **Gets re-queued** for a later, fresh round of attempts, or
- **Escalates** — pausing the pipeline with an ERROR log so a human can intervene.

This matters most for tasks with downstream dependents. If a keyword task feeds an
idea task, abandoning the keyword task would silently strand everything below it.
ARIA refuses to do that.

## Retry Strategy

### LLM Failures

LLM calls (timeouts, rate limits, malformed responses) retry with exponential
backoff. The first retry waits the initial delay (1 second), and each subsequent
delay multiplies by the backoff multiplier (default 2.0): 1s → 2s → 4s. ARIA makes
up to the immediate retry limit (default 3) before giving up on the current round.

### Search Failures

Search calls follow a simpler rule: exactly one retry after a 3-second delay. If the
retry also fails, the search is marked failed and the task moves on to the retry
queue. This mirrors the search provider's own graceful-degradation behavior.

## The Retry Queue

When a task exhausts its immediate retries, it does not disappear. It moves to the
**retry queue** with status `retry_queued`. The queue is processed only after every
other pending task in the current batch finishes. This prevents one stubborn task
from blocking healthy work behind it.

When a queued task is re-attempted, it gets a **fresh** set of immediate retries with
the backoff delay reset to 1 second. What is *not* reset is the **cumulative attempt
count** — that number is preserved across every retry-queue cycle so ARIA always
knows the true total of how many times a task has been tried.

## Persistence for Dependent Tasks

A task with downstream dependents is re-attempted across retry-queue cycles until it
either succeeds or its cumulative attempt count reaches the maximum total attempts
(default 5). The cumulative count is incremented and persisted on every attempt, so a
crash mid-run never loses track of how close a task was to escalation.

## Escalation

When a task's cumulative attempt count reaches the maximum total attempts without
success, ARIA escalates:

1. **Logs the failure at ERROR level** with the task identifier and last error.
2. **Pauses the pipeline** — it halts dispatch of new tasks rather than pressing on.
3. **Retains all persisted state** so the run can be resumed, or the task skipped,
   through manual intervention from the CLI.
4. **Marks downstream dependents** — every task that depended on the escalated task is
   set to `blocked_by_dependency`, making the blast radius visible at a glance.

## End-of-Run Summary

Every run ends with a summary the operator can read at a glance:

- **Total** tasks processed
- **Successful** completions
- **Failed** (exhausted all retries)
- **Retry-queued** (still pending a future attempt)
- **Blocked by dependency** (waiting on a failed upstream task)

## Design Decisions

- **Retry before skip** — recovery is always attempted before anything is given up.
- **Batch-aware queuing** — the retry queue only runs between batches, which avoids
  tight infinite loops on a persistently failing task.
- **Escalation over silence** — a paused pipeline is safer than quietly corrupted or
  incomplete results.
- **Human-in-the-loop** — after escalation, a person decides whether to resume or skip.
- **Configurable, not hardcoded** — every threshold (retry limit, multiplier, initial
  delay, max total attempts, halt-on-failure) comes from config (Req 07).

## What Users Don't Need to Know

- How the backoff delays are computed between attempts
- How the retry queue is ordered internally
- How the cumulative attempt count is persisted across crashes

## Dependencies

- Req 06 (State Persistence) — records task status, attempt counts, and dependents
- Req 07 (Configuration) — supplies the retry policy values
