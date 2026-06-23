# Tasks — Req 1: LLM Provider Abstraction

## Dependencies
- Req 5 (Config) — needs ConfigSchema and LLMConfig (provider type, model, params, timeout, retry limit)

## Tasks

- [ ] 1. Create LLMProvider abstract base class
  - Create `src/aria/providers/__init__.py` and `src/aria/providers/base.py`
  - Define ABC with `async generate(prompt: str, **kwargs) -> str` returning a text string and `healthcheck()`
  - Define `LLMProviderError` carrying provider name and attempt count
  - _Requirements: 1.1_

- [ ] 2. Implement OpenRouterProvider
  - Create `src/aria/providers/llm/openrouter.py`
  - Use `openai` SDK with `base_url="https://openrouter.ai/api/v1"`
  - Map config fields (model, temperature, max_tokens, request_timeout) to SDK params
  - _Requirements: 1.2_

- [ ] 3. Implement OllamaProvider
  - Create `src/aria/providers/llm/ollama.py`
  - Use `ollama` async client against the configured `base_url`
  - Handle connection to local instance
  - _Requirements: 1.3_

- [ ] 4. Implement CLIToolProvider
  - Create `src/aria/providers/llm/cli_tool.py`
  - Use `asyncio.create_subprocess_exec` for kiro-cli, codex, claude-code, gemini-cli
  - Capture stdout as response, stderr as error
  - _Requirements: 1.4_

- [ ] 5. Implement DirectAPIProvider
  - Create `src/aria/providers/llm/direct_api.py`
  - Support `anthropic` and `openai` native SDKs, routed on `config.sdk`
  - _Requirements: 1.5_

- [ ] 6. Create factory function
  - Create `src/aria/providers/llm/__init__.py`
  - `create_llm_provider(config: LLMConfig) -> LLMProvider`
  - Match on provider name, lazy import implementations, raise on unknown provider
  - _Requirements: 1.2, 1.3, 1.4, 1.5_

- [ ] 7. Add shared retry decorator with timeout handling
  - Trigger retry on connection error or exceeding `config.request_timeout` (default 60s, range 1–600s)
  - Exponential backoff (1s, 2s, 4s) up to `config.max_retries` (default 3, range 0–10)
  - On final failure raise `LLMProviderError` naming provider + attempt count; no fallback to another provider
  - _Requirements: 1.6, 1.7_

- [ ] 8. Define and validate LLMConfig bounds at load time
  - Add `LLMConfig` (in req-5 config schema) with `provider` Literal, temperature 0.0–2.0, max_tokens ≥ 1, request_timeout 1–600s, max_retries 0–10
  - Reject unsupported provider type or out-of-bounds parameter before starting any task, reporting the invalid field
  - _Requirements: 1.8, 1.9, 1.10_

- [ ] 9. Write unit tests
  - Test factory creates correct provider type per provider name
  - Test retry logic with mocked failures and timeout, asserting no provider fallback
  - Test `LLMProviderError` reports provider name and attempt count
  - Test each provider with mocked responses
  - Test config validation rejects bad provider and each out-of-bounds parameter
  - _Requirements: 1.1, 1.6, 1.7, 1.8, 1.9, 1.10_
