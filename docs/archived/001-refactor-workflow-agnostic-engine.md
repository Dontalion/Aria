# 001 — Refactor: Workflow-Agnostic Engine

## What Was Done

Removed hardcoded `Keyword_Pipeline` and `Idea_Pipeline` from the ARIA core engine specification. Both pipelines were previously encoded as system-level requirements (Req 3 and Req 4) with fixed thresholds, specific terminology (`Sub_Section`, `Idea_Record`), and contaminated downstream specs (Orchestrator, Config, Reviewer, Logging, Data Layer, Extensibility). The architecture now enforces a strict separation: the engine is a generic orchestrator that executes whatever workflow and skill files it loads at startup. Domain-specific behavior lives entirely in `workflows/` and `skills/` Markdown files.

## Files Changed

| File | Change |
|------|--------|
| `aria-research-agent/requirements.md` | Deleted Req 3 (Keyword Pipeline) and Req 4 (Idea Pipeline) as system requirements. Renumbered Req 5→3 through Req 19→17. Updated Glossary: replaced Keyword_Pipeline, Idea_Pipeline, Idea_Record, Sub_Section with WorkflowOutput, WorkflowPolicy, Section, Workflow_Step. Added architecture constraint banner. |
| `aria-research-agent/req-05-orchestrator/design.md` | Replaced Sub_Section/pipeline coupling language with generic Section/workflow-step language. Updated all Req X.Y cross-references to new numbering. |
| `aria-research-agent/req-06-state/design.md` | Added two-table model (Task + WorkflowOutput). Added `irrecoverable` status and crash-recovery behavior. Added `workflow_id`, `step_name`, `section`, `input_data` fields. |
| `aria-research-agent/req-07-config/design.md` | Added `skills_dir` and `workflows_dir` to config file schema. Replaced `input_file` with skill/workflow paths. Added WorkflowPolicy Precedence section (3-layer hierarchy + system key guard + conflict logging). |
| `aria-research-agent/req-11-logging/design.md` | Replaced Sub_Section/pipeline-start/keyword-processing language with Section/workflow-step/Task language. |
| `aria-research-agent/req-12-extensibility/design.md` | Fully rewritten. Replaced Idea_Record global config with per-step `outputs:` schema block. Replaced keyword/idea-step-specific template overrides with generic per-step `with:` template override. |
| `aria-research-agent/req-14-review-agent/design.md` | Fully rewritten. Reviewer now validates against active Workflow_Step `outputs:` schema at runtime. Removed all hardcoded Idea_Record and keyword/idea references. Activation is per-step (`review: true`). |
| `aria-research-agent/req-15-skills/design.md` | Updated Skill frontmatter schema to include explicit `prompt_template:` key. Updated Workflow example to use `needs:`, `with:`, `review:`, `outputs:` keys (Argo/Actions-aligned). Updated Req reference from 07 to 05. |
| `workflows/keyword-enrichment.md` | **NEW** — Workflow plugin file that replaces the former Keyword_Pipeline. All logic (search count, batch size, retry threshold, dedup scope) is declared here. |
| `workflows/idea-generation.md` | **NEW** — Workflow plugin file that replaces the former Idea_Pipeline. Output schema (title, description, resume_worthiness) is declared here. Review is activated per-step. |
| `docs/archived/001-refactor-workflow-agnostic-engine.md` | **THIS FILE** — Sprint record. |

## Decisions Made

1. **No hardcoded pipelines in the engine.** The `Keyword_Pipeline` and `Idea_Pipeline` are demoted from system requirements to workflow plugin files. They are the first-party examples of what a workflow looks like, not invariants.
2. **`Idea_Record` is not a Python class.** It is a runtime instance of whatever `outputs:` schema a workflow step declares. The term is retired from all specs.
3. **`Sub_Section` → `Section`.** Generalized. The engine does not know about keyword files or their structure.
4. **Workflow schema is Argo Workflows / GitHub Actions-aligned.** Keys: `name`, `skill`, `needs`, `with`, `review`, `outputs`. Learning curve is reduced because the pattern is familiar.
5. **Static skill resolution is fail-fast at startup; dynamic skill resolution is lazy-fail at execution time.** Dynamic workflows (generated via Req 14/15) may reference skills installed concurrently.
6. **Three-layer config precedence.** `with:` block > `config.yaml` > Pydantic defaults. System-level keys (provider, max_parallel, credentials) cannot be overridden in `with:` blocks.
7. **`irrecoverable` task status.** Tasks whose workflow file no longer exists on restart are set to `irrecoverable` instead of hanging or executing silently with a missing definition.
8. **Two-table persistence model.** Tasks table tracks lifecycle; WorkflowOutput table stores the actual generated records as opaque JSON linked by task_id. This makes the state store workflow-agnostic.
9. **Security sandboxing (VM isolation, dynamic policy elevation) deferred to Phase 2.**
