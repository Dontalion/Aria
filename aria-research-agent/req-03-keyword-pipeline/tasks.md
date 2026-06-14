# Tasks: Keyword Enrichment Pipeline

## Dependencies

- **Req 01** — LLM Provider (search-term generation, classification, keyword generation)
- **Req 02** — Search Provider (web search calls)
- **Req 06** — State Management (graph state persistence and resumability)

## Task List

- [ ] 1. Create `BasePipeline` ABC in `src/aria/pipelines/base.py`
  - Define abstract methods `build_graph()`, `run()`, `get_name()`
  - _Requirements: 3.1_

- [ ] 2. Implement `KeywordPipeline(BasePipeline)` skeleton and LangGraph wiring
  - File: `src/aria/pipelines/keyword_enrichment.py`
  - Wire nodes: read_keywords → generate_search_terms → search_parallel → classify_results → retry → generate_keywords → write_batch
  - _Requirements: 3.1_

- [ ] 3. Implement `read_keywords` node
  - Parse source file, collect keywords under the target Sub_Section into `KeywordInput` models
  - If the Sub_Section is absent or empty, log an error naming it and terminate that Sub_Section without creating or modifying any output file
  - _Requirements: 3.1, 3.2, 3.10_

- [ ] 4. Implement `generate_search_terms` node
  - Call LLMProvider to produce exactly 3 distinct `SearchTerm` per source keyword
  - _Requirements: 3.3_

- [ ] 5. Implement `search_parallel` node
  - Fan out the 3 searches concurrently via SearchProvider and wait for all 3 to return
  - _Requirements: 3.4_

- [ ] 6. Implement `classify_results` node
  - Call LLMProvider to assign each result exactly one label (accepted/rejected) against the research topic and record a reason
  - _Requirements: 3.5_

- [ ] 7. Implement retry conditional edge
  - When more than 50% of results are rejected, generate exactly 1 replacement term and run one more search + classify cycle; cap at 1 retry per keyword
  - _Requirements: 3.6_

- [ ] 8. Implement zero-accepted guard
  - When no accepted results remain after the retry is exhausted, skip keyword generation for that keyword, log the reason, and continue with the next keyword
  - _Requirements: 3.7_

- [ ] 9. Implement `generate_keywords` node
  - From ≥1 accepted result, produce exactly 10 new keywords, each distinct from source keywords and from keywords already generated in the same Sub_Section
  - _Requirements: 3.8_

- [ ] 10. Implement `write_batch` node
  - Append complete batches of ≤20 keywords per write to the separate output file; never write a partial batch; keep source files read-only
  - _Requirements: 3.9, 3.10_

- [ ] 11. Write unit tests for all nodes and retry logic
  - Cover 3-term generation, parallel search, classification, >50% retry (max 1), zero-accepted skip, exactly-10 distinct keywords, and ≤20 complete-batch writes
  - _Requirements: 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 3.10_
