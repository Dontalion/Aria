# Design: Progress Reporting & Logging

## Purpose

ARIA runs headless, often for long stretches in tmux or under nohup. Operators need
to know what it is doing without attaching to the process. ARIA produces two
complementary outputs: human-readable logs for someone watching a terminal, and a
machine-readable progress file that external tools, dashboards, or scripts can poll.

## Two Output Channels

### 1. Human-Readable Logs (stdout + File)

- Every entry is written to **stdout** and a **log file** at the same time, so logs
  survive a disconnected terminal and are also visible live.
- Entries are color-coded by severity on a TTY for quick scanning.
- Each entry carries an **ISO 8601 timestamp** and the **originating component name**,
  alongside a concise event message.
- The log file lives under `paths.logs_dir` from `config.yaml`.

### 2. Machine-Readable Progress (JSON File)

- A single JSON file, updated as the pipeline advances, at `{paths.logs_dir}/progress.json`.
- External dashboards, scripts, or UIs poll this file instead of parsing logs.
- It holds per-Sub_Section completion counts and an overall status, plus a freshness
  timestamp so consumers can detect staleness.

## What Gets Logged

ARIA logs these events, each with a timestamp and component name:

- **Pipeline start / end** — including overall status and duration
- **Keyword processing start / end** — which keyword, which step, a result summary
- **Errors** — full context, with stack traces surfaced at debug level
- **Retries** — attempt number, backoff duration, and the reason for the retry

## Log Levels

Configured in `config.yaml` under `logging.level`. ARIA supports **exactly four**
levels and defaults to `info` when none is configured:

| Level   | Use Case                                       |
|---------|------------------------------------------------|
| debug   | Detailed internals (API payloads, state dumps) |
| info    | Normal operation milestones (default)          |
| warning | Recoverable issues (retries, fallbacks)        |
| error   | Failures requiring attention                   |

If the configured level is anything other than these four, ARIA rejects the
configuration and emits an error naming the invalid value and the allowed values.

## Per-Sub_Section Progress

When a Sub_Section item completes, ARIA updates the progress file with the new
completed count and a refreshed `last_updated` timestamp. When a whole Sub_Section
finishes, ARIA also logs a summary stating completed items versus total items.

Each Sub_Section entry in the progress file carries:

- **name** — the Sub_Section identifier
- **completed** — items finished so far
- **total** — items expected
- **status** — one of `pending`, `running`, `done`, or `error`

## Progress File Schema (Conceptual)

The JSON file holds a list of Sub_Sections, each with its name, completed count, total
count, and status, plus a top-level `last_updated` ISO 8601 timestamp that lets
consumers tell how fresh the snapshot is.

## Safe, Atomic Updates

The progress file is written **atomically**: ARIA writes to a temporary file and then
replaces the real file in one step. Pollers therefore never read a half-written file —
they always see either the previous complete snapshot or the new complete snapshot.

## Resilience

Progress reporting is best-effort and must never take the pipeline down. If writing
the progress file fails for any reason, ARIA logs a **warning** and continues
executing rather than crashing.

## Design Decisions

- **Dual output** (stdout + file) so logs survive terminal disconnects.
- **JSON progress separate from logs** to keep polling simple and cheap.
- **Runtime-configurable level** so verbosity changes need no code edits.
- **Atomic writes** so external pollers never see corrupted state.
- **No external log aggregation** — plain local files only, no extra services.

## What Users Don't Need to Know

- How the structured logger renders to console versus file
- How the atomic temp-file replacement is implemented
- How log rotation thresholds are enforced internally

## Dependencies

- Req 07 (Configuration) — supplies `logging.level` and `paths.logs_dir`
