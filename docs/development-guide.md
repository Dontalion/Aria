# Development Guide — ARIA

## Package Manager

**uv** is the package manager for this project. Do NOT use pip or poetry.

```bash
# Install uv (if not installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create project
uv init aria
cd aria

# Add dependencies
uv add langgraph pydantic pydantic-ai httpx aiosqlite pyyaml typer rich

# Add dev dependencies
uv add --dev pytest pytest-asyncio ruff mypy

# Run the project
uv run python -m aria

# Run tests
uv run pytest
```

---

## Git Conventions

### Branch Strategy

- `main` — stable, always deployable
- `dev` — integration branch
- `feat/req-XX-short-name` — feature branches per requirement
- `fix/short-description` — bug fixes
- `refactor/short-description` — refactoring

### Commit Message Format

```
<type>(<scope>): <short description>

[optional body]

[optional footer]
```

**Types:**
| Type | When |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code restructuring (no behavior change) |
| `docs` | Documentation only |
| `test` | Adding/fixing tests |
| `chore` | Build, CI, tooling |
| `perf` | Performance improvement |

**Scope:** requirement number or module name (e.g., `req-01`, `providers`, `state`)

**Examples:**
```
feat(req-01): add OpenRouter LLM provider implementation
fix(state): handle SQLite lock on concurrent writes
docs(req-03): add tech-spec for keyword pipeline
refactor(providers): extract common retry logic to base class
test(req-02): add integration tests for Tavily search
```

### Commit Rules

1. One logical change per commit
2. Never commit secrets, API keys, or .env files
3. Run `uv run ruff check` before committing
4. Run `uv run pytest` before pushing
5. Keep commits small and reviewable

---

## Code Style

### Formatting & Linting

- **Ruff** for linting and formatting (replaces black + isort + flake8)
- Line length: 100 characters
- Python 3.11+ features allowed

### Type Hints

- ALL functions must have type hints (parameters + return)
- Use Pydantic models for data structures
- Use `typing` module for complex types

### Validation

- ALL external inputs validated with Pydantic
- Config validated on load
- API responses validated before processing
- Skill/workflow files validated on load

### Architecture Principles

1. **Dependency Injection** — providers injected, not hardcoded
2. **Interface Segregation** — small, focused interfaces
3. **Single Responsibility** — one class, one job
4. **Open/Closed** — extend via config/skills, not code changes
5. **Fail Loud** — errors logged clearly, never swallowed silently

---

## Database Best Practices

1. **Indexes** on frequently queried columns (task status, section_id, created_at)
2. **WAL mode** for SQLite (concurrent reads during writes)
3. **Connection pooling** for PostgreSQL
4. **Migrations** via Alembic or manual versioned SQL
5. **No ORM for hot paths** — raw SQL for performance-critical queries
6. **Batch operations** — bulk inserts/updates where possible
7. **Vacuum/analyze** scheduled for SQLite maintenance

---

## Testing Strategy

| Level | What | Tool |
|-------|------|------|
| Unit | Individual functions, providers | pytest |
| Integration | Pipeline end-to-end with mocked APIs | pytest + httpx mock |
| E2E | Full workflow with real (free) APIs | pytest (marked slow) |

---

## Task Completion Reports

After completing each task, create a report in `docs/archived/`:

**Naming:** `{index:03d}-{req-number}-{short-name}.md`

**Example:** `001-req-01-openrouter-provider.md`

**Template:**
```markdown
# Task Report: [Title]

- **Requirement:** Req-XX
- **Date:** YYYY-MM-DD
- **Status:** Completed

## What Was Done
[2-3 sentences]

## Files Changed
- `src/aria/providers/llm/openrouter.py` — created
- `tests/unit/test_openrouter.py` — created

## Decisions Made
[Any design decisions or trade-offs]

## Next Steps
[What depends on this, what's next]
```
