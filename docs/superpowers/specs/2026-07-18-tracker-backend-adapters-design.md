# Design: Pluggable Issue-Tracker Backends for Session Skills

**Date:** 2026-07-18
**Status:** Proposed

## Problem

`start-session.md` and `end-session.md` hardcode `gh issue ...` calls directly
in their prose (drift detection, `Closes #N` reconciliation). This works for
GitHub-backed repos but breaks for any repo using a different tracker — e.g.
a private work repo backed by a custom Jira CLI. There's currently no way to
swap the tracker without hand-editing the session skills themselves, and no
guidance for authoring a new backend.

## Scope

MVP covers exactly two skills: `start-session.md` (drift detection, listing
open issues) and `end-session.md` (`Closes #N` reconciliation). Other
wf-skills that touch issue trackers (`speckit-delivery-plan`, `agent-brief`,
etc.) are out of scope until they need it.

Tracker choice is one-per-repo, fixed at install time — not per-issue, not
runtime-switchable.

## Design

### Verb contract

Six canonical verbs, each with a short ID and a corresponding `##` header a
tracker skill implements:

| ID        | Header                                    | Used by                          |
|-----------|--------------------------------------------|-----------------------------------|
| `list`    | `## List open issues`                      | start-session drift check         |
| `view`    | `## View issue <ID>`                       | start-session, end-session verify |
| `close`   | `## Close issue <ID> with comment`         | end-session Closes #N reconciliation |
| `comment` | `## Comment on issue <ID>`                 | end-session progress notes        |
| `create`  | `## Create issue`                          | end-session untracked-work/TODO handling |
| `label`   | `## Add/remove label on issue <ID>`        | end-session in-progress labeling  |

A tracker skill does not need to implement all six. It declares the subset it
supports in frontmatter:

```markdown
---
name: tracker-jira
verbs: [list, view, comment]
---
```

`tracker-github/SKILL.md` (ships in wf-skills) declares all six, backed by
`gh issue ...`. A private `tracker-jira/SKILL.md` (never published to
wf-skills — company-specific) might declare only `[list, view, comment]`,
backed by a custom Jira CLI.

### Session skill changes

`start-session.md` and `end-session.md` no longer call `gh` directly. Instead:

1. Read the `tracker` field from `.wf-skills-manifest.json`.
2. If unset or `none`: skip all tracker-dependent steps (drift detection,
   `Closes #N` reconciliation) — same graceful-skip pattern pfms's
   `start-session.md` already uses when `gh auth status` fails.
3. If set: load `.agents/skills/tracker-<name>/SKILL.md`, check its declared
   `verbs:` list before using any verb.
   - Verb declared → follow that `##` section's instructions.
   - Verb not declared → skip that sub-step silently (not an error). E.g. if
     `tracker-jira` doesn't declare `close`, the Closes #N reconciliation step
     is skipped and the session summary just notes the issue reference
     without attempting to close it.

### Install-time flow

`wfctl install-skills` gains `--tracker <name>` — a free string, not a fixed
enum, since `scaffold-tracker` can produce any name. Only the literal value
`github` is special-cased (it's the one backend wf-skills ships). Omitted or
`none` disables tracker integration.

- `--tracker github`: `tracker-github` is copied from the wf-skills clone,
  using the exact same plan/backup/manifest mechanism already used for every
  other installed item (just another `src_rel, dst_rel` pair, conditionally
  added to the install plan).
- `--tracker <anything else>` (e.g. `jira`): wf-skills does not attempt to
  fetch or copy anything for it — the public repo only ships `github`.
  Instead, `install-skills` checks whether
  `.agents/skills/tracker-<name>/SKILL.md` already exists in the target repo
  and warns if not: `"selected tracker 'jira' but no tracker-jira/SKILL.md
  found — add yours first (see scaffold-tracker)"`.
- `--tracker none` / omitted: no tracker key is written; session skills
  behave as if unset.

The choice is persisted as a new top-level `"tracker"` key in
`.wf-skills-manifest.json`, a sibling of the existing per-agent entries
(`"claude"`, etc.) rather than nested under one:

```json
{
  "tracker": "github",
  "claude": {
    "repo": "https://github.com/aamarin/wf-skills",
    "ref": "main",
    "installed_at": "...",
    "items": [...]
  }
}
```

### Scaffolding: `scaffold-tracker` skill

A new skill (ships in wf-skills — it's generic, not tied to any backend)
that walks through authoring a new tracker adapter conversationally:

1. Ask for the tracker name (e.g. `jira`).
2. Ask which of the six verbs this backend supports.
3. For each supported verb, ask for the concrete command/instructions.
4. Write `.agents/skills/tracker-<name>/SKILL.md` from a template, with
   frontmatter (`verbs: [...]`) and `##` headers filled in for exactly the
   supported subset.
5. Run the structural self-check (below) before finishing.

This is the mechanism for "bring your own backend" — a private/work-specific
tracker adapter is generated and lives entirely in the target repo; it never
needs to touch the public wf-skills repo.

### Validation

No standalone validate command. Validation is a structural check — frontmatter
`verbs:` list vs. actual `##` headers present — reused at two points that
already exist in the design:

1. **End of `scaffold-tracker`** — self-check immediately after generating
   the file.
2. **`install-skills` tracker existence check** — already reading the file to
   confirm it exists; extended to also diff frontmatter against headers.

On mismatch, the check emits a precise diagnostic (e.g. `"frontmatter
declares 'close' but no '## Close issue <ID> with comment' header found"`)
and stops there — it does not auto-fix. The diagnostic is specific enough
that the agent can act on it immediately in conversation if the user asks,
but auto-patching a hand-authored, possibly-deliberate partial adapter
without the user seeing the mismatch first is the wrong default.

### Error handling

- Manifest references a tracker skill file that's missing at session-start
  time → warn, degrade to `none` behavior (skip tracker-dependent steps),
  don't hard-fail session start.
- Manifest with no `tracker` key (pre-existing installs) → treated as `none`.

## Out of scope

- Per-issue tracker selection within a single repo.
- Runtime tracker switching.
- Any wf-skills-shipped backend other than `tracker-github`.
- Auto-fix for validation mismatches.
- Extending the verb contract to skills beyond start/end-session.
