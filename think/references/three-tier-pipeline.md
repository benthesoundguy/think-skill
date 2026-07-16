# Three-Tier Pipeline: ClickUp → Kanban → Think

Architecture for autonomous agent-driven project execution. Each layer
has a distinct responsibility; no layer duplicates another's job.

```
┌──────────────────────────────────────────────────────────────────┐
│  TIER 1: ClickUp (The Plan)                                      │
│  Human-facing. Milestones, decisions, findings, spec work.       │
│  Managed by clickup-manager profile (ClickUp MCP tools only).    │
│  Polling: reads all lists, cross-references findings vs tasks.   │
│  Output: "M0 is in progress, no blocking findings, actionable."  │
├──────────────────────────────────────────────────────────────────┤
│  TIER 2: Kanban (The Execution Engine)                           │
│  Durable SQLite board shared across profiles.                    │
│  Managed by orchestrator profile (kanban tools only).            │
│  Input: milestone becomes ready → decompose into cards.          │
│  Output: cards completed with structured summaries.              │
├──────────────────────────────────────────────────────────────────┤
│  TIER 3: Think (Per-Card Quality)                                │
│  Methodology inside each worker card.                            │
│  think-alpha for focused investigation (researcher profile).     │
│  think-beta for gated quality analysis (reviewer profile).       │
│  Output: evidence-grounded findings, verified code.              │
└──────────────────────────────────────────────────────────────────┘
```

## When to Use This

This is for **multi-session, multi-profile, durable orchestration** —
work that spans hours or days, survives restarts, involves human-in-the-loop
unblocking, and requires an audit trail. Roughly 10% of cases.

Use `delegate_task` + think for quick in-session sub-agent work (90% of
cases). This pipeline is overhead that pays off when you have multiple
profiles, multiple milestones, and want walk-away execution.

## Profile Roles

### clickup-manager

The sensory layer. This profile has **only** the ClickUp MCP tools +
memory. It cannot execute implementation work.

Its job:
- Poll Agents PM space on a schedule
- Report: which milestones are actionable, which are blocked, which findings
  still gate which tasks
- Detect status inconsistencies (e.g. S2 says "ready" but a finding says
  "FAST full load blocked by Cloudflare — soft-gates S2")
- Write results back when workers complete — update milestone status, post
  findings, promote next-in-chain milestones
- Maintain a durable state snapshot of the space so the orchestrator never
  queries ClickUp cold

Concrete scan output shape:
```
Space: Agent PM
Scan at: 2026-07-15 20:00 UTC

Milestones:
  M0 (Prereqs): in progress — no blocking findings, actionable
  M1 (Schema DDL): in progress — no blocking findings, actionable
  M2 (read_node_by_id): blocked — blocking finding #F8 open
  S2 (Semantics sim): ready — ⚠ finding #F1 says FAST load blocked
  M3–M8: all blocked

Findings gating active milestones:
  F1: FAST full load blocked by Cloudflare — soft-gates S2
  F8: Addendum D forked on disk — gates M2
```

### orchestrator

The decomposition and routing layer. This profile has **only** kanban
tools (create, link, comment, list), memory, and possibly gateway. It
literally cannot execute implementation work.

Its job:
- Receive scan from clickup-manager
- For each actionable milestone: read the spec, decompose into Kanban cards
  with proper dependency chains (Research → Build → Review)
- Assign each card to the correct specialist profile by name
- Set parent→child links so the dispatcher auto-promotes
- Add a coordinating comment linking back to the ClickUp task URL
- Itself creates no output other than the card graph

A canonical decompose:
```
Input: M0 (Prereqs) from ClickUp — spec covers git bare repo,
       post-receive hook, n8n webhook

Output: 3 Kanban cards:
  Card 1: "Set up git bare repo on .10"
    → assignee: infra-ops
  Card 2: "Configure post-receive → n8n webhook"
    → assignee: infra-ops
    → parent: Card 1
  Card 3: "Verify end-to-end: push → webhook fires"
    → assignee: infra-ops
    → parent: Card 2
```

### Specialist Profiles (workers)

One profile per capability, restricted to the tools that capability needs:

