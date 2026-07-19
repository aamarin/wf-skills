---
name: scaffold-tracker
description: Author a new issue-tracker backend for `wfctl issue` by generating a `.agents/trackers/<name>.json` verb→command map. Use when a repo uses a tracker other than GitHub (e.g. a private Jira CLI) and needs the session skills to reconcile against it.
---

# Scaffold an issue-tracker backend

`wfctl issue <verb>` delegates issue operations to a per-repo backend defined by
`.agents/trackers/<name>.json`. GitHub ships as `github.json`. This skill authors
a config for any other tracker (a private Jira CLI, Linear, a custom script) so
the session skills work unchanged.

## The verb contract

The session skills speak six standard verbs. A backend implements the subset it
supports — the presence of a verb key is its declaration, so a backend cannot lie
about what it can do.

| Verb      | Meaning                     | Params available for `{...}` substitution |
|-----------|-----------------------------|-------------------------------------------|
| `list`    | list open issues            | (none)                                    |
| `view`    | show one issue              | `{id}`                                     |
| `close`   | close an issue with comment | `{id}`, `{comment}`                        |
| `comment` | comment on an issue         | `{id}`, `{body}`                           |
| `create`  | open a new issue            | `{title}`, `{body}`                        |
| `label`   | add/remove a label          | `{id}`, `{action}` (add\|remove), `{label}`|

Each verb maps to an **argv list** (never a shell string). `{name}` placeholders
are substituted per-token from the CLI options, so free text like a comment body
is always one inert argument — no shell injection, no quoting to get right.
Substitution is within-token, so `"--{action}-label"` becomes `--add-label`.

## Steps

1. **Ask for the tracker name** (lowercase, e.g. `jira`, `linear`). The file will
   be `.agents/trackers/<name>.json`.

2. **Ask which of the six verbs this backend supports.** Only include verbs the
   backend can actually do — omit the rest; `wfctl issue` skips unsupported verbs.

3. **For each supported verb, ask for the concrete command** as an argv list,
   using the `{...}` placeholders from the table above where the backend needs
   them. Example, for a Jira CLI invoked as `jiractl`:
   - `view`  → `["jiractl", "show", "{id}"]`
   - `comment` → `["jiractl", "comment", "{id}", "--message", "{body}"]`
   - (a tracker may map `{id}` onto its own key format, e.g. `PROJ-{id}`)

4. **Write** `.agents/trackers/<name>.json`:
   ```json
   {
     "verbs": {
       "view": ["jiractl", "show", "{id}"],
       "comment": ["jiractl", "comment", "{id}", "--message", "{body}"]
     }
   }
   ```

5. **Validate before finishing.** A malformed config does not crash `wfctl issue`
   — it *silently disables the tracker* (the loader treats invalid JSON as "no
   config" and every verb no-ops). So catch problems here. Run this guard against
   the file you wrote (replace `<name>`):

   ```bash
   python3 - "$PWD/.agents/trackers/<name>.json" <<'PY'
   import json, re, sys
   ALLOWED = {
       "list": set(), "view": {"id"}, "close": {"id", "comment"},
       "comment": {"id", "body"}, "create": {"title", "body"},
       "label": {"id", "action", "label"},
   }
   path = sys.argv[1]
   try:
       cfg = json.load(open(path))
   except (OSError, json.JSONDecodeError) as e:
       sys.exit(f"INVALID: {e}")
   verbs = cfg.get("verbs")
   if not isinstance(verbs, dict) or not verbs:
       sys.exit("INVALID: missing non-empty 'verbs' object")
   errs = []
   for verb, argv in verbs.items():
       if verb not in ALLOWED:
           errs.append(f"unknown verb '{verb}' (allowed: {sorted(ALLOWED)})"); continue
       if not isinstance(argv, list) or not argv or not all(isinstance(t, str) for t in argv):
           errs.append(f"'{verb}': must be a non-empty list of strings"); continue
       used = {m for tok in argv for m in re.findall(r"\{(\w+)\}", tok)}
       bad = used - ALLOWED[verb]
       if bad:
           errs.append(f"'{verb}': placeholder(s) {sorted(bad)} not allowed "
                       f"(allowed: {sorted(ALLOWED[verb]) or 'none'})")
   if errs:
       sys.exit("INVALID:\n- " + "\n- ".join(errs))
   print("OK:", ", ".join(verbs))
   PY
   ```

   It prints `OK: <verbs>` on success, or `INVALID:` with the specific problems
   and a non-zero exit. Fix any reported issue and re-run until it passes. Do not
   silently "fix" a deliberately omitted verb — omission is how a backend declares
   it doesn't support that operation.

6. **Point the repo at it:** run `wfctl install-skills --tracker <name>` (or edit
   the `"tracker"` key in `.wf-skills-manifest.json`) so `wfctl issue` uses this
   backend. Verify with a read-only call, e.g. `wfctl issue view <some-id>`.
