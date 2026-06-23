# Req 14 — Adversarial Review Agent — Design

## Purpose

The Review Agent is an optional second opinion. After the Executor generates a batch
of keywords or ideas, a separate reviewer — drawn from a different model family —
inspects each item for hallucinations, weak evidence, and shaky reasoning before it
is finalized. Using a different model family avoids the correlated blind spots that
two instances of the same model tend to share.

The reviewer is **disabled by default**. When it is off, batches flow straight to the
existing structural checks and output. When it is on, it adds a quality gate in front
of those checks.

## Enabling and Model Choice

- Review is **optional and off by default** — turning it on is a config change.
- The review model **must differ from the Executor's model or family**, so the
  reviewer brings an independent perspective rather than echoing the generator.
- A retry limit governs transient reviewer failures, and a revision cap bounds how
  many times an item can bounce back for rework.

## The Review Flow

When review is enabled, the Orchestrator routes **every item in a generated batch**
through the reviewer **before the corresponding task is marked completed**. The
reviewer assigns each item exactly one classification:

| Classification | Meaning                                  | What happens next                          |
|----------------|------------------------------------------|--------------------------------------------|
| **ACCEPT**     | Meets quality standards                  | Proceeds to the structural gate (see below)|
| **REVISE**     | Fixable issues                           | Returns to Executor with feedback          |
| **REJECT**     | Fundamentally flawed or hallucinated     | Discarded and logged                       |

For any **REVISE or REJECT**, the reviewer must produce **non-empty feedback text**
that names the specific issues to correct.

## Revision Loop

- A **REVISE below the max-revision count** sends the item back to the Executor
  **together with the reviewer's feedback**, and **increments that item's revision
  cycle count**. The Executor regenerates, and the new version is reviewed again.
- An item **still classified REVISE after the maximum cycles** is **discarded,
  excluded from the finalized output, and the reason is logged**.
- A **REJECT** item is **discarded, excluded from output, and the reason logged** —
  no revision attempt.

The maximum revision cycles is configurable from **0 to 5, defaulting to 2**.

## ACCEPT Is a Quality Gate Only (Key Behavior)

This is the crucial rule. An ACCEPT verdict is a **quality approval, nothing more**.
An item is **finalized only if it ALSO passes the structural rules** defined
elsewhere in the specification:

- **No duplicate title** within the same output file (per Req 4.11), and
- **Conformance to the configured Idea_Record schema** (per Req 12).

An item that the reviewer ACCEPTs but that **violates a structural rule is
discarded**, with the reason logged. Quality approval never overrides structural
correctness — both gates must pass.

## Failure Handling

- **Reviewer unreachable or no response** — the Orchestrator **retries the review up
  to the configured limit**. When retries are exhausted, it **logs the failure and
  retains the item flagged as "unreviewed"** rather than discarding it. A transient
  outage must not silently drop generated work.
- **Invalid review response** — if the reviewer returns something without a valid
  ACCEPT / REVISE / REJECT classification, the Orchestrator **treats the item as
  REVISE for the current cycle** and logs the invalid-response reason. This routes it
  back for another attempt instead of guessing a verdict.

## Configuration Validation

When review is **enabled**, ARIA validates the settings at load time:

- A **review model must be specified**.
- **max_revisions must be an integer from 0 to 5**.

If validation fails, ARIA **rejects the configuration** and logs an error naming the
invalid review setting. When review is disabled, these settings are not required.

## Configuration

```yaml
review:
  enabled: false           # disabled by default (Req 14.1)
  model: null              # required when enabled; must differ from executor
  max_revisions: 2         # integer 0–5, default 2
  retry_limit: 2           # review retries when reviewer is unreachable
```

## Dependencies

- Req 01 (LLM Provider) — the reviewer needs a second model, distinct from the
  Executor's.
- Req 05 (Orchestrator) — the review step is a routing node placed before task
  completion, feeding back into the Executor on REVISE.
- Req 07 (Configuration) — holds the enable flag, review model, max revisions, and
  retry limit, and validates them when review is enabled.
