# Multi-Agent Profile Teams — Architecture Pattern

How to set up multiple Hermes profiles that collaborate as an autonomous agent team,
each with a dedicated Discord bot, shared state, and cross-agent communication.

## When to use this pattern

You have multiple distinct responsibilities that should be owned by different agents:
- **Sensory layer** — watches an external system (ClickUp, GitHub, monitoring), detects changes, flags anomalies
- **Orchestrator layer** — reads the sensory layer's output, decomposes work into tasks, assigns to workers
- **Worker profiles** — execute individual tasks (research, code, review)

The pattern is for persistent, multi-session collaboration where agents need to talk to each other
and share state across runs. For single-session multi-agent work, use `delegate_task` instead.

## Architecture

### Profile roles

| Role | Toolset | Responsible for |
|---|---|---|
| **Sensory profile** (e.g. clickup-manager) | External-system MCP tools only. No terminal, no code execution. | Watch, detect changes, flag anomalies, escalate through tiers. Write results back. |
| **Orchestrator profile** (e.g. kanban-orchestrator) | Kanban tools only (create, link, assign, comment). No terminal, no code execution. | Read sensory reports, decompose milestones into Kanban cards, assign to workers, manage dependencies. |
| **Worker profiles** (e.g. researcher, clickup-dev, reviewer) | Full tools: terminal, file, web, browser. | Execute individual cards. Mark complete when done. |

### Shared state file

Each profile maintains a JSON state file under its profile's `plans/` directory.
Sensory and orchestrator profiles read each other's state files for cross-profile awareness.

```json
// /opt/data/profiles/clickup-manager/plans/state.json
{
  "last_scan": "ISO timestamp",
  "anomalies": {
    "task_id": {
      "type": "orphan_blocked|stale_status|stale_reference",
      "first_seen": "ISO timestamp",
      "last_seen": "ISO timestamp",
      "tier": 0,
      "comment_id": null
    }
  },
  "scan_count": 0
}
```

### Dedicated Discord bots

Each profile gets its own Discord bot application with a unique bot token.
This gives each agent its own identity, name, and avatar in the server.

```yaml
# In profile's .env
DISCORD_BOT_TOKEN=<unique-token>
DISCORD_ALLOW_ALL_USERS=true
```

**Limitation:** Discord allows 2 gateway connections per bot token. Profiles that need
interactive Discord must run their own gateway. Cron-only profiles don't need a gateway —
cron delivers directly to channels via the Discord API.

### Discord Configuration Flags

All settings go in the profile's `.env` file:

| Env var | Values | Default | Purpose |
|---|---|---|---|
| `DISCORD_BOT_TOKEN` | token string | — | Bot authentication (required) |
| `DISCORD_ALLOW_ALL_USERS` | true/false | false | Accept messages from any user |
| `DISCORD_ALLOWED_USERS` | comma-separated IDs | — | User allowlist |
| `DISCORD_ALLOW_BOTS` | none/mentions/all | **none** | Bot-to-bot message policy. **#1 reason cross-agent chat fails** — bots silently drop each other's messages until set to `all`. Use `all` (not `mentions`) — the `mentions` mode checks Discord's resolved mentions list, which doesn't always include the bot even when the @-mention is in the message body. |  
| `DISCORD_BOTS_REQUIRE_INLINE_MENTION` | true/false | false | When true, a bot-authored message only wakes this bot if the message content contains a literal `<@bot_id>` token. A Discord reply/quote ping does NOT count — this is the key fix for infinite bot reply loops. Pair with `DISCORD_ALLOW_BOTS=all` for: bots see all messages, but only act on inline @-mentions. |
| `DISCORD_AUTO_THREAD` | true/false | true | Auto-create threads on @mention. Set `false` to keep conversations in main channel. |
| `DISCORD_NO_THREAD_CHANNELS` | comma-separated channel IDs | — | Channels where threading is disabled even when `DISCORD_AUTO_THREAD=true` |
| `DISCORD_REPLY_TO_MODE` | off/first/all | first | Reply threading behavior |
| `DISCORD_HOME_CHANNEL` | channel ID | — | Set with `/sethome`. Bot processes ALL messages here. |
| `DISCORD_IGNORE_NO_MENTION` | true/false | true | Ignore messages without @mention |

