---
name: 'speckit-specify'
description: 'Create or update the feature specification from a natural language feature description. Uses GitHub issue numbers as NNN (collision-proof for parallel agents).'
compatibility: 'Requires spec-kit project structure with .specify/ directory and gh CLI'
metadata:
  author: 'pfms'
  source: '.claude/commands/speckit.specify.md'
---

## Outline

The text the user typed after `/speckit.specify` in the triggering message **is** the feature description. Assume you always have it available in this conversation. Do not ask the user to repeat it unless they provided an empty command.

Given that feature description, do this:

-1. **Pre-specify gate: confirm brainstorming is done**

    Check for an upstream design artifact before generating anything:

    ```
    if .agent/spec.md exists:
      → brainstorming artifact found; proceed to step 0
    else:
      → no brainstorming artifact found
      → prompt user:
          "No brainstorming artifact found at .agent/spec.md.
           For best results, run design work first:
             /superpowers:brainstorming  — requirements + design exploration
           It writes to .agent/spec.md; speckit picks it up automatically.

           Proceed with just the feature description anyway? (yes/no)"
      → if yes: proceed with description only (step 0 will skip)
      → if no: stop here; user will run brainstorming first
    ```

0. **Load pre-specify context from `.agent/spec.md` (if present)**

   Before generating anything, check for a design document from an upstream brainstorming
   or design skill session:

   ```
   if .agent/spec.md exists:
     → load it as additional context for spec generation
     → treat it as the detailed design intent; the user-typed description is supplemental
     → the formal spec.md should faithfully reflect decisions already made in .agent/spec.md
     → note in the spec's Assumptions section: "Pre-specify design context loaded from .agent/spec.md"
   else:
     → proceed with user-typed description only
   ```

   `.agent/spec.md` is the canonical handoff contract from any upstream design skill —
   superpowers brainstorming or any other domain skill. All of them write to
   `.agent/spec.md`; speckit picks it up here regardless of which skill produced it.

1. **Generate a concise short name** (2-4 words) for the branch:
   - Analyze the feature description and extract the most meaningful keywords
   - Create a 2-4 word short name that captures the essence of the feature
   - Use action-noun format when possible (e.g., "add-user-auth", "fix-payment-bug")
   - Preserve technical terms and acronyms (OAuth2, API, JWT, etc.)
   - Keep it concise but descriptive enough to understand the feature at a glance
   - Examples:
     - "I want to add user authentication" → "user-auth"
     - "Implement OAuth2 integration for the API" → "oauth2-api-integration"
     - "Create a dashboard for analytics" → "analytics-dashboard"
     - "Fix payment processing timeout bug" → "fix-payment-timeout"

2. **Determine feature number using GitHub issue (collision-proof)**:

   Sequential NNN numbering has a race condition when parallel agents both check
   at the same time. This workflow uses GitHub issue numbers as feature numbers
   instead — GitHub issue creation is atomic, guaranteeing uniqueness across all
   parallel agents.

   a. **Check if the user provided an issue number** in the feature description:
   - Look for patterns like `#251`, `issue 251`, or `--issue 251`
   - If found → extract the number (e.g., 251), use it as NNN. Skip to step 2d.

   b. **Search for an existing open GitHub issue** matching this feature:
   ```bash
   gh issue list --search "{short-name}" --state open --limit 5 --json number,title
   ```
   - Review results: if a clear match is found (title contains the short-name or
     key feature keywords) → confirm it is the right issue, use its number as NNN.
     Skip to step 2d.
   - If no match or ambiguous → proceed to step 2c.

   c. **Create a new GitHub issue** and use its number:
   ```bash
   gh issue create \
     --title "{short-name}: {feature description summary}" \
     --body "Speckit planning in progress. Spec, plan, and tasks will be linked here." \
     --label "enhancement"
   ```
   - Extract the issue number from the returned URL (e.g., `https://github.com/.../issues/265` → 265)
   - Use that number as NNN.

   d. **Run the create-new-feature script** with the GitHub issue number:
   ```bash
   .specify/scripts/bash/create-new-feature.sh --json "$ARGUMENTS" --number {GH_ISSUE_NUMBER} --short-name "{short-name}"
   ```

   **If the script fails with "branch already exists"** (common when running `/speckit.specify`
   from inside a worktree already created by `wm new` — the branch exists but the spec
   artifacts have not been initialised yet):

   1. Confirm you are on the correct branch: `git branch --show-current`
   2. Derive the paths manually — do not re-run the script:
      - `BRANCH_NAME = {GH_ISSUE_NUMBER}-{short-name}`
      - `FEATURE_DIR = specs/{BRANCH_NAME}`
      - `SPEC_FILE   = {FEATURE_DIR}/spec.md`
   3. Create the artifacts directory:
      ```bash
      mkdir -p specs/{BRANCH_NAME}/checklists
      ```
   4. Continue from step 3 (load spec template) using the derived paths.

   **IMPORTANT**:
   - You must only ever run the create-new-feature script once per feature
   - The JSON output will contain BRANCH_NAME and SPEC_FILE paths
   - The branch will be named `{GH_ISSUE_NUMBER}-{short-name}` (e.g., `251-extract-api-routes`)
   - The spec will live at `specs/{GH_ISSUE_NUMBER}-{short-name}/spec.md`
   - For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot")
   - **Why GitHub issue numbers**: they are globally unique, atomic, and align the
     speckit NNN with the GitHub issue tracking number — one numbering system instead
     of two. Existing sequentially-numbered spec dirs are unaffected.

