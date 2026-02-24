---
name: yk-dev-pipeline
description: >
  Full JS/TS development pipeline with 6 phases: brainstorm (or investigation), planning,
  implementation, code-review, testing, and documentation. Use this for any JS/TS project,
  feature, bug fix, or refactoring. Each phase produces artifacts consumed by the next.
  Semi-automatic — suggests the next step, user confirms. Triggers on: "start a new project",
  "build me a...", "let's develop...", "new feature", "start pipeline", "fix this bug",
  "debug this", "investigate", "refactor", "performance issue", "tech debt", or any request
  to build, fix, or improve a JS/TS application.
metadata:
  recommended_model: sonnet
---

# JS/TS Development Pipeline

You have access to a 6-phase development pipeline for building production-quality
JavaScript/TypeScript projects. Each phase has a dedicated skill with detailed instructions
and reference materials.

## Pipeline Overview

```
[1a. Brainstorm] ─┐
                  ├→ [2. Planning] → [3. Implementation] → [4. Code Review] → [5. Testing] → [6. Documentation]
[1b. Investigation]┘       ↓                 ↓                    ↓                  ↓                ↓
                        plan.md         working code        review.md + fix-plan   test-report.md    README + docs/
       ↓                                                         ↓ (if blocked)
    spec.md                                                 back to Implementation
```

Phase 1 has two alternatives:
- **Brainstorm** — for new features, new projects, adding functionality
- **Investigation** — for bug fixes, refactoring, performance issues, tech debt

## How to Use

### Starting Phase 1

Detect the user's intent and route to the correct Phase 1 skill:

**New feature / new project / adding functionality → Brainstorm:**
```
Read brainstorm/SKILL.md
```

**Bug fix / refactoring / performance / tech debt / debugging → Investigation:**
```
Read investigation/SKILL.md
```

Both skills produce `spec.md` as their output, so Phase 2 (Planning) works the same
regardless of which Phase 1 was used.

### Continuing the Pipeline

After each phase completes, suggest the next phase. The user confirms before proceeding.
When moving to the next phase, read that phase's skill:

```
Phase 1 complete (brainstorm or investigation) → "Ready for planning? I'll break this into tasks."
  Read planning/SKILL.md

Phase 2 complete → "Ready to start implementing? I'll work through the tasks."
  Read implementation/SKILL.md
  Read implementation/references/js-ts-best-practices.md
  Read implementation/references/databases-*.md (if applicable)

Phase 3 complete → "Ready for code review? I'll review everything."
  Read code-review/SKILL.md
  Read code-review/references/review-checklists.md

Phase 4 complete (approved) → "Ready for testing? I'll write comprehensive tests."
  Read testing/SKILL.md

Phase 4 complete (blocked) → "Issues found. I'll fix them and re-review."
  Read implementation/SKILL.md (to execute fix-plan.md)
  Then re-read code-review/SKILL.md (diff review)

Phase 5 complete → "Ready for documentation? I'll generate all project docs."
  Read documentation/SKILL.md

Phase 6 complete → "Pipeline complete! Project is built, reviewed, tested, and documented."
```

### Jumping to a Specific Phase

The user can skip phases or jump to any phase directly:

- "Just implement this" → Start at Phase 3 (but note missing spec/plan)
- "Review my code" → Start at Phase 4
- "Write tests for this" → Start at Phase 5
- "Generate docs" → Start at Phase 6
- "Continue from where we left off" → Check pipeline-state.json

### Pipeline State

All phases track progress in `pipeline-state.json`. Check this file to know where
the pipeline is and what's been completed:

```json
{
  "project_name": "{project name}",
  "created_at": "{ISO}",
  "current_phase": "implementation",
  "phases": {
    "brainstorm": { "status": "completed", "outputs": ["spec.md", "docs/plans/..."] },
    "planning": { "status": "completed", "outputs": ["plan.md"] },
    "implementation": { "status": "in-progress", "completed_tasks": [1, 2, 3] },
    "code-review": { "status": "pending" },
    "testing": { "status": "pending" },
    "documentation": { "status": "pending" }
  }
}
```

