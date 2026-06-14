# Req 12 — Extensibility for New Research Topics — Design

## Purpose

ARIA is not tied to any single subject. The research topic, the wording of its
prompts, and the shape of each generated idea are all configuration — never
hardcoded. Point ARIA at "AI engineering" today and "climate adaptation finance"
tomorrow, and the only thing that changes is the config file. This keeps the same
engine reusable across every research project a user runs.

## What This Covers

- A required research topic plus optional subtopics that flavor every prompt.
- Custom prompt templates for the keyword step and the idea step.
- A configurable structure for each Idea_Record (the fields ARIA captures).
- Clear, load-time rejection of anything malformed, so a bad topic, template, or
  schema never silently produces garbage.

## Research Topic and Subtopics

- The **research topic is required**. It must be a non-empty string of 1 to 200
  characters. ARIA refuses to start without one.
- **Subtopics are optional context** — a list of 0 to 50 strings, each 1 to 200
  characters — that narrow or steer the research.
- Both the topic and the subtopics are **injected into every generated prompt**.
  Because they flow through the prompts automatically, switching to a new domain
  is a **configuration-only change**: no code is touched, no templates are required.

## Custom Prompt Templates

ARIA phrases its work through templates. Two generation steps can be customized:

| Step                | What the template controls                          |
|---------------------|-----------------------------------------------------|
| Keyword generation  | How ARIA expands a seed keyword into new keywords   |
| Idea generation     | How ARIA turns accepted findings into Idea_Records  |

- A template can reference the configured **topic** and **subtopics** so its wording
  adapts to the domain.
- When a **custom template is provided** for a step, ARIA uses it **instead of** the
  built-in default for that step.
- When **no custom template is provided** for a step, ARIA falls back to the
  **shipped default** template for that step.
- This is a per-step override: a user can customize only the idea step, only the
  keyword step, both, or neither.

## Configurable Idea_Record Structure

The fields captured for each research output are defined in config, not in code.

- The structure is an **ordered set of 1 to 20 field definitions**.
- Each field definition names the field (**1 to 64 characters**) and marks whether
  it is **required**.
- When **no structure is defined**, ARIA applies the **default fields**: title,
  description, and resume-worthiness.
- Adding, removing, or reordering fields is a config edit — the data model ARIA
  validates against is rebuilt from the configured definitions at startup.

## Validation at Load Time

Extensibility is only safe if mistakes are caught before any work begins. ARIA
validates all three concerns when configuration loads, and refuses to start tasks
on any failure:

- **Topic / subtopics** outside their length or count bounds are rejected.
- **Templates** that cannot be parsed, or that reference a variable ARIA does not
  supply, are rejected with an error that **names the offending template**.
- **Idea_Record structure** is rejected when it has a missing or empty field name,
  a duplicate field name, or fewer than 1 or more than 20 fields — the error
  **names the offending field**.

In every case ARIA reports the specific problem and does **not** begin processing
tasks, so a misconfiguration fails fast and loud rather than partway through a run.

## User Experience

- Set `research_topic` (and optionally `subtopics`) in the config file and run.
- Override a prompt template only when the default wording does not fit the domain.
- Define Idea_Record fields only when the default three do not match what you want
  to capture; otherwise leave it blank and get the defaults.
- A typo in a template variable or a duplicate field name stops the run immediately
  with a message pointing at exactly what to fix.

## Relationship to Other Requirements

This requirement defines configuration and load-time behavior only; it has no hard
runtime dependency on other components. Its outputs are **consumed by the keyword
pipeline (Req 03) and the idea pipeline (Req 04)**, which use the injected topic,
the selected templates, and the dynamic Idea_Record structure when they run. The
structural schema also acts as a gate for the review agent (Req 14).

## Dependencies

- Req 07 (Configuration) — supplies and validates the topic, subtopics, template
  selections, and Idea_Record field definitions.
