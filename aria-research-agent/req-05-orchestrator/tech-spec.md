# Technical Specification: Orchestrator Graph

## Framework

LangGraph `StateGraph` with a `TypedDict` state schema, async execution, and checkpointing wired to the State_Store (Req 06). Concurrency is bounded by an `asyncio.Semaphore` sized from config.

## State Schema

```python
class OrchestratorState(TypedDict):
    tasks: list[dict]                    # planned task descriptors with status + deps
    current_section: str                 # section being processed
    results: dict[str, Any]              # accumulated outputs keyed by task id
    pending_user_decision: dict | None   # ambiguous-dependency prompt awaiting the user
    shutdown_flag: bool                  # stop accepting new work
```

`pending_user_decision`, when set, describes the two ambiguous sub-sections and the explanation shown to the user (Req 5.9); it is cleared once the user resolves the choice (Req 5.11).

## Graph Nodes

| Node | Responsibility |
|------|---------------|
| `plan_tasks` | Split the outline into task dicts with `status` + `dependencies` |
| `resolve_dependencies` | Classify each sub-section pair as independent / dependent / **ambiguous** (Req 5.8) |
| `await_user_decision` | Set `pending_user_decision`, halt scheduling of the pair, resume on user input (Req 5.9–5.11) |
| `execute_parallel` | Fan out ready tasks respecting `max_parallel` via `asyncio.Semaphore` (Req 5.1, 5.2, 5.5) |
| `check_dependencies` | Evaluate whether deferred tasks are now unblocked (Req 5.6, 5.7) |
| `write_output` | Collect results and write section output |
| `handle_errors` | Log failures, update state, decide retry or skip |

## Edges (Conditional Routing)

```
plan_tasks ──► resolve_dependencies
resolve_dependencies ──► await_user_decision   (if any pair is ambiguous)
resolve_dependencies ──► execute_parallel       (if all pairs resolved)
await_user_decision ──► execute_parallel         (after user decides)
execute_parallel ──► check_dependencies
check_dependencies ──► execute_parallel          (if more tasks ready)
check_dependencies ──► write_output              (if section complete)
write_output ──► END
Any node on error ──► handle_errors
handle_errors ──► check_dependencies             (continue remaining)
```

The router inspects `tasks` status fields, `pending_user_decision`, and `shutdown_flag`.

## Concurrency Control (Req 5.1–5.5)

```python
sem = asyncio.Semaphore(max_parallel)   # max_parallel from Req 07 Config

async def run_task(task):
    async with sem:
        return await dispatch(task)
```

`execute_parallel` selects ready tasks (`status == "pending"`, deps `completed`, not part of an unresolved ambiguous pair), launches them via `asyncio.gather` each guarded by the shared semaphore — so simultaneously running tasks never exceed `max_parallel` — and merges results into `state["results"]`; freed slots pull the next queued task.

## Ambiguous-Dependency Handling (Req 5.8–5.11)

```python
class DependencyVerdict(str, Enum):
    independent = "independent"
    dependent = "dependent"
    ambiguous = "ambiguous"

class PendingDecision(TypedDict):
    section_a: str
    section_b: str
    explanation: str          # plain-language reason the relationship is unclear
```

- `resolve_dependencies` produces a `DependencyVerdict` per sub-section pair:
  `independent` → run concurrently; `dependent` → ordered; `ambiguous` → neither runs until the user decides (no guess, no silent serialization).
- `await_user_decision` writes `pending_user_decision` and blocks scheduling of *both* sub-sections. The user's reply (`"concurrent"` | `"sequential"`) updates task dependencies, clears the field, and routes to `execute_parallel`.

## Configuration Validation (Req 5.3, 5.4)

```python
class ExecutionConfig(BaseModel):
    max_parallel: int = Field(3, ge=1)   # integer >= 1; invalid (<1) rejected at load
```

Loaded via Req 07. A value `< 1` raises a config error naming the field and exits before any task runs.

## Checkpointing Integration

State persists through the State_Store (Req 06). On resume the orchestrator reloads task state, skips `completed` tasks, and re-runs any task left `in_progress` (reset to `pending` by the State_Store on startup).

## File Layout

```
src/aria/orchestrator/
├── __init__.py
├── graph.py      # StateGraph definition, edge wiring, compile
├── state.py      # OrchestratorState TypedDict + PendingDecision helpers
└── nodes.py      # Node function implementations
```

## Key Dependencies

- `langgraph >= 0.2`
- `asyncio` (stdlib)
- `pydantic` for config validation (`max_parallel` from Req 07 Config)
- Internal: LLM provider (Req 01), Search provider (Req 02), StateStore (Req 06)
