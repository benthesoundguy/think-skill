---
name: think
description: "Router — dispatches to think-alpha (trust-based) or think-beta (gated) with sizing estimate. Projects complexity arc for multi-cycle work."
version: 1.3.0
author: benthesoundguy
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [thinking, planning, analysis, routing, sizing, metacognition]
    related_skills: [think-alpha, think-beta, plan, requesting-code-review]
---

# Think (Router)

Routes thinking tasks to the appropriate sub-skill with a sizing estimate. This skill dispatches — the actual workflow lives in **think-alpha** (trust-based) or **think-beta** (gated).

Triggers: same as both sub-skills — "think about X", "think on X", "think through X", "figure out X", "meta on X", "work through X", "plan X".
Count overrides: "N iterations/loops/times around/cycles" set Revision Count.

---

## Sizing (pre-dispatch)

Before routing, produce a complexity estimate for internal arc planning:

**Heuristic** (quick read of the message, not a full thinking pass):
- **Simple (1-2 cycles):** Short message, single domain, decision/comparison verb. Route to alpha.
- **Moderate (3-6 cycles):** Multi-faceted, needs investigation, 1-2 sub-problems. Route to beta.
- **Complex (7-12):** Cross-domain, many unknowns, likely needs agent escalation. Route to beta.

The estimate is a starting point. The arc adjusts dynamically during execution.

**Arc projection** (passed to the sub-skill as context):
- Cycles 1-3: Broad exploration (Wonder, Discovery)
- Cycles 4-6: Deep investigation, possible agent dispatch
- Cycles 7+: Convergence, Draft, Red Team

The arc lives in the working memory header. The sub-skill adjusts it as the session progresses.

---

## Core behavior

- Read the user's request.
- Determine which sub-skill to use (see routing below).
- Load that sub-skill with `skill_view()` and follow its workflow.
- Do not execute the workflow inline here — delegate to the sub-skill's instructions.

---

## Routing

**Default: think-beta** (gated, evidence-forcing, Draft Review with 3 criteria, per-criterion budgets, convergence detection).

Route to think-alpha when:
- User explicitly says "use alpha" or "quick think" or "quick thought"
- The request is a single well-scoped question answerable from context alone
- Speed matters over thoroughness

Route to think-beta when:
- The request mentions iterations, loops, cycles, or multi-step analysis
- The problem involves code, infrastructure, systems, or architecture
- Any ambiguity exists about complexity
- **Default for anything non-trivial**

---

## Sub-skill loading

After routing, call:
```
skill_view('think-alpha')  # for trust-based thinking
skill_view('think-beta')   # for gated thinking
```

Read the loaded skill's content and follow its instructions. The system prompt compels following loaded skill workflows.

See `references/dispatch-pattern.md` for the full technique — how skill_view-based routing works, when to use it, and how to extend it for future sub-skills.
See `references/industry-patterns.md` for how our approach compares to academic prompting (CoT/ToT/Self-Refine), model-level reasoning (o1/R1), multi-agent frameworks (MetaGPT, CrewAI, AutoGPT, Nexa), and Hermes Kanban — plus a novelty analysis.
See `references/three-tier-pipeline.md` for the ClickUp → Kanban → Think autonomous-loop architecture — profile roles, worker lifecycle, and concrete examples using live Agents PM milestones.
See `references/decision-gates.md` for using think-beta's Draft Review criteria (C1/C2/C3) as standalone validation gates in autonomous agent contexts — decide whether to comment, escalate, or fix without running the full think cycle.

See `references/multi-agent-teams.md` for setting up profile-based autonomous agent teams with dedicated Discord bots, shared state files, and cross-agent communication protocols — the architecture behind clickup-manager + kanban-orchestrator pipelines.

---

## Agent escalation

See `skill_view('think', 'references/agent-escalation.md')` for the full dispatch logic — spawn decision, tier selection, contract format, failure handling, wave management, and synthesis.

### Condensed contract

Quick-reference for constructing `delegate_task` calls without loading the full reference:

- **Spawn if:** concrete goal AND not resolvable with one command AND not tightly coupled.
- **Spawn timing by arc position:** Early cycles → Tier 2 agents for exploration. Mid cycles → Tier 1 agents for specific blockers. Late cycles → only if Red Team identifies a concrete gap.
- **Tier 1** = single correct answer → agent starts at Discovery. **Tier 2** = exploration needed → agent starts at Wonder.
- **Input:** `goal`, `toolsets`, `context` with Tier, Background, output format, and `skill_view('think-alpha')`.
- **Output:** Summary, Know, Confident, Uncertain, Next steps (tag unresolved `[UNRESOLVED]`).
- **Max:** 3 concurrent. Exceeding returns a tool error — batch in waves.