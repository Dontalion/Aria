# CLI Interface — Design

## Purpose

Provide a terminal-based control surface for ARIA that works in headless
environments — tmux sessions, nohup, cron jobs, SSH. No browser or GUI needed.
You drive every research run from the command line.

## Main Command

```
aria <subcommand> [options]
```

## Subcommands

### `aria run <section>`

Execute the pipeline for one section that exists in the loaded config. ARIA runs
**only** that section's pipeline and leaves the state of every other section
unchanged (Req 8.2).

### `aria run-all`

Execute all configured pipelines across all sections in sequence (Req 8.1).

### `aria status`

For each configured section, print completion progress in the form
`completed/total` (Req 8.3):

| Section           | Progress |
|-------------------|----------|
| Literature Review | 3/5      |
| Methodology       | 0/4      |

### `aria reset <section>`

Clear the persisted state for one section so it can be re-run from scratch. Like
`run`, it affects **only** the named section and leaves all other sections'
state untouched (Req 8.4).

### `aria chat`

Enter conversational mode for creating custom skills and workflows through
natural-language interaction with ARIA (see Req 16).

## Unknown Section Handling

If `run` or `reset` is given a section identifier that does not exist in the
loaded config, ARIA exits **without modifying any persisted state** and returns an
error that names the unknown section identifier (Req 8.5). Nothing is started, and
nothing is cleared.

## Headless Operation

- Every subcommand runs **without an attached TTY** (Req 8.6).
- When there is no TTY, output is written as **structured log records**, each
  including a timestamp, a log level, and an event description (Req 8.6).
- Rich formatting (colors, tables) activates **only when a TTY is detected**.
- Compatible with: tmux, screen, nohup, systemd services.

## Continuous Persistence

ARIA persists its current progress to the database **continuously as work
proceeds** (Req 8.7). At any moment the database reflects the most recent completed
work, so a run can always resume from that point after any interruption.

## Graceful Shutdown (SIGINT / SIGTERM)

When ARIA receives SIGINT (Ctrl+C) or SIGTERM, it:

1. **Stops accepting new work** — no new tasks are dispatched.
2. **Finishes persisting** the in-progress state to the database.
3. **Exits**, leaving any unfinished task in a non-completed status so it is
   resumed on the next run (Req 8.8).

There is **no fixed grace window** (for example, no 10-second countdown). Because
state is already persisted continuously (Req 8.7, and Req 6), shutdown is simply
"stop dispatching, finish the current write, and exit cleanly."

## Abrupt Termination

If ARIA is killed abruptly — power loss, network loss, `kill -9` — without a
graceful signal, the next run resumes from the last state persisted in the
database without repeating already-completed work (Req 8.9). The continuous
persistence model makes graceful and abrupt interruptions behave the same way on
resume.

## What Users Don't Need to Know

- How signal handlers coordinate with the orchestrator
- How the orchestrator checks for shutdown between tasks
- How TTY detection toggles Rich versus plain structured output
