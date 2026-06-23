---
name: generate-search-terms
description: >
  Given a seed keyword or input item and the configured research topic, produce
  a set of distinct search terms designed to surface diverse, relevant web results.
inputs:
  - name: seed_keyword
    type: string
    required: true
  - name: research_topic
    type: string
    required: true
  - name: terms_per_keyword
    type: integer
    required: false
    default: 3
outputs:
  - name: search_terms
    type: array
    items:
      type: string
prompt_template: |
  Research topic: {{ research_topic }}
  Seed keyword: {{ seed_keyword }}

  Generate exactly {{ terms_per_keyword }} distinct search terms derived from
  the seed keyword. Each term should surface a different angle or perspective
  on the topic. Return only the search terms, one per line.
tools: []
---
