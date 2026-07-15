---
description: Generate a per-task agent brief (.agent-runs/brief.md) from the active GitHub Issue or JIRA ticket. Scopes this agent to the task with hard stops and escalation criteria.
handoffs:
  - label: Start Working
    agent: init
    prompt: Load context and begin work within brief scope
---

Read `.agents/commands/speckit.brief.md` for the complete brief generation workflow.
