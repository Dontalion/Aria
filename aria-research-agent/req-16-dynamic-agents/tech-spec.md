# Req 16 — Dynamic Agent Creation and Lifecycle — Technical Spec

## Architecture

Dynamic agents are produced by an `AgentFactory`, run by a dynamic runtime, and
tracked in the State_Store. The Orchestrator (Req 3) is the only component allowed to
create agents; created agents cannot create others.

```
task needs specialist → AgentFactory.propose() → present definition to user
   ├─ approved  → activate → DynamicAgentRuntime.run(scoped tools) → on task end: cleanup
   └─ rejected  → discard definition + notify (no activation)            (Req 16.2, 16.3)
```

## Models (Req 16.1, 16.5, 16.7)

```python
class AgentDefinition(BaseModel):
    id: str; name: str; role: str
    model: str                                     # e.g., "gpt-4o", "claude-sonnet"
    tools: list[str]                               # tool IDs, subset of user-granted perms
    system_prompt: str
    lifecycle: Literal["temporary", "persistent"]
    created_at: datetime
    task_id: str | None = None                     # linked task for temporary agents

class AgentConfig(BaseModel):
    max_concurrent_temporary: int = Field(5, ge=1)        # Req 16.4
    default_lifecycle: Literal["temporary", "persistent"] = "temporary"
    temporary_timeout_minutes: int = Field(30, ge=1)      # Req 16.8
    persistent_dir: str = "./agents"
```

## AgentFactory (Req 16.1–16.4) — `src/aria/agents/factory.py`

```python
class AgentFactory:
    def propose(self, task: Task, registry: CapabilityRegistry) -> AgentDefinition:
        # Only invoked when no existing agent in the registry covers the task (Req 16.1).
        defn = self._llm_design(task)                      # name/role/model/tools/prompt
        defn.tools = [t for t in defn.tools if t in self.user_granted_tools]  # Req 16.1
        return defn                                        # NOT yet activated

    async def create(self, defn: AgentDefinition, approved: bool) -> DynamicAgent:
        if not approved:                                   # Req 16.3 — discard + notify
            raise AgentCreationCancelled(defn.name)
        if defn.lifecycle == "temporary" and self.live_temporary() >= self.cfg.max_concurrent_temporary:
            raise TooManyTemporaryAgents(self.cfg.max_concurrent_temporary)  # Req 16.4
        agent = DynamicAgent(defn, tools=self._bind_tools(defn.tools))
        self.state.record_agent(defn)                      # Req 16.7 — audit row
        return agent
```

- **No recursion:** `DynamicAgent` is constructed without an `AgentFactory` reference,
  so it has no API to spawn sub-agents.
- **Tool scoping:** `_bind_tools` resolves only IDs in `defn.tools`, themselves filtered
  to the user's up-front grants. Any tool outside the definition is unreachable.

## Dynamic Runtime (Req 16.8) — `src/aria/agents/dynamic.py`

```python
class DynamicAgent:
    async def run(self, task: Task) -> Any:
        async with asyncio.timeout(self.cfg.temporary_timeout_minutes * 60):  # Req 16.8
            return await self._execute(task)               # tools limited to self.tools
    async def cleanup(self):                               # on task/pipeline end (Req 16.5)
        await self._release_resources()
```

On `TimeoutError`, the runtime destroys the agent, releases resources, marks the audit
row terminated, and notifies the user (Req 16.8).

## Lifecycle Management (Req 16.5, 16.6)

| Action  | Temporary                         | Persistent                          |
|---------|-----------------------------------|-------------------------------------|
| Create  | in-memory only                    | JSON in `persistent_dir`            |
| Execute | scoped to one task/pipeline       | available across sessions           |
| Promote | → persistent (save definition)    | n/a                                 |
| Demote  | n/a                               | → archived (kept, inactive)         |
| Delete  | auto-destroyed on task end/timeout | manual remove definition           |

```python
def change_lifecycle(self, agent_id: str, action: Literal["promote","demote","delete"]):
    match action:                                          # Req 16.6
        case "promote": self._save_persistent(agent_id)    # temporary → persistent
        case "demote":  self._archive(agent_id)            # persistent → archived
        case "delete":  self._remove(agent_id)             # persistent → removed
```

## State Tracking (Req 16.7)

One audit row per agent use: `agent_id`, `task_id`, `lifecycle`, `created_at` (plus
`terminated_at`/`termination_reason`). Persistent definitions live as JSON under
`persistent_dir`; the history is queryable for audit.

## Key Files

| File                          | Responsibility                                |
|-------------------------------|-----------------------------------------------|
| `src/aria/agents/models.py`   | `AgentDefinition`, `AgentConfig`              |
| `src/aria/agents/factory.py`  | proposal, approval gate, concurrency cap      |
| `src/aria/agents/dynamic.py`  | runtime, scoped tools, timeout                |
| State_Store `agent_history`   | audit rows (id, task id, lifecycle, timestamp)|

## Dependencies

- **Req 1 (LLM Provider)** — model backing each dynamic agent.
- **Req 4 (State Persistence)** — agent-to-task audit history.
- **Req 5 (Configuration)** — concurrency cap, default lifecycle, timeout, paths.
