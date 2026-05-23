# agent-skills

A personal collection of agent skills used with Claude Code and Codex tools. Each subdirectory is a self-contained skill defined by a `SKILL.md` with frontmatter (`name`, `description`) and instructions the agent loads on demand.

## Skills

### [thought-experiment-flow](thought-experiment-flow/SKILL.md)

Payload-driven thought experiment for tracing whether an API or business flow will succeed before running it. Given a payload, the repositories and branches involved, and the endpoint chain, the skill walks execution step by step — inspecting real code paths, identifying preconditions, state changes, and failure modes — and produces an honest verdict (`SHOULD_SUCCEED`, `WILL_FAIL`, `PARTIAL_SUCCESS_RISK`, or `INCONCLUSIVE`) along with verification steps. Useful for safely rerunning APIs, predicting side effects, and reasoning about end-to-end flows without writing code.

## Using a skill

Drop the skill directory into your agent's skills location (e.g. `~/.claude/skills/<skill-name>/`) so the harness can discover it via its `SKILL.md` frontmatter. The agent will invoke the skill automatically when a user request matches its `description`, or you can reference it explicitly by name.

## Adding a skill

1. Create a new directory at the repo root.
2. Add a `SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: my-skill
   description: One-sentence trigger description that tells the agent when to use this skill.
   ---
   ```
3. Write the instructions below the frontmatter. Keep them concrete, action-oriented, and focused on when to use the skill, what inputs to collect, and what output to produce.
4. Add an entry to this README.
