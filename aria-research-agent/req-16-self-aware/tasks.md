# Tasks — Req 16: Self-Aware Conversational Skill/Workflow Creation

## Dependencies

- Req 15 (Skills/Workflows) — schema, validation, on-disk format, and hot-reload
- Req 01 (LLM Provider) — drives the conversation and Markdown generation
- Req 18 (Dynamic Agents) — the registry also includes agents

## Tasks

- [ ] 1. Implement the `CapabilityRegistry`
  - Create `src/aria/registry.py` aggregating installed tools, existing skills,
    existing workflows, and agents
  - Expose `describe()`, `find_skill()`, `find_workflow()`, `installed_tools()`,
    and `refresh()`
  - _Requirements: 16.9_

- [ ] 2. Add the `aria chat` CLI subcommand
  - Wire a Typer subcommand that launches the multi-turn session
  - _Requirements: 16.1_

- [ ] 3. Implement the chat session loop
  - Create `src/aria/chat/session.py` with a multi-turn REPL that exchanges turns
    within a single session (no cross-session memory)
  - _Requirements: 16.1_

- [ ] 4. Build registry-aware system prompts
  - Create `src/aria/chat/prompts.py`; embed the registry snapshot (tools, skills,
    workflows, agents) and the Req 15 schema rules into the system and generation
    prompts
  - _Requirements: 16.9_

- [ ] 5. Implement the pre-generation existence check
  - Before generating, query the registry for a skill/workflow that already provides
    the described capability
  - _Requirements: 16.2, 16.5_

- [ ] 6. Implement the existing-capability choice flow
  - When a match exists, inform the user and offer use-as-is / modify / create a
    variant, then act on the choice
  - _Requirements: 16.3, 16.5_

- [ ] 7. Implement the generator
  - Create `src/aria/chat/generator.py`; generate skill Markdown (name, description,
    input/output schema, prompt template) and workflow Markdown (name, description,
    ordered steps)
  - _Requirements: 16.4, 16.5_

- [ ] 8. Implement schema-validation retries
  - Validate generated Markdown with the Req 15 loaders; on failure regenerate up to
    2 additional times, then report a validation error and save no file
  - _Requirements: 16.7_

- [ ] 9. Implement the uninstalled-tool gate
  - Reject a generated file that references a tool not in the installed tool list,
    name the unavailable tool, and save no file
  - _Requirements: 16.8_

- [ ] 10. Implement save-and-notify on success
  - Write a valid file to the skills/ or workflows/ directory and notify the user of
    the created name and path (registration via Req 15 hot-reload)
  - _Requirements: 16.4, 16.5_

- [ ] 11. Implement post-creation refinement
  - Allow continued conversation to improve, trim, or delete the saved file; apply
    each change to the file, re-validating edits (no approval-timeout/auto-discard)
  - _Requirements: 16.6_

- [ ] 12. Implement self-description
  - When asked to describe itself, summarize available skills, workflows, and
    referenceable tools from the registry
  - _Requirements: 16.10_

- [ ] 13. Write unit tests
  - Existence check returns an existing capability and offers the three choices
  - New capability generates, validates, saves, and notifies the path
  - Invalid generation retried twice then errors saving nothing
  - File referencing an uninstalled tool rejected naming the tool, saving nothing
  - Refine/trim/delete applied to the saved file; self-description summarizes registry
  - _Requirements: 16.2, 16.3, 16.4, 16.6, 16.7, 16.8, 16.9, 16.10_

- [ ] 14. Write integration test for end-to-end creation
  - Drive a multi-turn chat that describes a new skill, saves it, then refines it,
    asserting the on-disk file matches the conversation
  - _Requirements: 16.1, 16.4, 16.6_
