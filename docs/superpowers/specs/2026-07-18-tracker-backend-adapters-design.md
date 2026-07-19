# Design: Pluggable Issue-Tracker Backends for Session Skills

**Date:** 2026-07-18 (revised 2026-07-19 — dispatcher model)
**Status:** Implemented

## Problem

`start-session` / `end-session` hardcoded `gh issue ...` calls. This breaks any
repo backed by a different tracker — notably a private work Jira CLI that can't
be shared publicly. The session skills need to be tracker-agnostic.

## Scope

Two skills: `start-session` (list open issues for drift check) and `end-session`
(reconcile the resolved issue). Tracker choice is one-per-repo, fixed at install
time. Six standard verbs: `list, view, close, comment, create, label`.

## Design (dispatcher, not markdown adapter)

An earlier draft made each backend a markdown *skill* the agent reads, with a
`verbs:` frontmatter declaration. That was rejected in favor of a deterministic
CLI dispatcher, because:

1. **Injection safety.** The `close`/`comment`/`create` verbs carry free text.
   An agent hand-composing `gh issue comment N --body "<text>"` in a shell is a
   quoting/injection hazard. A dispatcher using `subprocess` argv lists is safe
   by construction.
2. **Deterministic skips.** An unsupported verb is a no-op decided in code, not
   a prose instruction the agent might improvise around.
3. **No new coupling.** Session skills already require `wfctl`, so a `wfctl`
   subcommand adds no dependency the session surface didn't already have.

The design also collapses the earlier "capability declaration vs implementation"
validation problem: because a backend is a `verb → command` map, **the supported
verbs are the map's keys** — there is no separate declaration to drift, so a
backend cannot misdeclare its capabilities.

### Backend config

`.agents/trackers/<name>.json` — a map of verb to argv template. `{name}`
placeholders are substituted per-token (within-token, so `"--{action}-label"`
→ `--add-label`). `github.json` ships in wf-skills; custom backends are authored
locally via the `scaffold-tracker` skill and never touch the public repo.

```json
{
  "verbs": {
    "list": ["gh", "issue", "list", "--state", "open", "--limit", "20"],
    "view": ["gh", "issue", "view", "{id}"],
    "close": ["gh", "issue", "close", "{id}", "--comment", "{comment}"],
    "comment": ["gh", "issue", "comment", "{id}", "--body", "{body}"],
    "create": ["gh", "issue", "create", "--title", "{title}", "--body", "{body}"],
    "label": ["gh", "issue", "edit", "{id}", "--{action}-label", "{label}"]
  }
}
```

### Dispatcher (`wfctl issue <verb>`)

Implemented in wfctl (`wfctl/_tracker.py`, `dispatch()`), invoked as
`wfctl issue <verb> [id] [--comment/--body/--title/--label/--action]`:

1. Read `tracker` name from `.wf-skills-manifest.json`. Unset → notice, exit 0.
2. Load `.agents/trackers/<name>.json`. Missing/invalid → warn, exit 0.
3. Verb not in the map → "does not support" notice, exit 0.
4. Substitute `{...}` per argv token; missing param → `✗ requires --x`, exit 1.
5. `subprocess.run(argv, ...)` — no `shell=True`. Passthrough stdout; on failure
   print stderr and propagate the exit code. On success, log an `issue` event.

Steps 1–3 are the graceful-degrade path: a session must never fail because a
tracker step couldn't run.

### Install-time selection

`wfctl install-skills --tracker <name>`:
- `github` → copies `github.json` into the plan (same backup/manifest machinery
  as any other item) and records `manifest["tracker"] = "github"`.
- other name → records the name and warns if
  `.agents/trackers/<name>.json` doesn't exist yet.
- `none` → clears the key. Omitted → unchanged.

The `tracker` key is a top-level sibling of the per-agent manifest entries;
`uninstall-skills` (agent-scoped) leaves it in place.

### scaffold-tracker skill

Interviews for a backend name, the supported subset of the six verbs, and each
verb's argv template; writes `.agents/trackers/<name>.json`; self-checks that it
is valid JSON with non-empty argv arrays and only allowed placeholders.

## Out of scope

- Per-issue tracker selection within one repo; runtime switching.
- Any shipped backend other than `github.json`.
- Extending the verb vocabulary beyond the six, or to skills other than
  start/end-session.
