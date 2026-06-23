---
name: classify-search-results
description: >
  Evaluate each search result against the configured research topic and classify
  it as accepted or rejected. Records a short reason for each classification.
  Triggers one retry per input item when more than rejection_threshold fraction
  of results are rejected.
inputs:
  - name: results_by_term
    type: array
    required: true
  - name: research_topic
    type: string
    required: true
  - name: rejection_threshold
    type: number
    required: false
    default: 0.5
  - name: max_retries
    type: integer
    required: false
    default: 1
outputs:
  - name: classification
    type: object
    properties:
      accepted: { type: array }
      rejected: { type: array }
      reasons:  { type: object }
prompt_template: |
  Research topic: {{ research_topic }}

  For each search result below, classify it as "accepted" or "rejected" based
  on its relevance to the research topic. Record a one-sentence reason for
  each classification.

  Results:
  {{ results_by_term }}

  Return a JSON object with keys "accepted" (list of results), "rejected"
  (list of results), and "reasons" (dict mapping result index to reason string).
tools: []
---
