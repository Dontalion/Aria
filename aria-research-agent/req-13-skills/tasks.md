# Tasks — Req 13: Markdown-Defined Skills and Workflows

## Dependencies

- Req 5 (Configuration) — supplies the skills and workflows directory paths and
  their defaults (`skills/`, `workflows/`)

## Tasks

- [ ] 1. Add the skills/workflows path config
  - Read `skills_dir` (default `skills/`) and `workflows_dir` (default `workflows/`)
    from the Req 5 config
  - _Requirements: 15.1, 15.3_

- [ ] 2. Implement skill models
  - Create `src/aria/skills/models.py` with `SkillInput`, `SkillOutput`, and
    `SkillDefinition` (name, description, inputs, outputs, prompt_template,
    optional tool_requirements)
  - _Requirements: 15.2_

- [ ] 3. Implement workflow models
  - Create `src/aria/workflows/models.py` with `WorkflowStep` (id, skill, inputs
    mapping, depends_on) and `WorkflowDefinition` (name, description, ordered steps)
  - _Requirements: 15.4_

- [ ] 4. Implement `SkillLoader` loading
  - Create `src/aria/skills/loader.py`; load every `.md` in the skills directory,
    parsing YAML frontmatter with `python-frontmatter` and the body as the Jinja2
    prompt template
  - _Requirements: 15.1, 15.2_

- [ ] 5. Implement skill validation
  - Require non-empty name, non-empty description, input schema, output schema, and
    prompt template; treat tool requirements as optional
  - Emit each error naming the file path, the missing/invalid field, and the reason
  - _Requirements: 15.2, 15.5_

- [ ] 6. Implement `WorkflowLoader` loading
  - Create `src/aria/workflows/loader.py`; load every `.md` in the workflows
    directory, parsing name, description, and ordered steps with input/output
    mappings and dependency rules
  - _Requirements: 15.3, 15.4_

- [ ] 7. Implement workflow validation
  - Require a non-empty name and ≥1 ordered steps; validate each step references a
    loaded skill and has its input/output mappings and dependency rules
  - Emit each error naming the file path, the invalid field/reference, and the reason
  - _Requirements: 15.4, 15.5_

- [ ] 8. Implement unknown-skill and circular-dependency detection
  - Mark a workflow invalid when a step references a skill name not among the loaded
    skills, or when `depends_on` forms a cycle (topological sort / DFS)
  - Identify the source file path and the unresolved reference or circular dependency
  - _Requirements: 15.6_

- [ ] 9. Implement fail-fast startup
  - If any skill or workflow file fails validation, halt startup without processing
    any research request rather than running with invalid definitions
  - _Requirements: 15.7_

- [ ] 10. Implement hot-reload
  - Watch both directories with `watchdog`; on add/modify/remove, reload and
    re-validate the affected definitions within 5 seconds without a restart
  - _Requirements: 15.8_

- [ ] 11. Create example skill and workflow files
  - Add a sample skill (e.g. `summarize-source`) and a sample workflow (e.g.
    `deep-research`) demonstrating the frontmatter schema and step mappings
  - _Requirements: 15.2, 15.4_

- [ ] 12. Write unit tests
  - Skill missing each required field → invalid with file/field/reason; skill with no
    tools → valid
  - Workflow with zero steps, unknown skill reference, and circular dependency → invalid
  - Any failing file halts startup; all-valid files load
  - _Requirements: 15.2, 15.4, 15.5, 15.6, 15.7_

- [ ] 13. Write integration test for hot-reload
  - Add, modify, and remove a file while running; assert the affected definition
    reloads within 5 seconds without a restart
  - _Requirements: 15.1, 15.3, 15.8_
