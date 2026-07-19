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

4. Check open work via the configured issue tracker:
   ```bash
   wfctl issue list
   ```
   This runs the active backend's list command (GitHub, Jira, or a custom one).
   If no tracker is configured it prints a notice and no-ops — skip this step.

5. Report status to the user:
   - Current pipeline step (from `current.md`)
   - Last session's focus and its **Next Session TODO** (from `session-summary.md`)
   - Commits on this branch + any uncommitted changes
   - Open issues

6. Ask: "What are we working on today?" — defaulting to the top item from the last
   session's Next Session TODO if there was one.