| Profile | Tools | Skill |
|---------|-------|-------|
| researcher | web, browser, terminal (curl), file | think-alpha |
| clickup-dev | terminal (npm/bun/git), file, search, code_execution | — |
| reviewer | file, terminal (test/lint), search | think-beta |
| infra-ops | terminal (SSH/systemd/ZFS), file | — |

Each worker gets the `kanban_*` toolset auto-injected by the dispatcher
(no profile config needed). The worker calls `kanban_show()` on spawn,
does its work, calls `kanban_heartbeat()` during long ops, and calls
`kanban_complete()` or `kanban_block()` when done.

## The Autonomous Loop

```
[every 1-6 hours on cron]

  1. clickup-manager scans Agents PM
     → produces scan report (JSON in kanban comment)
  
  2. If no actionable milestones: LOOP ENDS (nothing changed)

  3. orchestrator reads scan report
     → for each actionable milestone:
       - creates Kanban cards with dependency chain
       - assigns to specialist profiles
       - links to ClickUp task URL
  
  4. Dispatcher picks up ready cards
     → spawns workers in dependency order
  
  5. Each worker:
     → kanban_show() on spawn
     → does the work (using think methodology)
     → kanban_heartbeat() during long ops
     → kanban_complete() with structured summary
  
  6. When a card chain completes:
     → clickup-manager reads the final card's summary
     → updates ClickUp milestone status + posts findings
     → promotes next milestone if conditions met
  
  7. GOTO 1
```

## When the Loop Breaks

Things that require human attention and how they surface:

| Failure mode | Signal | Action |
|---|---|---|
| Worker crashes mid-task | Dispatcher reclaims, auto-retry up to limit, then block | Human unblocks and respawns |
| Task blocked on "need input" | Card status = blocked + reason | Human unblocks when free |
| Orchestrator produces wrong cards | Output doesn't match milestone spec | Human edits/reassigns cards |
| ClickUp scan finds inconsistency | ⚠ flag in scan report | Human corrects the ClickUp status |
| No milestones changed | Scan shows zero delta from previous | Loop idles — nothing to do |
| Status says "ready" but finding blocks | Stale-status warning | Human reconciles status vs findings |

## Concrete Example: M0 (from live Agents PM)

```
ClickUp state:
  M0 status: "in progress"
  Blocking findings: none
  Gating decisions: none open
  Spec reference: clickup-mcp-fork-spec §3

clickup-manager scan → orchestrator:
  "M0: in progress, unblocked, spec §3 → actionable"

orchestrator decompose:
  Card A "Set up git bare repo on .10" → infra-ops
  Card B "Configure post-receive → n8n" → infra-ops (parent: A)
  Card C "Verify end-to-end push" → infra-ops (parent: B)

Dispatcher → infra-ops Card A:
  → kanban_show() → reads spec body
  → ssh .10 → git init --bare /var/git/vault.git
  → kanban_heartbeat(note="bare repo created, configuring hooks")
  → writes post-receive hook script
  → kanban_complete(summary="repo at /var/git/vault.git, hook wired")

Card B auto-promotes. infra-ops respawns, reads A's summary,
configures n8n webhook, tests. Completes.

Card C auto-promotes. Pushes test commit, confirms webhook fires.
Completes.

clickup-manager reads final summary:
  → updates M0 to "done"
  → posts finding: "M0 complete — bare repo + hook + webhook verified"
  → checks M1: "in progress, no blocking findings → needs human
     review before auto-promoting"
```

## When NOT to Use This Pipeline

- One-off investigation that finishes in one session → think + delegate_task
- Quick question → just ask the agent directly
- No agent execution needed → stays in ClickUp only
- Single profile, single session → Kanban overhead not justified

## Relationship to Industry Patterns

This pipeline sits between the extremes in `industry-patterns.md`:

- **MetaGPT-style rigid pipelines** — too inflexible; our profiles are
  discoverable maps, not fixed role queues
- **Plain delegate_task** — too ephemeral; no durability, cross-session
  survival, or human-in-the-loop

The three-tier pipeline is a specific instance of the general "plan →
execute → verify" pattern, tuned for ClickUp as the plan surface and
Hermes Kanban as the execution surface.
