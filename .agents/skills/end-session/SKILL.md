---
name: end-session
description: Use when finishing a development session - summarizes what was accomplished, closes wfctl session state, and surfaces uncommitted work before the context is lost.
compatibility: 'Requires wfctl to be installed'
---

# End Session

You are ending the current development session.

## Workflow

1. Summarize what was accomplished this session (files changed, decisions made,
   next steps).

2. Run `wfctl end` to close the session and write a summary scaffold:
   ```bash
   wfctl end
   ```

3. Report any uncommitted work so the user can decide whether to commit. Do not
   commit automatically.

4. Report: session closed, next steps, any blockers.
