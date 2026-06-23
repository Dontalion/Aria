# Tasks — Req 10: Extensibility for New Research Topics

## Dependencies

- Req 5 (Configuration) — supplies and validates topic, subtopics, template
  overrides, and WorkflowOutput field definitions

## Tasks

- [ ] 1. Add topic and subtopics to the config schema
  - Add `research_topic` (non-empty string, 1–200 chars) and `subtopics`
    (list of 0–50 strings, each 1–200 chars) to the config models
  - Reject out-of-bounds topic/subtopics at load time with a field-naming error
  - _Requirements: 12.1_

- [ ] 2. Inject topic and subtopics into the prompt render context
  - Build a shared render context exposing `topic` and `subtopics`
  - Ensure every generated prompt receives this context so topic change is config-only
  - _Requirements: 12.1_

- [ ] 3. Implement the template loader with sandboxed Jinja2
  - Create `src/aria/templates/loader.py` using `SandboxedEnvironment` + `StrictUndefined`
  - Ship default templates in `templates/default/` for the keyword and idea steps
  - _Requirements: 12.2, 12.4_

- [ ] 4. Implement per-step override resolution
  - For each step (`keyword`, `idea`): use the custom template when provided,
    otherwise fall back to the shipped default
  - _Requirements: 12.3, 12.4_

- [ ] 5. Implement template load-time validation
  - Parse each resolved template; on syntax error reject with an error naming the template
  - Detect references to unsupplied variables and reject naming the template
  - Ensure no task processing begins on failure
  - _Requirements: 12.5_

- [ ] 6. Implement dynamic WorkflowOutput builder
  - Create `src/aria/schema/dynamic.py` with `FieldDef` and `build_idea_record()`
  - Build the model via `create_model()` from ordered field definitions
  - Apply default fields (title, description, resume-worthiness) when none defined
  - _Requirements: 12.6_

- [ ] 7. Implement WorkflowOutput structure validation
  - Reject when fields < 1 or > 20, name empty/missing, or duplicate names exist
  - Error names the offending field; no task processing begins on failure
  - _Requirements: 12.7_

- [ ] 8. Wire template loader and dynamic schema into the pipelines
  - Pass the resolved keyword/idea templates and built WorkflowOutput model into keyword-enrichment workflow (see `workflows/keyword-enrichment.md`)/idea-generation workflow (see `workflows/idea-generation.md`)
  - _Requirements: 12.2, 12.3, 12.6_

- [ ] 9. Write unit tests
  - Topic/subtopics bounds: valid, empty topic, 201-char topic, 51 subtopics, 201-char subtopic
  - Template override: custom used when present, default used when absent
  - Template validation: unparsable template and unsupplied variable both rejected naming the template
  - WorkflowOutput: default fields when none; reject 0 fields, 21 fields, empty name, 65-char name, duplicate name
  - _Requirements: 12.1, 12.3, 12.4, 12.5, 12.6, 12.7_

- [ ] 10. Integration test: topic switch with zero code changes
  - Change only `research_topic`/`subtopics` and confirm prompts and outputs adapt
  - _Requirements: 12.1, 12.2_
