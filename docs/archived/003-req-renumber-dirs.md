# Archive: 003-req-renumber-dirs

## What Was Done
Renamed all remaining specification requirement directories (from `req-05-orchestrator` through `req-19-wiki`) under `aria-research-agent/` to match the new numbering layout defined in `requirements.md`. This shifts the directories down from `req-03-orchestrator` to `req-17-wiki` now that the keyword and idea generation pipelines are archived.

## Files Changed
The following directories and their internal files were renamed:
- `aria-research-agent/req-05-orchestrator` → [aria-research-agent/req-03-orchestrator](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-03-orchestrator)
- `aria-research-agent/req-06-state` → [aria-research-agent/req-04-state](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-04-state)
- `aria-research-agent/req-07-config` → [aria-research-agent/req-05-config](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-05-config)
- `aria-research-agent/req-08-cli` → [aria-research-agent/req-06-cli](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-06-cli)
- `aria-research-agent/req-09-output` → [aria-research-agent/req-07-output](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-07-output)
- `aria-research-agent/req-10-error-handling` → [aria-research-agent/req-08-error-handling](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-08-error-handling)
- `aria-research-agent/req-11-logging` → [aria-research-agent/req-09-logging](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-09-logging)
- `aria-research-agent/req-12-extensibility` → [aria-research-agent/req-10-extensibility](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-10-extensibility)
- `aria-research-agent/req-13-data-layer` → [aria-research-agent/req-11-data-layer](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-11-data-layer)
- `aria-research-agent/req-14-review-agent` → [aria-research-agent/req-12-review-agent](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-12-review-agent)
- `aria-research-agent/req-15-skills` → [aria-research-agent/req-13-skills](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-13-skills)
- `aria-research-agent/req-16-self-aware` → [aria-research-agent/req-14-self-aware](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-14-self-aware)
- `aria-research-agent/req-17-discovery` → [aria-research-agent/req-15-discovery](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-15-discovery)
- `aria-research-agent/req-18-dynamic-agents` → [aria-research-agent/req-16-dynamic-agents](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-16-dynamic-agents)
- `aria-research-agent/req-19-wiki` → [aria-research-agent/req-17-wiki](file:///home/dontalion/Desktop/project/ARIA/main/aria-research-agent/req-17-wiki)

## Decisions Made
- Renamed the directories directly via system `mv` command to match requirements.md renumbering.
- Confirmed that LLM and Search providers (req-01 and req-02) were kept unchanged as they are unaffected.
