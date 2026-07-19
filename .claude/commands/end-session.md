# End Session

You are ending the current development session. Produce a handoff good enough
that a fresh session (after `/clear`) can resume from the artifacts alone.

## Workflow

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
   any new TODO/FIXME as follow-up candidates. Commit subjects containing
   `Closes #N` / `Fixes #N` are the issues to reconcile in step 5.

3. **Close the session and write the summary scaffold:**
   ```bash
   wfctl end
   ```
   This writes `session-summary.md` in the state dir (`$(wfctl state-dir)`).

4. **Fill in `$(wfctl state-dir)/session-summary.md`** using the scan from step 2.
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

5. **Commit** any uncommitted work with a clear message referencing the active
   issue (`Closes #N` if it fully resolves one).

6. **Reconcile the issue tracker** (skip silently if no tracker is configured or a
   verb is unsupported — `wfctl issue` no-ops in both cases). Backend-agnostic —
   works on GitHub, Jira, or a custom tracker:

   ```bash
   # Verify / close the issue this session resolved (from the Closes #N in step 2)
   wfctl issue view <N>
   wfctl issue close <N> --comment "Completed this session."

   # Or, if work is only partially done, leave it open with a progress note:
   wfctl issue comment <N> --body "Partial progress: <what remains>"
   wfctl issue label <N> --action add --label in-progress

   # File any new work discovered this session:
   wfctl issue create --title "<title>" --body "<context>"
   ```

   Note: on GitHub, pushing a commit with `Closes #N` to the default branch
   already auto-closes the issue — use `wfctl issue view <N>` to confirm and only
   `close` explicitly when it did not (e.g. merged to a non-default branch).

7. **Report:** session closed, summary written, tracker updates made, next steps,
   any blockers. If ending because context is filling, remind the user they can
   `/clear` and `/start-session` to resume from the summary.
