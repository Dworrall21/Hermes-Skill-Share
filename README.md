# Hermes Skill Share

Reusable Hermes Agent skills distilled from completed work sessions.

## Skills

| Skill | Purpose |
|---|---|
| [`strict-response-contracts`](skills/autonomous-ai-agents/strict-response-contracts/SKILL.md) | Enforce exact strings, one-word limits, fixed sentence counts, machine-readable-only output, and other strict response contracts. |
| [`chatgpt-turn-completion-gate`](skills/autonomous-ai-agents/chatgpt-turn-completion-gate/SKILL.md) | Prevent MoA aggregation from starting while ChatGPT is still thinking or streaming; wait for an explicit terminal turn state. |

## Layout

Skills use the Hermes Agent convention:

```text
skills/<category>/<skill-name>/SKILL.md
```

Each skill should contain valid YAML frontmatter, explicit activation triggers, an actionable workflow, common pitfalls, and a verification checklist.
