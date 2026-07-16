---
name: think-alpha
description: "Lightweight structured thinking for quick analysis or sub-agent work. Optional Wonder/Discern — start at Discovery for directed work."
version: 2.0.1
author: benthesoundguy
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [thinking, analysis, agent, workflow, lightweight]
    related_skills: [think, think-beta, plan]
---

# Think Alpha

Lightweight thinking for quick analysis, single-cycle passes, or sub-agent work. **For directed work (sub-agents, focused questions), start at Phase 2 (Discovery).** The full phase chain is for multi-cycle exploration.

Triggers: loaded by `think` (router) for quick tasks and sub-agent delegation.

---

## Core behavior

- Do not implement or mutate. Investigate and report.
- Work silently. Do not output intermediate thinking cycles. Only the final output is shown. If the user asks for progress mid-cycle, summarize the current Draft from the scratch pad and continue.
- **Tier 1 agents (narrow tasks):** Default to Phase 2 (Discovery). May start at Phase 0 if the problem requires exploration.
- **Tier 2 agents (broad exploration):** Start at Phase 0 (Wonder). Full phase chain.

---

## Revision Count

Default 2 cycles (V3). Override from user. Max 12. Early exit if solid.

Multi-cycle: track threads + working memory in scratch pad. Check the router's sizing arc for projected cycle count — look under `skill_view('think')`'s Sizing section for arc guidance.

---

### Phase 0 — Wonder

Generate questions about the problem. Forced diversity: 1 from each type (Factual, Assumption, Gap, Inversion, Scope, Dependency, Consequence, History, Cost, Success). Then 3 from the opposite premise. Output: numbered list.

Arc-aware emphasis: early cycles → broad Wonder. Mid cycles → targeted Wonder. Late cycles → minimal Wonder (converge toward Draft and Red Team).

### Phase 1 — Discernment

Rate each 1-5. Address 3+. 5=blocks, 4=shapes direction, 3=worth addressing, 2=defer, 1=skip.

### Phase 2 — Discovery (required)

Minimum 2 commands. Record each with output. Use the right tool for the question — don't default to local files.
- **To validate an assumption about a live system:** `curl`, `browser_navigate`, or check documentation via browser.
- **To inspect code:** `read_file`, `search_files`, `grep`.
- **To research a concept:** `browser_navigate` (docs, specs, references), `curl` for APIs.
- **To check history:** `session_search`.
⚡ Measure, don't estimate — `wc -l`, `du`, `grep -c`, `find` for any numeric claim.

### Phase 3 — Gap Analysis & Wandering

Adversarial: what's still uncertain? Wandering: adjacent ideas, opposite directions. Output: gaps + tangents.

### Phase 4 — Draft (compressed)

Working draft — NOT final output. Bullets/tables. Structure: Know | Confident | Uncertain | Next steps | Risks. Minimum viable size.

### Phase 5 — Red Team

Minimum 2 specific citations per pass. Try to disprove claims, break direction, find edge cases. Must include the check — bare "couldn't disprove" fails.

### Phase 6 — Loop or Deliver

Nth cycle → Final Presentation. Else → return to Phase 0. Low insight density 2× consecutive → early exit prompt.

---

## Final Presentation

Format the last Draft into a user-friendly summary. The internal Draft structure (Know | Confident | Uncertain | Next steps | Risks) should be invisible — present findings in narrative form:

1. **Bottom line** — 1-2 sentences answering the original question.
2. **Key findings** — the most important claims, grouped by theme, with supporting citations.
3. **Open questions** — remaining uncertainties if relevant.
4. **Recommendations** — actionable next steps.
5. **Caveats** — risks and limitations to be aware of.

Do not reference cycles, Draft Reviews, or other internal mechanics. If agents were used, do not reference them in the main body — a brief note at the end is sufficient.

If this session involved multi-agent escalation, optionally `hindsight_retain` assessment tags: "thinking: decomposition [correct/overlapped/fragmented]" and "thinking: tier selection [correct/wrong — should have been Tier X for sub-problem Y]".

---

## Edge Cases

- **Tier 1:** Default to Phase 2, may Phase 0 if stuck.
- **Tier 2:** Full phase chain from Phase 0.
- **Empty phase:** Note "empty" and continue — Red Team catches gaps.
- **No conclusion:** Surface explicitly. Inconclusive is a valid finding.

## Output types

Analysis, findings, recommendations. For implementation plans, load `plan` skill for formatting.

## Agent escalation

This skill is used as the thinking engine for sub-agents. To spawn sub-agents from this session, load the escalation reference: `skill_view('think', 'references/agent-escalation.md')`.