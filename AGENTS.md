# ARIA — Agent Instructions

**ARIA** is a Python multi-agent research system built on LangGraph.
Source lives in `src/aria/`. Tests live in `tests/`. Docs live in `docs/`.

---

## Commands

```bash
uv run python -m aria          # run
uv run pytest                  # test
uv run ruff check .            # lint
uv run ruff format .           # format
uv run mypy src/               # type-check
```

Use **`uv`** exclusively. Never use `pip` or `poetry`.

---

## Hard Rules

### Never Do
- Commit or print API keys, secrets, or `.env` content.
- Put credentials in `config.yaml` or any config schema field.
- Put dynamic/runtime values (e.g. research topic) in `ConfigSchema`.
- Swallow exceptions silently — always raise or log with context.
- Use `print()` for application output — use `structlog`.
- Add dependencies without updating `pyproject.toml`.

### Always Do
- Add type hints to every function signature (parameters + return).
- Validate all external inputs with Pydantic before processing.
- Fail loud and early: validate config before any work starts.
- Use `uv run ruff check` and `uv run pytest` before any commit.
- Keep one logical change per commit.

---

## Architecture Constraints

| Concern | Rule |
|---------|------|
| **Providers** | Injected via `ConfigSchema`, never hardcoded. |
| **Credentials** | Read from env vars inside provider clients only, not in the loader. |
| **Config file** | `config.yaml` = static settings only (provider names, limits, paths). |
| **Runtime inputs** | Pass as function arguments through the call stack. |
| **Errors** | Field-level messages. Name the offending field or variable. |
| **Async** | All I/O-bound operations must be `async`. |
| **DB hot paths** | Raw SQL only — no ORM for performance-critical queries. |

---

## Module Map

```
src/aria/
  config/       schema.py, loader.py        — static config only
  providers/    base.py, llm/, search/      — all provider clients
  agents/       executor, reviewer, searcher — LangGraph agents
  pipelines/    keyword_enrichment, idea_generation
  orchestrator/ graph.py, state.py, nodes.py
  state/        database.py, models.py, cache.py
  output/       markdown, json, pdf, docx
  skills/       loader.py, schema.py
  workflows/    loader.py, schema.py
```

---

## Code Style

- **Line length:** 100 characters.
- **Pydantic v2** for all data structures.
- **`Literal[...]`** for every fixed-set field (provider names, etc.).
- **`Field(..., ge=N, le=M)`** for all bounded numeric fields.
- **`structlog`** for all logging — structured, JSON-compatible.
- **`tenacity`** for all retry logic — no hand-rolled retry loops.

---

## Commit Format

```
<type>(<scope>): <short description>
```

Types: `feat` `fix` `refactor` `docs` `test` `chore` `perf`  
Scope: requirement number or module name — e.g. `req-07`, `providers`, `state`

```
feat(req-01): add OpenRouter LLM provider
fix(state): handle SQLite lock on concurrent writes
test(req-07): add config validation unit tests
```

---

## After Each Task

Create `docs/archived/{index:03d}-{req}-{name}.md` with:
- What was done (2–3 sentences)
- Files changed
- Decisions made
