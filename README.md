# wf-skills

Reusable agent skills for Claude Code, Bob, and any markdown-capable AI agent.

Skills are plain markdown files (`SKILL.md`) — no framework required. Any agent
that reads markdown can use them. Command wrappers (thin shims that point at
the skill holding the actual workflow) live in `.claude/commands/`. That's the
one authored copy — Bob has its own command directory (`.bob/commands/`), but
`wfctl install-skills --agent bob` copies the same `.claude/commands/` content
there rather than maintaining a second hand-translated set that could drift out
of sync. (Bob's docs describe a narrower frontmatter schema for its commands —
`description` + `argument-hint`, no `handoffs` — but in practice it appears to
tolerate the extra Claude-specific fields fine.)

## Skills

### Session

| Skill | Description |
|-------|-------------|
| `start-session` | Initialize wfctl session state, infer the pipeline step, report open work |
| `end-session` | Summarize the session, close wfctl state, surface uncommitted work |

### Speckit pipeline

| Skill | Description |
|-------|-------------|
| `speckit-specify` | Create the feature spec from a description; uses GH issue numbers as NNN |
| `speckit-clarify` | Ask up to 5 targeted questions and encode answers back into the spec |
| `speckit-plan` | Generate implementation design artifacts from a spec |
| `speckit-tasks` | Generate a dependency-ordered tasks.md from design artifacts |
| `speckit-analyze` | Cross-artifact consistency and quality gate over spec/plan/tasks |
| `speckit-delivery-plan` | Decide PR boundaries, group tasks into issues, map parallelization waves |
| `speckit-implement` | Execute the implementation plan from tasks.md |
| `speckit-checklist` | Generate a custom checklist for the current feature |
| `speckit-constitution` | Create or update the project constitution |
| `speckit-orchestrate` | Read pipeline state and auto-advance or surface the next command |

### General

| Skill | Description |
|-------|-------------|
| `agent-brief` | Per-task scope brief protocol for headless agents — scope, escalation, done criteria |
| `brainstorming` | Explore intent, requirements, and design before any implementation |
| `idea-refine` | Sharpen a vague idea into an actionable concept; stress-test assumptions |
| `receiving-code-review` | Verify review feedback technically instead of agreeing reflexively |
| `verification-before-completion` | Run the checks and confirm output before claiming anything works |
| `finishing-a-development-branch` | Decide how to integrate completed work — merge, PR, or cleanup |
| `using-superpowers` | Establishes how to find and invoke skills |

## Installation

### Via wfctl (recommended)

```bash
wfctl install-skills
```

`--agent` selects where things land. Skills are agent-agnostic; only the
destination changes:

| `--agent` | Installs |
|-----------|----------|
| `claude` (default) | skills → `.agents/skills/`, command wrappers → `.claude/commands/` |
| `bob` | skills → `.bob/skills/`, command wrappers → `.bob/commands/` (same source content as Claude's) |
| `none` | skills → `.agents/skills/` only |

Also handles re-runs and removal: a pre-existing file that install-skills
overwrites is backed up first, and `wfctl uninstall-skills --agent <agent>`
removes what it installed and restores anything it backed up.

Files of the same name are overwritten — rerun to update, but local edits to
installed skills (not backed up by wfctl) are lost.

### Manual

```bash
git clone https://github.com/aamarin/wf-skills.git

# Claude Code
cp -r wf-skills/.agents/skills/* .agents/skills/
cp wf-skills/.claude/commands/* .claude/commands/

# Bob
cp -r wf-skills/.agents/skills/* .bob/skills/
cp wf-skills/.claude/commands/* .bob/commands/
```

### Any agent

Paste the contents of any `SKILL.md` directly into your agent's system prompt,
rules file (`CLAUDE.md`, `.cursorrules`, etc.), or conversation.

## Structure

```
.agents/
  skills/          ← agent-agnostic SKILL.md files (mirrors install target)
    speckit-specify/SKILL.md
    start-session/SKILL.md
    ...
.claude/
  commands/        ← Claude Code slash command shims → .agents/skills/
    speckit.specify.md
    start-session.md
    ...
```

The repo mirrors the project directory structure it installs into.

## Requirements

Skills that use `speckit-orchestrate` or the session commands require
[wfctl](https://github.com/aamarin/wfctl):

```bash
uv tool install git+https://github.com/aamarin/wfctl.git
```

## License

MIT
