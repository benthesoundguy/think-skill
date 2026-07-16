# Hermes Skill Dispatch via skill_view

## The pattern

A router skill dispatches to sub-skills at runtime using `skill_view()`:

1. Router loads (trigger matched).
2. Router classifies the problem.
3. Router calls `skill_view('think-beta')` (or `skill_view('think-alpha')`).
4. The loaded sub-skill's SKILL.md content is returned.
5. The system prompt says: *"If a skill matches or is even partially relevant to your task, you MUST load it with skill_view(name) and follow its instructions."*
6. The agent follows the loaded sub-skill's workflow.

No content duplication. The router is ~30 lines. All workflow logic lives in the sub-skills.

## When to use

- **Default branch vs explicit override** pattern (e.g., complex → beta, simple → alpha).
- **Progression model** — start strict, graduate to trust-based.
- **Future extension** — new sub-skills (gamma, delta) plug in via the same pattern without changing the router.

## Requirements

- The router and all sub-skills must have overlapping trigger phrases so the system prompt's "matches" clause activates.
- The sub-skill must be a real skill (created via `skill_manage(action='create')`) — not a loose file.
- The router must NOT contain the sub-skill's content inline. It dispatches; it doesn't duplicate.

## See also

- `think` (router) — implements this pattern for think-alpha and think-beta
