# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: TypeScript on Node.js 20+/22+; record any feature-specific
version constraints  
**Primary Dependencies**: Express 5, Vue 3, ZenStack, Prisma, PostgreSQL, Zod,
pnpm, Vitest; add feature-specific libraries only when required  
**Storage**: PostgreSQL via ZenStack/Prisma unless the feature explicitly uses a
different store  
**Testing**: Vitest plus feature-appropriate integration, contract, schema, or
UI validation commands  
**Target Platform**: Web application with Vue client and Express server
**Project Type**: Monorepo web application (`client/`, `server/`)  
**Performance Goals**: [feature-specific measurable target or NEEDS CLARIFICATION]  
**Constraints**: Workspace isolation, ZenStack policy enforcement, generated
artifact integrity, and minimal-complexity bias  
**Scale/Scope**: [feature-specific users, domains, workflows, or NEEDS CLARIFICATION]

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

- [ ] Workspace impact is defined: affected models, `workspaceId` boundaries,
      and cross-workspace risk are identified.
- [ ] Access-control impact is defined: affected ZenStack policies, audit-trail
      expectations, and authorization boundaries are listed.
- [ ] Schema integrity is defined: affected `.zmodel` files, tier placement,
      `bootstrap.zmodel` impact, generation steps, and migration needs are
      recorded.
- [ ] Evidence exists: relevant `.claude/context/*` files, TDRs, source
      references, or external authoritative research are cited.
- [ ] Validation plan exists: `pnpm type-check` plus the specific automated
      tests and checks needed for the changed surface are named.
- [ ] Complexity is justified: any added abstraction, infrastructure, or
      dependency has a measured or explicit reason the simpler path is
      insufficient.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```text
# [REMOVE IF UNUSED] Option 1: Single project (DEFAULT)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

# [REMOVE IF UNUSED] Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

# [REMOVE IF UNUSED] Option 3: Mobile + API (when "iOS/Android" detected)
api/
└── [same as backend above]

ios/ or android/
└── [platform-specific structure: feature modules, UI flows, platform tests]
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation                  | Why Needed         | Simpler Alternative Rejected Because |
| -------------------------- | ------------------ | ------------------------------------ |
| [e.g., 4th project]        | [current need]     | [why 3 projects insufficient]        |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient]  |
