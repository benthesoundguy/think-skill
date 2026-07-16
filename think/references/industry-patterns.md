# Multi-Agent Thinking — Industry Patterns

Collected during the think-system design session (2026-07-15). Research via browser/curl against academic and open-source approaches.

---

## Academic Prompting Techniques

These are single-shot prompt patterns — no state, no tool use, no agent loop. The think skill is an agentic, tool-augmented implementation of similar ideas, but these papers are stateless prompts, not a running workflow.

### Chain-of-Thought (Wei et al., 2022) / Tree-of-Thoughts (Yao et al., 2023)

- **Approach:** Multi-step reasoning via prompt engineering. CoT: "let's think step by step". ToT: explore multiple reasoning branches via BFS/DFS.
- **How it compares:** The think skill's phase chain (Wonder → Discovery → Draft → Red Team) is structurally similar to ToT's multi-branch exploration, but executed as an autonomous agent workflow with tool calls, real measurement, and persistent state between phases.
- **Key difference:** CoT/ToT are prompt patterns — no tools, no state, no agent. Every call is stateless. The think skill gives the agent an environment (files, commands, browser) to investigate claims during Discovery.

### Self-Refine (Madaan et al., 2023)

- **Approach:** Generate → Critique → Refine. The LLM produces an output, then critiques its own output, then refines based on that critique.
- **How it compares:** Draft → Red Team → Loop is the closest conceptual match in the think skill. Both involve self-critique and revision.
- **Key difference:** Self-Refine is a single-pass feedback loop with no front-end exploration. The think skill adds Wonder (divergent questioning), Discovery (tool-augmented investigation), and Gap Analysis before the critique phase. Self-Refine critiques immediately; the think skill gathers evidence first.

### Reflexion (Shinn et al., 2023)

- **Approach:** Agent reflects on past failures, stores reflections in episodic memory, and uses them to improve subsequent attempts. Used with programming tasks and decision-making.
- **How it compares:** The Red Team phase (try to disprove your own draft) shares the "check your work" ethos. Reflexion's episodic memory resembles what the think skill's revision budget tracks across cycles.
- **Key difference:** Reflexion learns across episodes (one attempt → reflect → retry). The think skill checks within a single thinking session. Reflexion is for long-term improvement; the think skill is for thoroughness in one go.

### Graph-of-Thoughts (Besta et al., 2023) / Language Agent Tree Search (Zhou et al., 2023)

- **Approach:** Model reasoning as a directed graph (GoT) or Monte Carlo tree search (LATS), allowing merging of reasoning paths and backtracking.
- **How it compares:** The think skill's phase loop (back to Wonder from Red Team) is a simpler version of graph-based backtracking. The escalation pattern (spawn sub-agents for deep exploration) parallels LATS's tree expansion.
- **Key difference:** GoT/LATS are formal search algorithms with explicit scoring. The think skill uses heuristic convergence detection and revision budgets instead. Much less mathematically rigorous, much more practical in an agent context.

---

## Model-Level Internal Reasoning

These happen inside the model's black box. The think skill is complementary scaffolding on top of whatever the model does internally.

### OpenAI o1/o3 / DeepSeek-R1 / Claude 3.7 Sonnet Extended Thinking

- **Approach:** Models generate internal "thinking tokens" before producing a visible answer. The chain of thought is hidden (o1) or shown (R1) but it's a model-level behavior, not agent-level.
- **How it compares:** Both involve multi-step reasoning before answering. The think skill's phase chain could theoretically be applied *on top* of a thinking model — the model's internal reasoning handles low-level logic while the skill handles the high-level cognitive arc: which tools to call, what questions to ask, when to stop and reconsider.
- **Key difference:** Model reasoning is invisible/immutable from the agent's perspective. The think skill structures what the agent *does* between LLM calls. You could run the think skill on top of o1 and get both layers.

---

## Multi-Agent Frameworks

Role specialization and handoff, not single-agent cognition. The think skill is primarily single-agent with optional escalation; these are inherently multi-role.

### MetaGPT (69k★ GitHub)

- **Approach:** Predefined roles (PM, Architect, Engineer, QA) with structured handoffs and produced artifacts (PRD, design docs, code).
- **How it compares to our system:** Our tier system (Tier 1 vs Tier 2) generalizes role specialization. Their artifact handoff is analogous to our agent contract/merge pattern.
- **Key difference:** They use rigid role pipelines. We use dynamic decomposition per problem.

