# Tasks — Req 08: CLI Interface

## Dependencies
- Req 05 (Orchestrator) — `run`/`run-all` dispatch pipelines and honor the shutdown flag
- Req 06 (State) — `status`/`reset` read and clear persisted state; continuous persistence and resume
- Req 07 (Config) — loads config, resolves section identifiers and `--config` path

## Tasks

- [ ] 1. Create Typer app skeleton
  - Create `src/aria/main.py` with the Typer app
  - Create `src/aria/cli/__init__.py` with helpers
  - Add `[project.scripts] aria = "aria.main:app"` to `pyproject.toml`
  - _Requirements: 8.1_

- [ ] 2. Implement `run` subcommand
  - Accept a single existing section identifier and `--config` override
  - Execute only that section's pipeline; leave all other sections' state unchanged
  - _Requirements: 8.1, 8.2_

- [ ] 3. Implement `run-all` subcommand
  - Execute all configured pipelines across all sections
  - _Requirements: 8.1_

- [ ] 4. Implement `status` subcommand
  - For each configured section, print completed/total
  - _Requirements: 8.1, 8.3_

- [ ] 5. Implement `reset` subcommand
  - Accept a single existing section identifier
  - Clear persisted State_Store entries only for that section; leave other sections unchanged
  - _Requirements: 8.1, 8.4_

- [ ] 6. Handle unknown section for `run` and `reset`
  - When the section identifier is not in the loaded config, exit non-zero without modifying any persisted state, naming the unknown identifier
  - _Requirements: 8.5_

- [ ] 7. Implement `chat` subcommand (stub)
  - Placeholder for conversational skill/workflow creation (Req 16)
  - _Requirements: 8.1_

- [ ] 8. Implement headless / structured output
  - Detect TTY; emit structured log records (timestamp, level, event) when no TTY
  - Activate Rich tables/colors only when a TTY is detected
  - _Requirements: 8.6_

- [ ] 9. Wire continuous persistence into dispatch
  - Ensure each task status transition is committed before the next, so the database always reflects the most recent completed work
  - _Requirements: 8.7_

- [ ] 10. Add signal handling and graceful shutdown
  - Register SIGINT/SIGTERM handlers that set a shutdown flag (stop accepting new work)
  - Orchestrator checks the flag between tasks, finishes persisting in-progress state, and exits
  - Leave any unfinished task non-completed; no fixed grace window
  - _Requirements: 8.8_

- [ ] 11. Verify resume after interruption
  - On startup, skip completed tasks and re-run non-completed tasks, covering graceful and abrupt termination identically
  - _Requirements: 8.9_

- [ ] 12. Write unit tests
  - Test each subcommand with `typer.testing.CliRunner`
  - Test `run`/`reset` on an unknown section exit non-zero and make no state change
  - Test `run`/`reset` affect only the named section
  - Test `status` prints completed/total per section
  - Test TTY vs non-TTY output selection
  - Test SIGINT/SIGTERM set the flag and leave the unfinished task non-completed
  - _Requirements: 8.2, 8.3, 8.4, 8.5, 8.6, 8.8, 8.9_
