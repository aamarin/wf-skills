# Issue Grouping Patterns — Detailed Guide

## Core Rule

One PR closes exactly one issue.

Use `tasks.md` as an implementation checklist inside the issue body. Do not create
one GitHub issue per task unless the user explicitly asks for that high-granularity
workflow.

## Pattern 1: Single Issue (Default)

**When to use:**
- XS/S/M features where one PR can deliver the full feature
- User stories are coupled or share verification paths
- A reviewer needs the full change to understand correctness
- Tasks are mostly implementation steps for one coherent outcome

**Grouping rule:** One issue contains the full task checklist. One PR closes it.

**Issue title format:** `[{NNN}] {feature or PR outcome}`

**Worktree base:** the issue's own worktree branches straight off the target
branch, same as any standalone issue. There's no parent epic and no planning
branch to inherit from — the "Epic Planning Branch as Worktree Base" convention
under Pattern 4 does not apply here.

**Example:** 13 tasks → 1 issue → 1 PR.

---

## Pattern 2: Sub-Feature Split

**When to use:**
- PR boundary analysis finds truly independent, mergeable sub-features
- Each sub-feature can be reviewed, tested, and merged without the others
- The user agrees to split an L-sized feature into multiple PRs

**Grouping rule:** One issue per PR-sized sub-feature.

**Issue title format:** `[{NNN}] {US title or phase description}`

**Example:**
- Issue A: `[338] Budget scenario tab shell`
- Issue B: `[338] Scenario comparison data wiring`
- Issue C: `[338] Scenario activation and validation`

Each issue closes with its own PR. If a parent epic issue exists, the final PR may
also close the parent after all child issues are complete.

**Worktree base:** if a parent epic issue exists, use the "Epic Planning Branch as
Worktree Base" convention documented under Pattern 4 — it applies to any pattern
with a parent epic + child issues, not just the Hierarchical structure.

---

## Pattern 3: Phase-Grouped

**When to use:**
- Feature is pure infrastructure/refactor with no distinct user-facing stories
- tasks.md phases are the natural grouping unit
- Feature has more than 3 phases and >10 tasks

**Grouping rule:** One issue per PR-sized phase. Merge setup/foundational/polish
into an adjacent issue when they have no standalone merge value.

**Issue title format:** `[{NNN}] Phase {N}: {phase title}`

**Constraint:** Do not create several phase issues if the result is still one PR.

---

## Pattern 4: Hierarchical (Multi-PR Features)

**When to use:**
- Feature requires 2+ PRs (L or XL scope)
- Each PR delivers a mergeable, independently testable increment
- Stakeholders need to track progress at the PR level

**Structure:**
```
Parent issue (#251): umbrella, stays open until all PRs merge
  └── Child A (#265): PR 1 — Story 1 tasks — closes with PR 1
  └── Child B (#266): PR 2 — Story 2 tasks — closes with PR 2
  └── Child C (#267): PR 3 — Story 3 tasks — closes with PR 3 (final PR also closes #251)
```

**Parent issue body must include a task list:**
```markdown
## Progress

- [ ] #265 Story 1: {description}
- [ ] #266 Story 2: {description}
- [ ] #267 Story 3: {description}
```

**PR closing convention:**
- PR 1 description: `Closes #265`
- PR 2 description: `Closes #266`
- PR 3 description: `Closes #267, Closes #251`

The parent close is allowed only on the final PR and only when the parent issue's
acceptance criteria are fully satisfied.

### Epic Planning Branch as Worktree Base

When `brainstorm`→`decompose` ran on a dedicated planning branch (the branch
`spec.md`/`plan.md`/`tasks.md`/`delivery.md` live on — typically the parent issue's
own branch, e.g. `251-extract-api-routes`), branch each child issue's worktree from
**that planning branch**, not from the target branch. This carries `specs/{feature}/`
along automatically, so the child PR's automated review (e.g. GitHub Copilot) can
check the implementation against spec/plan/contracts with no manual file copying:

```
wm add {child-issue}-{slug} --base {epic-planning-branch}
```

`specs/{feature}/` stays committed on the child branch for the life of its PR — it
is never deleted with a commit. At merge time, remove it from what actually lands
in the target branch by unstaging it out of the merge rather than committing its
removal, so the target branch never receives the spec directory even transiently:

```
git merge --squash <child-branch>
git reset -- specs/{feature}
git commit
```

Once the child branch merges, its worktree is normally deleted along with it —
which would otherwise take the on-disk `specs/{feature}/` copy with it. Use
workmux's `pre_remove` lifecycle hook (the same mechanism it documents for backing
up gitignored files before cleanup) to persist that state locally first, e.g.
copying it back into the main worktree's `specs/` directory, so the spec artifacts
survive after the child worktree and branch are gone.

The planning branch itself never merges to the target branch — it carries no
production code, only planning artifacts. Delete it once every child issue has
merged.

---

## Pattern 5: 1:1 Task-to-Issue (Rare)

**When to use:**
- The user explicitly requests task-per-issue tracking
- Each task is independently reviewable and likely to become its own PR
- Different agents or owners need independent GitHub assignment

**When NOT to use:**
- XS/S/M features
- Verification-only tasks
- Setup/polish tasks with no standalone value
- Any case where several issues would close from one PR

**Issue title format:** `{feature-id} T{NNN}: {task description verb phrase}`

## Issue Body Quality Rules

Regardless of pattern, every child issue must have:

1. **Summary sentence** — one human-scannable line
2. **Context** — feature ID, task IDs, phase, story, why this matters
3. **Entry Points** — specific files (not directories)
4. **Acceptance Criteria** — measurable, from task + spec validation steps
5. **Verification** — automated command or manual check for each criterion
6. **Dependencies** — task IDs or issue numbers, explicit
7. **References** — links to spec.md, plan.md, tasks.md
8. **Estimate** — rough hours or t-shirt size

Never include: `Assigned To`, `Status`, `Deadline` in the body (use GitHub native fields).

---

## Pattern Selection Quick Reference

```
How many PRs should this feature produce?
├── 1 → Single Issue
├── 2+ independent sub-features → Sub-Feature Split or Hierarchical
├── 2+ infra/refactor phases → Phase-Grouped
└── User explicitly requests task granularity → 1:1 Task-to-Issue
```
