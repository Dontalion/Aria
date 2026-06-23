---
name: generate-structured-outputs
description: >
  Given a set of accepted research results and the configured research topic,
  generate a batch of structured WorkflowOutput records conforming to the
  calling workflow step's outputs schema. The exact fields are defined by the
  workflow step, not by this skill.
inputs:
  - name: accepted_results
    type: array
    required: true
  - name: research_topic
    type: string
    required: true
  - name: output_schema
    type: object
    required: true
    description: >
      The outputs.schema from the calling workflow step. Injected automatically
      by the engine. Defines what fields each record must contain.
  - name: items_per_input
    type: integer
    required: false
    default: 10
outputs:
  - name: records
    type: array
    description: >
      Each element is a WorkflowOutput record conforming to output_schema.
prompt_template: |
  Research topic: {{ research_topic }}
  Accepted research sources: {{ accepted_results }}
  Required output fields: {{ output_schema }}

  Generate exactly {{ items_per_input }} structured research outputs based on
  the sources above. Each output must contain all required fields defined in
  the output schema. Outputs must be distinct from each other.

  Return a JSON array of objects, one per output.
tools: []
---

> **Note for workflow authors**: The `output_schema` input is injected
> automatically by the ARIA engine from the calling step's `outputs:` block.
> You do not need to set it manually in the `with:` block.
