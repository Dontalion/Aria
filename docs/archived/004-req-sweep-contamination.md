# Archive: 004-req-sweep-contamination

## What Was Done
Conducted a comprehensive sweep across all `tasks.md` and `tech-spec.md` specification files in the renamed `aria-research-agent/` directories to remove contaminated hardcoded pipeline references. Renumbered all requirement cross-references (`Req X`) to match the new 17-requirement schema, and replaced legacy terms (`Keyword_Pipeline`, `Idea_Pipeline`, `Sub_Section`, `Idea_Record`) with workflow-agnostic terms (`Research_Workflow`, `Section`, `WorkflowOutput`).

## Files Changed
Every `tasks.md` and `tech-spec.md` under the following specification directories was modified:
- `aria-research-agent/req-01-llm-provider/`
- `aria-research-agent/req-02-search-provider/`
- `aria-research-agent/req-03-orchestrator/`
- `aria-research-agent/req-04-state/`
- `aria-research-agent/req-05-config/`
- `aria-research-agent/req-06-cli/`
- `aria-research-agent/req-07-output/`
- `aria-research-agent/req-08-error-handling/`
- `aria-research-agent/req-09-logging/`
- `aria-research-agent/req-10-extensibility/`
- `aria-research-agent/req-11-data-layer/`
- `aria-research-agent/req-12-review-agent/`
- `aria-research-agent/req-13-skills/`
- `aria-research-agent/req-14-self-aware/`
- `aria-research-agent/req-15-discovery/`
- `aria-research-agent/req-16-dynamic-agents/`
- `aria-research-agent/req-17-wiki/`
- `aria-research-agent/req-archived-keyword-pipeline/`
- `aria-research-agent/req-archived-idea-pipeline/`

## Decisions Made
- Shipped a script to safely parse and adjust sequential list sequences/ranges of requirement numbers (e.g., matching standard listings like `Req X.Y, Z.W`).
- Standardized replacement of old pipeline requirements (`Req 3` and `Req 4`) with inline markdown references to the new workflow plugins (`workflows/keyword-enrichment.md` and `workflows/idea-generation.md`).
- Confirmed that the glossary definitions align exactly with the updated requirement terminology.
