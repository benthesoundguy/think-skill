---
name: think-beta
description: "Gated structured-thinking with deterministic phase routing. Draft Review forces quality before Red Team. Not code-specific."
version: 1.1.0-beta
author: benthesoundguy
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [thinking, planning, analysis, metacognition, gated, workflow]
    related_skills: [think, think-alpha, plan, requesting-code-review]
---

# Think Beta

A gated thinking skill. Same domain as `think` — work through a problem, question, or area of uncertainty — but with strict guardrails to prevent shortcut loops. Designed for multi-cycle sessions where an LLM would otherwise default to Draft → Red Team without expanding knowledge.

Triggers: same as `think` — "think about X", "think on X", "think through X", "figure out X", "meta on X", "work through X".
Count overrides: "N iterations/loops/times around/cycles" set Revision Count.

---

## Core behavior

- Do not implement or mutate. Thinking/planning only.
- Work silently. Do not output intermediate thinking cycles. Only the final output is shown. If the user asks for progress mid-cycle, summarize the current Draft from the scratch pad and continue.
- Deliverable saved to `.hermes/plans/` with timestamp.

### Scratch pad (persistent working memory)

At cycle 1, create a scratch file at `.hermes/scratch/YYYY-MM-DD_HHMMSS-think-slug.md`. This file holds working state between cycles:

```
Threads table
Working memory header
Current Draft
Last Draft Review result
Next tool assignment
Draft Review failure tracking (C1_strikes, C2_strikes, C3_strikes)
```

Update the scratch file after every phase that produces output. Read it at cycle start to resume state.

Delete the scratch file after Final Presentation. Keep it only if the user explicitly asks.

If interrupted mid-cycle (context compression), the scratch file survives and the next invocation reads it to restore state.

---

## Revision Workflow

### Count

Default: 2 cycles → V3 deliverable. Override: user-specified. Max: 12.
Single-cycle: if the user explicitly asks for 1 cycle, skip Draft Review and Red Team gates. Default is always 2.

### Threads (persistent across cycles)

```
T# | Description | Status | Blocks | Blocked by | Last finding
```

Max 5 active. Resolved threads don't count. "Blocked by" references Draft Review criteria that are preventing progress on this thread — this feeds Tool Selection deterministically.

---

## Cycle Structure

### Phase 0: Tool Selection

**Cycle 1:** Free choice. Default: Wonder (generate questions). If the user has already provided extensive context, skip to Draft (assemble known information).

**Arc-aware emphasis:** Check the projected arc from the router's sizing estimate. Early cycles (1-3 of projected): broad Wonder, exploratory Discovery. Mid cycles (4-6): targeted Wonder, confirmatory Discovery, heavier Draft. Late cycles (7+): minimal Wonder, Draft and Red Team focus, converge toward presentation.

**Subsequent cycles:** Dictated by the previous cycle's Draft Review failures:

| Failure pattern | Required tool |
|---|---|
| C1 only (no citations) | Discovery — minimum 2 commands |
| C2 only (no substantive delta) | Any tool — but must produce materially new content |
| C3 only (no external input) | Discovery — minimum 1 command |
| C1 + C2 | Discovery (citations), then Gap Analysis (find delta) |
| C1 + C3 | Discovery — minimum 3 commands |
| C2 + C3 | Discovery (external input), then Wonder (generate new angle) |
| All 3 | Discovery — minimum 3 commands, then Gap Analysis |

No discretion. The failure pattern selects the tool. Execute the required tool, then proceed to Draft.

---

### Phase 1: Expansion Tool

Selected by Phase 0. Use as directed:

| Tool | What to do | Output |
|---|---|---|
| Wonder | 10-15 questions: 1/type (Factual, Assumption, Gap, Inversion, Scope, Dependency, Consequence, History, Cost, Success), then 3 from opposite premise | numbered list |
| Discovery | Min 2 commands, record each. Don't default to local files — use browser/curl for live systems, read_file/search_files for code, browser for docs. ⚡ Measure, don't estimate: `wc -l`, `du`, `grep -c`, `find` | evidence |
| Gap & Wander | Adversarial + wandering: adjacent ideas, what-ifs, opposite directions | gaps & tangents |

