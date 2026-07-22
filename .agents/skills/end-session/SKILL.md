---
name: end-session
description: Use when finishing a development session - writes a session-summary handoff artifact, reconciles the issue tracker, and surfaces uncommitted work before the context is lost.
compatibility: 'Requires wfctl to be installed'
---

# End Session

You are ending the current development session. Produce a handoff good enough
that a fresh session (after `/clear`) can resume from the artifacts alone.

## Workflow

### Phase 1: Gather Facts (Mandatory)

1. **Capture the end timestamp** (used in the summary header; also the window for
   the git scan below):
   ```bash
   date -u +"%Y-%m-%dT%H:%M:%SZ"
   ```

2. **Scan this branch's work** to ground the summary in facts, not memory.
   `origin/HEAD` names the default branch when a remote is set; otherwise fall
   back to recent history so this never errors:
   ```bash
   BASE=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null || true)  # e.g. origin/main
   git log --format="%h %s" ${BASE:+${BASE}..HEAD} ${BASE:--20}   # branch commits, or last 20
   git diff --stat                                               # uncommitted changes in the tree
   git log -p ${BASE:+${BASE}..HEAD} ${BASE:--20} | grep -nE '^\+.*(TODO|FIXME)'  # TODOs added
   ```
   Read the commit subjects for what shipped, `--stat` for files touched, and note
   any new TODO/FIXME as follow-up candidates. The branch's own issue is already
   known — it's the branch key (`wfctl status` shows it), no scanning needed. If a
   commit subject says it *also* resolved another issue (your tracker's convention,
   e.g. GitHub `Closes #123` or Jira `Fixes PROJ-45`), note those as secondary
   issues to reconcile in Phase 3.

### Phase 2: Write Summary (Mandatory)

3. **Create the summary scaffold:**
   ```bash
   wfctl end
   ```
   This writes `session-summary.md` in the state dir (`$(wfctl state-dir)`).

4. **Fill in the summary file** - Use `write_to_file` or `apply_diff` to populate
   `$(wfctl state-dir)/session-summary.md` with concrete content from step 2.
   Keep it concrete — this is the next session's starting context:

   ```markdown
   # Session Summary: {YYYY-MM-DD} — {branch}

   **End time:** {timestamp from step 1}
   **Focus:** {one line — what this session was about}
   **Status:** {in progress | complete | blocked}

   ## What We Accomplished
   - {from commit subjects: what shipped}

   ## Decisions Made
   - {decision} — {why} (even small ones; future-you won't remember)

   ## Files Changed
   - `path` — {what changed} (from git diff --stat)

   ## Next Session TODO
   - [ ] {highest-priority next step}
   - [ ] {new TODO/FIXME found in step 2, if worth tracking}

   ## Blockers / Open Questions
   - {anything blocking, or "None"}
   ```

   **CRITICAL**: Do not leave placeholders like "(fill in)". Use the actual data
   from step 2's git scan. If there were no commits, say "No commits this session."
   If there are no blockers, say "None."

### Phase 3: User Actions (Optional - Prompt User)

5. **Prompt user about committing work** - If there are uncommitted changes from
   step 2's `git diff --stat`, ask the user:
   
   "There are uncommitted changes. Would you like me to commit them with a message
   referencing the active issue?"
   
   If yes, commit with a clear message referencing the active issue (include your
   tracker's close keyword if it has one, e.g. GitHub `Closes #N`).

6. **Prompt user about issue tracker updates** - Ask the user:
   
   "Would you like to update the issue tracker? Options:
   - Close the issue (if work is complete)
   - Add a progress comment (if work is partial)
   - Skip tracker updates"
   
   If user chooses to update, use `wfctl issue` commands (skip silently if no
   tracker is configured or a verb is unsupported — `wfctl issue` no-ops in both
   cases). The branch's issue key is from `wfctl status`. Verbs are backend-agnostic
   (GitHub, Jira, or a custom tracker):

   ```bash
   ISSUE=<branch issue key from `wfctl status`>

   # View first, then close only if still open. Idempotent: handles trackers that
   # already auto-closed the issue on merge, and those that don't.
   wfctl issue view "$ISSUE"
   wfctl issue close "$ISSUE" --comment "Completed this session."   # only if still open

   # If work is only partially done, leave it open with a progress note:
   wfctl issue comment "$ISSUE" --body "Partial progress: <what remains>"
   wfctl issue label "$ISSUE" --action add --label in-progress

   # Reconcile any secondary issues noted in step 2 the same way, and file new work:
   wfctl issue create --title "<title>" --body "<context>"
   ```

### Phase 4: Report (Mandatory)

7. **Report to user:**
   - Session closed, summary written to `$(wfctl state-dir)/session-summary.md`
   - What was accomplished (from the summary)
   - Next steps (from the summary's TODO section)
   - Any blockers (from the summary)
   - Tracker updates made (if any from step 6)
   - Remind: If ending because context is filling, they can `/clear` and
     `/start-session` to resume from the summary.

## Error Handling

- If `wfctl end` fails, report the error and do not proceed to step 4
- If git commands fail, use fallback values (e.g., "Unable to scan git history")
- If tracker commands fail, report but continue (tracker updates are optional)