3. Load `.specify/templates/spec-template.md` to understand required sections.

4. Follow this execution flow:
   1. Parse user description from Input
      If empty: ERROR "No feature description provided"
   2. Extract key concepts from description
      Identify: actors, actions, data, constraints
   3. For unclear aspects:
      - Make informed guesses based on context and industry standards
      - Only mark with [NEEDS CLARIFICATION: specific question] if:
        - The choice significantly impacts feature scope or user experience
        - Multiple reasonable interpretations exist with different implications
        - No reasonable default exists
      - **LIMIT: Maximum 3 [NEEDS CLARIFICATION] markers total**
      - Prioritize clarifications by impact: scope > security/privacy > user experience > technical details
   4. Fill User Scenarios & Testing section
      If no clear user flow: ERROR "Cannot determine user scenarios"
   5. Generate Functional Requirements
      Each requirement must be testable
      Use reasonable defaults for unspecified details (document assumptions in Assumptions section)
   6. Define Success Criteria
      Create measurable, technology-agnostic outcomes
      Include both quantitative metrics (time, performance, volume) and qualitative measures (user satisfaction, task completion)
      Each criterion must be verifiable without implementation details
   7. Identify Key Entities (if data involved)
   8. Return: SUCCESS (spec ready for planning)

5. Write the specification to SPEC_FILE using the template structure, replacing placeholders with concrete details derived from the feature description while preserving section order and headings.

6. **Specification Quality Validation**: After writing the initial spec, validate it against quality criteria:

   a. **Create Spec Quality Checklist**: Generate a checklist file at `FEATURE_DIR/checklists/requirements.md` using the checklist template structure with these validation items:

   ```markdown
   # Specification Quality Checklist: [FEATURE NAME]

   **Purpose**: Validate specification completeness and quality before proceeding to planning
   **Created**: [DATE]
   **Feature**: [Link to spec.md]

   ## Content Quality

   - [ ] No implementation details (languages, frameworks, APIs)
   - [ ] Focused on user value and business needs
   - [ ] Written for non-technical stakeholders
   - [ ] All mandatory sections completed

   ## Requirement Completeness

   - [ ] No [NEEDS CLARIFICATION] markers remain
   - [ ] Requirements are testable and unambiguous
   - [ ] Success criteria are measurable
   - [ ] Success criteria are technology-agnostic (no implementation details)
   - [ ] All acceptance scenarios are defined
   - [ ] Edge cases are identified
   - [ ] Scope is clearly bounded
   - [ ] Dependencies and assumptions identified

   ## Feature Readiness

   - [ ] All functional requirements have clear acceptance criteria
   - [ ] User scenarios cover primary flows
   - [ ] Feature meets measurable outcomes defined in Success Criteria
   - [ ] No implementation details leak into specification

   ## Notes

   - Items marked incomplete require spec updates before `/speckit.clarify` or `/speckit.plan`
   ```

   b. **Run Validation Check**: Review the spec against each checklist item:
   - For each item, determine if it passes or fails
   - Document specific issues found (quote relevant spec sections)

   c. **Handle Validation Results**:
   - **If all items pass**: Mark checklist complete and proceed to step 7

   - **If items fail (excluding [NEEDS CLARIFICATION])**:
     1. List the failing items and specific issues
     2. Update the spec to address each issue
     3. Re-run validation until all items pass (max 3 iterations)
     4. If still failing after 3 iterations, document remaining issues in checklist notes and warn user

   - **If [NEEDS CLARIFICATION] markers remain**:
     1. Extract all [NEEDS CLARIFICATION: ...] markers from the spec
     2. **LIMIT CHECK**: If more than 3 markers exist, keep only the 3 most critical (by scope/security/UX impact) and make informed guesses for the rest
     3. For each clarification needed (max 3), present options to user in this format:

        ```markdown
        ## Question [N]: [Topic]

        **Context**: [Quote relevant spec section]

        **What we need to know**: [Specific question from NEEDS CLARIFICATION marker]

        **Suggested Answers**:

        | Option | Answer                    | Implications                          |
        | ------ | ------------------------- | ------------------------------------- |
        | A      | [First suggested answer]  | [What this means for the feature]     |
        | B      | [Second suggested answer] | [What this means for the feature]     |
        | C      | [Third suggested answer]  | [What this means for the feature]     |
        | Custom | Provide your own answer   | [Explain how to provide custom input] |

        **Your choice**: _[Wait for user response]_
        ```

     4. **CRITICAL - Table Formatting**: Ensure markdown tables are properly formatted:
        - Use consistent spacing with pipes aligned
        - Each cell should have spaces around content: `| Content |` not `|Content|`
        - Header separator must have at least 3 dashes: `|--------|`
        - Test that the table renders correctly in markdown preview
     5. Number questions sequentially (Q1, Q2, Q3 - max 3 total)
     6. Present all questions together before waiting for responses
     7. Wait for user to respond with their choices for all questions (e.g., "Q1: A, Q2: Custom - [details], Q3: B")
     8. Update the spec by replacing each [NEEDS CLARIFICATION] marker with the user's selected or provided answer
     9. Re-run validation after all clarifications are resolved

   d. **Update Checklist**: After each validation iteration, update the checklist file with current pass/fail status

