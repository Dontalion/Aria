# Design: Parallel Execution & Orchestration

## Purpose

Run multiple research tasks at the same time so the overall process finishes
faster. Instead of handling one keyword or one section after another, ARIA fans
work out across parallel lanes and only waits where a true dependency exists.
The goal is speed without ever guessing about ordering: when ARIA is unsure
whether two pieces of work can safely run together, it asks you rather than
risk doing the wrong thing.

## Core Concept — Workflow as a Graph

The orchestrator coordinates every stage of work as a connected flow. Each stage
reads what happened before it and decides what to run next. A shared state object
travels through the flow, so the orchestrator always knows which tasks are
pending, which are running, which are done, and which are waiting on something.

## Parallelism Rules

| Scope | Rule |
|-------|------|
| Keywords within one sub-section | Run in parallel, up to `max_parallel` (Req 5.1) |
| Independent sub-sections (no shared data) | Run in parallel (Req 5.2) |
| Keyword pipeline and Idea pipeline for *different* independent sections | Run concurrently (Req 5.5) |

In every case the number of tasks running at the same time never exceeds the
configured limit. Extra ready tasks wait in a queue and start as soon as a
running task finishes and frees a slot.

## The Concurrency Limit

A single configurable value, `max_parallel`, caps how many tasks run at once.

- Default is **3**.
- It must be a whole number of **1 or greater**.
- A value below 1 is rejected when the configuration loads, with a clear error,
  before any task starts (Req 5.3, 5.4).

This prevents flooding the LLM provider or search API with too many simultaneous
requests, and keeps behavior predictable.

## Dependency Rules

- The Idea pipeline for a given section **waits** until the Keyword pipeline for
  that same section has finished, with its output written and verified
  (Req 5.6).
- A dependent task whose prerequisite has not completed successfully stays
  deferred — it is never started early (Req 5.7).
- A sub-section that references another sub-section's output waits for that
  output before it runs.

## Ambiguous Dependencies — Ask, Don't Guess (KEY BEHAVIOR)

Before scheduling, the orchestrator checks each pair of sub-sections to decide
whether they are independent — meaning neither one contains a task that depends
on a task or output of the other (Req 5.8).

Sometimes it genuinely cannot tell. When that happens, ARIA does **not** guess,
and it does **not** quietly fall back to running them one after another. Instead:

1. It presents a plain-language explanation of the ambiguous dependency to you —
   which two sub-sections are involved and why their relationship is unclear
   (Req 5.9).
2. It **waits for your decision**. While waiting, it starts **neither** of the two
   sub-sections concurrently, so nothing happens behind your back (Req 5.10).
3. Once you decide, it schedules the two sub-sections **concurrently or
   sequentially** exactly as you chose, then continues (Req 5.11).

This keeps a human in the loop precisely where automated judgment is unreliable,
while everything ARIA *can* confidently parallelize keeps running at full speed.

## Recovery

Progress is saved continuously through the State_Store (see Requirement 6). If
the process is interrupted, the next run picks up from the last saved state:
completed work is skipped, and any task that was mid-flight is re-run. No
duplicate API calls for work that already finished.

## Flow Summary

```
Plan Tasks ──► Execute (parallel, capped) ──► Check Dependencies
                          ▲                          │
                          │                  (ambiguous pair?)
                          │                          │
                          │                          ▼
                          │                 Ask user & wait
                          │                          │
                          └──────────────────────────┘
                                     │
                                     ▼
                              Write Output ──► Done
```

Errors at any stage route to a dedicated error-handling path that logs the
failure, marks the task as failed in state, and lets the rest of the flow
continue where possible.

## What Users Don't Need to Know

- How the underlying graph engine schedules and routes work
- How the concurrency limit is enforced internally
- How checkpoint state is stored and reloaded

ARIA stays out of the way when work is clearly parallel, and speaks up only when
a real decision needs a human.