### CrewAI (20k★ GitHub)

- **Approach:** "Crews" of role-based agents with "Flows" for event-driven orchestration.
- **How it compares:** Their "Flow" pattern (event-driven chaining) is what our synthesis phase does — merge agent outputs, flag contradictions, produce unified Draft.
- **Key difference:** Code-first setup vs our skill-driven approach. We do flow manually rather than through an event system.

### AutoGPT

- **Approach:** Block-based workflow builder with pre-configured agents. Low-code agent creation.
- **How it compares:** Their block concept maps roughly to our phases — each phase is a block in the thinking pipeline.
- **Key difference:** Our phases are better optimized for LLM cognition (Wonder → Discover → Draft → Red Team is intentionally ordered). Their blocks are generic.

### Nexa Thinking Framework (NexaEthos, 2026)

- **Approach:** Full-stack (FastAPI + React + Tauri) multi-agent orchestration app with chain-of-thought, project canvas, and research lab workspaces. Agents collaborate via a shared canvas with @mention routing.
- **How it compares:** Superficially similar naming ("thinking framework"), but a different target entirely — it's a user-facing desktop/web app for human-guided multi-agent work, not an agent's own reasoning skill.
- **Key difference:** The think skill is embedded in the agent's runtime (loaded and followed autonomously). Nexa is a standalone application the user opens. Different layers of the stack.

### Hermes Kanban (built-in)

- **Approach:** Durable SQLite-backed multi-agent work queue with dispatcher, workers, heartbeat, failure limits.
- **How it compares:** Kanban's dispatcher does what our Agent Escalation section describes — but with durable state and automatic retry.
- **When to use it over delegate_task:** Multi-session, multi-profile, or long-running orchestration (>1 hour). Not for single-session multi-agent work.
- **Setup cost:** Requires board init, dispatcher config, worker profile config. Overhead justified only for durable work.

---

## Agent Thinking Skills (Emerging Category, 2025–2026)

A new category that emerged in 2025–2026: **SKILL.md files that define a structured reasoning protocol an agent loads and follows autonomously.** Unlike academic prompts (stateless), model reasoning (black box), or multi-agent frameworks (role specialization), these are behavioral protocols embedded in the agent's own runtime. These are the closest analogues to the think skill — same technical category, different designs.

As of July 2026, nine public repos exist. The think skill is the only one that combines two-tier routing, Draft Review with per-criterion budgets, adversarial Red Team, agent escalation, convergence detection, and arc-aware phase behavior.

| Skill | Stars | Created | Phases | Unique approach |
|---|---|---|---|---|
| **cc-thinking-skills** (tjboudreaux) | 689★ | ~2025 | 39 individual skills | Library of mental models with a "thinking-model-router" that sends the agent to the right single skill. Not a multi-phase pipeline — each skill is one technique. |
| **hegelian-dialectic-skill** (KyleAMathews) | 562★ | ~2025 | 3-agent dialectic | Two AI subagents commit to opposing positions; a third orchestrates synthesis. Subagents are the *core mechanism*, not optional escalation. |
| **comprehensive-thinking-skill** (syzkillall) | 33★ | May 2026 | 5 layers (五重审视) | Hegelian dialectics + "master theory systems" + anti-formalism discipline. Chinese-language. Knowledge-centric — assumes domain expertise via external theory. Monolithic one-pass pipeline. |
| **cascade-thinking-mcp** (drewdotpro) | 21★ | Jul 2025 | Branching trees | MCP server for managing branching thought trees. A *tool* the agent calls, not a behavioral protocol the agent follows. |
| **self-inspect-mcp** (ejentum) | 6★ | Jun 2026 | 1 question | Single metacognition question tool. Keyless, deterministic. "Ask yourself one question" — no workflow, no phases. |
| **grant-thinking-skill** (Agents365-ai) | 5★ | ~2026 | Domain-specific | Grant funding reasoning. Noteworthy for supporting Hermes Agent format (metadata.hermes namespace). |
| **knowledge-skills** (deciqAI) | 3★ | Jun 2026 | 227 individual skills | Library of composable mental models (First Principles, Inversion, Bayesian reasoning). Individual techniques, not a structured pipeline. |
| **deep-thinking-skill** (shade-solutions) | 2★ | Apr 2026 | 7 phases | Closest analogue to the think skill. Phases: Activation → Problem Intelligence → Decomposition → Systematic Reasoning → Multi-Path → Failure Mode → Confidence Scoring. Problem-solving methodology orientation. |
| **agent-thinking-skills** (DerekYRC) | 1★ | Jul 2026 | 2 skills | first-principles (re-derive from fundamentals) + adversarial-review (parallel code review agents). Code-focused. |

