# Tasks — Req 5: Configuration Management

## Dependencies
- None (foundation — no upstream requirements)

## Tasks

- [ ] 1. Create config package skeleton
  - Create `src/aria/config/__init__.py`
  - Add core dependencies: `pydantic`, `pyyaml`
  - _Requirements: 7.1_

- [ ] 2. Implement Pydantic config schema
  - Create `src/aria/config/schema.py`
  - Define `ConfigSchema`, `LLMConfig`, `SearchConfig`, `ExecutionConfig`, `PathsConfig`
  - LLM provider `Literal["openrouter", "ollama", "cli_tool", "direct_api"]`
  - Search provider `Literal["duckduckgo", "tavily", "brave"]`
  - `execution.max_parallel` integer 1–10 (default 3), `execution.retry_limit` integer 0–5
  - **Do not include API keys or credentials in the schema** — these belong in the environment
  - **Do not include `research_topic` or any per-run value in the schema** — these are runtime inputs
  - _Requirements: 7.2, 7.5_

- [ ] 3. Implement config loader with default path resolution
  - Create `src/aria/config/loader.py`
  - Resolve path from `--config` flag, defaulting to `./config.yaml`
  - When no file exists at the resolved path, build config from documented defaults without terminating
  - Loader does **not** perform `${ENV_VAR}` substitution — no env var references in config files
  - _Requirements: 7.1, 7.5, 7.6_

- [ ] 4. Implement environment variable resolution in provider clients
  - Each provider client (LLM, search) reads its required environment variable at initialisation
  - When a required variable is unset, raise a field-level `ConfigError` naming the variable and exit non-zero before any research
  - Variables to resolve: `OPENROUTER_API_KEY`, `TAVILY_API_KEY`, `BRAVE_API_KEY`
  - _Requirements: 7.4, 7.8_

- [ ] 5. Wire validation and field-level errors
  - Convert Pydantic `ValidationError` into a message naming the offending field
  - Missing required value → name the missing field, exit non-zero before any research
  - Out-of-bounds or out-of-set value → name the field and the expected constraint, exit non-zero before any research
  - _Requirements: 7.3, 7.7_

- [ ] 6. Add documented example config file
  - Create `config.yaml.example` with every section and field documented
  - File contains **no credentials** — add a comment directing users to set API keys via `.env`
  - Defaults align with schema (max_parallel 3, retry_limit 2, search max_results 5)
  - Add `.env.example` listing all required environment variables with placeholder values
  - _Requirements: 7.5_

- [ ] 7. Write unit tests
  - Test schema validation for valid input and each out-of-bounds value (max_parallel 0/11, retry_limit -1/6)
  - Test unsupported LLM and search provider values are rejected with a field-level error
  - Test missing required field produces a field-naming error and non-zero exit
  - Test missing config file resolves to documented defaults without terminating
  - Test each provider client raises `ConfigError` when its required env var is absent
  - Test each provider client initialises correctly when its env var is set
  - _Requirements: 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8_
