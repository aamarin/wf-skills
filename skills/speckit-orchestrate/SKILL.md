---
name: 'speckit-orchestrate'
description: 'Read pipeline state after a speckit step completes, then auto-advance or surface the next command based on the step auto flag.'
---

## Steps

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
