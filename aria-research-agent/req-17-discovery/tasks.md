# Tasks — Req 17: Tool/MCP/Workflow Discovery and Auto-Integration

## Dependencies

- Req 07 (Configuration) — supplies the registry list, relevance threshold, rate
  limits, and the discovery-history file location

## Tasks

- [ ] 1. Add the `discovery` config section
  - Registry list (MCP, GitHub, package indexes), relevance threshold (default 0.5),
    per-registry rate limit (10/hour), cache TTL, and history file path
  - _Requirements: 17.1, 17.2_

- [ ] 2. Implement the `Recommendation` model
  - Create `src/aria/discovery/models.py` with name, source, description,
    relevance_score (0.0–1.0), expected_improvement, integration_steps,
    install_command, and config_snippet
  - _Requirements: 17.2_

- [ ] 3. Implement the adapter base and rate limiter
  - Define an `Adapter` ABC (`name`, `search(topic)`, `rate_limiter`) with a
    token-bucket limiter capped at 10 queries/hour/registry
  - _Requirements: 17.1_

- [ ] 4. Implement the MCP registry adapter
  - Create `src/aria/discovery/adapters/mcp.py` — keyword + category search
  - _Requirements: 17.1_

- [ ] 5. Implement the GitHub search adapter
  - Create `src/aria/discovery/adapters/github.py` — topic + star-based ranking
  - _Requirements: 17.1_

- [ ] 6. Implement the PyPI search adapter
  - Create `src/aria/discovery/adapters/pypi.py` — keyword + download-count ranking
  - _Requirements: 17.1_

- [ ] 7. Implement the discovery engine search
  - Create `src/aria/discovery/engine.py`; on user request or scheduled scan, query
    each registry for the current topic, dedup across adapters
  - _Requirements: 17.1_

- [ ] 8. Implement recommendation filtering
  - Present only items with relevance ≥ threshold and not already in approved/rejected
    history, including name, source, description, expected improvement, integration steps
  - _Requirements: 17.2_

- [ ] 9. Implement unreachable-registry handling
  - When a registry cannot be reached, skip it, continue the others, and notify the
    user which registry was unavailable
  - _Requirements: 17.6_

- [ ] 10. Implement the confirmation gate
  - Never install, configure, or integrate any tool/MCP/workflow without explicit user
    confirmation
  - _Requirements: 17.3_

- [ ] 11. Implement approved-integration flow
  - On approval, snapshot the tool registry, run the install, register the config, and
    verify accessibility before reporting success
  - _Requirements: 17.4_

- [ ] 12. Implement integration-failure rollback
  - On install/config/verification failure, abort, restore the tool registry to its
    prior state, and notify the user with the failure reason
  - _Requirements: 17.5_

- [ ] 13. Implement the approval history
  - Persist each approve/reject decision (tool name, source, status, timestamp) to
    `config/discovery_history.json`; exclude any tool whose latest status is rejected
    from future recommendations
  - _Requirements: 17.7_

- [ ] 14. Write unit tests
  - Relevance threshold and history filtering exclude known/rejected tools
  - Rate limiter caps at 10 queries/hour/registry
  - Unreachable registry skipped while others continue, user notified
  - No integration occurs without confirmation; rejected tool never re-recommended
  - _Requirements: 17.1, 17.2, 17.3, 17.6, 17.7_

- [ ] 15. Write integration test for approve → integrate → verify
  - Approve a recommendation; assert install + register + verify, then success report
  - Simulate a verification failure; assert abort, registry restored, user notified
  - _Requirements: 17.4, 17.5_
