# LLM Provider Abstraction — Design

## Purpose

Allow ARIA users to swap LLM providers without changing any pipeline logic.
You pick a provider, set your API key, and the system handles the rest. Every
pipeline and agent stays ignorant of which backend runs underneath.

## Supported Providers (Priority Order)

1. **OpenRouter** (primary) — Access 300+ models through a single OpenAI-compatible API.
   One key, one endpoint, maximum flexibility.
2. **Ollama** (local) — Run models on your own machine. No API key needed.
   Good for offline work, privacy-sensitive tasks, or cost-free experimentation.
3. **CLI Tools** — Use existing AI tools you already have installed:
   kiro-cli, codex, claude-code, gemini-cli. ARIA calls them as subprocesses.
4. **Direct APIs** — Native SDKs for Anthropic and OpenAI when you want
   provider-specific features or already have those keys.

## How It Works (User Perspective)

1. Open `config.yaml`
2. Set your provider and model
3. Set the API key as an environment variable
4. Run ARIA — it connects automatically

That's it. No code changes, no imports to swap, no adapter wiring.

## Config Examples

```yaml
# OpenRouter (recommended default)
llm:
  provider: openrouter
  model: anthropic/claude-sonnet-4
  api_key_env: OPENROUTER_API_KEY
  temperature: 0.3          # valid range 0.0–2.0
  request_timeout: 60       # seconds, valid range 1–600
  max_retries: 3            # valid range 0–10

# Ollama (local, no key needed)
llm:
  provider: ollama
  model: llama3:8b
  base_url: http://localhost:11434

# CLI tool (uses installed binary)
llm:
  provider: cli_tool
  tool: claude-code
  args: ["--model", "sonnet"]

# Direct API (native SDK)
llm:
  provider: direct_api
  sdk: anthropic
  model: claude-sonnet-4-20250514
  api_key_env: ANTHROPIC_API_KEY
```

## What Every Request Returns

You send a prompt string, you get a completion text string back. The interface
is identical regardless of which provider is active.

## Timeout and Retry Behavior

- A request that returns a connection error, or that does not respond within the
  configured **request timeout** (default 60s, valid range 1–600s), triggers a retry.
- Retries use **exponential backoff** (1s, 2s, 4s) up to the configured retry
  limit (default 3, valid range 0–10).
- After all retries fail, ARIA marks the task as **failed** and reports a clear
  error stating which provider failed and how many attempts were made.
- **No silent fallback** to a different provider — explicit is better than implicit.

## Config Validation (Loud and Early)

When the LLM configuration is loaded, ARIA checks it before any pipeline task runs:

- The provider type must be one of `openrouter`, `ollama`, `cli_tool`, `direct_api`.
- `temperature` must be within 0.0–2.0.
- `max_tokens` must be ≥ 1.
- `request_timeout` must be within 1–600 seconds.
- `max_retries` must be within 0–10.

If the provider is unsupported, or any parameter falls outside its bounds, ARIA
**rejects the configuration at load time** and reports exactly which field is
invalid — without starting a single pipeline task.

## Design Decisions

- OpenRouter is the default because one key unlocks the widest model selection.
- Ollama covers offline and privacy-first use with zero API cost.
- CLI tools reuse binaries the user already trusts, no new credentials needed.
- No silent fallback keeps failures honest — you always know which provider broke.
- Validating at load time fails fast, so a typo never wastes an expensive run.

## What Users Don't Need to Know

- The abstract interface behind the scenes
- How subprocess communication works for CLI tools
- SDK version differences between providers
- Connection pooling or session management

The provider layer is invisible when it works, and loud when it doesn't.
