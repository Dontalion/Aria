# Req 18 — Dynamic Agent Creation and Lifecycle — Design

## Purpose

Some research tasks need a specialist that none of ARIA's standing agents provide —
a "FinTech keyword expert," a "patent-language summarizer," a "benchmark comparison
analyst." Rather than forcing the user to pre-build every conceivable agent, ARIA
creates one on demand, shaped exactly to the task in front of it. Each dynamic agent
is a small, well-scoped worker defined by a name, a role, a model, a tool list, and a
prompt.

Creation is never silent and never automatic. ARIA proposes the agent, the user
approves it, and only then does it run. Every agent has a defined lifespan — most
disappear the moment their task is done, while a few earn a place in the user's
permanent toolkit.

## When a Dynamic Agent Is Created

ARIA creates a new specialized agent only when a Task requires behavior that **no
existing agent provides** (Req 18.1). It does not spin up duplicates of agents it
already has. The proposed agent is described by five things: its **name**, its
**role**, the **model** it will run on, the **tools** it may call, and the **prompt**
that governs its behavior.

Two hard boundaries shape every dynamic agent:

- **No recursive spawning.** A dynamic agent can do its job, but it **cannot create
  or spawn sub-agents** of its own. Only ARIA's Orchestrator creates agents, which
  keeps the agent graph flat and predictable.
- **Tools are limited to the definition.** An agent may use **only the tools listed
  in its definition**, and those tools come from the **permissions the user granted
  up front**. A dynamic agent can never reach beyond what the user already allowed.

## The Approval Gate (Key Behavior)

Before an agent does anything, ARIA presents the **complete definition** — name,
role, model, tools, and prompt — to the user and **withholds activation until the
user explicitly approves** it (Req 18.2). Nothing runs on an unapproved definition.

If the user **rejects** the proposal, ARIA does three things (Req 18.3):

1. It does **not** activate the agent.
2. It **discards** the proposed definition entirely.
3. It **notifies** the user that agent creation was cancelled.

## The Concurrency Limit

Temporary agents are cheap, but not free. ARIA caps the number of **concurrent
temporary agents at 5**. If creating a new temporary agent would push the count past
that limit, ARIA **rejects the creation request, leaves all existing temporary agents
untouched, and notifies the user** that the maximum of 5 has been reached (Req 18.4).
No silent eviction of a running agent to make room.

## Lifecycle Modes

Every agent runs in one of two modes (Req 18.5):

| Mode           | Lifespan                                              |
|----------------|-------------------------------------------------------|
| **Temporary**  | Destroyed when its task or pipeline completes         |
| **Persistent** | Saved to disk and reusable across future sessions     |

The user can change an agent's mode at any time (Req 18.6):

- **Promote** a temporary agent → persistent (its definition is **saved** for reuse).
- **Demote** a persistent agent → **archived** (kept but inactive).
- **Delete** a persistent agent → its definition is **removed**.

## Timeout and Cleanup

A temporary agent is not allowed to linger. If one stays active **longer than its
configured timeout (default 30 minutes)**, ARIA **destroys it, releases its
resources, and notifies the user** that it was terminated due to timeout (Req 18.8).
This protects the system from stuck or runaway workers holding resources.

## Auditability

Whenever an agent is used for a task, the State_Store records four facts: the
**agent identifier**, the **associated task identifier**, the **lifecycle mode**, and
the **creation timestamp** (Req 18.7). This gives a complete, queryable history of
which agent worked on which task and when — so any run can be audited after the fact.

## User Experience

- ARIA: "This task needs domain-specific FinTech enrichment. I'd like to create a
  `FinTechKeywordExpert` agent (model: gpt-4o; tools: web_search, summarize). Approve?"
- User approves → the agent activates and runs the task.
- Task completes → temporary agent is destroyed automatically; ARIA asks whether to
  keep it. "Promote `FinTechKeywordExpert` to a persistent agent for reuse?"
- User declines a different proposal → "Agent creation cancelled. Nothing was activated."

## Configuration

```yaml
agents:
  max_concurrent_temporary: 5     # hard cap on live temporary agents (Req 18.4)
  default_lifecycle: temporary    # temporary | persistent (Req 18.5)
  temporary_timeout_minutes: 30   # auto-destroy after this idle/active window (Req 18.8)
  persistent_dir: "./agents"      # where promoted agent definitions are saved
```

## Dependencies

- **Req 01 (LLM Provider)** — each dynamic agent runs on a configured model through
  the unified LLM interface.
- **Req 06 (State Persistence)** — the State_Store records the agent-to-task history
  (id, task id, lifecycle mode, timestamp) for auditability.
- **Req 07 (Configuration)** — holds the concurrency cap, default lifecycle, timeout,
  and persistent storage location.
