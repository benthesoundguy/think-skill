# Think Skill

A structured thinking system for AI agents. Three skills that form a complete cognitive workflow for reasoning through complex problems.

- **think** — Router. Estimates problem complexity and dispatches to the right thinking mode.
- **think-alpha** — Lightweight, trust-based thinking for quick analysis and sub-agent work.
- **think-beta** — Gated, evidence-forcing thinking for deep analysis with quality gates.

## Why this exists

LLMs are fast but not careful. They race to conclusions, skip exploration, and rarely check their own work. Standard prompting patterns like "think step by step" improve things slightly but produce shallow reasoning with no verification and no way to know when the answer is actually trustworthy.

The Think Skill solves this by embedding a reasoning protocol directly into the agent's behavior. When loaded, it transforms how the agent approaches problems — from single-shot generation to a structured cognitive process.

## How it works

When you ask the agent to think about something, the router first estimates the complexity of the request:

- **Simple** — quick decision, single domain, answerable from context. Routes to alpha.
- **Moderate** — multi-faceted, needs some investigation. Routes to beta.
- **Complex** — cross-domain, many unknowns, likely needs multi-cycle work. Routes to beta.

This complexity estimate also projects how many reasoning cycles the work will likely require, so the agent paces itself appropriately through the phases.

### think-alpha (Lightweight)

Designed for quick analysis where you already trust the context. Phases:

- **Wonder** — Generate diverse questions about the problem (factual, assumptions, gaps, inversions, scope, dependencies, consequences, history, cost, success criteria).
- **Discern** — Rate each question by importance. Address the critical ones.
- **Discovery** — Investigate using tools. Measure, don't estimate.
- **Draft** — Compress findings into a working draft.
- **Red Team** — Attempt to disprove your own conclusions. Minimum 2 attempts with citations.
- **Loop or Deliver** — Check for convergence. If the analysis is solid, deliver. If not, loop back.

### think-beta (Gated)

Designed for deep analysis where thoroughness matters. Same underlying phases as alpha but with formal quality gates between them.

- **Tool Selection** — Each cycle chooses what to do next based on the previous cycle's gaps.
- **Expansion** — Generate new questions, investigate with tools, or explore adjacent ideas.
- **Draft** — Compressed working draft with four categories: what you Know, what you're Confident about, what's Uncertain, and Risks.
- **Draft Review** — The agent checks its own work against three criteria:
  - Are claims grounded in real evidence?
  - Is there substantive new information versus the previous cycle?
  - Was external input consumed and reflected in the draft?
  
  Each criterion has a budget of 3 failures. When a criterion fails, the agent is routed back to fix the specific gap — not start over. If a criterion exhausts its budget, the agent produces a candid exit document instead of faking confidence.
- **Red Team** — Read actual files. Try to disprove claims. Test edge cases. Minimum 2 specific citations.
- **Loop or Exit** — If Red Team finds nothing wrong, exit early — the analysis is solid. Otherwise loop for another cycle.

## Usage

Trigger the skill naturally:

- "Think about whether we should migrate to PostgreSQL 17"
- "Think through the scalability bottleneck in our auth service"
- "Figure out why the payment worker throws duplicate key errors"
- "Work through the trade-offs between RAG and fine-tuning"
- "Quick think on whether this API change is backward compatible"

The agent sizes the request, selects the right mode, and works through the phases autonomously. You get a final answer with findings, evidence, and remaining uncertainties — not a single unverified guess.

## Architecture

```
User request
    │
    ▼
┌─────────────┐
│   think     │  Router — sizes and routes
│  (router)   │
└──────┬──────┘
       │
       ├── Simple / Quick  ──► think-alpha (lightweight)
       │
       └── Complex / Deep  ──► think-beta (gated)
```

### Persistent state

During multi-cycle analysis, the agent maintains a scratch pad with:
- Active threads (max 5, with dependencies tracked)
- Working memory from cycle to cycle
- Draft Review failure counts per criterion
- Current cycle number and projected arc

This scratch pad survives interruptions. If the agent runs out of context mid-analysis, the next invocation picks up where it left off.

### Sub-agent escalation

For problems too broad for a single agent, the skill has built-in dispatch logic to spawn sub-agents for parallel investigation, then synthesize their findings. This preserves the thinking structure while scaling across agents.

## What's in the box

```
think-skill/
├── README.md
├── LICENSE                      # MIT
├── think/                       # Router
│   ├── SKILL.md
│   └── references/
│       ├── agent-escalation.md     # Sub-agent dispatch logic
│       ├── dispatch-pattern.md     # Skill routing technique
│       ├── industry-patterns.md    # How this relates to other approaches
│       └── three-tier-pipeline.md  # Kanban integration
├── think-alpha/                 # Lightweight thinking
│   └── SKILL.md
└── think-beta/                  # Gated thinking
    ├── SKILL.md
    └── references/
        ├── design-rationale.md     # Why the gates exist
        └── gated-agent-patterns.md # Patterns for gate design
```

## License

MIT
