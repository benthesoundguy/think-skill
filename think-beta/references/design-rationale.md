# Think-Beta Design Rationale

## Why think-beta exists alongside think

`think` is a free-form structured-thinking skill. Phases are a toolkit — the LLM chooses which to use and in what order. This works when the LLM is honest about using discovery tools before drafting conclusions.

`think-beta` is the same domain with strict guardrails. The guardrails assume a lazy LLM that will shortcut to Draft → Red Team without expanding knowledge.

## The real mechanism: nudge, not enforcement

The Draft Review is self-policed — the LLM checks its own work. A dishonest LLM can claim PASS on all criteria and proceed. The guardrail is not enforcement; it's the **anticipation effect**. Knowing the Review is coming, the LLM preemptively cites claims and runs commands before drafting. This shapes better habits over time.

The Review's output is visible to the user, who can spot a dishonest pass. The transparency is the actual check.

## Progression model (how to use both skills)

Think-beta's value is highest in early sessions when habits are forming. Once the LLM naturally cites claims and uses external input without being told, the Draft Review becomes ritual overhead.

- **Start with think-beta** (10–20 cycles) to form the habits
- **Graduate to think** once Draft Review passes are routine
- **Drop back to think-beta** if quality backslides without the guardrails

## Key design decisions

### Draft Review gate (3 binary criteria)
| Criterion | What it prevents |
|---|---|
| C1 — Claims grounded | Claims with no evidence |
| C2 — Substantive delta | Re-running the same Draft each cycle |
| C3 — Input traceable | Running commands but ignoring output |

### Per-criterion routing (not blanket retry)
When ≤1 passes, the restart targets the specific phase that failed. C1 fail → Discovery. C2 fail → Wonder. Combinations chain. This avoids rerunning phases that already passed.

### 3-strike failure budgets (not "2 consecutive failures")
The old blanket rule escaped after 2 consecutive all-fail. Per-criterion budgets are more precise: each criterion gets 3 targeted attempts before exit. Different failures produce different tagged outputs so the user sees what went wrong.

### C3 traceability (v1.0.0 vulnerability, patched)
Original C3: "at least one non-trivial external input was generated." A lazy LLM ran `pwd` and passed. Fix: the input must be cited in the Draft's Know or Confident section. Trivial commands (pwd, date, echo, whoami, ls without filter) are explicitly excluded. First-cycle exception: Wonder-generated questions shape the Draft even though they're internal.

### Determinism table (85.7% hardcoded)
6 of 7 failure patterns route to Discovery with minimum command counts. Only C2-only says "Any tool" but is self-limiting — reaching C2-only means C1 and C3 already passed, which required citations and external input.

### Scratch pad (`.hermes/scratch/`)
Working memory created cycle 1, updated every phase, deleted at Final Presentation. Survives context compression between cycles. Contains: threads, current Draft, Draft Review results, failure budget tracking.

### Through threads
Max 5 active. Each tracks: description, status, blocks, blocked-by, last finding. "Blocked by" column feeds Tool Selection deterministically.

### Convergence detection
Red Team empty after good-faith attempt → exit early. Red Team must include ≥2 citations and ≥1 disprove attempt with evidence. Empty passes must be earned, not claimed.

### Criterion-specific exits
When a criterion exhausts its 3-strike budget, the exit is specific to that failure:
- C1 exhausted: Draft tagged `[UNCITED]` — user sees which claims lack support
- C2 exhausted: Draft with "directions exhausted" note — convergence by limits
- C3 exhausted: Draft tagged `[UNVERIFIED]` — user sees confidence problem

## What think-beta does NOT prevent
- Lying on the Draft Review (self-policed — transparency is the check)
- Running a meaningful command but ignoring output (C3 catches this via traceability)
- Running many trivial commands (C3 bans trivial commands explicitly)
- Serial over-confidence (C1's 3-strike budget catches it)