7. Report completion with branch name, spec file path, checklist results, and readiness for the next phase (`/speckit.clarify` or `/speckit.plan`).

   Also remind the user of the branch's role in the delivery workflow:

   > **Planning branch convention**: `{NNN}-{feature-name}` is a **planning branch** — it will become a lightweight planning PR (`specs/{NNN}-{feature-name}/` only, no code) that merges to `dev` before any implementation begins. Implementation branches are created off `dev` after that planning PR merges.
   >
   > **One PR = one issue.** The GitHub issue created for this feature closes when its single implementation PR merges (`Closes #{NNN}` in the PR description). If the feature is too large for one PR, flag it during `/speckit.decompose` — do not pre-split; discuss first.

**NOTE:** The script creates and checks out the new branch and initializes the spec file before writing.

## Quick Guidelines

- Focus on **WHAT** users need and **WHY**.
- Avoid HOW to implement (no tech stack, APIs, code structure).
- Written for business stakeholders, not developers.
- DO NOT create any checklists that are embedded in the spec. That will be a separate command.

### Section Requirements

- **Mandatory sections**: Must be completed for every feature
- **Optional sections**: Include only when relevant to the feature
- When a section doesn't apply, remove it entirely (don't leave as "N/A")

### For AI Generation

When creating this spec from a user prompt:

1. **Make informed guesses**: Use context, industry standards, and common patterns to fill gaps
2. **Document assumptions**: Record reasonable defaults in the Assumptions section
3. **Limit clarifications**: Maximum 3 [NEEDS CLARIFICATION] markers - use only for critical decisions that:
   - Significantly impact feature scope or user experience
   - Have multiple reasonable interpretations with different implications
   - Lack any reasonable default
4. **Prioritize clarifications**: scope > security/privacy > user experience > technical details
5. **Think like a tester**: Every vague requirement should fail the "testable and unambiguous" checklist item
6. **Common areas needing clarification** (only if no reasonable default exists):
   - Feature scope and boundaries (include/exclude specific use cases)
   - User types and permissions (if multiple conflicting interpretations possible)
   - Security/compliance requirements (when legally/financially significant)

**Examples of reasonable defaults** (don't ask about these):

- Data retention: Industry-standard practices for the domain
- Performance targets: Standard web/mobile app expectations unless specified
- Error handling: User-friendly messages with appropriate fallbacks
- Authentication method: Standard session-based or OAuth2 for web apps
- Integration patterns: RESTful APIs unless specified otherwise

### Success Criteria Guidelines

Success criteria must be:

1. **Measurable**: Include specific metrics (time, percentage, count, rate)
2. **Technology-agnostic**: No mention of frameworks, languages, databases, or tools
3. **User-focused**: Describe outcomes from user/business perspective, not system internals
4. **Verifiable**: Can be tested/validated without knowing implementation details

**Good examples**:

- "Users can complete checkout in under 3 minutes"
- "System supports 10,000 concurrent users"
- "95% of searches return results in under 1 second"
- "Task completion rate improves by 40%"

**Bad examples** (implementation-focused):

- "API response time is under 200ms" (too technical, use "Users see results instantly")
- "Database can handle 1000 TPS" (implementation detail, use user-facing metric)
- "React components render efficiently" (framework-specific)
- "Redis cache hit rate above 80%" (technology-specific)
