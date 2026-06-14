# Search Provider Abstraction — Design

## Purpose

Allow ARIA to swap web search backends without changing any pipeline logic.
The research agents don't know or care which search engine runs underneath —
they just ask for results and get them back in a consistent format.

## Supported Providers

| Provider | Use Case | API Key Required |
|----------|----------|-----------------|
| DuckDuckGo | Development & testing | No (free) |
| Tavily | Production (structured AI results) | Yes |
| Brave Search | Production alt (independent index, privacy-first) | Yes |

## How It Works (User Perspective)

1. Open `config.yaml`
2. Set the provider name
3. Add an API key if the provider requires one
4. Done — ARIA uses that provider for all searches

No code changes. No restarts beyond reloading config.

## Config Example

```yaml
search:
  provider: "tavily"        # Options: duckduckgo, tavily, brave
  api_key: "${TAVILY_API_KEY}"   # Required for tavily and brave; blank for duckduckgo
  max_results: 5
```

## What Every Search Returns

Each search call returns a list of **0 up to `max_results`** results (default 5).
Every result contains:

- **title** — page title, always non-empty
- **url** — full link to the source
- **snippet** — short text excerpt describing the page content

The format is identical regardless of which provider is active. A query with no
matches simply returns an empty list rather than an error.

## Retry Behavior

If a search request fails (network error or error response):

1. Wait 3 seconds
2. Retry once
3. If the retry **succeeds**, return its results
4. If the retry also fails, return an empty result set (0 results)

The pipeline continues gracefully — no crashes from search failures.

## Config Validation (Loud and Early)

When the search configuration is loaded, ARIA checks it before any search runs:

- The provider must be one of `duckduckgo`, `tavily`, `brave`.
- An API key credential must be present for **Tavily** and **Brave Search**.

If the provider is unsupported, or a required API key is missing, ARIA logs an
error identifying the invalid field and **exits before performing any search**.
DuckDuckGo needs no key, so it is exempt from the credential check.

## Design Decisions

- DuckDuckGo is the default so new users can run ARIA immediately with zero setup.
- Tavily is recommended for production because its results are pre-structured for AI.
- Brave is offered as a privacy-conscious alternative with its own independent index.
- One retry keeps things simple; more retries would slow down the research loop.
- Empty results on failure let downstream agents handle missing data gracefully.
- Validating the provider and credentials up front fails fast, so a misconfigured
  key never wastes a run or surfaces as a confusing mid-search error.
