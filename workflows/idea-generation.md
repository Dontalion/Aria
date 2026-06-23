---
name: idea-generation
description: >
  Generates structured research outputs (project ideas, summaries, analyses)
  from enriched keywords using web research. Produces WorkflowOutput records
  conforming to the configurable outputs schema, grouped by section.
version: "1.0"
steps:
  - name: generate_search_terms
    skill: generate-search-terms
    with:
      terms_per_keyword: 3      # exactly 3 search terms per keyword
    outputs:
      schema:
        type: array
        items:
          type: object
          properties:
            keyword:      { type: string, required: true }
            search_terms: { type: array,  items: { type: string }, required: true }

  - name: web_search
    skill: parallel-web-search
    needs: [generate_search_terms]
    with:
      parallel: true
    outputs:
      schema:
        type: array
        items:
          type: object
          properties:
            keyword:     { type: string, required: true }
            search_term: { type: string, required: true }
            results:     { type: array,  required: true }

  - name: classify_results
    skill: classify-search-results
    needs: [web_search]
    with:
      rejection_threshold: 0.50
      max_retries: 1
    outputs:
      schema:
        type: array
        items:
          type: object
          properties:
            keyword:  { type: string, required: true }
            accepted: { type: array,  required: true }
            rejected: { type: array,  required: true }
            reason:   { type: string, required: true }

  - name: generate_ideas
    skill: generate-structured-outputs
    needs: [classify_results]
    review: true                  # route every output record through the Reviewer Agent
    with:
      items_per_input: 10         # exactly 10 records per qualifying keyword
      batch_size: 5               # append in batches of ≤5; no partial batches
      dedup_field: title          # enforce title uniqueness per output file (case-insensitive)
      dedup_scope: file           # uniqueness enforced across the entire output file
    outputs:
      schema:
        type: array
        items:
          type: object
          properties:
            title:
              type: string
              required: true
              unique: true        # enforces case-insensitive dedup across the output file
            description:
              type: string
              required: true
            resume_worthiness:
              type: string
              required: true
              description: >
                Why this idea is resume-worthy; must name at least one specific technology.
---

<!-- Workflow notes -->

This workflow file replaces what was previously hardcoded as the `Idea_Pipeline`
in the ARIA engine. The engine is a generic orchestrator; all domain-specific logic
lives in this file.

**Dependency**: This workflow is designed to run after `keyword-enrichment.md`
for the same section. The Orchestrator enforces this by checking the `needs:` of
any step that declares it. In the default configuration, `keyword-enrichment` must
complete for a section before `idea-generation` is scheduled for the same section.
To declare this cross-workflow dependency, use the Orchestrator's section-level
dependency configuration.

**Review gate**: The `generate_ideas` step has `review: true`, meaning every
generated output record is routed through the Reviewer Agent before being written.
The Reviewer validates quality (ACCEPT/REVISE/REJECT) and the engine validates
structural conformance against the `outputs:` schema above. Both gates must pass.

**Custom schema**: To change the output structure (e.g. add a `tools_needed` field
or remove `resume_worthiness`), edit the `outputs.schema` block of `generate_ideas`
above. No Python code changes are needed.
