# Req 14 — Adversarial Review Agent — Technical Spec

## Architecture

The reviewer is an agent backed by its own `LLMProvider` (Req 01), wired into the
orchestration graph (Req 05) as a routing node placed **before** a task is marked
completed. It is constructed only when `review.enabled` is true.

```
generate_batch → review_node ─┬─ accept  → structural_gate ─┬─ pass → finalize
                              │                              └─ fail → discard+log
                              ├─ revise (< max) → regenerate (with feedback)
                              ├─ revise (≥ max) / reject → discard+log
                              ├─ unreachable → retry → flag "unreviewed" + retain
                              └─ invalid     → treat as revise + log
```

## Models (Req 14.4, 14.9, 14.12)

```python
class ReviewResult(BaseModel):
    classification: Literal["accept", "revise", "reject"]
    feedback: str; confidence: float = 0.0

    @model_validator(mode="after")
    def _feedback_required(self):
        if self.classification in ("revise", "reject") and not self.feedback.strip():
            raise ValueError("feedback must be non-empty for revise/reject")   # Req 14.4
        return self

class ReviewConfig(BaseModel):
    enabled: bool = False                          # Req 14.1 — disabled by default
    model: str | None = None
    max_revisions: int = Field(2, ge=0, le=5)      # Req 14.9
    retry_limit: int = Field(2, ge=0, le=10)

    @model_validator(mode="after")
    def _validate_when_enabled(self):              # Req 14.12
        if self.enabled and not self.model:
            raise ConfigError("review.model is required when review is enabled")
        return self
```

The factory ensures the review provider differs from the executor provider/family
(Req 14.3); executor and reviewer use separate `LLMConfig` blocks.

## Routing Node (Req 14.2, 14.5–14.7, 14.10, 14.11)

```python
async def review_node(state):
    reviewer, item = state["reviewer"], state["item"]
    for _ in range(reviewer.cfg.retry_limit + 1):
        try:
            state["review"] = await reviewer.review(item, state["context"])
            return state
        except ProviderUnreachable:
            continue                               # Req 14.10 — retry
        except ParseError as e:                    # Req 14.11 — invalid response → REVISE
            log.warning("invalid review response: %s; treating as REVISE", e)
            state["review"] = ReviewResult(classification="revise",
                                           feedback="invalid reviewer response")
            return state
    log.error("reviewer unreachable after retries; retaining item flagged unreviewed")
    state["review"], state["unreviewed"] = None, True   # Req 14.10 — retain, not discard
    return state

def route_review(state):
    if state.get("unreviewed"):
        return "finalize_unreviewed"               # retained, not discarded
    r = state["review"]
    if r.classification == "accept":
        return "accept"                            # → structural gate, NOT auto-finalize
    if r.classification == "revise" and state["revision_count"] < state["max_revisions"]:
        state["revision_count"] += 1               # Req 14.5 — increment + feed back
        return "revise"
    return "discard"                               # Req 14.6 (revise≥max) / 14.7 (reject)
```

## Structural Gate — ACCEPT Is Quality-Only (Req 14.8, Key Behavior)

```python
def structural_gate(item, output_file, schema) -> bool:
    # ACCEPT passed the QUALITY gate; structural rules are independent and also required.
    if output_file.has_title(item["title"]):       # Req 4.11 — no duplicate title (case-insensitive)
        log.info("discarded ACCEPT item: duplicate title %r", item["title"]); return False
    try:
        schema(**item)                             # Req 12 — Idea_Record conformance
    except ValidationError as e:
        log.info("discarded ACCEPT item: schema violation: %s", e); return False
    return True                                    # finalize only when BOTH gates pass
```

## Dependencies

- Req 01 (LLM Provider) — second model instance for the reviewer.
- Req 05 (Orchestrator) — review routing node and feedback edge to the Executor.
- Req 07 (Configuration) — `review` settings and their enabled-time validation.
