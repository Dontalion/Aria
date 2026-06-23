# Req 17 — Tool, MCP, and Workflow Discovery and Auto-Integration — Design

## Overview

ARIA does not stay frozen with the tools it shipped with. This requirement lets it
look outward — at the wider ecosystem of MCP servers, open-source projects, and
published packages — and suggest capabilities that could make the current research
better. The guiding rule is simple: ARIA may *discover and recommend* freely, but it
*never installs or changes anything* without the user explicitly saying yes.

The goal is to give the user the benefit of the ecosystem without the chore of manual
hunting, while keeping them firmly in control of what actually gets added to the system.

## How Discovery Works

Discovery runs in one of two ways: the user asks for it directly, or a scheduled scan
kicks it off. Either way ARIA searches the configured external registries — an MCP
registry, GitHub, and package indexes — for tools and workflows that match the current
research topic. To stay a good citizen of those services, ARIA caps itself at no more
than ten queries per hour per registry.

Each candidate it finds is scored for relevance on a 0.0–1.0 scale. ARIA only surfaces
items whose relevance is at or above a configurable threshold (default 0.5) and that
are not already in its approved or rejected history. For every recommendation it shows
the user the name, the source it came from, a description, the expected improvement to
the research, and the steps that would be needed to integrate it. That way the user
sees both what the tool is and why it might help, before deciding anything.

## Nothing Happens Without a Yes

The most important behavior is the confirmation gate. ARIA will not install, configure,
or integrate any external tool, MCP server, or workflow without explicit user
confirmation. A recommendation is just a suggestion until the user approves it.

When the user *does* approve an integration, ARIA carries it through carefully:

1. It runs the installation.
2. It registers the tool's configuration into its tool registry.
3. It verifies the tool is actually accessible before declaring success.

Only after that verification does ARIA report completion. If any of those steps fails —
installation, configuration, or the accessibility check — ARIA aborts the integration,
restores the tool registry to exactly the state it was in before the attempt, and
notifies the user with an error describing what went wrong. The system is never left in
a half-integrated state.

## When a Registry Is Down

External services are not always reachable. If a registry cannot be contacted — because
of network trouble or an error on its end — ARIA does not fail the whole discovery run.
It skips the unreachable registry, continues searching the others, and tells the user
which registry was unavailable so they know the picture is partial.

## Remembering Decisions

Every approve or reject decision is remembered. ARIA persists a record of each one,
capturing the tool name, its source, the status (approved or rejected), and a
timestamp. That history serves two purposes: it keeps an auditable trail of what was
decided, and it ensures a tool the user has rejected is never recommended again. Once
something is on the rejected list, it stays off future recommendation lists.

## User Experience

- The user triggers discovery on demand, or lets a scheduled scan find candidates.
- ARIA presents each relevant find with its name, source, description, expected benefit,
  and integration steps — for example, "I found a paper-retrieval MCP server that could
  improve source gathering. Integrate it?"
- The user approves or declines. Nothing is installed unless they approve.
- On approval, ARIA installs, registers, and verifies, then confirms success — or rolls
  back cleanly and explains the failure.
- Previously rejected tools never reappear in later suggestions.

## Constraints

- Network access is required for registry searches.
- Searches are rate-limited to at most ten queries per hour per registry.
- ARIA only recommends items relevant to the configured research topic and above the
  relevance threshold.
- Integration is all-or-nothing: success is reported only after verification, and any
  failure restores the prior registry state.

## Success Criteria

- ARIA surfaces genuinely relevant tools for the configured research topic.
- Zero installations or configuration changes happen without explicit user confirmation.
- A failed integration always leaves the tool registry exactly as it was before.
- A rejected tool is never recommended again.

## Dependencies

- Req 07 (Configuration) — supplies the registry list, relevance threshold, rate
  limits, and the location of the discovery history file.
