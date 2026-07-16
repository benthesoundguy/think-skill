# Decision Gates: Using Think-Beta's Draft Review Criteria in Autonomous Agents

When think-beta runs its Draft Review, it checks three binary criteria:

- **C1 (Grounded):** Every claim in Know/Confident has a specific citation.
- **C2 (Delta):** This cycle produced substantively new content vs the prior.
- **C3 (External input):** At least one non-trivial external source was consumed.

This reference covers a **narrower application**: using C1/C2/C3 as standalone
validation gates before an autonomous agent takes an action — without running
the full think-beta Wonder→Discovery→Draft→Review→Red Team cycle. The gates
become a lightweight "should I act?" filter.

## When to Use Decision Gates

Any time an autonomous agent (cron job, watcher script, automated monitor)
detects a candidate change and needs to decide whether to:

- Write a comment on a task
- Escalate to a human channel
- Change a status field
- Post a notification

The gates prevent the agent from acting on bad evidence, duplicate flags, or
insufficient investigation. This is the **gate pattern**: validate before you
write, not after.

## Applying the Gates

For every candidate action, run three checks:

### C1 — Evidence

Can the agent point to specific, citeable data that confirms the issue?

```
PASS: "Task M6 (86baxz6e2) has status='blocked', dependencies=[], 
       description='' (empty), no finding references this task ID."
FAIL: "M6 is blocked without reason." (no citations)
```

**Rule of thumb:** the agent should be able to produce an exact field:value
pair from the source system. If the claim can't be backed by a specific read,
C1 fails.

### C2 — Delta

Has this same issue been flagged before?

The agent needs durable cross-session state — a state file, a memory record,
or a comment thread on the affected entity. Check:

```
PASS: Agent reads state file → anomaly type "orphan_blocked" on task X has
      tier=1, comment_id=90140227464789. This is the same anomaly — move to
      tier 2 escalation.
PASS: Agent reads state file → no record of this anomaly on task X. New
      finding — proceed to tier 1 comment.
FAIL: Agent has no state file. Cannot determine if this was flagged before.
```

### C3 — External Input

Has the agent actually read the full source data, or is it operating on a
surface-level summary?

```
PASS: Agent called get_task_details(task_id) and read custom fields, 
      comments, and status history before concluding.
FAIL: Agent only saw the task name in a list output.
```

## Escalation Tiers (Using Gates Across Sessions)

The gates aren't one-shot. Each session re-checks C1-C3 and progresses the
escalation tier:

| Tier | Action | C1-C3 Check | State Update |
|------|--------|-------------|--------------|
| 0 (detected) | Candidate anomaly passes C1-C3 for first time | All pass | Record in state: tier=1 |
| 1 (commented) | Write a comment on the task. Next scan: re-check | C1 (anomaly still exists?), C2 (comment has no reply?), C3 (re-read to confirm) | tier=1 → tier=2 if no response |
| 2 (pinged) | Post an @mention in the escalation channel. Next scan: re-check | Same C1-C3 re-check + check for response | tier=2 → tier=3 if still no response |
| 3 (fixed) | Fix directly. Write a correction note. | Auto-fix | tier=3 resolved |

### C2 Re-check Logic

On subsequent scans, C2 changes meaning slightly:
- **First occurrence:** C2 = "has this ever been flagged?" (check state file)
- **Subsequent (tier 1):** C2 = "has the task been updated since my comment?"
  (check comments for a reply, check if status changed)
- **Subsequent (tier 2):** C2 = "has the escalation channel message gotten a
  response?" (check the Discord/Slack thread)

## State File Schema

The agent needs durable cross-session memory. A JSON state file works:

```json
{
  "last_scan": "2026-07-15T22:00:00Z",
  "scan_count": 12,
  "anomalies": {
    "task_id_abc": {
      "type": "orphan_blocked",
      "first_seen": "2026-07-15T20:00:00Z",
      "last_seen": "2026-07-15T22:00:00Z",
      "tier": 1,
      "comment_id": "90140227464789",
      "details": "M6 blocked with no upstream deps and no finding"
    }
  }
}
```

**Rules:**
- Read at start of every scan
- Write at end of every scan (even if nothing changed)
- Don't re-flag an anomaly at the same tier — only escalate
- If the anomaly is resolved (entity status changed appropriately),
  mark it resolved and remove from active tracking

## Concrete Example: M6 Orphan Blocked

```json
// Scan 1 state (after action):
"86baxz6e2": {
  "type": "orphan_blocked",
  "first_seen": "2026-07-15T20:00:00Z",
  "last_seen": "2026-07-15T20:00:00Z",
  "tier": 1,
  "comment_id": "90140227464789",
  "details": "M6 flagged — blocked with no upstream deps, empty description, no finding"
}

// Scan 2 re-check:
// C1: M6 still blocked, still no finding → pass
// C2: Comment on task has no reply → pass (escalate)
// C3: Re-read full task details → still no explanation → pass
// Action: escalate to tier 2 — post @mention in #pm-gateway
// State update: tier → 2

// Scan 3 re-check (if no response):
// C1: M6 still blocked
// C2: No reply to comment, no @mention response
// C3: Confirm still unexplained
// Action: tier 3 — fix directly (set status to "needs input"
//         or similar, add correction comment)
// State: tier → 3, resolved: true
```

## Relationship to Full Think-Beta Workflow

Decision gates are NOT a replacement for the full think workflow. They are a
**lighter variant** for scenarios where:

| Full think-beta | Decision gates |
|----------------|----------------|
| Exploration: Wonder → Discovery → Draft | Validation: C1-C3 on a candidate |
| Multi-cycle with convergence detection | Single-pass before action |
| Generates analysis | Decides whether to act |
| Know / Confident / Uncertain / Next steps | Pass / Fail per criterion |

Use the full workflow when investigating an unknown. Use decision gates when
you already have a candidate action and need to validate before executing.

## Implementation Pattern (Cron Agent)

```python
# Pseudocode for a cron-driven agent using decision gates
state = read_state_file()
current = read_source_system()
changes = diff(state, current)

for change in changes:
    c1 = check_evidence(change)   # Can I cite specific field:value?
    c2 = check_delta(change, state)  # Already flagged?
    c3 = check_full_read(change)  # Did I read the full entity?
    
    if all([c1, c2, c3]):
        tier = state.get_tier(change.entity)
        if tier == 0:
            write_comment(change)
            state.set_tier(change.entity, 1)
        elif tier == 1 and not has_response(change):
            post_escalation(change)
            state.set_tier(change.entity, 2)
        elif tier == 2 and not has_response(change):
            fix_directly(change)
            state.set_tier(change.entity, 3, resolved=True)
    else:
        log_discard(change, c1_result, c2_result, c3_result)

write_state_file(state)
```
