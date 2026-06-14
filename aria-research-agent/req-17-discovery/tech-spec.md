# Req 17 — Tool/MCP/Workflow Discovery and Auto-Integration — Technical Spec

## Architecture

A `DiscoveryEngine` fans out across pluggable registry adapters (MCP registry, GitHub,
package indexes), scores and deduplicates candidates, and filters them against the
approve/reject history. Integration is gated behind explicit user confirmation; on
approval the engine installs, registers into the tool registry, and verifies
accessibility before reporting success, rolling back on any failure. Every decision is
persisted to `config/discovery_history.json`.

```
trigger (user request | scheduled scan)
   → each adapter: search(topic) [≤10/hr/registry]; unreachable → skip+notify (Req 17.6)
   → score (0.0–1.0) + dedup; keep relevance ≥ threshold (0.5) AND not in history (Req 17.2)
   → present recommendation (name, source, description, improvement, steps)
   → confirm? ── no ──→ record reject (Req 17.7)
              └─ yes ─→ install → register → verify
                           ├─ ok   → record approve, report success (Req 17.4)
                           └─ fail → abort + restore registry + notify (Req 17.5)
```

## Recommendation Model (Req 17.2)

```python
# src/aria/discovery/models.py
class Recommendation(BaseModel):
    name: str
    source: Literal["mcp", "github", "pypi"]
    description: str
    relevance_score: float = Field(ge=0.0, le=1.0)
    expected_improvement: str           # why it helps the research
    integration_steps: list[str]        # steps required to integrate
    install_command: str
    config_snippet: dict
```
## Discovery Engine (Req 17.1, 17.2, 17.6)

```python
# src/aria/discovery/engine.py
class DiscoveryEngine:
    def __init__(self, adapters, history: DiscoveryHistory, threshold: float = 0.5): ...
    async def discover(self, topic: str) -> list[Recommendation]:
        recs = []
        for adapter in self.adapters:
            if not adapter.rate_limiter.allow():     # ≤10/hour/registry (Req 17.1)
                continue
            try:
                recs += await adapter.search(topic)
            except RegistryUnreachable as e:
                notify(f"{adapter.name} unavailable: {e}")   # Req 17.6 skip + continue
        return [r for r in dedupe(recs)              # Req 17.2 — threshold + not-known
                if r.relevance_score >= self.threshold
                and not self.history.is_known(r.name, r.source)]
```

## Registry Adapters (Req 17.1)

| Adapter | File | Source | Search method |
|---------|------|--------|---------------|
| MCP | `src/aria/discovery/adapters/mcp.py` | MCP registry | keyword + category |
| GitHub | `src/aria/discovery/adapters/github.py` | GitHub API | topic + star rank |
| PyPI | `src/aria/discovery/adapters/pypi.py` | PyPI API | keyword + downloads |

Each adapter shares an `Adapter` ABC (`name`, `search(topic)`, `rate_limiter`) with its
own token-bucket limiter capped at 10 queries/hour.

## Confirmation Gate and Integration (Req 17.3, 17.4, 17.5)

```python
async def integrate(self, rec: Recommendation) -> None:
    if not await confirm(rec):                       # Req 17.3 — never without confirmation
        self.history.record(rec, status="rejected"); return
    snapshot = self.tool_registry.snapshot()         # prior state for rollback
    try:
        run(rec.install_command)                     # 1. install
        self.tool_registry.add(rec.name, rec.config_snippet)   # 2. register
        if not await self.tool_registry.verify(rec.name):      # 3. verify accessible
            raise IntegrationError("tool not accessible after install")
    except Exception as e:                            # Req 17.5 — abort + restore + notify
        self.tool_registry.restore(snapshot)
        notify(f"integration of {rec.name} failed: {e}; registry restored"); return
    self.history.record(rec, status="approved")      # Req 17.7
    notify(f"{rec.name} installed and verified")      # Req 17.4 — report after verify
```

## Approval History (Req 17.7)

`DiscoveryHistory` persists `[{tool_name, source, status, timestamp}, ...]` to
`config/discovery_history.json`, exposing `record(rec, status)`, `is_known(name, source)`
(present in approved OR rejected), and `is_rejected(name, source)`. `discover()` excludes
any tool whose latest recorded status is rejected, so a rejected tool is never
re-recommended (Req 17.7). Records are appended atomically.

## Rate Limiting and Caching

Token-bucket limiter per adapter caps at 10 queries/hour/registry (Req 17.1). Search
results are cached 24h to avoid redundant queries; cap and TTL are configurable via
`config.yaml` (Req 07).

## Files

`src/aria/discovery/engine.py` (orchestration, scoring, dedup, integration),
`models.py` (`Recommendation`), `adapters/{mcp,github,pypi}.py`, and
`config/discovery_history.json` (persisted approve/reject decisions).

## Dependencies

- Req 07 (Configuration) — registry list, relevance threshold, rate limits, and the
  discovery-history file location.
