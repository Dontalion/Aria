# Req 15 — Markdown-Defined Skills and Workflows — Design

## Purpose

Skills and workflows let users teach ARIA new behavior without touching code. A **skill** is a single reusable capability — a prompt with named inputs and outputs. A **workflow** is an ordered recipe that chains skills together with dependencies and data passed between steps. Both live as plain Markdown files in folders the user controls, so they are easy to write, read, share, and version.

The guiding idea is "definitions are data, not code." Anything domain-specific — how to summarize a source, how to run a deep-research sequence — is a Markdown file, never baked into the engine.

## Where Definitions Live

- Skills load from a configurable **skills directory** (default `skills/`).
- Workflows load from a configurable **workflows directory** (default `workflows/`).
- Both directories and their defaults come from configuration (Req 07), so a user can point ARIA at any location.

At startup ARIA reads **every** `.md` file in each directory and turns each into a definition — nothing is loaded selectively or lazily. What is in the folder is what ARIA knows.

## What a Skill Must Contain

Every skill file must define all of these, or it is invalid:

- a **non-empty name** — how other files and the user refer to it,
- a **non-empty description** — what the skill does, in plain language,
- an **input schema** — the named inputs the skill expects,
- an **output schema** — the named outputs the skill produces,
- a **prompt template** — the instructions sent to the model.

**Tool requirements are optional:** a skill may list tools it needs, but one with no tools is perfectly valid.

## What a Workflow Must Contain

Every workflow file must define:

- a **non-empty name**,
- an **ordered sequence of one or more steps** (zero steps is invalid),
- for each step, the **skill it references**,
- the **input/output mappings** between steps — how one step's outputs feed the next step's inputs,
- the **dependency rules** between steps — which steps must finish first.

## Validation at Startup

When ARIA starts it checks every loaded skill and workflow. For each problem it produces an error message naming three things:

1. the **source file path**,
2. the **specific missing or invalid field or reference**, and
3. the **reason** it failed.

Two workflow-specific problems always make a workflow invalid: a step that **references a skill name that was not loaded**, and a **circular dependency** between steps (A waits on B, B waits on A).

### Failing Fast

If **any** skill or workflow file fails validation, ARIA **halts startup** and processes no research request. It will not run with some files valid and others broken — a single bad file stops the whole launch so problems surface immediately rather than midway through a run.

## Hot Reload

Users edit these files while ARIA is running, so changes take effect without a restart. When a skill or workflow file is **added, modified, or removed** in a watched directory, ARIA detects the change and reloads the affected definitions **within 5 seconds** — a new skill becomes available, an edited one updates in place, and a removed one disappears, all live.

## Example — Skill

```markdown
---
name: summarize-source
description: Summarize a research source into key points
inputs:
  - name: source_text
    type: string
    required: true
outputs:
  - name: summary
    type: string
tools: []            # optional
---
Summarize the following source into 3-5 key points: {{ source_text }}
```

## Example — Workflow

```markdown
---
name: deep-research
description: Search, summarize, then synthesize
steps:
  - id: search
    skill: search-web
  - id: summarize
    skill: summarize-source
    inputs: { source_text: "{{ search.results }}" }
    depends_on: [search]
  - id: synthesize
    skill: synthesize
    inputs: { points: "{{ summarize.summary }}" }
    depends_on: [summarize]
---
```

## Dependencies

- Req 07 (Configuration) — supplies the skills and workflows directory paths and their defaults (`skills/`, `workflows/`).
