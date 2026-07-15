---
description: Start a brainstorming session. Wraps superpowers:brainstorming + idea-refine and writes the result to .agent/spec.md for speckit pickup.
handoffs:
  - label: Start Specify
    agent: speckit.specify
    prompt: The spec is ready in .agent/spec.md. Run specify.
    send: true
---

Read `.agent/AGENT.md` if it exists for project overrides, then invoke and follow the `brainstorming` skill exactly.

**Output:** Write the final approved spec to `.agent/spec.md`. This is required for `/speckit.specify` to pick it up automatically.

After the brainstorming session concludes, invoke the `idea-refine` skill to sharpen the chosen direction into an actionable one-pager. Save the idea-refine output to `.agent/spec.md`.
