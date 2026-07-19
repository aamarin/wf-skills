# wf-skills

Reusable agent skills for Claude Code, Bob, and any markdown-capable AI agent.

Skills are plain markdown files (`SKILL.md`) — no framework required. Any agent
that reads markdown can use them. Command wrappers (thin shims that point at
the skill holding the actual workflow, so a slash-command UI has something to
list) are agent-agnostic too and live in `.agents/commands/` — the one
authored copy. Each agent's install destination is different (Claude reads
`.claude/commands/`, Bob reads `.bob/commands/`), but the content is identical,
so `wfctl install-skills` copies the same source to both rather than
maintaining a separate hand-translated set per agent that could drift out of
sync. (Bob's docs describe a narrower frontmatter schema for its commands —
`description` + `argument-hint`, no `handoffs` — but in practice it appears to
tolerate the extra Claude-specific fields fine.)

## Skills

### Session

| Skill | Description |
|-------|-------------|
| `start-session` | Initialize wfctl session state, infer the pipeline step, report open work |
| `end-session` | Summarize the session, close wfctl state, surface uncommitted work |
| `using-wfctl` | Reference for wfctl's full command surface — use when you need a command the other skills don't already call for you |

### Speckit pipeline

| Skill | Description |
|-------|-------------|
| `speckit-specify` | Create the feature spec from a description on a branch already named for an existing issue |
| `speckit-clarify` | Ask up to 5 targeted questions and encode answers back into the spec |
| `speckit-plan` | Generate implementation design artifacts from a spec |
| `speckit-tasks` | Generate a dependency-ordered tasks.md from design artifacts |
| `speckit-analyze` | Cross-artifact consistency and quality gate over spec/plan/tasks |
| `speckit-delivery-plan` | Decide PR boundaries, group tasks into issues, map parallelization waves |
| `speckit-implement` | Execute the implementation plan from tasks.md |
| `speckit-checklist` | Generate a custom checklist for the current feature |
| `speckit-constitution` | Create or update the project constitution |
| `speckit-orchestrate` | Read pipeline state and auto-advance or surface the next command |
| `scaffold-tracker` | Author a `.agents/trackers/<name>.json` backend so `wfctl issue` works with a non-GitHub tracker (e.g. a private Jira CLI) |

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

`--agent` selects where things land. Skills are agent-agnostic; only the
destination changes:

| `--agent` | Installs |
|-----------|----------|
| `claude` (default) | skills → `.agents/skills/`, command wrappers → `.claude/commands/` |
| `bob` | skills → `.bob/skills/`, command wrappers → `.bob/commands/` (same source as Claude's) |
| `none` | skills → `.agents/skills/` only |

`--tracker github` also copies `.agents/trackers/github.json` and records the
choice in `.wf-skills-manifest.json`. For a custom backend, author
`.agents/trackers/<name>.json` (see `scaffold-tracker`) then run
`wfctl install-skills --tracker <name>`; `--tracker none` clears the choice.

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
cp wf-skills/.agents/commands/* .claude/commands/

# Bob
cp -r wf-skills/.agents/skills/* .bob/skills/
cp wf-skills/.agents/commands/* .bob/commands/
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
  commands/        ← agent-agnostic command shims → .agents/skills/, copied to
    speckit.specify.md   each agent's own command directory at install time
    start-session.md
    ...
  trackers/        ← issue-tracker backends (verb→command maps)
    github.json
    ...
```

`.agents/skills/` mirrors its install target directly. `.agents/commands/`
doesn't — it's the one authored source that `install-skills` copies into
`.claude/commands/` (Claude) or `.bob/commands/` (Bob) depending on `--agent`.

## Requirements

Skills that use `speckit-orchestrate` or the session commands require
[wfctl](https://github.com/aamarin/wfctl):

```bash
uv tool install git+https://github.com/aamarin/wfctl.git
```

## License

MIT
