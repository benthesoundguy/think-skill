# Gated Agent Patterns — Using Think-Beta for External-System Monitoring

## When to use this pattern

You're building an autonomous agent that monitors an external system (ClickUp, GitHub, Jira, Slack, a database, an API) and needs to:

1. Detect changes without polling too aggressively
2. Validate anomalies before acting on them
3. Escalate through progressive tiers
4. Maintain durable state across cron sessions
5. Stay silent when nothing needs attention

This pattern combines a lightweight change-detector with a think-beta-gated investigator.

## Architecture

```
┌─────────────────────┐      every N minutes      ┌─────────────────────┐
│  Watcher (script)   │ ──── silent unless ──────→ │  Target channel     │
│  no_agent=true      │      delta detected        │  (Discord/Telegram) │
│  Python/curl        │                             └─────────────────────┘
└─────────┬───────────┘
          │ writes
          ▼
┌─────────────────────┐      on schedule or       ┌─────────────────────┐
│  State file (JSON)  │ ←──── trigger ─────────── │  Investigator       │
│  ~/.hermes/plans/   │                             │  (LLM + think-beta) │
└─────────────────────┘                             │  C1/C2/C3 gates     │
          ▲                                         │  Escalation tiers   │
          │ reads                                   └─────────────────────┘
┌─────────────────────┐
│  Investigator       │
│  (LLM + think-beta) │
│  C1/C2/C3 gates     │
│  Escalation tiers   │
└─────────────────────┘
```

## Key components

### Watcher script (no_agent cron job)

A lightweight, LLM-free script that:
- Polls the external system via REST API or direct database query
- Compares current state against a stored snapshot
- Outputs nothing if nothing changed (silent exit)
- Outputs delta text if something changed (delivered as cron message)

```bash
hermes cron create "every 5m" \
  --name "my-watcher" \
  --no-agent \
  --script "my-watcher.py" \
  --deliver "discord:#my-channel"
```

The script MUST exit silently (no stdout) when there are no changes. The cron
system only delivers non-empty stdout from no_agent jobs.

### State file

A durable JSON file at `~/.hermes/plans/<name>-state.json` that stores:
- `last_scan`: ISO timestamp
- Task/milestone/issue states keyed by ID with name and status
- Written by the watcher after every tick

### Investigator (LLM + think-beta)

A scheduled cron job that:
- Reads the watcher's state file to find what changed
- Loads the think skill at session start via `skills: ["think"]`
- For each delta, runs C1-C3 Draft Review gates before acting:
  - **C1 (Evidence):** Can I cite the exact data showing this anomaly?
  - **C2 (Delta):** Has this been flagged before? Check task comments or state.
  - **C3 (Input):** Have I read the full details before concluding?
- Only acts on anomalies that pass all three gates

```bash
hermes cron create "0 8,18 * * *" \
  --name "my-investigator" \
  --skills "think" \
  --prompt "Your investigation prompt here..."
```

### Durable memory (state file per agent)

Each agent maintains its own state file tracking:
- Anomalies keyed by task ID with type, first_seen, last_seen, tier, and comment_id
- Escalation history
- Dedup: if a task is already in state at a given tier, don't re-flag

```json
{
  "last_scan": "2026-07-15T22:30:00Z",
  "anomalies": {
    "task_123": {
      "type": "orphan_blocked",
      "first_seen": "2026-07-15T22:30:00Z",
      "last_seen": "2026-07-15T22:30:00Z",
      "tier": 1,
      "comment_id": "90140227464789"
    }
  },
  "scan_count": 5
}
```

## The three-tier escalation

For each validated anomaly, escalate through progressive tiers:

| Tier | Action | Condition |
|------|--------|-----------|
| 0 — Detected | Raw data collected, anomaly candidate identified | C1-C3 pass |
| 1 — Comment | Write a comment on the affected entity | First detection, no prior comment |
| 2 — Ping | Post an @mention in the alert channel | Same anomaly persists, no response |
| 3 — Fix | Correct the issue directly | 3 scans without response |

Store the current tier in the state file. Only move forward, never backwards.
Dedup: if same anomaly + same tier already in state, skip.

## Example: ClickUp Manager agent

The canonical implementation monitors a ClickUp workspace for:
- **Orphan blocked milestones** — blocked with no upstream dep and no finding
- **Stale status milestones** — all deps done but still blocked
- **Stale decision references** — decision references a task whose status contradicts the intent

The watcher polls every 5 minutes via ClickUp REST API. The investigator runs
at 8 AM / 6 PM with think-beta loaded, applies C1-C3 gates, and escalates
through the three-tier system.

## Pitfalls

- **Watcher credential management:** The no_agent script needs its own auth
  (API key, OAuth token). Store in a profile config or a dedicated secrets file,
  not in the script itself.
- **State file collision:** Multiple agents writing to the same state file
  will overwrite each other. Each agent needs its own state file path.
- **False positive on first run:** The first watcher run shows all tasks as
  "new". This is expected — the next run will only show actual deltas.
- **Think skill in cron:** Use `skills: ["think"]` in the cron job config, NOT
  `/skill think` (slash commands don't work in cron sessions).
- **Gateway vs cron:** Cron jobs deliver to Discord channels directly without
  needing a running gateway. Only interactive chat requires a gateway.
- **Discord bot token limit:** One bot token supports 2 concurrent gateway
  connections. For N agent profiles, create N Discord bot applications and
  give each profile its own token.
