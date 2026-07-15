# Start Session

You are starting a development session in the current git worktree.

## Workflow

1. Run `wfctl start` to initialize agent session context and infer the current pipeline step.

2. Read `$(wfctl state-dir)/current.md` to resume from last session.

3. Check open work:
   ```bash
   gh issue list --state open --limit 20
   ```

4. Report status to user:
   - Current pipeline step (from `current.md`)
   - Last session focus (if any)
   - Open issues

5. Ask: "What are we working on today?"
