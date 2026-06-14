# Tasks — Req 14: Adversarial Review Agent

## Dependencies

- Req 01 (LLM Provider) — second model instance for the reviewer
- Req 05 (Orchestrator) — review routing node and feedback edge to the Executor
- Req 07 (Configuration) — `review` settings and enabled-time validation

## Tasks

- [ ] 1. Add the `review` config section
  - `enabled` (default false), `model`, `max_revisions` (int 0–5, default 2),
    `retry_limit`
  - Validate when enabled: a model must be specified and max_revisions is 0–5;
    reject naming the invalid setting
  - _Requirements: 14.1, 14.9, 14.12_

- [ ] 2. Implement `ReviewResult` and `ReviewConfig` models
  - Classification enum accept/revise/reject; require non-empty feedback for
    revise/reject
  - _Requirements: 14.4, 14.9_

- [ ] 3. Implement `ReviewerAgent` backed by a distinct LLM provider
  - Build the review prompt (hallucination, evidence, coherence, completeness)
  - Ensure the review model/family differs from the Executor's
  - _Requirements: 14.1, 14.3, 14.4_

- [ ] 4. Implement response parsing
  - Parse a valid classification + feedback; raise on responses without a valid
    ACCEPT/REVISE/REJECT
  - _Requirements: 14.4, 14.11_

- [ ] 5. Integrate the review node into the orchestration graph
  - Route every item in a generated batch through review before the task is marked
    completed
  - _Requirements: 14.2_

- [ ] 6. Implement the revision loop
  - REVISE below max → return to Executor with feedback and increment the cycle count
  - REVISE at/after max → discard, exclude from output, log the reason
  - _Requirements: 14.5, 14.6_

- [ ] 7. Implement reject handling
  - REJECT → discard, exclude from output, log the rejection reason
  - _Requirements: 14.7_

- [ ] 8. Implement the structural gate after ACCEPT (key behavior)
  - Treat ACCEPT as quality-only; finalize only if the item also has no duplicate
    title (Req 4.11) and conforms to the Idea_Record schema (Req 12)
  - Discard an ACCEPT item that violates a structural rule and log the reason
  - _Requirements: 14.8_

- [ ] 9. Implement reviewer-unreachable handling
  - Retry up to the configured limit; when exhausted, log the failure and retain the
    item flagged "unreviewed" rather than discarding it
  - _Requirements: 14.10_

- [ ] 10. Implement invalid-response handling
  - Treat an invalid review response as REVISE for the current cycle and log the reason
  - _Requirements: 14.11_

- [ ] 11. Implement disabled-review bypass
  - When `review.enabled` is false, skip the review node and route batches straight
    to the structural checks and output
  - _Requirements: 14.1_

- [ ] 12. Write unit tests
  - Accept/revise/reject paths with mocked reviewer responses
  - Non-empty feedback enforced for revise/reject
  - ACCEPT but duplicate title → discarded; ACCEPT but schema violation → discarded
  - Revise increments cycle; still revise after max → discarded
  - Unreachable → retried then retained as unreviewed; invalid response → treated as revise
  - Enabled-time validation: missing model and max_revisions out of 0–5 rejected
  - _Requirements: 14.4, 14.5, 14.6, 14.7, 14.8, 14.9, 14.10, 14.11, 14.12_

- [ ] 13. Write integration test for a full review cycle
  - End-to-end revise → regenerate → accept → structural gate → finalize
  - _Requirements: 14.2, 14.5, 14.8_
