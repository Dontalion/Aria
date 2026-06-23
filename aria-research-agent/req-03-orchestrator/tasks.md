# Tasks: Parallel Execution & Orchestration

## Dependencies

- **Req 1** — LLM Provider (called inside execute nodes for generation/classification)
- **Req 2** — Search Provider (called inside execute nodes for web search)
- **Req 4** — State Management (StateStore checkpoint persistence; resume/skip logic)
- **Req 5** — Config (`ExecutionConfig.max_parallel`, validated at load time)

## Task List

- [ ] 1. Define `OrchestratorState` TypedDict and `PendingDecision` helpers
  - File: `src/aria/orchestrator/state.py`
  - Fields: `tasks: list[dict]`, `current_section: str`, `results: dict[str, Any]`, `pending_user_decision: dict | None`, `shutdown_flag: bool`
  - Add `DependencyVerdict` enum (`independent`, `dependent`, `ambiguous`) and `PendingDecision` TypedDict (`section_a`, `section_b`, `explanation`)
  - _Requirements: 5.8, 5.9_

- [ ] 2. Implement `plan_tasks` node
  - File: `src/aria/orchestrator/nodes.py`
  - Split the research outline into discrete task dicts, each carrying `status` and `dependencies`
  - Seed initial statuses (`pending`) and persist planned tasks through the StateStore
  - _Requirements: 5.1, 5.2_

- [ ] 3. Implement `resolve_dependencies` node
  - For each pair of Sections, classify the relationship as `independent`, `dependent`, or `ambiguous`
  - `independent` → eligible to run concurrently; `dependent` → ordered (downstream deferred until upstream completed and verified)
  - When any pair is `ambiguous`, do NOT guess and do NOT silently serialize — route to `await_user_decision`
  - _Requirements: 5.6, 5.7, 5.8_

- [ ] 4. Implement `await_user_decision` node (ambiguous-dependency user prompt flow)
  - Set `state["pending_user_decision"]` with the two sub-sections and a plain-language explanation of why the relationship is unclear
  - Halt scheduling of BOTH sub-sections while the decision is pending; start neither concurrently
  - On user reply (`"concurrent"` | `"sequential"`), update task dependencies accordingly, clear `pending_user_decision`, and route to `execute_parallel`
  - _Requirements: 5.9, 5.10, 5.11_

- [ ] 5. Implement `execute_parallel` node with `asyncio.Semaphore` concurrency cap
  - Create `sem = asyncio.Semaphore(max_parallel)` sized from `ExecutionConfig` (Req 5)
  - Select ready tasks (`status == "pending"`, deps `completed`, not part of an unresolved ambiguous pair)
  - Fan out via `asyncio.gather`, each task guarded by the shared semaphore so simultaneously running tasks never exceed `max_parallel`; queue extras and start them as slots free
  - Merge results back into `state["results"]`
  - _Requirements: 5.1, 5.2, 5.5_

- [ ] 6. Implement `check_dependencies` node
  - Evaluate deferred tasks and mark them ready once all dependency tasks are `completed` (output written and verified)
  - Keep dependent Idea tasks deferred until every Keyword task for the same section is completed and verified
  - Route back to `execute_parallel` while work remains, or to `write_output` when the section is complete
  - _Requirements: 5.6, 5.7_

- [ ] 7. Implement `write_output` node
  - Collect accumulated results and write the section output, then route to `END`
  - _Requirements: 5.1_

- [ ] 8. Implement `handle_errors` node
  - Log the failure, update task state, and decide retry or skip; route back to `check_dependencies` so unrelated tasks continue
  - _Requirements: 5.1, 5.2_

- [ ] 9. Validate `ExecutionConfig.max_parallel` at load time
  - Define `max_parallel: int = Field(3, ge=1)` (integer ≥ 1, default 3)
  - Reject a value `< 1` with a clear error naming the field and exit before any task runs
  - _Requirements: 5.3, 5.4_

- [ ] 10. Wire the graph in `graph.py`
  - File: `src/aria/orchestrator/graph.py`
  - Add nodes and conditional edges: `plan_tasks → resolve_dependencies`; `resolve_dependencies → await_user_decision` (if ambiguous) or `→ execute_parallel` (if all resolved); `await_user_decision → execute_parallel`; `execute_parallel → check_dependencies`; `check_dependencies → execute_parallel` or `→ write_output`; `write_output → END`; any node on error `→ handle_errors → check_dependencies`
  - Router inspects task statuses, `pending_user_decision`, and `shutdown_flag`; compile the graph
  - _Requirements: 5.6, 5.7, 5.8, 5.9, 5.10, 5.11_

- [ ] 11. Integrate checkpointing with the StateStore (resume logic)
  - Persist task state through the StateStore (Req 4); on resume, reload task state, skip `completed` tasks, and re-run any task left `in_progress` (reset to `pending` by the StateStore on startup)
  - _Requirements: 5.1_

- [ ] 12. Write integration tests
  - Mock LLM + search; assert parallel fan-out never exceeds `max_parallel` (semaphore cap)
  - Assert independent sub-sections run concurrently and dependent tasks wait for verified upstream completion
  - Assert ambiguous pairs set `pending_user_decision`, start neither sub-section while waiting, and schedule per the user's `concurrent`/`sequential` choice
  - Assert `max_parallel < 1` is rejected at load time; assert crash mid-run resumes from last checkpoint without redoing completed work
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9, 5.10, 5.11_

## Acceptance Criteria

- Graph executes tasks in parallel up to `max_parallel`, enforced by `asyncio.Semaphore`
- Dependent tasks wait until prerequisites complete with output written and verified
- Ambiguous dependencies prompt the user and block both sub-sections until resolved — no guessing, no silent serialization
- Invalid `max_parallel` (< 1) is rejected at load time before any task runs
- Crash mid-run → restart resumes from last checkpoint without re-doing completed work
- Errors in one task do not halt unrelated tasks