**Note:** The first phase key is either `brainstorm` OR `investigation` — they are
alternatives, never both. If investigation was Phase 1, the state will have an
`investigation` key instead of `brainstorm`.

### State Validation

When reading `pipeline-state.json`, validate before proceeding:

- **Missing artifacts:** If a phase says "completed" but its outputs don't exist on disk,
  warn the user and suggest re-running that phase.
- **Blocked without fix plan:** If code-review or testing is "blocked" but `fix-plan.md` /
  `fix-test-plan.md` is missing, flag the inconsistency.
- **Stale state:** If `current_phase` doesn't match the most recently completed phase + 1,
  ask the user which phase they want to continue from.
- **Out-of-order completion:** If a later phase is completed but an earlier one isn't,
  note this and ask if it's intentional (e.g., user jumped to a phase directly).
- **Both brainstorm and investigation:** If `pipeline-state.json` contains both a
  `brainstorm` and `investigation` key, flag this as unusual — they are alternatives.
  Ask the user which one applies.

### Continuation Logic

When the user says "next step", "continue", "go ahead", or similar:

1. Read `pipeline-state.json`
2. Find `current_phase` and phase statuses
3. Route to the next action:

| Current state | Next action |
|---|---|
| No pipeline-state.json | Ask: new project (brainstorm) or investigate existing code? |
| brainstorm/investigation: completed, planning: pending | Start Planning |
| planning: completed, implementation: pending | Start Implementation |
| implementation: completed, code-review: pending | Start Code-Review |
| code-review: completed (approved), testing: pending | Start Testing |
| code-review: blocked | Start Implementation (fix mode — reads fix-plan.md) |
| testing: completed, documentation: pending | Start Documentation |
| testing: blocked | Start Implementation (test-fix mode — reads fix-test-plan.md) |
| documentation: completed | Pipeline complete — summarize and suggest new work |
| Any phase: in-progress | Resume that phase |

### Intent Detection Rules

When routing a new user request (not "next step" / "continue"):

**Rule 1 — Pipeline state takes priority.** If `pipeline-state.json` exists with an active pipeline, check if the request relates to the current pipeline before starting a new one.

**Rule 2 — Build-something intent => Brainstorm.** Keywords: "build", "create", "develop", "new feature", "new project", "add a feature", "I want to make", "I have an idea".

**Rule 3 — Fix-something intent => Investigation.** Keywords: "fix", "bug", "broken", "debug", "refactor", "slow", "performance", "tech debt", "optimize", "root cause", "something is wrong", "not working".

**Rule 4 — Explicit phase name => that phase directly.** "brainstorm" => Brainstorm, "investigate" => Investigation, "plan" => Planning, "implement" => Implementation, "review" => Code-Review, "test" => Testing, "document"/"docs" => Documentation.

**Rule 5 — Has artifacts => skip to appropriate phase.** User provides spec => Planning. User provides plan => Implementation. Code exists but no spec/plan => Code-Review, Testing, or Documentation as requested.

## Phase Summary

### Phase 1a: Brainstorm
**Skill:** `brainstorm/SKILL.md`
**Purpose:** Deep-dive into requirements through conversational questioning.
**Output:** `spec.md` (concise summary) + detailed design doc
**Key:** One question at a time, checks existing project context first, proposes approaches with trade-offs, YAGNI ruthlessly.
**When:** New features, new projects, adding functionality.

### Phase 1b: Investigation
**Skill:** `investigation/SKILL.md`
**Reference:** `investigation/references/investigation-patterns.md`
**Purpose:** Systematic investigation of bugs, performance issues, refactoring needs, or tech debt.
**Output:** `spec.md` (adapted for fixes) + investigation report
**Key:** Reproduce first, trace root causes with evidence, multiple hypotheses, propose fix strategies with trade-offs.
**When:** Bug fixes, refactoring, performance optimization, tech debt, debugging.

