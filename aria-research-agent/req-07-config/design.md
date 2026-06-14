# Configuration Management — Design

## Purpose

The config system lets users customize ARIA without touching code. A single YAML
file controls the LLM provider, the search provider, parallelism, input/output
paths, and retry policy. Sensible defaults mean ARIA can run with almost zero
config — just set the credentials it needs through environment variables.

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
  api_key: ${OPENROUTER_API_KEY}

search:
  provider: duckduckgo            # duckduckgo | tavily | brave
  api_key: ${TAVILY_API_KEY}      # required for tavily and brave
  max_results: 5

execution:
  max_parallel: 3                 # whole number, 1 to 10
  retry_limit: 2                  # whole number, 0 to 5

paths:
  input_file: ./keywords.md
  output_dir: ./output

research_topic: AI engineering project ideas
```

## What Can Be Configured

| Section          | Controls                                                     |
|------------------|--------------------------------------------------------------|
| `llm`            | Provider, model, credentials, optional model parameters      |
| `search`         | Search provider and credentials                              |
| `execution`      | Maximum parallelism (1–10) and retry limit (0–5)             |
| `paths`          | Input file paths and the output directory structure          |
| `research_topic` | The topic injected into every generated prompt               |

## Supported Provider Values

- **LLM providers:** `openrouter`, `ollama`, `cli_tool`, `direct_api`
- **Search providers:** `duckduckgo`, `tavily`, `brave`

Anything outside these sets is rejected at load time (see Error Handling below).

## Environment Variable Support

Any value can reference an environment variable using `${ENV_VAR_NAME}` syntax. This
keeps secrets out of the config file (Req 7.4). At load time ARIA substitutes each
reference with the variable's value. If a referenced variable is **not set**, ARIA
reports the variable name and exits with a non-zero status before any research begins
(Req 7.8).

## Defaults

- Config path: `config.yaml` in the project root
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
3. A **referenced environment variable is unset** — the message names the variable.

Example messages:

```
Config error: required field 'research_topic' is missing
Config error: execution.max_parallel must be between 1 and 10. Got: 0
Config error: execution.retry_limit must be between 0 and 5. Got: 7
Config error: llm.provider must be one of [openrouter, ollama, cli_tool, direct_api]. Got: "opanai"
Config error: environment variable OPENROUTER_API_KEY is not set
```

## What Users Don't Need to Know

- How YAML is parsed and validated internally
- How environment-variable substitution is implemented
- How default values are merged with provided values

The config layer is invisible when it works, and loud and specific when it doesn't.
