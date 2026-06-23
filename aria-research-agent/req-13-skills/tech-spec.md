# Req 13 — Markdown-Defined Skills and Workflows — Technical Spec

## Architecture

Skills and workflows are plain Markdown files with YAML frontmatter, loaded from
two configurable directories (defaults `skills/` and `workflows/`, supplied by
Req 5). At startup the loaders read **every** `.md` file in each directory, parse
frontmatter with `python-frontmatter`, and build validated Pydantic models. The
Markdown body becomes a Jinja2 prompt template. A filesystem watcher (`watchdog`)
keeps the in-memory definitions in sync while ARIA runs.

```
config (Req 5) → SkillLoader.load_all() → {name: SkillDefinition}
                → WorkflowLoader.load_all(skills) → {name: WorkflowDefinition}
                → validate all → any failure → halt startup
watchdog events (create/modify/delete) → reload affected defs within 5s
```

## Models (Req 13.2, 13.4)

```python
# src/aria/skills/models.py
class SkillInput(BaseModel):
    name: str                       # non-empty
    type: str = "string"            # "string" | "list" | "dict"
    required: bool = True

class SkillOutput(BaseModel):
    name: str                       # non-empty
    type: str = "string"

class SkillDefinition(BaseModel):
    name: str                       # non-empty (Req 13.2)
    description: str                # non-empty (Req 13.2)
    inputs: list[SkillInput]        # input schema (required)
    outputs: list[SkillOutput]      # output schema (required)
    prompt_template: str            # Markdown body, Jinja2 (required)
    tool_requirements: list[str] = []   # OPTIONAL — empty is valid

# src/aria/workflows/models.py
class WorkflowStep(BaseModel):
    id: str
    skill: str                      # must reference a loaded SkillDefinition.name
    inputs: dict[str, str] = {}     # input/output mappings between steps
    depends_on: list[str] = []      # dependency rules

class WorkflowDefinition(BaseModel):
    name: str                       # non-empty (Req 13.4)
    description: str
    steps: list[WorkflowStep]       # ordered, length ≥ 1 (Req 13.4)
```

## Loaders (Req 13.1, 13.3)

```python
# src/aria/skills/loader.py
class SkillLoader:
    def __init__(self, skills_dir: Path): ...
    def load_all(self) -> dict[str, SkillDefinition]: ...   # every *.md
    def load_file(self, path: Path) -> SkillDefinition: ...  # frontmatter + body
    def validate(self, skill: SkillDefinition, path: Path) -> list[str]: ...

# src/aria/workflows/loader.py
class WorkflowLoader:
    def __init__(self, workflows_dir: Path, skills: dict[str, SkillDefinition]): ...
    def load_all(self) -> dict[str, WorkflowDefinition]: ...
    def validate(self, wf: WorkflowDefinition, path: Path) -> list[str]: ...
```

`load_file` splits the document: `frontmatter.load(path)` yields the metadata mapping
and `.content` yields the prompt template body.

## Validation (Req 13.5, 13.6, 13.7)

Each error message names three things — **file path**, the **missing/invalid field or
reference**, and the **reason**:

```
skills/summarize.md: field 'description' is empty — description is required
workflows/deep.md: step 'summarize' references skill 'sumarize' — no such loaded skill
workflows/deep.md: steps [a→b→a] form a circular dependency — cannot order steps
```

- A skill missing a non-empty name, non-empty description, input schema, output
  schema, or prompt template is invalid (tool requirements are never required).
- A workflow with zero steps, a step referencing an unknown skill name (Req 13.6), or
  a cycle in `depends_on` (detected via topological sort / DFS) is invalid.
- **Fail-fast (Req 13.7):** if **any** file fails validation, the loader raises and ARIA
  halts startup — it never runs with valid and invalid definitions mixed.

## Hot Reload (Req 13.8)

A `watchdog.Observer` watches both directories. On a create/modify/delete event the
affected definition is reloaded and re-validated within 5 seconds, without a restart —
a new file appears, an edited one replaces in place, a deleted one is dropped. A newly
invalid edit surfaces the same file/field/reason error.

## Files

| File | Purpose |
|------|---------|
| `src/aria/skills/models.py` | `SkillDefinition`, `SkillInput`, `SkillOutput` |
| `src/aria/skills/loader.py` | skill loading, frontmatter parsing, validation, watch |
| `src/aria/workflows/models.py` | `WorkflowDefinition`, `WorkflowStep` |
| `src/aria/workflows/loader.py` | workflow loading, reference + cycle validation |

## Dependencies

- Req 5 (Configuration) — supplies the skills/workflows directory paths and their
  defaults (`skills/`, `workflows/`).
