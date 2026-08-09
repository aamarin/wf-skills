# AGENTS.md

Project overrides for agents working in this repository. Read by
`/speckit.brainstorm` at the start of a session. Optional by design — a repo
without this file proceeds silently.

## `.agents/` is the source; `.claude/` is generated

Edit skills and commands under `.agents/`. `wfctl install-skills` copies them
into `.claude/` (and any other agent's directory), so edits made in `.claude/`
are overwritten on the next install and never reach consumers.

This repo installs its own skills into itself, so `.gitignore` lists many
`.agents/…` and `.claude/…` paths. That does not untrack the sources — files
already tracked stay tracked. Check `git ls-files` before assuming a path is
ignored.

## Pipeline artifacts are gitignored

Everything under `specs/` — `design.md`, `spec.md`, `plan.md`, `tasks.md`,
`brief.md`, `escalation.md` — is local. The implementation is what ships.
`wfctl archive-story` snapshots the tree into the XDG state dir at worktree
teardown, so nothing is lost by not committing it.

Never write an instruction that requires committing a path under `specs/`.

## Vendored skills keep their upstream names

`brainstorming` and `idea-refine` are ported into `.agents/skills/` so consumer
repos get them without depending on the superpowers or agent-skills plugins.
Commands invoke them **unnamespaced** — `brainstorming`, not
`superpowers:brainstorming`. Adding a namespace reintroduces the dependency this
repo exists to avoid.

## One writer per artifact

Each pipeline artifact has exactly one writing instruction across all skills and
commands. Before adding a write, search for an existing writer of that path — a
second one silently destroys the first's output.
