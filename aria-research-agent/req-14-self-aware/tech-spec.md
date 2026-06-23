# Req 14 — Self-Aware Conversational Skill/Workflow Creation — Technical Spec

## Architecture

A Typer subcommand (`aria chat`) launches a multi-turn session. The `ChatSession`
drives the conversation through the LLM provider (Req 1), injecting a
`CapabilityRegistry` snapshot into the system prompt so every turn is grounded in what
ARIA already has. When the user describes a capability, the session runs the
**existence check** against the registry *before* any generation. New definitions are
produced by the `Generator`, validated against the Req 13 schema, then saved directly
to the skills/workflows directory.

## Capability Registry (Req 14.9, 14.10)

```python
# src/aria/registry.py
class CapabilityRegistry:
    def __init__(self, skills, workflows, agents, tools): ...
    def describe(self) -> dict:                      # {tools, skills, workflows, agents}
        ...                                          # used for self-description (Req 14.10)
    def find_skill(self, capability: str) -> SkillDefinition | None: ...      # existence check
    def find_workflow(self, capability: str) -> WorkflowDefinition | None: ...
    def installed_tools(self) -> set[str]: ...       # for tool-availability check (Req 14.8)
    def refresh(self) -> None: ...                   # re-read after add/modify/delete
```

The registry aggregates the Req 13 skill/workflow loaders, the Req 16 agent registry,
and the installed tool list. `describe()` returns a structured summary; `find_*` backs
the pre-generation existence check (Req 14.2, 14.5).

## Chat Session (Req 14.1, 14.2, 14.3, 14.5, 14.6, 14.10)

```python
# src/aria/chat/session.py
class ChatSession:
    def __init__(self, llm: LLMProvider, registry: CapabilityRegistry, gen: Generator): ...
    async def run(self) -> None: ...                 # multi-turn REPL loop
    async def handle_describe_request(self, intent) -> str:   # Req 14.10 self-summary
    async def handle_create_request(self, intent) -> None:
        existing = self.registry.find_skill(intent) or self.registry.find_workflow(intent)
        if existing:                                 # Req 14.3 / 14.5
            choice = await self.prompt_choice(["use-as-is", "modify", "variant"])
            return await self.apply_choice(choice, existing, intent)
        await self.create_new(intent)                # Req 14.4 / 14.5
    async def refine(self, target_file: Path, change) -> None: ...   # Req 14.6
```

Session context lives only for the current process (no cross-session memory). The
`describe` intent returns `registry.describe()` rendered as available skills, workflows,
and referenceable tools (Req 14.10).## Generator (Req 14.4, 14.7, 14.8)

```python
# src/aria/chat/generator.py
class Generator:
    MAX_RETRIES = 2                                  # 2 *additional* attempts (Req 14.7)
    async def generate(self, kind: Literal["skill", "workflow"], intent, registry):
        for attempt in range(self.MAX_RETRIES + 1):
            md = await self.llm.generate(build_prompt(kind, intent, registry))
            errors = validate_markdown(kind, md)     # Req 13 schema
            if errors:
                continue                             # regenerate
            missing = referenced_tools(md) - registry.installed_tools()
            if missing:                              # Req 14.8
                raise ToolNotInstalled(sorted(missing))   # reject, save nothing
            return md
        raise GenerationFailed(errors)               # Req 14.7 — error, save nothing
```

- **Schema retries (Req 14.7):** validation uses the Req 13 loaders. Failure triggers up
  to 2 additional regenerations; if it still fails, raise and write no file.
- **Tool gate (Req 14.8):** any referenced tool absent from `registry.installed_tools()`
  rejects the file naming the tool; nothing is saved.
- **Save + notify (Req 14.4):** a valid file is written to `skills/` or `workflows/` and
  the user is told the name and path. Saving relies on Req 13's hot-reload to register it.

## System Prompt (Req 14.9)

`src/aria/chat/prompts.py` exposes `build_system_prompt(registry)` and
`build_generation_prompt(kind, intent, registry)`. The system prompt embeds the registry
snapshot (installed tools, existing skills + descriptions, existing workflows +
descriptions, agents) plus the Req 13 schema rules, so both the existence check and
generation are registry-aware.

## Refinement (Req 14.6)

After creation, `refine()` edits the saved file in place for each requested change
(improve / trim / delete), re-validating edits and re-resolving tool references; a delete
request removes the file. There is **no approval-timeout or auto-discard** path — files
persist on creation and are iterated afterward.

## Files

| File | Responsibility |
|------|----------------|
| `src/aria/chat/session.py` | multi-turn chat loop, existence check, refine/delete |
| `src/aria/chat/generator.py` | Markdown generation, schema retries, tool gate, save |
| `src/aria/chat/prompts.py` | system + generation prompt construction (registry-aware) |
| `src/aria/registry.py` | `CapabilityRegistry` — tools/skills/workflows/agents |

## Dependencies

- Req 13 (Skills/Workflows) — schema, validation, on-disk format, hot-reload.
- Req 1 (LLM Provider) — conversation and Markdown generation.
- Req 16 (Dynamic Agents) — registry includes agents for self-description + checks.