---

### Phase 2: Draft

Compressed working draft stored in the scratch pad.

**Structure:**
```
Know: [claims with specific citations]
Confident: [strong beliefs without full proof]
Uncertain: [specific unknown items]
Next steps: [actions for next cycle]
Risks: [what could derail the analysis]
```

**Rules:**
- NOT final output. Bullets or tables only.
- Must be materially different from the previous cycle's Draft (if one exists). Same text verbatim = skip straight to Loop (stagnation detected).
- Maximum ~50 lines. If the Draft exceeds this, compact: move secondary items to a note, keep the top 3-5 per section.

After writing the Draft, update the scratch pad with the full text.

---

### Phase 3: Draft Review

Check 3 binary criteria. The LLM must be honest — padding wastes cycles.

**C1 — Claims are grounded**
Every statement in "Know" or "Confident" has a specific reference: file:line, command output text, API response, session ID, search result. A claim like "the endpoint works" without "curl returned 200 on 2026-07-15" fails.
Pass: zero uncited claims. Fail: one or more.

**C2 — Substantive delta from previous cycle**
Compared to the previous cycle's Draft, at least one item is substantively new — a new claim added to Know/Confident/Uncertain, an item moved between categories (Uncertain → Confident), or a thread progressed. Adding a citation to an existing claim does not count. Adding a new claim does.
Pass: substantive change exists. Fail: same substance, different words.
First cycle: auto-passes (nothing to compare to).

**C3 — External input is traceable in the Draft**
At least one non-trivial external input was consumed this cycle, AND its result is referenced in the Draft's Know or Confident section (as a claim, a citation, or a direct reference). "I thought about it" does not count. Running `pwd`, `ls` without a filter, `date`, `echo`, `whoami`, or `cat` on unrelated files does not count — these produce no information that advances the analysis.
First cycle: passes if Wonder was run (questions shape the Draft even though they are internally generated).

**Result and Routing:**

**≥2 pass** → Red Team. **≤1 pass** → Failure routing below (record in scratch pad):

| Failure | Restart from | Required before re-Draft |
|---|---|---|
| C1 only | Discovery → Draft | Find evidence for uncited claims. If no evidence exists, remove the claim or move it to Uncertain. |
| C2 only | Wonder → Draft | Generate at least 5 new questions from a different angle. If no new angle emerges, this is directional convergence — skip to Phase 4 with current Draft. |
| C3 only | Discovery (min 3 commands) → Draft | Each command must be cited in the Draft. If all 3 fail, list what was tried. |
| C1 + C2 | Discovery → Wonder → Draft | Find evidence AND a new angle. |
| C1 + C3 | Discovery (min 3 commands) → Draft | Commands must produce evidence. Remove uncited claims that find no support. |
| C2 + C3 | Discovery (min 3) → Wonder → Draft | Generate new input then find a new angle from it. |
| All 3 | Full restart: Wonder → Discovery → Draft | Build properly this cycle. |

Each restart targets the phase that failed. Do not rerun phases that already passed.

**Failure Budgets:** 3 strikes per criterion, reset on pass.
```
C1_strikes: 0/3   C2_strikes: 0/3   C3_strikes: 0/3
```
Strike 3 → criterion-specific exit (below).

**Criterion-Specific Exit (Budget Exhausted):**

When a criterion exhausts its budget, loops stop immediately regardless of cycle count. Produce:

- **C1 exhausted:** Current Draft with all Know/Confident claims tagged `[UNCITED]`.
- **C2 exhausted:** Current Draft with note: "Analysis could not be advanced further — available directions appear exhausted."
- **C3 exhausted:** Current Draft with all claims tagged `[UNVERIFIED — no external source]`.
- **Multiple exhausted:** Lowest-index tag takes precedence (C1 over C2 over C3). All three simultaneously → full Known Unknowns document.

After producing the exit document, proceed to Final Presentation.

---

### Phase 4: Red Team (Active)

Read actual files. Attempt to break the Draft.

