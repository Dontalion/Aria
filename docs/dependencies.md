# Dependencies — ARIA (Autonomous Research & Idea Agent)

## Runtime Dependencies

| Package | Version | Purpose | Requirement |
|---------|---------|---------|-------------|
| `langgraph` | ^0.3 | Multi-agent orchestration, state management, checkpointing | Req 5, 6 |
| `langchain-core` | ^0.3 | Base abstractions for LangGraph | Req 5 |
| `pydantic` | ^2.9 | Data validation, config schemas, input/output models | All |
| `pydantic-ai` | ^0.1 | AI-specific validation, structured outputs | Req 3, 4, 14 |
| `httpx` | ^0.28 | Async HTTP client for API calls | Req 1, 2 |
| `aiosqlite` | ^0.20 | Async SQLite for state persistence | Req 6, 13 |
| `sqlalchemy` | ^2.0 | ORM + raw SQL, database migrations | Req 6, 13 |
| `alembic` | ^1.14 | Database schema migrations | Req 13 |
| `pyyaml` | ^6.0 | YAML config file parsing | Req 7 |
| `typer` | ^0.15 | CLI framework (Click-based, type-safe) | Req 8 |
| `rich` | ^13.9 | Terminal formatting, progress bars, tables | Req 8, 11 |
| `structlog` | ^24.4 | Structured logging (JSON + human-readable) | Req 11 |
| `tenacity` | ^9.0 | Retry logic with exponential backoff | Req 10 |
| `jinja2` | ^3.1 | Prompt template rendering | Req 12, 15 |

## LLM Provider Dependencies

| Package | Version | Purpose | When Needed |
|---------|---------|---------|-------------|
| `openai` | ^1.60 | OpenRouter + OpenAI direct API (OpenAI-compatible) | Req 1 — OpenRouter/OpenAI |
| `anthropic` | ^0.40 | Anthropic Claude direct API | Req 1 — Anthropic |
| `ollama` | ^0.4 | Local Ollama model integration | Req 1 — Ollama |

## Search Provider Dependencies

| Package | Version | Purpose | When Needed |
|---------|---------|---------|-------------|
| `duckduckgo-search` | ^7.0 | Free web search (no API key) | Req 2 — Development |
| `tavily-python` | ^0.5 | Tavily AI search API | Req 2 — Production |
| `brave-search` | ^0.3 | Brave Search API | Req 2 — Production alt |

## Output Format Dependencies

| Package | Version | Purpose | When Needed |
|---------|---------|---------|-------------|
| `python-docx` | ^1.1 | DOCX export | Req 9 — DOCX output |
| `weasyprint` | ^63.0 | PDF export from HTML/CSS | Req 9 — PDF output |
| `markdown` | ^3.7 | Markdown parsing/rendering | Req 9 — All formats |

## Optional Data Layer Dependencies

| Package | Version | Purpose | When Needed |
|---------|---------|---------|-------------|
| `chromadb` | ^0.5 | Vector store for semantic search | Req 13 — Vector Store |
| `redis` | ^5.2 | Distributed cache (optional upgrade from in-memory) | Req 13 — Cache Layer |
| `networkx` | ^3.4 | Local graph database (lightweight) | Req 13 — Graph DB |
| `neo4j` | ^5.26 | Production graph database | Req 13 — Graph DB (prod) |

## Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `pytest` | ^8.3 | Test framework |
| `pytest-asyncio` | ^0.24 | Async test support |
| `pytest-cov` | ^6.0 | Coverage reporting |
| `ruff` | ^0.8 | Linting + formatting (replaces black, isort, flake8) |
| `mypy` | ^1.13 | Static type checking |
| `respx` | ^0.22 | HTTP mocking for httpx |
| `factory-boy` | ^3.3 | Test data factories |

---

## Dependency Groups (uv)

```toml
[project]
name = "aria"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "langgraph>=0.3",
    "langchain-core>=0.3",
    "pydantic>=2.9",
    "pydantic-ai>=0.1",
    "httpx>=0.28",
    "aiosqlite>=0.20",
    "sqlalchemy>=2.0",
    "alembic>=1.14",
    "pyyaml>=6.0",
    "typer>=0.15",
    "rich>=13.9",
    "structlog>=24.4",
    "tenacity>=9.0",
    "jinja2>=3.1",
    "openai>=1.60",
    "duckduckgo-search>=7.0",
    "markdown>=3.7",
]

[project.optional-dependencies]
anthropic = ["anthropic>=0.40"]
ollama = ["ollama>=0.4"]
tavily = ["tavily-python>=0.5"]
brave = ["brave-search>=0.3"]
pdf = ["weasyprint>=63.0"]
docx = ["python-docx>=1.1"]
vector = ["chromadb>=0.5"]
redis = ["redis>=5.2"]
graph = ["networkx>=3.4"]
graph-prod = ["neo4j>=5.26"]
all = [
    "aria[anthropic,ollama,tavily,brave,pdf,docx,vector,redis,graph]"
]

[tool.uv]
dev-dependencies = [
    "pytest>=8.3",
    "pytest-asyncio>=0.24",
    "pytest-cov>=6.0",
    "ruff>=0.8",
    "mypy>=1.13",
    "respx>=0.22",
    "factory-boy>=3.3",
]
```

---

## Installation Commands

```bash
# Minimal (development with DuckDuckGo + OpenRouter)
uv sync

# With Tavily search
uv sync --extra tavily

# With PDF/DOCX export
uv sync --extra pdf --extra docx

# With vector store
uv sync --extra vector

# Everything
uv sync --extra all

# Dev tools
uv sync --dev
```

---

## System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Python | 3.11 | 3.12+ |
| RAM | 2 GB | 4 GB+ |
| Disk | 500 MB | 2 GB (for vector store) |
| Network | Required (for API calls) | Stable broadband |
| OS | Linux, macOS, Windows (WSL) | Linux |

---

## External Services (API Keys)

| Service | Required? | Env Variable | Free Tier |
|---------|-----------|-------------|-----------|
| OpenRouter | Yes (primary LLM) | `OPENROUTER_API_KEY` | Some free models |
| Ollama | No (local alternative) | — | Fully free (local) |
| DuckDuckGo | No (default search) | — | Free, no key |
| Tavily | No (production search) | `TAVILY_API_KEY` | 1000 credits/month |
| Brave Search | No (alt production search) | `BRAVE_API_KEY` | ~1000 queries/month |
| Anthropic | No (direct Claude) | `ANTHROPIC_API_KEY` | Pay-per-use |
| OpenAI | No (direct GPT) | `OPENAI_API_KEY` | Pay-per-use |

---

## Dependency Update Policy

1. Pin major versions in `pyproject.toml` (e.g., `>=0.3,<1.0`)
2. Run `uv lock --upgrade` monthly to get patch updates
3. Test after every dependency update before committing lock file
4. LangGraph updates may require migration — test thoroughly
