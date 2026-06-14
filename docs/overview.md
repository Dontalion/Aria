# Project Overview — Standalone Research Agent

## Project Name
**ARIA** (Autonomous Research & Idea Agent)

## What Is This?
A standalone Python multi-agent research system that automates research workflows — keyword enrichment, idea generation, source completion, and more. Built on LangGraph with provider-agnostic LLM and search backends.

---

## Repository Structure

```
aria/                                   ← Main project (implementation)
├── pyproject.toml                      ← Project config (uv package manager)
├── config.yaml                         ← Default configuration
├── README.md
├── src/
│   └── aria/
│       ├── __init__.py
│       ├── main.py                     ← CLI entry point
│       ├── config/                     ← Configuration management
│       │   ├── __init__.py
│       │   ├── schema.py              ← Pydantic config models
│       │   └── loader.py              ← YAML + env var loading
│       ├── providers/                  ← LLM & Search abstractions
│       │   ├── __init__.py
│       │   ├── base.py                ← Abstract interfaces
│       │   ├── llm/
│       │   │   ├── openrouter.py
│       │   │   ├── ollama.py
│       │   │   ├── cli_tool.py
│       │   │   └── direct_api.py
│       │   └── search/
│       │       ├── duckduckgo.py
│       │       ├── tavily.py
│       │       └── brave.py
│       ├── agents/                     ← Agent definitions
│       │   ├── __init__.py
│       │   ├── base.py                ← Base agent class
│       │   ├── executor.py            ← Main execution agent
│       │   ├── reviewer.py            ← Adversarial review agent
│       │   ├── searcher.py            ← Web search agent
│       │   └── dynamic.py             ← Dynamic agent factory
│       ├── pipelines/                  ← Research pipelines
│       │   ├── __init__.py
│       │   ├── keyword_enrichment.py
│       │   ├── idea_generation.py
│       │   └── base.py                ← Base pipeline class
│       ├── orchestrator/               ← LangGraph orchestration
│       │   ├── __init__.py
│       │   ├── graph.py               ← Main workflow graph
│       │   ├── state.py               ← Graph state definitions
│       │   └── nodes.py               ← Graph node implementations
│       ├── state/                      ← Persistence layer
│       │   ├── __init__.py
│       │   ├── database.py            ← SQLite/PostgreSQL
│       │   ├── models.py              ← SQLAlchemy/Pydantic models
│       │   ├── cache.py               ← Cache layer
│       │   └── migrations/
│       ├── output/                     ← Output formatting
│       │   ├── __init__.py
│       │   ├── formatter.py           ← Base formatter
│       │   ├── markdown.py
│       │   ├── json_out.py
│       │   ├── pdf.py
│       │   └── docx.py
│       ├── skills/                     ← Skill loading & management
│       │   ├── __init__.py
│       │   ├── loader.py
│       │   └── schema.py
│       ├── workflows/                  ← Workflow loading & management
│       │   ├── __init__.py
│       │   ├── loader.py
│       │   └── schema.py
│       ├── wiki/                       ← Per-research wiki
│       │   ├── __init__.py
│       │   ├── store.py
│       │   └── query.py
│       └── discovery/                  ← MCP/tool discovery
│           ├── __init__.py
│           ├── registry.py
│           └── integrator.py
├── skills/                             ← User-defined skill files (Markdown)
│   ├── keyword-enrichment.md
│   ├── idea-generation.md
│   └── review.md
├── workflows/                          ← User-defined workflow files (Markdown)
│   ├── full-enrichment.md
│   └── quick-ideas.md
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
└── docs/
    ├── overview.md
    ├── development-guide.md
    └── archived/                       ← Completed task reports
        └── .gitkeep
```

---

## Spec Structure

```

├── requirements.md                     ← All 19 requirements
├── overview.md                         ← This file
├── development-guide.md                ← Dev rules, git conventions
├── dependencies.md                     ← External dependencies list
├── framework-comparison.md             ← Framework decision document
├── req-01-llm-provider/
│   ├── design.md                       ← Non-technical design
│   ├── tech-spec.md                    ← Technical specification
│   └── tasks.md                        ← Implementation tasks
├── req-02-search-provider/
│   ├── design.md
│   ├── tech-spec.md
│   └── tasks.md
├── ... (req-03 through req-19)
└── docs/
    └── archived/                       ← Task completion reports
```

---

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | Python 3.11+ | AI ecosystem, async support |
| Framework | LangGraph | State management, parallel, checkpointing |
| Package Manager | uv | Speed, modern Python packaging |
| LLM (primary) | OpenRouter | 300+ models, one API key |
| Search (dev) | DuckDuckGo | Free, no API key needed |
| Search (prod) | Tavily / Brave | Structured results for agents |
| Database | SQLite → PostgreSQL | Simple start, scalable later |
| Validation | Pydantic / Pydantic AI | Type safety, schema validation |
| CLI | Click or Typer | Modern Python CLI |
| Config | YAML + env vars | Human-readable, secrets-safe |

---

## Implementation Phases

| Phase | Requirements | Goal |
|-------|-------------|------|
| 1 — Foundation | 1, 2, 7, 8, 11 | Providers + Config + CLI + Logging |
| 2 — Core Pipelines | 3, 4, 5, 6, 10 | Keyword + Idea + Parallel + State + Retry |
| 3 — Output & Quality | 9, 14 | Multi-format + Review Agent |
| 4 — Intelligence | 13, 15, 16, 19 | Data Layer + Skills + Self-Aware + Wiki |
| 5 — Ecosystem | 12, 17, 18 | Extensibility + Discovery + Dynamic Agents |
