# Configuration Management — Design

## Purpose

The config system lets users customize ARIA without touching code. A single YAML
file controls the LLM provider, search provider, parallelism, and input/output
paths. Sensible defaults mean ARIA can run with almost zero config.

Credentials and other secrets are managed **exclusively** through environment
variables (a `.env` file or the shell environment). They are **not** fields in
`config.yaml`.

Dynamic, per-run values such as the research topic are **not** part of the config
file either — they are provided at runtime via CLI arguments or the calling context.

## Separation of Concerns

| Source                        | What it controls                                                   |
|-------------------------------|--------------------------------------------------------------------|
| `config.yaml`                 | Static settings: provider names, parallelism, retry, paths         |
| Environment variables / `.env`| Secrets: API keys, tokens, credentials                             |
| CLI arguments / runtime input | Dynamic values: research topic, per-run overrides                  |

This boundary means `config.yaml` is safe to commit to version control, the `.env`
file is never committed (it is in `.gitignore`), and runtime inputs change without
touching any file.

## Config File Location

ARIA loads configuration from a YAML file at a configurable path. The default is
`config.yaml` in the project root (Req 7.1). You can point ARIA at a different file
with the `--config` CLI flag.

When **no file exists** at the resolved path, ARIA does **not** stop. It initializes
using documented default values and continues (Req 7.6). A missing config file is a
normal, supported way to run.

## File Format (YAML)

```yaml
llm:
  provider: openrouter            # openrouter | ollama | cli_tool | direct_api
  model: anthropic/claude-sonnet-4

search:
  provider: duckduckgo            # duckduckgo | tavily | brave
  max_results: 5

execution:
  max_parallel: 3                 # whole number, 1 to 10
  retry_limit: 2                  # whole number, 0 to 5

paths:
  input_file: ./keywords.md
  output_dir: ./output
```

Notice that **no API keys appear in this file**. Keys are read from environment
variables at runtime (see Environment Variables below).

## What Can Be Configured

| Section     | Controls                                                       |
|-------------|----------------------------------------------------------------|
| `llm`       | Provider name and model identifier                             |
| `search`    | Search provider name and maximum results per query             |
| `execution` | Maximum parallelism (1–10) and retry limit (0–5)               |
| `paths`     | Input file path and output directory                           |

## Supported Provider Values

- **LLM providers:** `openrouter`, `ollama`, `cli_tool`, `direct_api`
- **Search providers:** `duckduckgo`, `tavily`, `brave`

Anything outside these sets is rejected at load time (see Error Handling below).

## Environment Variables

Credentials are read directly from the environment — never from `config.yaml`.
ARIA looks for the following variables:

| Variable               | Used by                        | Required when              |
|------------------------|--------------------------------|----------------------------|
| `OPENROUTER_API_KEY`   | LLM provider `openrouter`      | provider is `openrouter`   |
| `TAVILY_API_KEY`       | Search provider `tavily`       | provider is `tavily`       |
| `BRAVE_API_KEY`        | Search provider `brave`        | provider is `brave`        |

If a required variable is **not set**, ARIA reports the variable name and exits
with a non-zero status before any research begins (Req 7.8).

A `.env` file in the project root is the recommended way to set these locally.
It must not be committed to version control.

## Runtime Inputs (Dynamic Values)

Values that change with every run — such as the research topic — are not stored in
`config.yaml`. They are supplied at runtime:

- **Research topic**: passed as a CLI argument (`--topic "…"`) or read from the
  input file specified by `paths.input_file`.
- Other per-run overrides may be added as CLI flags in future requirements.

This keeps `config.yaml` stable between runs and avoids accidental topic leakage
across sessions.

## Defaults

- Config path: `config.yaml` in the project root
- LLM provider: `openrouter`
- Parallelism: `3` simultaneous tasks
- Retry limit: `2` retries
- Search results: `5`
- Output: written under `./output`

Any value not explicitly set falls back to its documented default (Req 7.5).

## Error Handling (Loud and Early)

ARIA validates configuration **before** starting any research. It exits with a
non-zero status code in three cases, each with a field-level message (Req 7.3, 7.7,
7.8):

1. A **required value is missing** — the message names the missing field.
2. A **value is out of bounds** or outside its allowed set — the message names the
   field and the expected constraint.
3. A **required environment variable is unset** — the message names the variable.

Example messages:

```
Config error: llm.provider is required
Config error: execution.max_parallel must be between 1 and 10. Got: 0
Config error: execution.retry_limit must be between 0 and 5. Got: 7
Config error: llm.provider must be one of [openrouter, ollama, cli_tool, direct_api]. Got: "opanai"
Config error: environment variable OPENROUTER_API_KEY is not set
```

## What Users Don't Need to Know

- How YAML is parsed and validated internally
- How environment variables are resolved at runtime
- How default values are merged with provided values

The config layer is invisible when it works, and loud and specific when it doesn't.