### Phase 2: Planning
**Skill:** `planning/SKILL.md`
**Purpose:** Break the spec into an ordered task list with dependencies and acceptance criteria.
**Output:** `plan.md` with task dependency graph
**Key:** Every task has files to create, dependencies, acceptance criteria. Tasks grouped into phases.

### Phase 3: Implementation
**Skill:** `implementation/SKILL.md`
**References:**
- `implementation/references/js-ts-best-practices.md`
- `implementation/references/databases-sql.md`
- `implementation/references/databases-nosql.md`
- `implementation/references/databases-redis.md`

**Purpose:** Execute the plan — write code, install deps, verify acceptance criteria.
**Output:** Working code, passing builds
**Key:** Critical plan review before coding, tech stack detection, batch execution (3 tasks per batch), quality gate after every batch (types + build + lint + tests), hard STOP on blockers.

### Phase 4: Code Review
**Skill:** `code-review/SKILL.md`
**Reference:** `code-review/references/review-checklists.md`
**Purpose:** Strict review across 18 areas — find every issue.
**Output:** `review.md` + `fix-plan.md`
**Key:** 4-level verdict (Block / Request Changes / Approve+suggestions / Approve). If blocked, loops back to implementation. Diff mode for re-reviews.

### Phase 5: Testing
**Skill:** `testing/SKILL.md`
**Purpose:** Write comprehensive tests — unit, integration, e2e, edge cases, benchmarks.
**Output:** Test files + `test-report.md`
**Key:** Auto-detect test runner (default Vitest), >80% coverage target, generate-run-fix loop, production bugs documented.

### Phase 6: Documentation
**Skill:** `documentation/SKILL.md`
**Purpose:** Generate all project documentation from actual code.
**Output:** README.md, API docs, architecture, contributing, changelog, deployment guide
**Key:** Every statement verified against code. Adapts format to project type. No aspirational docs.

## Handling User Requests

| User says | Action |
|---|---|
| "Build me a REST API for..." | Start Phase 1a (Brainstorm) |
| "I want to build..." / "New feature" | Start Phase 1a (Brainstorm) |
| "Fix this bug" / "Debug this" | Start Phase 1b (Investigation) |
| "Refactor this" / "Clean up this code" | Start Phase 1b (Investigation) |
| "Performance issue" / "This is slow" | Start Phase 1b (Investigation) |
| "Investigate" / "Find the root cause" | Start Phase 1b (Investigation) |
| "Tech debt" / "Why is this broken?" | Start Phase 1b (Investigation) |
| "I have a spec, let's plan" | Start Phase 2 (Planning), read their spec |
| "Here's my plan, start coding" | Start Phase 3 (Implementation), read their plan |
| "Review this code" | Start Phase 4 (Code Review) |
| "Write tests" | Start Phase 5 (Testing) |
| "Generate docs" | Start Phase 6 (Documentation) |
| "Fix the review issues" / "Address the feedback" | Start Phase 3 (Implementation) in fix mode — read `fix-plan.md` |
| "Fix the failing tests" / "Fix test failures" | Start Phase 3 (Implementation) in test-fix mode — read `fix-test-plan.md` |
| "Continue" / "Next step" | Check pipeline-state.json, proceed to next phase |
| "What's the status?" | Read pipeline-state.json, summarize progress |
| "Start over" | Reset pipeline-state.json, start Phase 1 |
| "Skip to testing" | Jump to Phase 5, note skipped phases |

## Important Rules

1. **Always read the skill file before starting a phase.** The skills contain detailed
   instructions, checklists, and examples that are essential for quality output.
2. **Read reference files when the skill tells you to.** Best practices, checklists, and
   database references exist for a reason.
3. **Follow the skill's process step by step.** Don't skip steps.
4. **Update pipeline-state.json** after each phase and significant milestone.
5. **Suggest the next phase** but wait for user confirmation before proceeding.
6. **If the user provides their own spec/plan/code**, adapt — don't force them through
   earlier phases they've already done.
