---
name: 'speckit-orchestrate'
description: 'Read pipeline state after a speckit step completes, then auto-advance or surface the next command based on the step auto flag.'
---

## Steps

0. **Epic-inherited spec check** — run before trusting wfctl's own inference.
   `wfctl`'s `resolve_spec_dir` only matches `specs/<current-branch>/` exactly, so
   a sub-issue worktree branched from an epic (per that epic's `delivery.md` →
   `speckit-delivery-plan`'s "Epic Planning Branch as Worktree Base" convention)
   has no spec dir under its own branch name — `wfctl status`/`resume` will
   always report "no spec dir found" and default to `brainstorm`, even when the
   epic's spec/plan/tasks/decompose are already committed and this sub-issue is
   mid- or post-implementation.

   Check: does `specs/<current-branch>/` exist? If not:
   - Resolve the active tracker's key format: read `key_pattern` from whichever
     `.agents/trackers/*.json` exists (default `\d+` — GitHub's bare-numeric
     default — if no tracker config or no `key_pattern` field). Build a match
     regex `#?{key_pattern}` — optional leading `#`, since GitHub issues are
     conventionally written `#123` in prose while other trackers' keys (e.g.
     `PROJ-123`) never take one.
   - Glob `specs/*/delivery.md`; in each one's "Issue Grouping Map" table, search
     every row for that regex. A row whose match equals the current issue's key
     means that `delivery.md`'s directory is the real spec dir, and the row's
     `Tasks` column is this sub-issue's task range. (Older delivery.md files may
     predate the standardized `{issue-key}`-leads-the-cell format — the regex
     search-anywhere-in-row approach still finds them.)
   - If found, ignore wfctl's brainstorm default:
     - For a GitHub-backed repo, `gh pr list --head {current-branch}` is a cheap
       non-blocking nicety — an open/merged PR means the sub-issue is past
       `implement`; report that instead ("PR #N open, awaiting review" / "PR #N
       merged — story complete"). Skip this check for other trackers; the six
       standard verbs don't include a by-branch change lookup.
     - No PR found (or non-GitHub tracker) → sub-issue is at `implement`
       (spec/plan/tasks/decompose are already done at the epic level); next
       command is `/speckit.implement` scoped to the resolved task range, not
       `/speckit.brainstorm`.

1. Run `wfctl status` and display the output so the user can see the updated pipeline position.

2. Run `wfctl resume` to re-infer the pipeline step and write `$(wfctl state-dir)/next-step.md`.

3. Read `$(wfctl state-dir)/next-step.md` and extract:
   - `command`: value after `Next step: ` (e.g. `/speckit.plan`)
   - `auto`: `true` or `false` from the `auto:` line

4. Branch on the result:

   **Story complete** (file contains "Story complete"):
   - Display: "Story complete — open PR or run `/end-session`."
   - Stop.

   **`auto: true`**:
   - Strip the leading `/` from command (e.g. `/speckit.plan` → `speckit.plan`)
   - Output on its own line: `EXECUTE_COMMAND: {command-without-slash}`

   **`auto: false`**:
   - Display: "Next: run `{command}` when ready."
   - Stop.
