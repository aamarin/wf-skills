# Delivery Plan: {Feature Name} ({NNN})

**Feature**: `{NNN}-{feature-name}` | **Date**: {DATE}
**Source**: `specs/{NNN}-{feature-name}/tasks.md` ({N} tasks)
**Parent issue**: #{PARENT_ISSUE}

---

## PR Decomposition

| PR | Tasks | Files Touched | Size | Merge Condition |
|----|-------|--------------|------|----------------|
| #{PR_NUMBER} | {task range} | `{file1}` ({created/modified}), `{file2}` ({created/modified}) | {XS/S/M/L} | {condition} |

**Rationale**: {Single PR / Multiple PRs}. {One sentence explaining why — mutual dependency, independent stories, etc.}

**PR closes**: `{Closes #{issue-for-this-PR}}`

If this is the final PR for a parent epic and all parent acceptance criteria are
satisfied, add the parent close separately: `{Closes #{parent}}`.

---

## Issue Grouping Map

| Issue | Tasks | Title | Estimate | Closes With |
|-------|-------|-------|----------|-------------|
| Issue A | {task IDs} | `[{NNN}] {group description}` | {estimate} | PR #{N} |
| Issue B | {task IDs} | `[{NNN}] {group description}` | {estimate} | PR #{N} |
| Issue C | {task IDs} | `[{NNN}] {group description}` | {estimate} | PR #{N} |

**Grouping pattern**: {Single issue / Sub-feature split / Phase-grouped / Hierarchical / 1:1 explicit}
**Rationale**: {One sentence explaining the pattern choice}

---

## Parallelization Waves

| Wave | Mode | Tasks | Gate / Notes |
|------|------|-------|-------------|
| 0 | Sequential | {tasks} | {gate condition or "no dependencies"} |
| 1 | Parallel | {task} ‖ {task} | {constraint or "no dependencies"} |
| 2 | Parallel | {task} ‖ {task} | {constraint} |
| 3 | Sequential | {task} | {gate condition} |
| 4 | Sequential | {task} → {task} | {ordering reason} |

**Single-agent order** (recommended for {XS/S} features):
{T001} → {T002} → ... → {T013}

---

## Agent Fanning Instructions

{For XS features:}
Single agent recommended for this {size} feature. Wave table above provided for
reference and template reuse.

{For M+ features:}
**Wave {N} fanning ({N} agents):**

**Agent A prompt:**
```
{copy-paste agent prompt}
```

**Agent B prompt:**
```
{copy-paste agent prompt}
```

**Fan-in gate after Wave {N}:** `{command}`
