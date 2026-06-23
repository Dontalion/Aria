# Req 19 — Per-Research Wiki with Selective Inclusion — Design

## Purpose

Every research project accumulates knowledge as it runs — key findings, decisions
made, sources consulted, approaches that were tried and rejected, and connections
between topics. Without somewhere to keep that knowledge, ARIA re-discovers the same
facts run after run. The Wiki is a **per-project knowledge base** that captures this
knowledge over time so future runs build on what's already known instead of starting
from scratch.

The user stays in control of what goes in. By default ARIA **proposes** entries and
waits for approval; the user can flip on an automatic mode when they trust the system
to curate on its own.

## What the Wiki Holds

The Wiki is stored as **structured Markdown or in the database** (configurable). Its
entries are categorized (Req 19.1):

- **Key findings**
- **Decisions made**
- **Sources consulted**
- **Rejected approaches**
- **Cross-references between topics**

Each entry carries a topic, content, optional source, a confidence score, tags, and a
timestamp. Every entry is bounded: its content **must not exceed the configured
maximum entry size** (default **2,000 characters**), keeping entries concise and
quotable rather than sprawling.

## Proposing Entries by Notability

Not everything a pipeline produces deserves a permanent place. ARIA uses a
**notability threshold** (a confidence score from 0.0 to 1.0, default **0.8**). When a
pipeline produces a finding whose confidence is **greater than or equal to the
threshold**, ARIA **proposes that finding as a single Wiki entry** (Req 19.2).
Findings below the threshold are not proposed.

## Selective Inclusion (Key Behavior)

How a proposed entry is handled depends on the **auto-wiki mode**:

- **Auto-wiki disabled (default).** ARIA proposes the entry and adds it **only after
  the user explicitly approves** it. If the user **rejects** it, ARIA **discards the
  proposed entry and leaves the Wiki unchanged** (Req 19.3).
- **Auto-wiki enabled.** ARIA adds proposed entries **without asking for
  confirmation** (Req 19.4).

This gives the user a curated knowledge base by default, with an opt-in fast path
when they want hands-off accumulation.

## Querying the Wiki

When the user asks "what do we know about X?", ARIA returns the **matching entries
ranked by relevance**, up to the **maximum query result count** (default **5**)
(Req 19.5). If a query matches **nothing**, ARIA returns an **empty result set with a
clear "no matching entries found" indication** rather than an error or a guess
(Req 19.9).

## Context Injection at Run Start

The Wiki earns its keep at the start of every run. When the **Orchestrator starts a
research run**, it **injects the relevant Wiki entries for the run's topic into the
agent prompts**, capped at the **maximum context entry count** (default **10**)
(Req 19.6). This is how ARIA avoids re-discovering known information — the agents
begin already aware of what the project has learned, without flooding the prompt.

## Size Limits and Archiving

The Wiki does not grow without bound. When the number of **active entries reaches the
configured maximum Wiki size**, ARIA **archives the oldest entries** so the active
size stays within the maximum (Req 19.10). Archived entries are retained but no longer
count toward the active set or context injection.

## Write Failures

If writing a proposed entry **fails**, ARIA **returns an error indicating the addition
failed and leaves the existing Wiki contents unchanged** (Req 19.8). A failed write is
never silent and never partially applied — the Wiki is always in a consistent state.

## Configuration

```yaml
wiki:
  location: "./aria_data/wiki"    # storage location (Req 19.7)
  backend: "markdown"             # "markdown" or "database" (Req 19.7)
  auto_wiki: false                # add without confirmation when enabled (Req 19.3, 19.4)
  notability_threshold: 0.8       # 0.0–1.0, min confidence to propose (Req 19.2)
  max_entry_size: 2000            # characters per entry (Req 19.1)
  max_query_results: 5            # results returned per query (Req 19.5)
  max_context_entries: 10         # entries injected at run start (Req 19.6)
  max_wiki_size: 1000             # active entries before archiving oldest (Req 19.10)
```

## Dependencies

- **Req 07 (Configuration)** — holds storage location, backend, auto-wiki mode,
  notability threshold, max entry size, max query results, max context entries, and
  max wiki size.
- **Req 13 (Data Layer)** — optional Vector_Store for semantic relevance ranking
  beyond keyword matching.
- **Req 05 (Orchestrator)** — injects relevant Wiki entries into agent prompts at the
  start of each research run.
