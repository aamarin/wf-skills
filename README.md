# wf-skills

Reusable agent skills for Claude Code, Bob, and any markdown-capable AI agent.

Skills are plain markdown files (`SKILL.md`) — no framework required. Any agent
that reads markdown can use them. Agent-specific command wrappers (e.g. Claude
slash commands) are in `commands/`.

## Skills

| Skill | Description |
|-------|-------------|
| `agent-brief` | Per-task scope brief protocol for headless agents — defines scope, escalation, and done criteria |
| `speckit-orchestrate` | Read pipeline state and auto-advance or surface the next speckit command |
| `speckit-analyze` | Analyze spec artifacts for consistency and quality |
| `speckit-checklist` | Generate a pre-implementation checklist from the active spec |
| `speckit-clarify` | Clarify ambiguous requirements before specifying |
| `speckit-constitution` | Define project-level constraints and principles for agent behavior |
| `speckit-delivery-plan` | Generate a delivery plan from spec and tasks |
| `speckit-implement` | Execute the implementation plan from tasks.md |
| `speckit-plan` | Generate implementation design artifacts from a spec |
| `speckit-tasks` | Generate a dependency-ordered tasks.md from design artifacts |

## Installation

### Via wfctl (recommended)

```bash
wfctl install-skills
```

Copies `skills/` → `.agents/skills/` and `commands/` → `.claude/commands/`.

### Manual

```bash
# Clone and copy skills into your project
git clone https://github.com/MarinVentures/wf-skills.git
cp -r wf-skills/skills/* .agents/skills/
cp wf-skills/commands/* .claude/commands/
```

### Any agent

Paste the contents of any `SKILL.md` directly into your agent's system prompt,
rules file (`CLAUDE.md`, `.cursorrules`, etc.), or conversation.

## Structure

```
skills/          ← agent-agnostic SKILL.md files
  speckit-orchestrate/SKILL.md
  agent-brief/SKILL.md
  ...
commands/        ← Claude Code slash command wrappers
  start-session.md
  speckit.implement.md
  ...
```

## Requirements

Skills that use `speckit-orchestrate` or session commands require
[wfctl](https://github.com/MarinVentures/wfctl) to be installed:

```bash
uv tool install git+https://github.com/MarinVentures/wfctl.git@v0.2.0
```

## License

MIT
