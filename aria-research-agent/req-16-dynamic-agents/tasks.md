# Req 16 — Dynamic Agent Creation and Lifecycle — Tasks

## Dependencies

- Req 1 (LLM Provider) — model backing each dynamic agent
- Req 4 (State Persistence) — agent-to-task audit history
- Req 5 (Configuration) — concurrency cap, default lifecycle, timeout, paths

## Tasks

- [ ] 18.1 Define `AgentDefinition` and `AgentConfig` Pydantic models in `src/aria/agents/models.py` (name, role, model, tools, prompt, lifecycle, created_at, task_id) _(Req 16.1, 16.5)_
- [ ] 18.2 Implement `AgentFactory.propose()` in `src/aria/agents/factory.py` — create a definition only when no existing agent covers the task; design name/role/model/tools/prompt via LLM _(Req 16.1)_
- [ ] 18.3 Enforce tool scoping: filter the proposed tool list to the user's up-front granted permissions; bind only those tools at runtime _(Req 16.1)_
- [ ] 18.4 Enforce no recursive spawning: construct `DynamicAgent` without factory access so agents cannot create sub-agents _(Req 16.1)_
- [ ] 18.5 Implement the approval gate: present the full definition and withhold activation until explicit user approval _(Req 16.2)_
- [ ] 18.6 Implement rejection path: do not activate, discard the proposed definition, notify cancellation _(Req 16.3)_
- [ ] 18.7 Enforce concurrency cap: reject creation that would exceed 5 concurrent temporary agents, leave existing agents unchanged, notify the user _(Req 16.4)_
- [ ] 18.8 Implement `DynamicAgent` runtime in `src/aria/agents/dynamic.py` with task-scoped execution and tool restriction _(Req 16.1, 16.5)_
- [ ] 18.9 Implement lifecycle changes: promote temporary→persistent (save), demote persistent→archived, delete persistent (remove) _(Req 16.5, 16.6)_
- [ ] 18.10 Implement temporary-agent cleanup on task/pipeline completion _(Req 16.5)_
- [ ] 18.11 Implement timeout enforcement (default 30 min): destroy the agent, release resources, notify the user _(Req 16.8)_
- [ ] 18.12 Record audit rows in the State_Store: agent id, task id, lifecycle mode, creation timestamp _(Req 16.7)_
- [ ] 18.13 Add the `agents` config section (max_concurrent_temporary, default_lifecycle, temporary_timeout_minutes, persistent_dir) with validation _(Req 16.4, 16.5, 16.8)_
- [ ] 18.14 Write unit tests for factory proposal, approval/rejection, concurrency cap, tool scoping, and lifecycle transitions _(Req 16.1–16.6)_
- [ ] 18.15 Write integration test: create temporary agent, execute task, verify cleanup, audit row, and timeout destruction _(Req 16.7, 16.8)_
