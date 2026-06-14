# Tasks — Req 02: Search Provider Abstraction

## Dependencies
- Req 07 (Config) — needs ConfigSchema and SearchConfig (provider, api_key, max_results)

## Tasks

- [ ] 1. Create SearchProvider ABC and SearchResult model
  - Create `src/aria/providers/search/base.py`
  - Define `SearchResult(BaseModel)` with non-empty `title`, `url`, `snippet`
  - Define ABC with `async search(query, max_results=5) -> list[SearchResult]` returning 0..max_results
  - _Requirements: 2.1_

- [ ] 2. Implement DuckDuckGoProvider
  - Create `src/aria/providers/search/duckduckgo.py`
  - Use `duckduckgo-search` library (`AsyncDDGS`)
  - Map results to SearchResult model; no API key needed
  - _Requirements: 2.2_

- [ ] 3. Implement TavilyProvider
  - Create `src/aria/providers/search/tavily.py`
  - Use `tavily-python` SDK with the configured API key
  - Map response to SearchResult model
  - _Requirements: 2.3_

- [ ] 4. Implement BraveSearchProvider
  - Create `src/aria/providers/search/brave.py`
  - Use `httpx` direct HTTP to the Brave Search API with the configured API key
  - Map JSON `web.results[]` to SearchResult model
  - _Requirements: 2.4_

- [ ] 5. Create factory function
  - Create `src/aria/providers/search/__init__.py`
  - `create_search_provider(config: SearchConfig) -> SearchProvider`
  - Match on provider name, raise on unknown provider
  - _Requirements: 2.2, 2.3, 2.4_

- [ ] 6. Add retry wrapper
  - 1 retry after a 3-second delay on network error or error response
  - Return the retry's results on success; return an empty (0-result) list on final failure
  - _Requirements: 2.5, 2.6_

- [ ] 7. Define and validate SearchConfig at load time
  - Add `SearchConfig` (in req-07 config schema) with `provider` Literal (duckduckgo, tavily, brave) and `api_key`
  - Require an API key credential for Tavily and Brave; reject unsupported provider or missing key, log the invalid field, and exit before any search
  - _Requirements: 2.7, 2.8, 2.9_

- [ ] 8. Write unit tests
  - Test factory creates correct provider per provider name
  - Test retry returns empty list on double failure and returns results on retry success
  - Test each provider with mocked responses
  - Test SearchResult validation rejects empty titles and caps results at max_results
  - Test config validation rejects unsupported provider and missing Tavily/Brave key
  - _Requirements: 2.1, 2.5, 2.6, 2.7, 2.8, 2.9_
