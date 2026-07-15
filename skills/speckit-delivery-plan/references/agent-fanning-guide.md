# Agent Fanning Guide — Wave-Based Parallelization

## When to Fan Agents

Fan agents when:
- Feature is M or larger (5+ files) AND independent waves exist
- You have a single wave with 3+ parallel-safe tasks
- Time savings > overhead of coordinating multiple agents

Do NOT fan for:
- XS features (2 files) — single agent is faster
- Waves where all tasks are sequential — no benefit
- Tasks that share state mid-execution

---

## Wave Identification

### Step 1: Build the dependency graph

For each task pair (A, B): does B require A's output to exist?
- YES → A must complete before B (sequential edge)
- NO → A and B are parallel candidates

### Step 2: Topological sort → wave assignments

All tasks with no remaining dependencies = same wave.

```
Example (018):
T003 (no deps)                    → Wave 0
T001, T002 (depend on Wave 0 done) → Wave 1 [parallel]
T004, T005 (depend on Wave 0 done) → Wave 2 [parallel]
T006 (depends on T004 AND T005)   → Wave 3 (fan-in gate)
T007 (depends on T006)            → Wave 4
T008 (depends on T007)            → Wave 4 (sequential after T007)
T009, T010, T011 (deps on T008)   → Wave 5 [parallel]
T012, T013 (deps on Wave 5)       → Wave 6 [parallel]
```

### Step 3: Identify fan-worthwhile waves

A wave is worth fanning when:
- It has 3+ parallel-safe tasks, OR
- Each task takes >30 minutes and they total >1 hour

---

## Agent Prompt Templates

### Single-file creation agent

```
You are implementing task {T_ID} for feature {NNN}-{feature-name}.

Task: Create {file_path}
Spec: {feature_dir}/spec.md
Plan: {feature_dir}/plan.md (see Implementation Sequence, Step {N})

Requirements:
- {requirement 1 from task description}
- {requirement 2 from task description}

When complete:
- Signal "T{ID} complete" in your final message
- Do NOT run type-check yet — parallel agents are modifying other files
- Fan-in gate: all Wave {N} agents must complete before type-check runs
```

### Single-file modification agent

```
You are implementing task {T_ID} for feature {NNN}-{feature-name}.

Task: Modify {file_path}
Current file content available at: {file_path}

Changes required:
- {specific change 1}
- {specific change 2}

Constraints:
- Leave {unchanged section} exactly as-is
- Verify change with: {validation command} (run ONLY after receiving fan-in signal)

Signal "T{ID} complete" when edits are saved.
```

### Verification agent (grep/type-check)

```
You are running verification task {T_ID} for feature {NNN}-{feature-name}.

Run these commands in order:
1. {command 1} — expected output: {expected}
2. {command 2} — expected output: {expected}

Report: "T{ID} PASS" if all outputs match, "T{ID} FAIL: {discrepancy}" if not.
Do not modify any files.
```

---

## Fan-in Protocol

After dispatching Wave N parallel agents, wait for all to signal completion
before proceeding to Wave N+1.

```
Dispatcher: "Agents T004 and T005 — begin Wave 2"

[agents work in parallel]

Agent T004: "T004 complete — server/src/routes/index.ts created"
Agent T005: "T005 complete — server/src/index.ts modified"

Dispatcher: "Both T004 and T005 complete. Running Wave 3 gate:
             pnpm --filter server type-check"
```

---

## 018 Agent Fanning Reference

Feature 018 is XS — single agent recommended. If fanning Wave 2 for practice:

**Agent A (T004):**
```
Create server/src/routes/index.ts with registerApiRoutes(app: Application): void.
Import all 14 domain route modules from server/src/index.ts (current imports).
Register them in order: /api/debt, /api/debts (alias), /api/bills, /api/transactions,
/api/budgets, /api/categories, /api/payment-schedule, /api/accounts, /api/dashboard,
/api/forecast, /api/planned-income, /api/reports, /api/scenarios, /api/tenants.
Do NOT run type-check — Agent B is modifying index.ts simultaneously.
Signal "T004 complete" when file is saved.
```

**Agent B (T005):**
```
Modify server/src/index.ts:
1. Remove all 14 domain route module imports (debtRoutes through tenantRoutes).
2. Add: import { registerApiRoutes } from './routes'
3. Replace the 14 app.use('/api/...') block with: registerApiRoutes(app)
   Position: after app.use(attachEnhancedClient), before the Sentry conditional.
4. Leave health routes, middleware, error handler, 404 handler, app.listen unchanged.
Do NOT run type-check — Agent A is creating routes/index.ts simultaneously.
Signal "T005 complete" when file is saved.
```

**After both complete:** `pnpm --filter server type-check`
