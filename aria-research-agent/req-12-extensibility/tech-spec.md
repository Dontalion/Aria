# Req 12 — Extensibility for New Research Topics — Technical Spec

## Architecture

Domain knowledge lives in three places — config values, Jinja2 templates, and a
dynamically built Pydantic model — never in core code. All three are resolved and
validated at config load time, before any pipeline task runs.

```
config.yaml ──► ConfigSchema (Req 07)
                  ├─ research_topic / subtopics  ──► prompt context (loader.py)
                  ├─ template overrides          ──► TemplateLoader (templates/loader.py)
                  └─ idea_record fields           ──► dynamic IdeaRecord (schema/dynamic.py)
```

## Topic and Subtopics (Req 12.1)

```python
class TopicConfig(BaseModel):
    research_topic: str = Field(..., min_length=1, max_length=200)
    subtopics: list[str] = Field(default_factory=list, max_length=50)

    @field_validator("subtopics")
    @classmethod
    def _bounded(cls, v):
        if any(not (1 <= len(s) <= 200) for s in v):
            raise ValueError("each subtopic must be 1–200 characters")
        return v
```

`research_topic` and `subtopics` are injected into the render context of **every**
generated prompt, so a topic change is config-only.

## Template System (Req 12.2–12.5)

- **Engine**: Jinja2 `SandboxedEnvironment` with `undefined=StrictUndefined`, so any
  reference to an unsupplied variable raises rather than rendering blank.
- **Injected variables**: `topic`, `subtopics`, plus step-specific context.

```python
def resolve_template(step: str, cfg) -> Template:        # step ∈ {keyword, idea}
    src = cfg.overrides.get(step)                        # custom body/path
    if src is None:
        src = DEFAULT_TEMPLATES[step]                    # Req 12.4 — default fallback
    return env.from_string(src)                          # Req 12.3 — custom overrides default

def validate_template(name: str, source: str, supplied: set[str]) -> None:
    try:
        ast = env.parse(source)                          # Req 12.5 — unparsable → reject
    except TemplateSyntaxError as e:
        raise ConfigError(f"template {name!r} failed to parse: {e}")
    missing = meta.find_undeclared_variables(ast) - supplied
    if missing:                                          # Req 12.5 — unsupplied var → reject
        raise ConfigError(f"template {name!r} references unsupplied variables: {sorted(missing)}")
```

Both failure paths name the offending template and abort before any task runs.

## Dynamic Idea_Record (Req 12.6, 12.7)

```python
class FieldDef(BaseModel):
    name: str = Field(..., min_length=1, max_length=64)
    required: bool = True

DEFAULT_FIELDS = [FieldDef(name="title"), FieldDef(name="description"),
                  FieldDef(name="resume_worthiness")]

def build_idea_record(fields: list[FieldDef] | None) -> type[BaseModel]:
    fields = fields or DEFAULT_FIELDS                    # Req 12.6 — default when none
    if not (1 <= len(fields) <= 20):                     # Req 12.7
        raise ConfigError(f"idea_record must define 1–20 fields, got {len(fields)}")
    seen, spec = set(), {}
    for f in fields:
        if not f.name.strip():                           # Req 12.7 — missing/empty name
            raise ConfigError("idea_record field name must be non-empty")
        if f.name in seen:                               # Req 12.7 — duplicate name
            raise ConfigError(f"duplicate idea_record field name: {f.name!r}")
        seen.add(f.name)
        spec[f.name] = (str, ... if f.required else None)
    return create_model("IdeaRecord", **spec)            # Req 12.6 — create_model()
```

## Config Example

```yaml
research_topic: "AI engineering"
subtopics: ["agents", "RAG", "evaluation"]
templates:
  overrides: {idea: "templates/custom/idea_finance.j2"}   # keyword uses default
idea_record:
  fields:
    - {name: title, required: true}
    - {name: feasibility, required: false}
```

## Validation Summary

| Concern            | Rule                                             | Criteria |
|--------------------|--------------------------------------------------|----------|
| Topic              | non-empty, 1–200 chars                           | 12.1     |
| Subtopics          | 0–50 items, each 1–200 chars                     | 12.1     |
| Template override  | parses; no unsupplied variable; names template   | 12.5     |
| Idea_Record fields | 1–20 fields, name 1–64, non-empty, no duplicates | 12.7     |

## Dependencies

- Req 07 (Configuration) — loads and validates topic, subtopics, template
  overrides, and Idea_Record field definitions; the built model and resolved
  templates are consumed by the Req 03 and Req 04 pipelines.
