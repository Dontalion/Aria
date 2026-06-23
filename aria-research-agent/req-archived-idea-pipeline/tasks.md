# Tasks: Idea Generation Pipeline

## Dependencies

- **Req 1** — LLM Provider (search-term generation, classification, idea generation)
- **Req 2** — Search Provider (web search execution)
- **keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)** — Keyword Enrichment Pipeline (shares `BasePipeline`; supplies the keyword output consumed for the same section)
- **Req 4** — State Management (pipeline state persistence and resumability)

## Task List

- [ ] 1. Implement `IdeaPipeline(BasePipeline)` skeleton and LangGraph wiring
  - File: `src/aria/pipelines/idea_generation.py`
  - Wire nodes: read_keywords → generate_search_terms → search_parallel → classify → retry → generate_ideas → deduplicate → write_batch
  - _Requirements: 4.1_

- [ ] 2. Implement `read_keywords` node
  - Read all keywords grouped by Section
  - If the file is missing, empty, or cannot be parsed into Sections, log an error naming the cause and terminate without writing any output file
  - _Requirements: 4.1, 4.2_

- [ ] 3. Implement `generate_search_terms` node
  - Call LLMProvider to produce exactly 3 search terms derived from each keyword
  - _Requirements: 4.3_

- [ ] 4. Implement `search_parallel` node
  - Execute all 3 searches concurrently via SearchProvider
  - _Requirements: 4.4_

- [ ] 5. Implement `classify` node
  - Call LLMProvider to label each result accepted/rejected by relevance to the configured research topic
  - _Requirements: 4.5_

- [ ] 6. Implement retry conditional edge
  - When more than half of results are rejected, regenerate one search term and re-execute the searches; cap at 1 retry per keyword
  - _Requirements: 4.6_

- [ ] 7. Implement zero-accepted guard
  - When no accepted results remain after the retry is exhausted, skip WorkflowOutput generation for that keyword, log the reason, and continue with the next keyword
  - _Requirements: 4.7_

- [ ] 8. Implement `generate_ideas` node with structured LLM output
  - From ≥1 accepted result, produce exactly 10 `IdeaRecord` instances using structured-output (Pydantic schema binding)
  - Ensure each record contains all fields of the configured schema (default: title, description, resume_worthiness naming ≥1 specific technology)
  - _Requirements: 4.8, 4.9_

- [ ] 9. Implement configurable `IdeaRecord` schema
  - Accept a schema dict at pipeline init and build the model via `create_model()`; fall back to default fields when none is provided
  - _Requirements: 4.9, 4.12_

- [ ] 10. Implement `deduplicate` node
  - Read existing titles from the target output file; drop any idea whose title already exists, using case-insensitive comparison
  - _Requirements: 4.11_

- [ ] 11. Implement finalization gating
  - Treat Reviewer ACCEPT as a quality gate only; write an idea only when it ALSO passes deduplication and schema conformance
  - _Requirements: 4.9, 4.11_

- [ ] 12. Implement `write_batch` node (max 5 records per append)
  - Split surviving ideas into chunks of ≤5; acquire the file lock before each append
  - _Requirements: 4.10_

- [ ] 13. Write unit tests
  - Cover 3-term generation, >50% retry (max 1), zero-accepted skip, exactly-10 idea generation, configurable schema override, case-insensitive dedup, and batch splitting (10 ideas → 2 batches of 5)
  - _Requirements: 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11_
