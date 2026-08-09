---
disable-model-invocation: true
description: Start a brainstorming session. Wraps the brainstorming + idea-refine skills, whose output lands in specs/<branch>/design.md for speckit pickup.
handoffs:
  - label: Start Specify
    agent: speckit.specify
    prompt: The design document is ready in specs/<branch>/design.md. Run specify.
    send: true
---

Read `AGENTS.md` at the repository root for project overrides. It is optional —
if the file is absent, proceed silently. Then invoke and follow the
`brainstorming` skill exactly.

Create the destination directory. `<branch>` below is a placeholder — resolve it
from the current branch rather than writing it literally:

```bash
eval "$(wfctl feature-paths)"
mkdir -p "$FEATURE_DIR"
```

`$FEATURE_DIR` is this branch's spec directory; the design document is
`$FEATURE_DIR/design.md`.

After the brainstorming session concludes, invoke the `idea-refine` skill to
sharpen the chosen direction into an actionable one-pager.

**Output:** `specs/<branch>/design.md`, written by `idea-refine` — once, at final
fidelity. `brainstorming` carries its approved design here in context rather than
saving it first; a second write to that path destroys the approved design.
`/speckit.specify` reads the file from there.
