---
name: parallel-web-search
description: >
  Execute a list of search terms concurrently via the configured Search_Provider
  and return the raw results for each term.
inputs:
  - name: search_terms
    type: array
    required: true
    items:
      type: string
  - name: max_results
    type: integer
    required: false
    default: 5
outputs:
  - name: results_by_term
    type: array
    items:
      type: object
      properties:
        search_term: { type: string }
        results:     { type: array }
tools:
  - search_provider
---

This skill delegates to the configured Search_Provider. It runs all searches
in the `search_terms` list concurrently and waits for all to return before
producing output. The `parallel: true` parameter in the workflow step's `with:`
block enables this behavior (it is the default).
