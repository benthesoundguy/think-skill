# Agent Escalation — Multi-Agent Orchestration

Reference for spawning sub-agents. Loaded via `skill_view('think', 'references/agent-escalation.md')` from any thinking session.

---

## When to spawn

All three must be true:

1. **Concrete goal** — Can you write a specific goal for this sub-problem? If not, don't spawn.
2. **Cost justifies delegation** — Can you resolve it with a single command right now? If yes, just run it. Don't spawn.
3. **Independent** — Is the sub-problem loosely coupled to your current analysis? If tightly coupled (splitting would require re-integration), resolve in context. Don't spawn.

## Tier selection

- **Tier 1 (narrow tasks):** The sub-problem has a single correct answer. "What does this endpoint return?" → agent starts at Phase 2 (Discovery) in think-alpha.
- **Tier 2 (broad exploration):** The sub-problem needs exploration. "Design the schema." → agent starts at Phase 0 (Wonder), full think-alpha chain.

## Agent input contract

Pass via `delegate_task` context. Include the right `toolsets`:

| Task type | Recommended toolsets |
|---|---|
| Technical investigation (API, code) | `["terminal", "file"]` |
| Research (find docs, compare options) | `["web", "terminal"]` |
| Combined investigation | `["web", "terminal", "file"]` |

Contract format:
```
goal: "Investigate [sub-problem]"
context: |
  Tier: 1 or 2
  Background: [relevant context from the main session]
  Output format:
    Summary: [2-3 sentences]
    Know: [claims with citations]
    Confident: [strong beliefs without full proof]
    Uncertain: [specific unknowns]
    Next steps: [actions, including unresolved items tagged [UNRESOLVED]]
  Use skill: skill_view('think-alpha') and follow its workflow
toolsets: ["terminal", "file"]  # or ["web", ...] as appropriate
```

## Failure handling

- **Timeout:** Note partial result. Investigate remaining items locally in the main session.
- **Malformed output (no Know/Confident/Uncertain):** Log error. Optionally re-dispatch with stricter formatting instructions.
- **Contradiction in agent output (Know: X and Uncertain: X):** Treat as Uncertain — low confidence dominates.

## Wave management

- Max **3 concurrent agents** per wave via the `tasks` array. Exceeding the limit returns a tool error — does not silently truncate.
- While agents run, continue working on remaining undispatched sub-problems (Gap Analysis, Draft of non-dispatched portions).
- If 4+ sub-problems, batch: spawn 3, synthesize, spawn next wave for remaining.
- **Layer 2:** If agents return items in Next steps tagged `[UNRESOLVED]`, decide per item: spawn next wave, defer to user, or investigate directly.

## Synthesis (standalone)

Works for merging any two analyses — from agents, parallel discovery, or multiple Draft cycles. Does not require agent dispatch.

1. **Merge:** Combine findings from both sources. Know + Confident → Know. Uncertain + Uncertain → Uncertain. **Select top 3-5 claims per source rather than copying verbatim.**
2. **Flag contradictions:** Source A says X, Source B says not-X → test via Discovery. If can't test, add to Uncertain: "X is disputed between [sources]."
3. **Flag gaps:** Areas covered by one source but not the other → note as scope gap. Spawn if critical, defer otherwise.
4. **Produce unified Draft** incorporating all findings.

## Synthesis (after each wave)

1. **Merge:** Agent Know + Agent Confident → Main Know. Merge across all agents. Select top 3-5 claims per agent.
2. **Flag contradictions:** Agent A says X, Agent B says not-X → test via main session Discovery. If can't test, add to Uncertain: "X is disputed between [subsystems]."
3. **Flag gaps:** Areas no agent covered → note as scope gap or spawn if critical.
4. **Handle Unresolved:** Items tagged `[UNRESOLVED]` in Next steps: decide per item — spawn Layer 2 agent, defer to user, or investigate directly. If resolved by Layer 2, update Next steps: `[Resolved via Layer 2] Y — brief finding`.
5. **Produce unified Draft** incorporating all agent findings.
