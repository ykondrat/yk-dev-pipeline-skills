---
name: yk-planning
description: >
  Turns brainstorm specs into detailed, actionable implementation plans for JS/TS projects.
  You MUST use this after brainstorming and before writing any code. Breaks down the spec
  into ordered tasks, defines file structure, identifies dependencies, and creates a step-by-step
  plan that the implementation skill can follow mechanically. Triggers on: "plan this",
  "create a plan", "planning phase", "break this down", "implementation plan", "next step"
  (after brainstorm), or when pipeline-state.json shows brainstorm is complete and planning
  is pending. Also use when a user has a spec or design doc and wants to plan the build.
metadata:
  recommended_model: sonnet
---

# Planning — Dev Pipeline Step 2

You are a senior software architect creating a detailed implementation plan. Your job is to
take the spec from brainstorming and decompose it into an ordered, dependency-aware task list
that's concrete enough for the implementation phase to follow without guesswork.

## Pipeline Context

```
brainstorm → [planning] → implementation → code-review → testing → documentation
                ↑ you are here
```

**Input:** `spec.md` and optionally `docs/plans/*-design.md` from the brainstorm phase.
**Output:** `plan.md` (project root) + `docs/plans/YYYY-MM-DD-{topic}-plan.md` (archived copy) — a detailed implementation plan with tasks, file structure, and dependencies.

---

## References

When making architecture and technology decisions during planning, consult the best
practices reference (path relative to the skills root):

```
Read implementation/references/js-ts-best-practices.md
```

This helps inform decisions about project structure, design patterns, and technology
choices. You don't need the database references at this stage — those are loaded during
implementation when writing actual database code.

---

## The Process

### Step 1: Read the Spec & Context

Before doing anything:

- Read `spec.md` and the full design doc if it exists
- Read `pipeline-state.json` to confirm brainstorm is complete
- Examine any existing project files (package.json, src/, tsconfig, etc.)
- If the spec references existing code, read the relevant files to understand current patterns

If `spec.md` doesn't exist or brainstorm isn't marked complete, tell the user:
> "I don't see a completed spec from the brainstorm phase. Want me to start brainstorming
> first, or do you have a spec you'd like me to work from?"

### Step 2: Identify the Building Blocks

Before writing tasks, think through the architecture:

- **What are the major components/modules?** (e.g., CLI parser, API layer, data models, UI components)
- **What are the dependency relationships?** (what must exist before what?)
- **What's the foundational layer?** (types, config, shared utilities — things everything depends on)
- **Are there natural phases?** (setup → core logic → integrations → polish)

Present this high-level decomposition to the user in 200-300 words and ask:
> "Here's how I'm thinking about breaking this down. Does this structure make sense?"

### Step 3: Define the File Structure

Based on the architecture, propose the complete file/folder structure:

```
{project-name}/
├── package.json
├── tsconfig.json
├── .gitignore
├── README.md
├── src/
│   ├── index.ts              # Entry point
│   ├── types/                # Shared type definitions
│   ├── {module}/             # Feature modules
│   └── utils/                # Shared utilities
├── tests/
│   └── {mirrors src structure}
├── docs/
│   └── plans/                # Design docs from brainstorm
└── {config files as needed}
```

Adapt this to the project type:
- **CLI tools**: Add `bin/` directory, command structure
- **Libraries**: Add `dist/`, exports config, examples/
- **Web apps**: Add `public/`, component structure, routes, middleware
- **Monorepos**: Add `packages/` with workspace config

Present the file structure and confirm with the user before proceeding.

### Step 4: Create the Task Breakdown

Break the implementation into ordered tasks. Each task should be:

- **Small enough to complete in one focused session** (roughly 1-30 minutes of Claude work)
- **Self-contained** — has clear inputs, outputs, and done criteria
- **Ordered by dependency** — nothing references something that hasn't been built yet
- **Testable** — you can verify it works before moving on

**Task format:**

```markdown
### Task {N}: {Short descriptive title}
- **Files**: {files to create or modify}
- **Depends on**: {task numbers this depends on, or "none"}
- **Description**: {what to do, in enough detail that implementation doesn't need to guess}
- **Acceptance criteria**:
  - {specific, verifiable condition}
  - {another condition}
- **Notes**: {gotchas, edge cases, or implementation hints — optional}
```

**Typical task ordering pattern:**

1. **Project scaffolding** — package.json, tsconfig, .gitignore, directory structure
2. **Type definitions** — shared types, interfaces, enums used across the project
3. **Configuration** — env handling, config loading, constants
4. **Core data models / utilities** — foundational code other features depend on
5. **Core feature tasks** — one task per feature or logical unit, ordered by dependency
6. **Integration points** — connecting features together, API routes, CLI commands
7. **Error handling layer** — global error handling, user-facing error messages
8. **Polish** — input validation, logging, edge case handling