**Requirements:**
- ≥2 specific citations (exact file:line, exact quote, exact reference to a command output)
- ≥1 attempt to disprove a Know claim or break a proposed direction
  - Must include the check. "I tried curl — got 200" passes. Bare "I couldn't disprove it" fails.

**Look for:**
- Unchecked assumptions
- Missing edge cases
- Cost/time underestimates
- Broken fallback chains
- Criterion that Draft Review missed (cross-check)

**Result:**
- ≥2 citations + ≥1 valid disprove attempt → Red Team passed. Proceed to Phase 5.
- Less than both → Red Team failed. Run it again with more rigor. This cycle does not count toward the iteration limit.
- No citations found, all claims held up under testing → **Convergence detected.** The Draft is solid. Exit early.

Record Red Team findings in the scratch pad for the next cycle's Wonder fuel.

---

### Phase 5: Loop or Exit

- **Convergence** (Red Team returned nothing) → Final Presentation.
- **Criterion budget exhausted** (any single Draft Review criterion hit 3 strikes) → criterion-specific exit + Final Presentation.
- **Nth cycle reached** (default 2) → Final Presentation.
- **Fewer than N cycles** → return to Phase 0 (Tool Selection). Record the Draft Review failures as the agenda for the next cycle.

---

### Escape Hatch (Criterion Budget Exhausted)

Triggered when any single Draft Review criterion accumulates 3 failures. This replaces the old "2 consecutive failures" blanket rule — per-criterion budgets are more precise.

Each exhausted criterion produces a specific output — see Phase 3 (Draft Review) for the full table.

The escape is irreversible for this session. Proceed to Final Presentation with the exit document.

---

### Convergence

Triggered when Red Team finds no actionable issues after a good-faith attempt.

Proceed directly to Final Presentation. Do not run remaining cycles — the Draft has been tested and held. Further cycles would produce diminishing returns.

---

### Final Presentation

Format the last Draft into a user-friendly summary. The internal Draft structure (Know | Confident | Uncertain | Next steps | Risks) should be invisible — present findings in narrative form:

1. **Bottom line** — 1-2 sentences answering the original question.
2. **Key findings** — the most important claims, grouped by theme, with supporting citations.
3. **Open questions** — remaining uncertainties if relevant.
4. **Recommendations** — actionable next steps.
5. **Caveats** — risks and limitations to be aware of.

Do not reference cycles, Draft Reviews, or other internal mechanics. If agents were used, do not reference them in the main body — a brief note at the end is sufficient.

If this session involved multi-agent escalation, optionally `hindsight_retain` assessment tags: "thinking: decomposition [correct/overlapped/fragmented]" and "thinking: tier selection [correct/wrong — should have been Tier X for sub-problem Y]".

Save to `.hermes/plans/` with timestamp.

Delete the scratch pad file (`.hermes/scratch/...`).

---

## Edge Cases

- **Single cycle:** User explicitly says "1 cycle" or similar. Skip Draft Review and Red Team gates. Run: Tool → Draft → Presentation.
- **Empty phase:** If an expansion tool produces nothing useful, note "empty" in the scratch pad and proceed to Draft anyway. Draft Review will likely flag it.
- **Scratch pad corruption:** If the scratch pad file cannot be read or is empty, start fresh with a new file and run Phase 0 normally.
- **Thread overflow:** If more than 5 threads emerge, defer the lowest-priority thread. Priority = blocked-by count (a thread blocking others is higher priority).

---

## Output types

- Analysis / review → findings, evidence, recommendations
- Implementation plan → tasks, paths, verification (use `plan` skill for formatting)
- Known unknowns → what's known, what's blocked, resolution path
- Decision record → options, tradeoffs, rationale

---

## Agent Escalation

See `skill_view('think', 'references/agent-escalation.md')` for the full dispatch logic — spawn decision, tier selection, contract format, failure handling, wave management, and synthesis.

## Reference files

- `references/design-rationale.md` — why think-beta exists, how the gates evolved, C3 vulnerability and fix, scratch pad pattern
- `references/gated-agent-patterns.md` — using C1/C2/C3 gates in autonomous cron agents that monitor external systems, with state file, three-tier escalation, and silent-unless-changed delivery