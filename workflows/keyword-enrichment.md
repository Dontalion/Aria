---
name: keyword-enrichment
description: >
  Expands a small set of seed keywords into a research-validated keyword set by
  using web search and intelligent classification. Produces new keywords grouped
  by the same section structure as the input file.
version: "1.0"
steps:
  - name: generate_search_terms
    skill: generate-search-terms
    with:
      terms_per_keyword: 3      # exactly 3 distinct search terms per seed keyword
    outputs:
      schema:
        type: array
        items:
          type: object
          properties:
            seed_keyword: { type: string, required: true }
            search_terms: { type: array, items: { type: string }, required: true }

  - name: web_search
    skill: parallel-web-search
    needs: [generate_search_terms]
    with:
      parallel: true            # all searches for one keyword run concurrently
    outputs:
      schema:
        type: array
        items:
          type: object
          properties:
            seed_keyword:  { type: string, required: true }
            search_term:   { type: string, required: true }
            results:       { type: array,  required: true }

  - name: classify_results
    skill: classify-search-results
    needs: [web_search]
    with:
      rejection_threshold: 0.50   # retry triggered when >50% are rejected
      max_retries: 1              # at most 1 retry per seed keyword
    outputs:
      schema:
        type: array
        items:
          type: object
          properties:
            seed_keyword: { type: string, required: true }
            accepted:     { type: array,  required: true }
            rejected:     { type: array,  required: true }
            reason:       { type: string, required: true }

  - name: generate_keywords
    skill: generate-keywords
    needs: [classify_results]
    review: false
    with:
      items_per_input: 10         # exactly 10 new keywords per qualifying seed keyword
      batch_size: 20              # append in batches of ≤20 keywords; no partial batches
      dedup_scope: section        # keywords must be distinct within the same section
    outputs:
      schema:
        type: array
        items:
          type: object
          properties:
            section:  { type: string, required: true }
            keyword:  { type: string, required: true }
---

<!-- Workflow notes -->

This workflow file replaces what was previously hardcoded as the `Keyword_Pipeline`
in the ARIA engine. The logic lives here, not in Python. The engine is a generic
orchestrator that executes whatever steps are declared above.

**Section isolation**: Each section in the input file is processed independently.
If a section is absent or contains zero keywords, the `generate_search_terms` step
logs an error identifying the missing section and skips it without writing output.

**Deduplication**: The `generate_keywords` step ensures that generated keywords are
distinct from both the seed keywords and all other keywords already generated within
the same section.

**Output format**: Keywords are written to a separate output file (never the source
file) organized by the same section structure as the input. The source file is
read-only throughout execution.