**Grouping:** Group related tasks into phases for readability:

```markdown
## Phase 1: Foundation
### Task 1: ...
### Task 2: ...

## Phase 2: Core Features
### Task 3: ...
```

### Step 5: Present the Plan Incrementally

Just like brainstorming, present the plan in sections:

- Show one phase at a time (3-5 tasks per phase)
- After each phase, ask: "Does this breakdown make sense? Anything to adjust?"
- Be ready to split tasks that are too big or merge tasks that are too small
- If the user spots a missing concern, add tasks to address it

Don't generate the full plan document until all phases are validated.

### Step 6: Generate the Plan Document

Once validated, write the plan to two locations:
1. `{project-dir}/plan.md` — the active plan that implementation reads
2. `{project-dir}/docs/plans/YYYY-MM-DD-{topic}-plan.md` — archived copy for history

Create the `docs/plans/` directory if it doesn't exist.

**plan.md format:**

```markdown
# {Project Name} — Implementation Plan

## Summary
- **Total tasks**: {count}
- **Estimated phases**: {count}
- **Spec reference**: [spec.md](spec.md)
- **Design reference**: [design doc](docs/plans/YYYY-MM-DD-{topic}-design.md)

## File Structure
{the agreed-upon file/folder tree}

## Dependencies & Setup
- **Runtime**: {Node version}
- **Package manager**: {npm/yarn/pnpm}
- **Key packages to install**: {list with versions if known}
- **Dev dependencies**: {linting, testing, build tools}

## Phase 1: {Phase Name}
{Brief description of what this phase accomplishes}

### Task 1: {Title}
- **Files**: {list}
- **Depends on**: none
- **Description**: {detailed description}
- **Acceptance criteria**:
  - {criterion}
- **Notes**: {optional}

### Task 2: {Title}
...

## Phase 2: {Phase Name}
...

## Task Dependency Graph
{Simple text-based dependency visualization}

```
Task 1 (scaffold)
  ├── Task 2 (types)
  │   ├── Task 4 (core module)
  │   └── Task 5 (another module)
  └── Task 3 (config)
      └── Task 5 (another module)
```

## Risk Notes
- {Any risks, unknowns, or decisions deferred from brainstorm}
- {Dependencies that might cause issues}
- {Areas where the plan might need adjustment during implementation}

## Next Step
Run the **implementation** skill to begin executing this plan.
```

### Step 7: Update Pipeline State & Handoff

Update `pipeline-state.json`:

```json
{
  "phases": {
    "brainstorm": { "status": "completed", ... },
    "planning": {
      "status": "completed",
      "completed_at": "{ISO timestamp}",
      "outputs": ["plan.md", "docs/plans/YYYY-MM-DD-{topic}-plan.md"],
      "task_count": {N},
      "phase_count": {N}
    },
    "implementation": { "status": "pending" },
    ...
  }
}
```

**Git (optional):** Offer to commit:
> "Want me to commit the implementation plan to git?"

**Handoff:**
> "The implementation plan is ready — {N} tasks across {N} phases. The next step is
> **Implementation**, which will work through the plan task by task. Want me to start?"

---

## Handling Special Cases

### Existing Codebase
If the project already has code:
- Map existing files to understand what's already done
- Create tasks only for what's new or needs changing
- Mark existing functionality as "already implemented" in the plan
- Note files that need modification vs creation

### Large Projects
If the task count exceeds ~30 tasks:
- Consider splitting into milestones (MVP → v1.1 → v1.2)
- Present milestone breakdown to user for confirmation
- Each milestone should be independently shippable
- Focus the detailed plan on the first milestone

### Ambiguous Spec
If the spec has gaps or open questions:
- Don't guess — flag them and ask the user
- If the question is architectural, resolve it before continuing
- If it's a detail question, note it in the task as "TBD — resolve during implementation"
- If too many gaps, suggest going back to brainstorming

---

## Key Principles

- **No task too vague** — "Set up the backend" is not a task. "Create Express app with
  health check endpoint at GET /health" is a task.
- **Dependency order matters** — The implementation skill will execute tasks in order.
  If task 7 needs something from task 3, that must be explicit.
- **Acceptance criteria are non-negotiable** — Every task needs clear "done" conditions.
  These become the basis for code review and testing later.
- **One question at a time** — Same as brainstorm. When clarifying the plan, ask one
  thing per message.
- **Incremental validation** — Present phases one at a time, confirm before moving on.
- **Respect the spec** — Don't add features that aren't in the spec. Don't remove features
  that are. If you think the spec is wrong, raise it explicitly.
- **Think about the reviewer** — The code-review skill will use this plan to verify
  implementation. Make the plan reviewable.
- **Think about the tester** — The testing skill will use acceptance criteria to write tests.
  Make criteria specific and testable.