### How the think skill differs from the category

| Feature | think skill | deep-thinking-skill (closest) | comprehensive-thinking-skill (most popular) | cc-thinking-skills (most starred) |
|---|---|---|---|---|
| **Two-tier routing** (alpha/beta) | ✅ | — | — | Has a router, but sends to *different* skills, not gated vs light versions of one skill |
| **Draft Review w/ per-criterion budgets** (C1/C2/C3, 3 strikes) | ✅ | Has "Absolute Rules" (textual list, no budget) | Has "anti-formalism discipline" (process adherence, no binary pass/fail) | — |
| **Adversarial Red Team** (file reads, min citations, disprove requirement) | ✅ | "Verify" step is self-check, no file/citation requirement | "Strongest opposing view" in layer 4, no citation requirement | — |
| **Agent escalation within thinking** | ✅ | — | — | — (hegelian-dialectic has subagents as core mechanism, different concept) |
| **Convergence detection** (early exit) | ✅ | Runs all 7 phases every time | Runs all 5 layers every time | N/A (skill library) |
| **Arc-aware phases** (behavior changes by cycle) | ✅ | — | — | — |

### Category trajectory

The "agent thinking skill" category is accelerating — 9 repos in 12 months, with 3 of those (deep-thinking-skill, comprehensive-thinking-skill, agent-thinking-skills) being multi-phase reasoning protocols specifically. This suggests the category will grow and the think skill's specific feature combination may be replicated. The think skill's current novelty lies in:
1. Being the **first** to combine all six unique features
2. Having the **most sophisticated quality-control machinery** (Draft Review with budgets, not just rules)
3. Being the **only one designed as an epistemological framework** (what do I know? what am I uncertain about? how do I test my conclusions?) rather than a problem-solving methodology

---

## Novelty Analysis (derived from user conversation, 2026-07-15)

The individual components are all known in isolation — phase-based reasoning (CoT/ToT), self-critique (Self-Refine/Reflexion), multi-attempt with convergence (LATS, MCTS), sub-agent dispatch (multi-agent frameworks).

What makes the think skill novel is the **specific combination** of:

1. **Ordered cognitive arc** — divergent exploration (Wonder) → measurement (Discovery) → adversarial critique (Red Team) → loop with explicit convergence check. This order is intentional and optimized for LLM cognition.
2. **Embedded as a bootable protocol** — the agent loads it via skill_view() and follows it autonomously. It's not a separate system, a paper, or a standalone app — it's how the agent thinks, running inside the agent's own runtime.
3. **Meta-routing by trust level** — alpha (quick, trusted context) vs beta (gated, evidence-forcing, multi-cycle). The system decides how much reasoning rigor to apply based on context.
4. **Revision budgets with convergence detection** — explicit per-cycle budgets, 3-strike pattern, insight-density-based early exit.
5. **Sub-agent dispatch within the thinking workflow** — not a separate system, but a phase of the thinking process: "this is too broad for one agent, spawn workers."

**Bottom line:** Academic papers have the pieces in isolation. No one shipped them as a working agent skill in a production system. The think skill is a new OS primitive: *how an agent should think about hard problems.*

---

## Summary — When each approach fits

| Approach | Best for |
|---|---|---|
| Our ad-hoc delegate_task (via Agent Escalation) | Single-session multi-agent work. 90% of cases. |
| Hermes Kanban | Multi-session, multi-profile, durable orchestration. 10% of cases. See `references/multi-agent-teams.md` for the full profile-based architecture pattern. |
| ClickUp-Manager + Kanban pipeline | Full autonomous loop: ClickUp=plan, Kanban=engine, Think=quality. Three layers, each doing what it's best at. Requires: (1) clickup-manager profile (MCP-only read/write, no implementation tools), (2) orchestrator profile (kanban-only create/link/assign, no implementation tools), (3) shared state file for cross-profile memory, (4) dedicated Discord bot per profile. See `references/multi-agent-teams.md`. |
| Fixed role pipelines (MetaGPT style) | Known, repeatable workflows with predictable roles. Not prioritized. |
| Event-driven flows (CrewAI style) | Complex branching logic. Our manual flow is sufficient for current needs. |