### Channel routing

| Channel | Purpose | Active Agents |
|---|---|---|
| `#general` | User-facing main chat | Default profile |
| `#agent-a` | Sensory agent's operational channel | Agent A |
| `#agent-b` | Orchestrator's channel | Agent B |
| `#pm-gateway` | Cross-agent communication | Both agents |

### Cross-agent communication protocol

Agents communicate via @-mentions in a shared channel (#pm-gateway).
With `DISCORD_ALLOW_BOTS=all` and `DISCORD_BOTS_REQUIRE_INLINE_MENTION=true`, bots see all messages from other bots but only act on messages with a literal `<@bot_id>` in the body.

- Agent A posts to #pm-gateway with `<@agent_b_user_id>` in the body → Agent B's gateway routes it (inline mention detected) → Agent B processes and responds
- Agent B responds in the same channel with `<@agent_a_user_id>` in the body → Agent A sees it
- Agent B's response to Agent A does NOT contain Agent A's @-mention (it's a reply, not an inline mention) → Agent A does NOT wake up → no loop
- SOUL.md for each agent lists the other agent's user ID for reference
- Cron deliveries include @-mentions so the target bot's gateway routes the message

### Three-tier escalation (for compliance violations)

| Tier | Action | When |
|---|---|---|
| 0 (detected) | C1-C3 validation gates pass. Anomaly recorded in state file. | First scan |
| 1 (commented) | Comment on the task: observation + discrepancy + question. State updated to tier=1. | Same scan |
| 2 (pinged) | @-mention the responsible agent in #pm-gateway. State updated to tier=2. | Next scan, no response |
| 3 (fixed) | Fix the issue directly, add correction comment. State updated to tier=3 resolved. | 3rd scan, no response |

**Dedup:** State file tracks current tier per anomaly. Never re-perform the same tier on the same anomaly.

### C1-C3 validation gates (before acting)

| Gate | Check |
|---|---|
| C1 (Evidence) | Can you cite the exact task data showing this anomaly? Task ID, field, value. |
| C2 (Delta) | Has this same anomaly been flagged before? Check state file and task comments. |
| C3 (Input) | Have you read the full task details (custom fields, gating status) before concluding? |

## MCP Server Wiring Per-Profile

Profiles configure their own MCP servers in their profile's `config.yaml`:

```yaml
mcp_servers:
  myapi:
    command: node
    args:
      - /path/to/mcp-server/build/index.js
    enabled: true
    env:
      API_KEY: "${API_KEY}"
```

### Auth modes

- **Stdio (API key)** — preferred for portability across profiles. MCP server reads API key from env var. Simple, no OAuth flow. Use `command` + `args` in profile config.
- **HTTP (OAuth)** — requires cached OAuth tokens in `~/.hermes/mcp-tokens/`. Tokens are **global** (not per-profile). If a profile can't find the token, symlink: `ln -s /path/to/mcp-tokens /opt/data/profiles/<name>/mcp-tokens`

### Cron job caveat

Cron jobs run under the **default profile**, not the named profile. If the cron needs profile-specific MCP tools, configure them globally in the main `config.yaml` under `mcp_servers`, or use a `no_agent` script that calls the API directly via curl/requests with a stored token.

## Setup steps

1. **Create profiles:**
   ```bash
   hermes profile create agent-name --description "Role description"
   cp /opt/data/.env /opt/data/profiles/agent-name/.env
   # Clean old tokens that were copied:
   sed -i '/DISCORD_BOT_TOKEN/d' /opt/data/profiles/agent-name/.env
   # Clean deprecated env vars that were copied (they produce startup warnings):
   sed -i '/^TERMINAL_CWD=/d' /opt/data/profiles/agent-name/.env
   sed -i '/^DISCORD_BOT_TOKEN=MTUyNDQ1NjYxNDQyODY3MjE4MA/d' /opt/data/profiles/agent-name/.env
   ```

2. **Write SOUL.md** defining the agent's role, channels it monitors, and cross-agent protocol.

3. **Create Discord bot applications** (one per profile with dedicated identity):
   - Discord Developer Portal → New Application
   - Bot → Add Bot → Reset Token → Enable Message Content Intent
   - OAuth2 → URL Generator → bot + applications.commands scopes
   - Invite to server
   - Add token to profile's `.env` as `DISCORD_BOT_TOKEN`

4. **Create a lightweight change-detection script** (no_agent cron) for periodic polling:
   ```python
   # Watcher script — runs every N minutes, silent unless a delta is found
   # Compares current API state against stored snapshot
   # Outputs nothing if no change; outputs delta if change detected
   ```

5. **Create agent-driven cron jobs** for deeper compliance scanning:
   ```bash
   hermes cron create "0 8,18 * * *" --skills think --prompt "..."
   ```

6. **Start gateways:**
   ```bash
   hermes -p profile-name gateway run --replace
   ```

7. **Verify Discord connection:**
   ```bash
   grep -i 'connected' /opt/data/profiles/profile-name/logs/gateway.log
   # Expected: "Connected as Bot Name#1234" and "✓ discord connected"
   ```

## Cron-to-Agent Job Pattern

A durable two-tier pattern for automated agents: lightweight watchers + LLM-powered scanners.

### Tier 1: no_agent watcher script
- Python script at `~/.hermes/scripts/<name>.py`
- Runs every 5m via `cronjob` with `no_agent=True`
- Reads API, compares against stored JSON state file
- Silent exit (no stdout) if nothing changed
- Prints delta if change detected → delivered to Discord
- State file at `~/.hermes/plans/<watcher-state>.json`

### Tier 2: LLM-powered scanner cron
- Runs on schedule (e.g., twice daily) via `cronjob` with `skills=["think"]`
- Reads watcher state file for deltas
- Uses think skill gates (C1-C3) to validate anomalies
- Outputs report or escalates based on findings
- Writes state at end for cross-session dedup

### Key differences from interactive sessions
- **No slash commands** — `/skill think` doesn't work in cron. Use `skills=["think"]` parameter.
- **No prior context** — every run is a fresh session. The prompt must be fully self-contained.
- **SOUL.md is NOT loaded** — cron sessions don't load the profile's SOUL.md. All instructions must be in the prompt.
- **Silence is valid** — if nothing changed, output nothing. The cron system supports silence (empty stdout = no delivery).

## Pitfalls

- **DISCORD_ALLOW_BOTS defaults to "none"** — this is the #1 reason cross-agent chat fails. Bots silently drop all messages from other bots until this is changed. Pair with `DISCORD_BOTS_REQUIRE_INLINE_MENTION=true` to prevent reply loops.
- **Reply loops:** With `DISCORD_ALLOW_BOTS=all`, bots reply to each other infinitely because Discord's reply feature auto-adds the replied-to bot to `message.mentions`. Fix: set `DISCORD_BOTS_REQUIRE_INLINE_MENTION=true` — requires a literal `<@bot_id>` in the message body. Reply-pings don't count.
- **Thread isolation:** Discord creates separate threads per user message. Bots in different threads can't see each other. Use `DISCORD_AUTO_THREAD=false` or `DISCORD_NO_THREAD_CHANNELS` for shared-channel cross-agent communication.
- **@-mention requirement:** Gateways only process @-mentioned messages outside the home channel. Always @-mention the target bot by user ID in cross-agent messages.
- **Token cleanup:** When copying `.env` from main profile, old tokens accumulate. Each profile's `.env` must have exactly one `DISCORD_BOT_TOKEN`. Clean with `sed -i '/old_token/d'` after `cp`.
- **Gateway limits:** 2 Discord connections per bot token. One for main gateway, one for one profile gateway. For more profiles, use separate bot applications with distinct tokens.
- **State file discipline:** Every scan must read state at start and write at end. Without this, the agent loses cross-session memory and re-flags the same anomalies each run.
- **Cron vs gateway:** Cron delivers to Discord channels directly without needing a running gateway. Profiles that are cron-only still produce output — they just can't do interactive chat.
- **MCP tokens are global:** HTTP/OAuth MCP servers store tokens in `~/.hermes/mcp-tokens/`, not per-profile. Use stdio MCP servers with API key env vars for per-profile isolation.
- **Cron job profile:** cronjob tool doesn't have a profile parameter. Cron sessions run under the default profile. If the cron needs profile-specific MCP tools, configure them globally or use a no_agent script.
