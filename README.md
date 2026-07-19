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
| `scaffold-tracker` | Author a `.agents/trackers/<name>.json` backend so `wfctl issue` works with a non-GitHub tracker (e.g. a private Jira CLI) |

## Issue trackers

The session commands (`start-session`, `end-session`) touch an issue tracker
through a fixed vocabulary of six verbs — `list, view, close, comment, create,
label` — dispatched by `wfctl issue <verb>`. Each backend is a
`.agents/trackers/<name>.json` map of verb → argv template; the supported-verb
set is just the map's keys. `github.json` ships here. For any other tracker,
author one with the `scaffold-tracker` skill and select it at install time.

`wfctl issue` builds real argv lists (never a shell string), so a comment body
containing quotes or `$(...)` is one inert argument. An unsupported verb or an
unconfigured tracker no-ops instead of failing the session.

```jsonc
// .agents/trackers/github.json
{ "verbs": {
  "close": ["gh", "issue", "close", "{id}", "--comment", "{comment}"],
  "label": ["gh", "issue", "edit", "{id}", "--{action}-label", "{label}"]
} }
```

## Installation

### Via wfctl (recommended)

```bash
wfctl install-skills                    # skills + commands
wfctl install-skills --tracker github   # also install the GitHub tracker backend
```

Copies `.agents/skills/` → `.agents/skills/` and `.agents/commands/` →
`.claude/commands/`. `--tracker github` also copies `.agents/trackers/github.json`
and records the choice in `.wf-skills-manifest.json`. For a custom backend,
author `.agents/trackers/<name>.json` (see `scaffold-tracker`) then run
`wfctl install-skills --tracker <name>`; `--tracker none` clears the choice.

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
.agents/
  skills/          ← agent-agnostic SKILL.md files (mirrors install target)
    speckit-orchestrate/SKILL.md
    agent-brief/SKILL.md
    ...
  trackers/        ← issue-tracker backends (verb→command maps)
    github.json
    ...
.claude/
  commands/        ← Claude Code slash command wrappers
    start-session.md
    speckit.implement.md
    ...
```

The repo mirrors the project directory structure it installs into.
For Bob or other agents, copy `.agents/skills/` into `.bob/skills/` or equivalent.

## Requirements

Skills that use `speckit-orchestrate` or session commands require
[wfctl](https://github.com/MarinVentures/wfctl) to be installed:

```bash
uv tool install git+https://github.com/MarinVentures/wfctl.git@v0.2.0
```

## License

MIT
