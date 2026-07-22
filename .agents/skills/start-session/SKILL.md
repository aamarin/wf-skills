---
name: start-session
description: Use when starting a development session in a git worktree - initializes wfctl session state, loads handoff artifacts from the last session, and reports open work before any code is touched.
compatibility: 'Requires wfctl to be installed'
---

# Start Session

You are starting a development session in the current git worktree. If the last
session ended with `/end-session` and `/clear`, the artifacts below are your only
memory of it — load them before doing anything else.

## Workflow

1. Run `wfctl start` to initialize agent session context and infer the current
   pipeline step.

2. **Load the handoff artifacts** from the state dir (`$(wfctl state-dir)`):
   - `current.md` — the resume point (issue, status, step, next action).
   - `session-summary.md` — the last session's handoff (accomplishments,
     decisions, and **Next Session TODO**). This is the primary context after a
     `/clear`; read it fully. If absent, this is the first session on the branch.

3. **Surface work done on this branch** so you can see where things stand:
   ```bash
   BASE=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null || true)
   git log --format="%h %s" ${BASE:+${BASE}..HEAD} ${BASE:--20}
   git status --short
   ```

4. Check open work via the configured backends:
   ```bash
   wfctl issue list     # open issues (scoped to you if the tracker sets {me})
   wfctl change list     # open PRs / patchsets (your changes under review)
   ```
   Each runs the active backend's list command (GitHub, Jira, Gerrit, or a custom
   one). If a backend isn't configured — or doesn't implement the verb — it prints
   a notice and no-ops, so skip whatever comes back empty.

5. **Check alignment** — does the branch's work match what's tracked? Correlate
   the commits (step 3) with the open issues/changes (step 4). It's a heads-up
   read, not an audit or a gate:
   - **Aligned** — a commit references an issue/change (`#N`, `Closes #N`, or a
     tracker key like `PROJ-123`). Nothing to flag.
   - **Likely done** — an open issue whose work the commits appear to complete.
     Flag it to close (`wfctl issue view <id>` to confirm), don't close it yourself.
   - **Untracked** — committed work (especially new feature files) that matches no
     open issue. Surface it so the user decides: open an issue, fold it into an
     existing one, or leave it (infra / one-off).

   Only surface the non-aligned items. If everything lines up, say so in one line
   and move on.

6. Report status to the user:
   - Current pipeline step (from `current.md`)
   - Last session's focus and its **Next Session TODO** (from `session-summary.md`)
   - Commits on this branch + any uncommitted changes
   - Open issues and open changes (PRs / patchsets)
   - **Alignment**: aligned, or the likely-done / untracked items from step 5

7. Ask: "What are we working on today?" — defaulting to the top item from the last
   session's Next Session TODO if there was one.
