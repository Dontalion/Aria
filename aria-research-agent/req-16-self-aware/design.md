# Req 16 — Self-Aware Conversational Skill and Workflow Creation — Design

## Overview

Most users should never have to hand-write a skill or workflow Markdown file or learn
its schema. This requirement lets a user simply *talk* to ARIA. In a conversational
CLI chat mode the user describes, in plain language, a capability they want — "I want
something that summarizes a research source into key points" — and ARIA turns that
description into a real, saved skill or workflow.

What makes this "self-aware" is that ARIA knows what it can already do. Before it
builds anything new, it looks at its own registry of tools, skills, workflows, and
agents to see whether the capability already exists. This avoids duplicate clutter and
lets ARIA reuse or adapt what is already there instead of generating near-identical
copies.

## How It Works

A user starts the chat mode and describes what they want across as many turns as they
like. ARIA holds the conversation, asks clarifying questions when something is unclear,
and stays grounded in its own capabilities throughout the session.

The single most important behavior is the **existence check**. When the user describes
a new skill (or workflow), ARIA first checks its internal registry for an existing one
that already provides that capability — *before* generating anything.

- **If a matching capability already exists**, ARIA does not silently build a copy. It
  tells the user it already exists and offers three choices: use the existing one
  as-is, modify the existing one, or create a different variant alongside it. ARIA then
  acts on whichever choice the user makes.
- **If nothing matches**, ARIA generates the Markdown with all the required schema
  elements (name, description, input/output schema, and prompt template for a skill;
  name, description, and ordered steps for a workflow), **saves it directly** into the
  skills or workflows directory, and tells the user what was created and where the file
  lives.

## Living With the Result

Creation is not a one-shot event. After a skill or workflow is saved, the user can keep
chatting to improve it, trim parts out, or delete it entirely. Each requested change is
applied directly to the saved file, so the conversation and the file on disk stay in
step. There is **no approval-timeout or auto-discard step** — files are saved when they
are created, and the user refines them afterward at their own pace.

## Staying Safe

Because generated files become real definitions, ARIA guards what it writes:

- **Schema safety.** A generated file that does not pass schema validation is
  regenerated up to two more times. If it still fails after those attempts, ARIA reports
  the validation failure and saves nothing — a broken file never lands on disk.
- **Tool safety.** If a generated skill or workflow references a tool that is not
  installed, ARIA rejects it, names the unavailable tool, and saves nothing. ARIA only
  creates capabilities it can actually run.

## Knowing Itself

ARIA maintains an internal registry of its currently available skills, workflows,
agents, and tools. That registry is included in the context ARIA uses both when
checking whether a capability already exists and when generating a new one, so its
output is always informed by what it already has. The same self-knowledge powers
introspection: when the user asks ARIA to describe itself in chat, it answers with a
summary of its available skills, available workflows, and the tools it can reference.

## User Experience

- The user runs the chat mode and types what they want in everyday language.
- ARIA replies conversationally, asking follow-up questions only when needed.
- When a capability already exists, ARIA surfaces it and offers the use / modify /
  variant choice rather than quietly duplicating it.
- When it is genuinely new, ARIA writes the file, confirms the name and location, and
  invites the user to keep refining.
- At any time the user can ask "what can you do?" and get a plain summary of skills,
  workflows, and tools.

## Constraints

- Conversation context is limited to the current session (no cross-session memory).
- ARIA cannot create a capability that references a tool it does not have installed.
- Generated files must pass schema validation before being saved, with at most two
  regeneration retries.

## Success Criteria

- A user can create a working, saved skill purely through conversation.
- ARIA reliably detects an existing capability and offers reuse instead of duplicating.
- No invalid or tool-missing file is ever written to disk.
- Asked to describe itself, ARIA accurately summarizes its own skills, workflows, and
  tools.

## Dependencies

- Req 15 (Skills and Workflows) — the schema, loaders, and on-disk format that
  generated files must conform to.
- Req 01 (LLM Provider) — drives the conversation and the generation of Markdown.
- Req 18 (Dynamic Agents) — the registry also includes agents, so self-description and
  the existence check cover agents as well.
